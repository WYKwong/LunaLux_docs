# 官方模型扣费与调用流程（/api/official-model/submit）

## 概述
- 本文描述当用户选择官方模型后，服务端如何依据 `model_config_table` 与 `user_wallet_table` 完成计费审批与原子扣费，并与大模型调用并发执行，最终将模型回复与更新后的钱包信息一并返回给前端。
- 表名均通过环境变量配置：`MODEL_CONFIG_TABLE_NAME`、`USER_WALLET_TABLE_NAME`（详见 `.env.local`）。
  - 注意：默认表名为 `model_config_table`。如果未配置或表不存在，将返回 `MODEL_NOT_FOUND`，请确保创建该表并填充模型配置。

## 时序与关键步骤
1. 前端在模型侧边栏启用官方模型模式，写入：`localStorage.officialModelEnabled=true`、`officialModelName`、`officialModelId`、`userId`。
2. 工作流中 `LLMNodeTools.invokeLLM` 发现官方模式后，改为调用服务端接口：`POST /api/official-model/submit`。
3. 服务端 `app/api/official-model/submit/route.ts` 处理：
   - 校验会话（请求头 `x-user-id/x-session-token`），回退使用 body 的 `userId`。
   - 使用 `modelId` 以 O(1) 从 `model_config_table` 读取配置；
   - 使用 `userId` 以 O(1) 从 `user_wallet_table` 读取钱包；
   - 具体表结构与主键设计见《DynamoDB 表结构设计》：`description/dynamodb_tables.md`。
   - 计费审批顺序：
     - 免费：若 `freeLevel >= 0` 且 `wallet.luxLevel >= freeLevel` → 免费通过；
     - 星元：否则若 `allowStar===true` 且 `starCost >= 0` 且 `wallet.starCoin >= starCost` → 采用星元；
     - 月华：否则若 `allowLuna===true` 且 `lunaCost >= 0` 且 `wallet.lunaCoin >= lunaCost` → 采用月华；
     - 否则拒绝（不支持或余额不足）。
   - 一旦通过审批：并发执行两件事（Promise.all）：
     - 调用官方大模型（OpenAI/Gemini）获取内容；
     - 若不是免费，使用 DynamoDB `ADD` 负数 + 条件表达式进行原子扣费（并返回最新钱包）。
   - 汇总结果，返回大模型内容与更新后钱包信息。

## 请求参数（Body）
```json
{
  "systemMessage": "...",
  "userMessage": "...",
  "language": "zh",
  "llmType": "openai" | "gemini",
  "modelName": "gpt-5" | "gemini-2.5-pro" | "...",
  "modelId": "001",
  "userId": "<cognito-sub>",
  "temperature": 0.7,
  "topP": 0.7,
  "topK": 40,
  "maxTokens": 2048,
  "frequencyPenalty": 0,
  "presencePenalty": 0,
  "repeatPenalty": 1.1,
  "baseUrl": "https://...",
  "raw": "可选的原始调试文本"
}
```

## 响应示例
- 成功：
```json
{
  "success": true,
  "content": "模型生成的文本",
  "wallet": {
    "userId": "...",
    "luxLevel": 0,
    "starCoin": 95,
    "lunaCoin": 100
  },
  "billing": { "method": "star", "cost": 5, "modelId": "001" }
}
```
- 失败（部分）：
```json
{ "success": false, "code": "USER_ID_REQUIRED" }
{ "success": false, "code": "MODEL_ID_REQUIRED" }
{ "success": false, "code": "MODEL_NOT_FOUND" }
{ "success": false, "code": "LLM_CONFIG_INVALID" }
{ "success": false, "code": "WALLET_NOT_FOUND" }
{ "success": false, "code": "PAYMENT_NOT_SUPPORTED" }
{ "success": false, "code": "INSUFFICIENT_FUNDS" }
{ "success": false, "code": "BILLING_FAILED" }
```

## 并发与原子扣费
- 原子扣费通过 `UpdateCommand` 实现：`SET updatedAt = :now ADD <balanceField> :neg`，并使用条件表达式确保余额充足：`<balanceField> >= :cost`。
- 若在审批通过后，另一路并发请求先一步扣减导致条件失败，将抛出 `ConditionalCheckFailedException`，接口返回 `BILLING_FAILED`（HTTP 402）。
- 为避免越权调用，官方模式下前端不会在失败时回退到直连模型，确保扣费路径被强制执行。

## 实现位置
- 路由：`app/api/official-model/submit/route.ts`
- 表工具：`lib/server/dynamodb.ts`
- 模型配置仓库：`lib/server/model-config-repo.ts`
- 钱包仓库：`lib/server/user-wallet-repo.ts`
- 前端转发调用：`lib/nodeflow/LLMNode/LLMNodeTools.ts`

## 约定与注意
- `freeLevel = -1` 表示该模型不对任何等级免费；仅当 `freeLevel >= 0` 且用户 `luxLevel >= freeLevel` 时免费。
- `allowStar/allowLuna` 为 `false` 或对应 `Cost < 0` 均视为不支持该支付方式。
- 审批与扣费均基于以 O(1) 方式获取的配置与钱包记录，满足性能要求。

## Token 用量打印（可选）
- 环境变量：`analyze_token`（布尔，默认关闭）。
- 当用户采用“官方模型”路径时，若 `analyze_token=true`，后端会在终端打印本次调用的 Token 用量：
  - OpenAI：从 `chat.completions` 响应体的 `usage` 字段读取 `prompt_tokens`、`completion_tokens`、`total_tokens`。
  - Gemini：从返回对象的 `response.usageMetadata`（或 `usage_metadata` 兼容字段）读取 `promptTokenCount`、`candidatesTokenCount`、`totalTokenCount` 并规范化输出。
- 实现位置：`lib/official-llm/adapter.ts` 中的 `GeminiOfficialAdapter.generate` 与 `OpenAIOfficialAdapter.generate`。
- 仅在“官方模型”路径生效；直连用户自带 Key 的调用不打印。