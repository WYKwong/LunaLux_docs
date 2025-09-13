# 用户钱包（user_wallet_table）集成说明

## 概述

- 新增 DynamoDB 表 `user_wallet_table` 管理用户会员等级与余额。
- 采用与用户表一致的 PK/SK 结构（具体主键设计见《DynamoDB 表结构设计》：`description/dynamodb_tables.md`）。
- 登录后初始化并同步钱包到本地，页面刷新不重复拉取，和用户资料同步策略一致。

## 表结构

- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `user_wallet_table` 小节）。

## 服务端实现

- `lib/server/dynamodb.ts`
  - 新增：`getUserWalletTableName()`、`buildWalletPk(userId)`
- `lib/server/user-wallet-repo.ts`
  - `initUserWallet(userId)`：幂等创建，默认余额 100/100，等级 0
  - `getUserWallet(userId)`：读取钱包
  - `updateUserWalletBalances(userId, updates)`：更新余额或等级
  - `deductWalletBalance(userId, method, amount)`：使用 `ADD` 负数 + 条件表达式原子扣减 `starCoin` 或 `lunaCoin`
  - `creditWalletBalance(userId, method, amount)`：原子增加对应余额（用于兑换码/邀请结算）
- API：
  - `POST /api/user/init`：在创建/补全 Profile 时，同时幂等初始化 Wallet
  - `GET /api/wallet/me`：按当前会话返回钱包信息

## 前端实现

- `lib/data/user-profile-sync.ts`
  - 新增类型 `UserWalletInfo`
  - `readLocalUserWallet()` / `writeLocalUserWallet()`
  - `fetchUserWalletFromServer()` / `syncWalletAfterLogin()`
- `contexts/AuthContext.tsx`
  - 登录成功后：先持久化会话 → `syncProfileAfterLogin(email)` → `syncWalletAfterLogin()`
  - 初次/刷新：优先从本地读取 `wallet` 恢复 UI；仅在登录流程拉取
- UI：`app/user/page.tsx`
  - 本地读取并展示钱包（徽章 + 星/月余额），支持 i18n
  - 图标抽离：星元/月华图标提取到 `components/icons/StarCoinIcon.tsx`、`components/icons/LunaCoinIcon.tsx`
  - 同页展示邀请信息（邀请码/总邀请/未发放人数）

## 国际化

- `app/i18n/locales/en.json` / `zh.json`
  - 新增：`account.wallet`、`account.lux`、`account.superLux`、`account.ultraLux`、`account.starCoin`、`account.lunaCoin`

## 环境变量

- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节。

## 兼容性与迁移

- 老用户：首次登录时 `/api/user/init` 会幂等补全 Profile 与 Wallet。
- 刷新策略：沿用“本地优先 + 仅校验会话”的策略，避免每次刷新请求数据库。

## 测试要点

- 新注册：钱包默认 `luxLevel=0`，`starCoin=100`，`lunaCoin=100`
- 登录后刷新页面：钱包显示仍然存在且不重复请求
- 升级会员或充值路径可基于 `updateUserWalletBalances` 扩展

---

## 进一步阅读
- 官方模型扣费路径与原子扣费实现：`description/official_model_billing_flow.md`


