# Official Model Billing & Submit Flow (/api/official-model/submit)

## Overview
- Server approves billing from `model_config_table` and `user_wallet_table` and calls official LLMs in parallel

## Steps
1) Frontend enables official mode and stores selection in localStorage
2) `LLMNodeTools.invokeLLM` calls `POST /api/official-model/submit`
3) Route:
   - Validate session (`x-user-id/x-session-token`) or fallback to body `userId`
   - O(1) get model config and wallet
   - Approval order: free → star → luna; reject otherwise
   - Parallel: official LLM call + atomic wallet deduction (negative ADD with condition)
   - Return content + updated wallet

## Request body
See CN doc for full fields; includes system/user messages, llmType/modelName/modelId, userId, parameters, baseUrl, raw.

## Responses
- Success: `{ success, content, wallet, billing: { method, cost, modelId } }`
- Errors: `USER_ID_REQUIRED`, `MODEL_ID_REQUIRED`, `MODEL_NOT_FOUND`, `LLM_CONFIG_INVALID`, `WALLET_NOT_FOUND`, `PAYMENT_NOT_SUPPORTED`, `INSUFFICIENT_FUNDS`, `BILLING_FAILED`

## Atomic deduction
- `SET updatedAt = :now ADD <balanceField> :neg` with condition `<balanceField> >= :cost`
- Concurrent conflicts raise `ConditionalCheckFailedException` (respond 402)

## Notes
- Free level requires `wallet.luxLevel >= freeLevel`
- Do not fallback to direct keys on billing failure
- Optional token usage logging via env
