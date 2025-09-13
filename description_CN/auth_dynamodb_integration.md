# Auth: Cognito + DynamoDB user_table 集成说明

## 概述

本方案在沿用现有 Cognito 登录态的基础上，引入 DynamoDB `user_table` 作为用户资料的权威数据源（username、email、createdAt）。
- 邮箱注册与验证仍通过 Cognito 完成。
- 注册邮箱验证成功后：自动登录、初始化 `user_table` 记录，并要求用户设置唯一的 `username`。
- 前端本地读取用户信息时，优先从 `user_table` 读取，保持 O(1) 检索（同 userId 主键）。

## 表结构

- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `user_table`、`user_wallet_table` 小节）。

## 接口与服务

- DynamoDB 客户端：`lib/server/dynamodb.ts`
- 用户仓储：`lib/server/user-repo.ts`
  - `initUserProfile(userId, email)`：幂等创建 Profile
  - `getUserProfile(userId)`：读取 Profile
  - `setUsername(userId, username)`：事务设置用户名与占位；若为改名则同时释放旧用户名占位，仅当旧占位的 `userId` 与当前用户一致时才删除；大小写变更（`lower` 相同）将仅更新占位与资料中的展示名。

### API 路由

- `POST /api/user/init`
  - Header 需带 `x-user-id`、`x-session-token`，服务端校验单点登录。
  - Body: `{ email, referralCode? }`
  - 作用：初始化 `USER#userId / PROFILE`；幂等初始化 `WALLET` 与 `LEDGER_SUMMARY`；幂等初始化 `INVITE_PROFILE`；如提供 `referralCode` 且有效，则原子为邀请人累计 `totalInvites/unissuedRewardCount`。

- `POST /api/user/username`
  - Header 需带 `x-user-id`、`x-session-token`。
  - Body: `{ username }`
  - 作用：设置或修改用户名（含唯一性校验）。
    - 如新用户名已被他人占用：409。
    - 如仅大小写变更：成功更新展示名。
    - 如为改名：新名预占成功后，更新资料并释放旧名。
  - 成功响应：`{ success: true, user: { id, email, username } }`。前端应直接使用该 `user` 覆盖本地缓存与全局状态，无需额外 GET `/api/user/me`。

- `GET /api/user/me`
  - Header 需带 `x-user-id`、`x-session-token`。
  - 作用：按 `userId` O(1) 查询 Profile。若注册表缺失且客户端持有有效 Cognito cookie，自修复会重签 `sessionToken`。

- `GET /api/wallet/me`
  - 作用：按 `userId` O(1) 查询 Wallet。
- `GET /api/invite/me`：按 `userId` O(1) 查询邀请档案；若不存在则幂等初始化
- `POST /api/invite/settle/me`：根据 `.env` 阈值结算未发放邀请奖励，更新钱包与汇总

- `GET /api/model-config/list`
  - Header 同上（仅校验会话）。
  - 作用：列出 `model_config` 中的全部模型配置，供前端缓存展示。

## 前端改动

- `contexts/AuthContext.tsx`
  - 登录成功后先本地保存 `sessionToken/userId`（`applyUser`）。随后通过统一的资料同步模块 `lib/data/user-profile-sync.ts` 执行 `syncProfileAfterLogin(email)`：确保后端存在 `PROFILE` 并读取最新资料回填本地。
  - 刷新页面时不再每次强制拉取资料；`checkAuthStatus` 优先从本地缓存恢复用户信息，并通过 `GET /api/session/validate` 仅校验会话有效性（不触库）。
  - 修改：`updateUsername` 在成功后直接使用接口返回的 `user` 更新本地状态与缓存，不再触发 `syncProfileFromServer()` 额外拉取。
  - 登录与 `applyUser` 时：若 `username` 为空，则派发 `require-set-username` 事件，由全局布局强制打开用户名设置面板。

- 新增/扩展 `lib/data/user-profile-sync.ts`
  - Profile 同前。
  - 新增钱包：`syncWalletAfterLogin()`、`readLocalUserWallet()`、`writeLocalUserWallet()`、`fetchUserWalletFromServer()`。
  - 新增模型配置：`syncModelConfigsAfterLogin()`、`readLocalModelConfigs()`、`writeLocalModelConfigs()`、`fetchModelConfigsFromServer()`。
  - 新增汇总与邀请档案：`syncLedgerSummaryAfterLogin()`、`syncInviteProfileAfterLogin()`，并提供本地读写函数。

- `components/LoginModal.tsx`
  - 增加 `set-username` 步骤：邮箱验证成功后自动登录，随后要求设置唯一用户名。
  - 用户表初始化由登录流统一处理，注册确认后无需再次调用 `/api/user/init`。
  - 支持 `forceSetUsername` 强制模式：当用户名缺失时从 `MainLayout` 传入，禁止手动关闭，直至设置成功自动关闭。
  - 设置成功后关闭并进入已登录状态。

## 环境变量

- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节。

## 本地存储与会话

- 仍沿用现有单点登录头：`x-user-id` 与 `x-session-token`（见 `utils/session-client.ts`）。
- 用户信息的权威来源为 DynamoDB `user_table`，但刷新时优先读本地缓存，减少不必要的网络请求；Cognito 仅用于认证与令牌。
- 服务端具备“会话自修复”能力：当仅内存注册表丢失时，不会错误地判定为过期；会通过 Cookie 验证恢复。

## 错误与国际化

- 用户名占用返回 409（`USERNAME_TAKEN`）。
- 已添加 i18n：`auth.loginModal.usernameTaken`。

 
