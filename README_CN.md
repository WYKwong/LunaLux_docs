# Lunalux（中文）

Lunalux 是一个以“角色”为中心的 AI 对话应用，提供可落地、可扩展的生产级架构。

## 架构总览
- 前端（Next.js App Router）：`/character`（对话）、`/character-market`（市场）、`/create-character`、`/user`、`/settings`
- 对话工作流：`preset → context → worldbook → llm → regex → plugin`
- 后端（API Routes）：鉴权、官方模型网关、角色社区、用户/钱包/资产汇总/邀请/兑换码
- 数据（DynamoDB + S3）：O(1) 读写、条件原子更新、Dirty+批处理（热度/时间重排）

## 模块（含跳转）

- 鉴权与会话
  - Cognito + DynamoDB 用户资料；单活登录；头部接管与基于 Cookie 的自修复
  - 文档：[鉴权与 user_table](./auth_dynamodb_integration.md)、[资料同步与本地化](./user_info_sync_and_localization.md)

- 官方模型计费
  - 审批顺序：免费 → 星元 → 月华；模型调用与扣费并发；资产汇总同步
  - 文档：[扣费流程](./official_model_billing_flow.md)、[模型配置表](./model_config_schema_and_integration.md)

- 角色社区与市场
  - 预签/直传、Finalize 缩略图、热度/时间列表、下载、点赞；按语言本地缓存
  - 文档：[S3 + DynamoDB 流程](./character_table_and_s3_workflow.md)、[市场缓存](./character_market_caching.md)、[S3 CORS 回退](./s3_cors_and_upload_fallback.md)、[SK 迁移](./character_table_sk_migration.md)

- 对话工作流与绑定
  - 节点化管线；后处理异步；官方模型可附带社区绑定 id
  - 文档：[对话工作流](./character_chat_workflow.md)、[绑定与用量](./character_binding_and_usage.md)

- 用户数据与操作
  - 钱包（lux/star/luna）、资产汇总、邀请档案、兑换码
  - 文档：[钱包](./user_wallet_integration.md)、[资产汇总](./ledger_summary_integration.md)、[邀请](./invite_profile_integration.md)、[兑换码](./coupons_table_and_redeem_flow.md)

- 数据模型
  - 表、键、字段与访问模式
  - 文档：[DynamoDB 表](./dynamodb_tables.md)

- 性能与热度策略
  - 本地优先、请求减少、Presigned、Dirty+批、会话自修复
  - 文档：[性能优化](./performance_optimization.md)、[RangeHeat 策略](./range_heat_policy.md)

## 文档
- 中文：`description_CN/`
- 英文：见根目录 [README.md](./README.md) 与 `description_EN/`