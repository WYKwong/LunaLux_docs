# 用户信息同步与本地化策略

## 目标
- 在“单会话”前提下，避免刷新时重复拉取用户资料。
- 登录后统一完成用户资料的后端初始化与前端缓存写入。
- 当资料发生变更（如修改用户名）或重新登录时，才从后端读取并刷新本地缓存。

## 权威数据源
- 权威来源：DynamoDB `user_table`（结构见《DynamoDB 表结构设计》）
- 认证来源：Cognito（仅用于令牌与会话）

## 模块与职责
- `lib/data/user-profile-sync.ts`
  - Profile：
    - `readLocalUserProfile()` / `writeLocalUserProfile()` / `clearLocalUserProfile()`
    - `initUserProfileOnServer(email)` → POST `/api/user/init`
    - `fetchUserProfileFromServer()` → GET `/api/user/me`
    - `syncProfileAfterLogin(email)` / `syncProfileFromServer()`
  - Wallet（新增）：
    - `readLocalUserWallet()` / `writeLocalUserWallet()`
    - `fetchUserWalletFromServer()` → GET `/api/wallet/me`
    - `syncWalletAfterLogin()`：登录后与 Profile 同时拉取并写入本地
  - Invite Profile（新增）：
    - `readLocalInviteProfile()` / `writeLocalInviteProfile()`
    - `fetchInviteProfileFromServer()` → GET `/api/invite/me`
    - `syncInviteProfileAfterLogin()`：登录后与 Wallet 同步拉取并写入本地

## 新增会话验证端点
- `GET /api/session/validate`：仅基于请求头 `x-user-id/x-session-token` 校验会话有效性；
  - 有效：返回 `{ success: true, userId }`；
  - 无头/无效：返回 401 或 460（由拦截器处理）。
  - 不访问数据库，避免刷新时触发资料拉取。

## 生命周期与调用点
- 登录成功（`/api/auth/signin`）：
  1) 前端持久化 `sessionToken/userId`
  2) 调用 `syncProfileAfterLogin(email)`（含 `/api/user/init` 幂等创建 Profile 与 Wallet）
  3) 并行调用 `syncWalletAfterLogin()`、`syncLedgerSummaryAfterLogin()`、`syncInviteProfileAfterLogin()`；仅首次登录拉取，刷新不重复拉取
  4) 若资料或返回结果中的 `username` 为空，派发 `require-set-username`，由全局布局强制打开不可手动关闭的“设置用户名”面板

- 页面刷新：
  - 优先读取本地缓存恢复 UI（要求同时存在 `userId` 与 `sessionToken`）；
  - 调用 `/api/session/validate` 严格校验会话（若 460 则由拦截器处理过期）；
  - 不从数据库拉取资料。

- 资料变更（如设置用户名）：
  - 成功后直接使用 `/api/user/username` 返回的 `user` 覆盖本地缓存（`writeLocalUserProfile`）与上下文（`applyUser`），不再额外请求 `/api/user/me`。
  - 用户改名成功后，后端会释放其旧用户名占位，其他用户可再次申请该旧名；若仅大小写变更，旧名占位不会变化，仅更新展示名。
  - 若 `applyUser` 时检测到用户名缺失，也会派发 `require-set-username` 统一触发 UI 强制设置流程。

## 本地键位
- `userId`, `email`, `username`, `isLoggedIn`, `sessionToken`
- 额外 UI 键位（与登录无强关联）：`displayUsername`

提示：登录态判定以 `isLoggedIn === "true"`、存在 `userId` 且存在 `sessionToken` 为准；`username` 可以为空（例如新注册在设置用户名之前），UI 需要容错处理空用户名。
注意：当 `username` 为空且会话有效时，前端将强制弹出“设置用户名”面板以补全资料。

## 拦截器策略
- `utils/session-client.ts`：
  - 仅对“同源且非 `/api/auth/*`”请求注入 `x-user-id/x-session-token`；
  - 收到 460 时，仅对非 auth 端点派发 `session-expired` 事件，避免刷新时误清登录态。
  - 与用户名强制规则配合：会话仍有效但资料缺失用户名时，由 `AuthContext` 派发 `require-set-username`，与拦截器无冲突。

## 错误处理
- `/api/user/init` 或 `/api/user/me` 网络失败时：
  - 登录流程不阻塞；若资料获取失败，保留 Cognito 返回的用户最小集并稍后重试。
- 会话失效（HTTP 460）：
  - 由拦截器触发 `session-expired`，UI 弹窗处理并清理本地项。

## 与现有实现的衔接
- `contexts/AuthContext.tsx`：
  - 刷新时采用“本地优先 + `/api/session/validate` 严格校验”策略；
  - 登录与改名后采用 `user-profile-sync` 进行一致性刷新。
- `utils/session-client.ts`：拦截器继续向同源请求注入会话头，但排除 `/api/auth/*`。

## 进一步阅读
- 兼容性与迁移说明：见 `improve_description/user_info_migration_and_compatibility.md`
- 前端改动参考：`description/auth_dynamodb_integration.md` 中“前端改动”


