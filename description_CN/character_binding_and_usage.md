#+ 角色卡本地绑定（active）与 TotalLunaCoinUsed 计数规则

## 1. 本地绑定数据结构与存储

- 本地角色卡记录结构：`CharacterRecord`
  - 扩展字段：`marketBinding?: { active: boolean; characterId?: string }`
  - 存储介质：IndexedDB（对象仓库：`CHARACTERS_RECORD_FILE`）
  - 读写 API：
    - `LocalCharacterRecordOperations.updateMarketBinding(characterId, { active, characterId? })`
    - `LocalCharacterRecordOperations.getMarketBinding(characterId)`
- 设定规则：
  - 通过“导入角色”（本地 PNG）创建的角色卡：`active=false`
  - 通过“角色社区/立即对话”导入的角色卡：`active=true`，并保存社区 `characterId`

参考实现：
- `lib/data/roleplay/character-record-operation.ts`
- `function/character/import.ts`
- `components/MarketCharacterCard.tsx`

## 2. 对话流程中的绑定使用

- 在进入 `/character` 页面后，前端会将当前本地角色卡的 `id` 写入 `sessionStorage.currentCharacterLocalId`，供 LLM 节点读取绑定（只在官方模型模式下需要）。
- 对话时：
  - 若 `active=false`：不携带任何社区 `characterId`，完全本地化对话。
  - 若 `active=true`：在官方模型模式（`LLMNodeTools.invokeLLM` → `POST /api/official-model/submit`）中，会携带社区 `characterId`。

参考实现：
- `app/character/page.tsx`（写入 `currentCharacterLocalId`）
- `lib/nodeflow/LLMNode/LLMNodeTools.ts`（读取绑定并随请求传递 `characterId`）

## 3. 服务端计费与 TotalLunaCoinUsed/DialogueCount 计数

- 路由：`POST /api/official-model/submit`
- 计费决策顺序：会员免费（`free`）→ 星元（`star`）→ 月华（`luna`）。
- 社区角色用量累计（`character_table_EN/character_table_CN.Metadata.TotalLunaCoinUsed`）：
  - 当存在有效的 `characterId` 且该条目存在：
    - 免费（会员）与星元路径：不累加。
    - 月华路径（luna）：累加本次扣费金额。
  - DynamoDB 更新采用“两步更新”，避免 UpdateExpression 文档路径重叠：
    1) `SET Metadata = if_not_exists(Metadata, {})`
    2) `SET UpdatedAt = :now ADD Metadata.TotalLunaCoinUsed :amt`
  - 若多语言分表中查无此 `characterId`：
    - 不报错、不回滚计费逻辑；仅跳过 `TotalLunaCoinUsed` 更新。
    - 在成功响应体中返回 `deactivateBinding=true`，提示前端自动将本地 `active` 关闭。

- 对话计数与会员热度（`character_table_EN/character_table_CN.Metadata.DialogueCount` 与 `HeatScore`）：
  - 每次对话（无论计费路径）均执行：
    - `Metadata.DialogueCount += 1`
    - 按 `luxLevel` 通过环境变量（`LUX0_DIALOGUE_HEAT_ADD`、`LUX1_DIALOGUE_HEAT_ADD`、`LUX2_DIALOGUE_HEAT_ADD`）为 `HeatScore` 增量，并置脏（进入批处理）。
  - 实现函数：`incrementDialogueCountAndMembershipHeat(characterId, luxLevel, language)`。

参考实现：
- `app/api/official-model/submit/route.ts`
- `lib/server/character-repo.ts`（`addToCharacterTotalLunaCoinUsed`、`addHeatAndMarkDirty` 与 `getCharacterByIdRaw` 支持按 language 选择多表）

## 4. 前端自愈与错误呈现

- 当服务端返回 `deactivateBinding=true`（且对话返回成功）：
  - 前端调用 `LocalCharacterRecordOperations.updateMarketBinding(currentLocalId, { active: false })`，后续对话不再携带 `characterId`。
- 若出现支付失败（402 等），前端不会抛出异常中断工作流；而是返回一条用户可读的 Billing 信息文本（例如：`【Billing】Payment required`）。

参考实现：
- `lib/nodeflow/LLMNode/LLMNodeTools.ts`

## 5. 环境变量

```bash
# LunaCoin 热度增量倍数（仅对 LunaCoin 消耗生效）
LUNACOIN_HOT_INC_MULTIPLIER=2

# Dialogue membership heat per lux level
LUX0_DIALOGUE_HEAT_ADD=0
LUX1_DIALOGUE_HEAT_ADD=0
LUX2_DIALOGUE_HEAT_ADD=0

# 统一的市场窗口 TTL（毫秒），用于前端缓存与后端 presign 参考
CHARACTER_MARKET_REFRESH_INTERVAL=600000
```

## 6. 典型时序（简述）

1) 本地导入：`active=false` → 纯本地对话 → 不触发计数。
2) 市场导入：`active=true`、有 `characterId` → 官方模型调用 → 计费（或免费）与用量累加并发执行。
3) 若社区卡已删除：官方模型仍返回内容，`deactivateBinding=true`；前端将本地 `active=false`。


