Ledger Summary（`ledger_summary_table`）集成说明

概述
- 新增 DynamoDB 表 `ledger_summary_table`，以 O(1) 主键模型存储用户全局汇总：
  - `totalStarCoinUsed`：累计消耗星元
  - `totalStarCoinGain`：累计获得星元
  - `totalLunaCoinUsed`：累计消耗月华
  - `totalLunaCoinGain`：累计获得月华
- 登录后从服务端获取并本地化缓存，`/user` 页面显示统计信息（支持 i18n）。
- 官方模型扣费流中，审批通过后（非免费）与模型调用并发，原子扣费钱包的同时增量更新汇总表。

表结构与字段
- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `ledger_summary_table` 小节）。

服务端实现
- lib/server/dynamodb.ts
  - 新增：`getLedgerSummaryTableName()`、`buildLedgerSummaryPk(userId)`
- lib/server/ledger-summary-repo.ts
  - initLedgerSummary(userId, initialStar, initialLuna)：幂等创建
  - getLedgerSummary(userId)：读取
  - incrementLedgerSummary(userId, { starUsed, starGain, lunaUsed, lunaGain })：使用 ADD 原子自增并更新 updatedAt
- 被邀请奖励发放：通过 `creditWalletBalance(userId, "star", amount)` 增加钱包余额，同时调用 `incrementLedgerSummary(userId, { starGain: amount })` 记录历史获得。
- API：
  - `GET /api/ledger/summary/me`：按当前会话返回汇总信息
  - `POST /api/user/init`：在幂等初始化 Wallet 后，同步初始化 Ledger Summary（以初始星元/月华计入 Gain）
  - `POST /api/official-model/submit`：审批通过后，若 star/luna 扣费，分别调用 `incrementLedgerSummary` 增量更新 Used；免费不更新
  - `POST /api/coupon/redeem`：兑换成功后，按奖励种类调用 `incrementLedgerSummary` 增量更新 Gain

前端实现
- lib/data/user-profile-sync.ts
  - 新增类型：UserLedgerSummaryInfo
  - readLocalLedgerSummary() / writeLocalLedgerSummary()
  - fetchLedgerSummaryFromServer() / syncLedgerSummaryAfterLogin()
- contexts/AuthContext.tsx
  - 登录后：syncProfileAfterLogin(email) → syncWalletAfterLogin() → syncLedgerSummaryAfterLogin() → syncModelConfigsAfterLogin()
- lib/nodeflow/LLMNode/LLMNodeTools.ts
  - 官方模型模式调用成功后：若返回 ledgerSummary，写入本地缓存
- UI：app/user/page.tsx
  - 新增显示：totalStarCoinUsed/totalStarCoinGain/totalLunaCoinUsed/totalLunaCoinGain

国际化
- app/i18n/locales/en.json / zh.json
  - 新增键位：
    - account.totalStarCoinUsed
    - account.totalStarCoinGain
    - account.totalLunaCoinUsed
    - account.totalLunaCoinGain

环境变量
- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节。

官方模型扣费与并发
- 审批与扣费路径沿用 description/official_model_billing_flow.md：
  - 采用 Promise.all 并发执行：LLM 调用 + 钱包原子扣费 +（若非免费）Ledger 汇总自增
  - 钱包扣费若 ConditionalCheckFailedException：返回 BILLING_FAILED
  - 免费路径不更新 Ledger 汇总

注意事项
- 注册/初始化：若 Wallet 已存在但 Ledger 缺失，将按当前 Wallet 余额补建 Ledger 并计入初始 Gain。
- O(1) 访问：所有读取均按 PK/SK 精确定位，无需扫描。
- 本地缓存：登录后仅拉取一次，页面刷新不重复请求；官方模型调用返回会刷新本地缓存。

