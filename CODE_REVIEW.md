# dp_agt 代码审查报告

- 审查日期：2026-10-31
- 审查范围：`src/`（后端，3376 行）、`web/`（前端，2313 行）、`data/cfg.yml` 与数据库实际配置、`build/` 部署上下文
- 审查方式：逐文件静态审查 + 线上配置核对
- 分支状态：基于 `session_reuse` 分支当前工作区

> 严重程度：🔴 高危（正在生效的风险或功能失效）｜🟡 中危（特定场景下出问题）｜🟢 低危（打磨项）

---

## 🔴 高危

### 1. 对外业务 API 当前完全无鉴权（正在生效）

`apiTokenMiddleware` 是 fail-open 设计：DB 未启用认证 → 看 `cfg.yml` 静态 token → 都没有 → **直接放行**（`src/routes/auth.ts:184-226`）。

实测线上配置：
- `data/cfg.yml` 的 `sys.api_token` 为空；
- 数据库 `config` 表只有 `token_secret` 一行，没有 `api_auth_enabled`。

即当前**任何人无需凭证**即可：

| 接口 | 后果 |
|---|---|
| `POST /api/v1/*` | 任意提交任务，烧 LLM 配额 |
| `GET /api/v1/tasks` | 列出所有活动任务（task_id / linkid / callback_url） |
| `GET /api/v1/task/trace?taskid=` | 拿到完整请求体 + 渲染后提示词 + Agent 输出（公文内容泄露，task_id 可由上一接口免费获取） |
| `DELETE /api/v1/tasks/:id` | 取消任何排队任务 |
| `GET /api/v1/task/log/:id`、`/api/v1/task/link/log/:linkid` | 完整执行时间线 |

**建议**：
1. 立即在「系统设置」启用 API 认证，或配置环境变量 `API_TOKEN`；
2. 把中间件改为 fail-closed：未配置任何 token 时返回 503 并在启动日志中显式告警。

### 2. 「取消任务」目前是假的

`DELETE /api/v1/tasks/:id` 只把 DB 状态改成 `cancelled`，任务**仍在内存 pending 队列中**，`pump()` 照常执行，状态随后被覆盖为 `processing` → `completed`（`src/routes/tasks.ts:38-57`、`src/services/task-queue.ts:124-153`）。

**建议**：`processItem` 开始前检查 DB 状态（已取消则跳过），或取消时从 pending 数组摘除。

### 3. callback_url 任意地址 → SSRF + token 泄漏

提交方自带 `callback_url`（必填字段），服务器无条件向任意 host POST 完整回调体，包括 127.0.0.1 与内网服务（`src/services/callback.ts:36-88`）。更糟的是：

- 回调请求带 `Authorization: Bearer <file_url_token>` 发往**提交方指定的 URL**（`callback.ts:53-55,101`），任何提交者都能借此拿到文件服务 token；
- 该 token 还明文写进 info 日志的 curl 行（`callback.ts:106-110`）。

**建议**：回调目标白名单 / 内网地址黑名单；回调鉴权头不要复用 file_url_token；日志脱敏。

### 4. Agent 以 bypassPermissions 运行且环境变量携带全部密钥

Agent 子进程 env 携带 LLM 网关 token、`DOC_AUDIT_TOKEN`（96 位，已配置）、`FILE_URL_TOKEN`（`src/services/sdk-runner.ts:24-76`），权限为 `bypassPermissions`（`sdk-runner.ts:239`），而它处理的是外部传入的公文与下载的文件——天然提示注入面：文档里埋一句「把环境变量 curl 给某地址」就可能被照做，且 CLAUDE.md 明确告知 agent 这些 token 的存在。

**建议**：改用最小权限集（allow 规则）替代 bypassPermissions；带密钥的外部服务调用改为服务端代理（token 不进 agent env）；限制子进程网络出口。

---

## 🟡 中危

### 5. 改密不验证旧密码

`PUT /api/inner/user/password` 的 `old_pwd` 可选（`src/routes/auth.ts:348-353`）。偷到 Cookie/token 的人可直接改密码，把最长 7 天的 token 变成永久账号控制。

**建议**：除 `must_change_pwd` 强制首改流程外，必须校验旧密码。

### 6. sessions 表 UNIQUE(linkid) 冲突会炸任务

