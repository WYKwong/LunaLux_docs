# 邀请系统（invite_profile_table）集成说明

## 概述

- 新增 DynamoDB 表 `invite_profile_table` 管理每个用户的邀请码和邀请进度。
- 采用 PK/SK 结构（具体主键设计见《DynamoDB 表结构设计》：`description/dynamodb_tables.md`）。
- 支持：
  - 注册完成时为每位用户生成唯一邀请码
  - 使用邀请码注册时，原子累加邀请人的 `totalInvites` 与 `unissuedRewardCount`
  - 登录后拉取邀请档案本地化展示
  - 里程碑结算：每 N 人结算一次奖励，自动入账钱包与汇总表

## 表结构

- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `invite_profile_table` 小节）。

## 环境变量

- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节（业务阈值类配置继续在本文件列出）。
  - `INVITED_USER_REWARD_STAR`
  - `INVITE_REWARD_THRESHOLD`
  - `INVITE_MILESTONE_REWARD_STAR`

## 服务端实现

- `lib/server/dynamodb.ts`
  - `getInviteProfileTableName()`、`buildInviteProfilePk(userId)`、`buildInviteCodePk(inviteCode)`
- `lib/server/invite-profile-repo.ts`
  - `initInviteProfile(userId)`：事务生成邀请码 + 建档（幂等）
  - `getInviteProfileByUserId(userId)` / `getInviteCodeOwner(code)`
  - `incrementInviterCountersByCode(code)`：O(1) 验证并原子自增邀请人的计数
  - `settleInviteMilestonesForUser(userId, threshold, reward)`：按阈值批量结算，钱包与 `ledger_summary` 同步入账
- `app/api/user/init`
  - 完成 Profile/Wallet/Ledger 初始化后，幂等初始化 InviteProfile；若携带 `referralCode`（Body 或 Cookie）则执行 `incrementInviterCountersByCode`
- `app/api/invite/me`
  - 返回当前用户的邀请档案（不存在则补建）
- `app/api/invite/settle/me`
  - 根据阈值与奖励配置结算；返回结算次数与奖励金额

## 前端实现

- `lib/data/user-profile-sync.ts`
  - `UserInviteProfileInfo` 类型
  - 本地读写：`readLocalInviteProfile()` / `writeLocalInviteProfile()`
  - 拉取：`fetchInviteProfileFromServer()` / `syncInviteProfileAfterLogin()`
- `contexts/AuthContext.tsx`
  - 登录成功后并行执行 `syncInviteProfileAfterLogin()`，仅首次拉取
- `components/LoginModal.tsx`
  - 注册表单新增“邀请码（可选）”输入框并支持 i18n
  - 将值暂存至 `localStorage.pendingReferralCode`，注册调用 `/api/auth/signup` 会随 Body 传递，同时服务端也会在成功时写入短期 Cookie 以确保 `/api/user/init` 能读取
- `app/user/page.tsx`
  - 展示邀请码、总邀请人数、未发放奖励人数（本地缓存）

## 结算与奖励逻辑

- 使用邀请码注册的用户：
  - 立即获得 `INVITED_USER_REWARD_STAR` 星元，O(1) 增加钱包并调用 `incrementLedgerSummary(... starGain ...)`。
- 邀请人登录时：
  - 调用 `/api/invite/settle/me` 将 `unissuedRewardCount` 按 `INVITE_REWARD_THRESHOLD` 结算为 `INVITE_MILESTONE_REWARD_STAR`，支持一次结算多次；钱包与汇总表同步更新。

## 安全与并发

- 所有计数更新使用 DynamoDB `ADD` 原子操作；
- 邀请码使用单独的 `INVITE_CODE#` 占位实现 O(1) 反查与唯一性；
- 初始化与占位使用事务确保无重码；
- 结算前后均读取最新值以避免重放。