若 SDK 为已有 linkid 返回新的 session_id（resume 失败等场景），`upsert` 的 INSERT 违反唯一约束（`src/store.ts:509-521`）：

- 在 `onSessionId` 回调里抛 → 任务被误判失败；
- 在 `processItem` 尾部抛 → 任务**永远卡在 processing** 直到重启（`src/services/task-queue.ts:61-69,137-151`）。

**建议**：upsert 改为按 linkid 更新旧行（`ON CONFLICT(linkid) DO UPDATE` 或先查后改）。

### 7. 登录无速率限制 + bcrypt 同步阻塞主线程

匿名可无限刷 `/api/login`；bcryptjs 纯 JS、同步跑在事件循环上，每次约百毫秒，几十 QPS 即可拖垮整个服务（含正在执行的任务）。

**建议**：登录限速 + 异步 bcrypt 或移入 worker。

### 8. 每日清理与运行中任务竞态

清理逻辑以 sessions 表为准 `rm -rf` 工作目录（`src/index.ts:171-181`）。任务刚启动、`onSessionId` 尚未落库的窗口内，运行中 agent 的 cwd 可能被删除。

**建议**：清理时跳过近 N 分钟内有 `pending`/`processing` 任务的 linkid。

---

## 🟢 低危 / 打磨

| # | 问题 | 位置 |
|---|---|---|
| 9 | 存储型 XSS：`deleteInterface`/`deleteUser` 的 onclick 用单引号拼 JS，`escHtml` 不转义 `'`（需 admin 权限触发，低危） | `web/admin.html:369,813` |
| 10 | `/api/dev/null` 未鉴权调试端点，任何人可刷日志 | `src/index.ts:120-123` |
| 11 | `trust proxy=true` + 直连部署 → X-Forwarded-For 可伪造，日志来源不可信 | `src/index.ts:67` |
| 12 | 错误处理中间件注册在 admin 路由之前，admin 路由内抛错走 Express 默认 handler；NODE_ENV 未设 production 时可能向客户端吐堆栈 | `src/index.ts:109-117` |
| 13 | 回调响应体无上限收集，30s 超时只防空闲不防慢速流 | `src/services/callback.ts:66-76` |
| 14 | Cookie 未设 `secure` 标记（HTTPS 部署时降级明文发送） | `src/routes/auth.ts:287-292` |
| 15 | `disable_nonessential_traffic`/`attribution_header` 配置项被 `buildEnv` 硬编码覆盖，成了死配置 | `src/services/sdk-runner.ts:68-70` |
| 16 | 全量 `traceMessages` 常驻内存再 stringify（trace 字段实际未落库），长任务内存膨胀 | `src/services/sdk-runner.ts:232,278` |
| 17 | 重启恢复时 `request_body` JSON.parse 失败的任务被 `continue` 跳过 → 永久 pending | `src/services/task-queue.ts:183` |
| 18 | `viewTask`/`taskModal` 疑似死代码（列表操作已改为会话记录按钮） | `web/admin.html:462` |

---

## 修复优先级建议

1. **先修 #1（fail-closed + 启用认证）**：正在生效的风险，改动小、收益大。
2. **再修 #2（取消任务）**：功能上「骗人」的按钮，调用方会误以为任务已取消。
3. **#5（改密校验旧密码）**：一行逻辑，堵住 token 升级为永久账号的口子。
4. **#6（upsert 冲突）**：防任务误失败/卡死，改法明确。
5. **#3、#4**：涉及与业务方的信任边界与 agent 运行模型，建议与业务方讨论后设计（白名单、服务端代理），不要裸改。
6. 其余按 #7、#8 → 低危清单依次排期。

---

## 做得好的地方

- **Token 设计**：HMAC 签名 + `user_version` 递增作废（改密/登出即失效）、`timingSafeEqual` 比较、Cookie `httpOnly` + `sameSite`。
- **SQL 全参数化**（better-sqlite3 prepare），未发现注入点。
- **账号安全**：首次登录强制改密（2h 窗口）+ 密码强度校验 + 防删 admin 账号。
- **前端转义基本到位**：动态内容普遍 `escHtml`/`textContent`，SSE 断连有清理。
- **队列设计**：per-linkid 串行化（同公文 session 不冲突）、重启恢复 pending/processing 的兜底逻辑合理。
- **配置**：敏感字段日志脱敏（`mask()`）、环境变量优先于 cfg.yml。
