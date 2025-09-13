# Local Binding (marketBinding) and TotalLunaCoinUsed Rules

## 1) Local binding structure and storage
- Local record: `CharacterRecord`
  - Extra field: `marketBinding?: { active: boolean; characterId?: string }`
  - Store: IndexedDB (object store `CHARACTERS_RECORD_FILE`)
  - APIs:
    - `LocalCharacterRecordOperations.updateMarketBinding(characterId, { active, characterId? })`
    - `LocalCharacterRecordOperations.getMarketBinding(characterId)`
- Rules:
  - Imported from local PNG: `active=false`
  - Imported via community “Start Chat”: `active=true` and store community `characterId`

Refs: `lib/data/roleplay/character-record-operation.ts`, `function/character/import.ts`, `components/MarketCharacterCard.tsx`

## 2) Binding usage in chat
- On entering `/character`, write local id to `sessionStorage.currentCharacterLocalId` for LLM node to read (only for official model path).
- During chat:
  - `active=false`: do not send any community `characterId` (pure local chat)
  - `active=true`: official model path (`LLMNodeTools.invokeLLM` → `POST /api/official-model/submit`) includes `characterId`

Refs: `app/character/page.tsx`, `lib/nodeflow/LLMNode/LLMNodeTools.ts`

## 3) Server-side billing and counters
- Route: `POST /api/official-model/submit`
- Decision order: membership free → star → luna
- Usage accumulation for community card (`Metadata.TotalLunaCoinUsed` in EN/CN tables):
  - When valid `characterId` exists:
    - Free/star: do not add
    - Luna: add the charged amount
  - Two-step DynamoDB update to avoid path overlap:
    1) `SET Metadata = if_not_exists(Metadata, {})`
    2) `SET UpdatedAt = :now ADD Metadata.TotalLunaCoinUsed :amt`
  - If not found in multilingual tables:
    - Do not fail billing; just skip the update
    - Return `deactivateBinding=true` in success response; frontend flips local `active=false`

- Dialogue count and membership heat (`Metadata.DialogueCount`, `HeatScore`):
  - Every chat increments `DialogueCount += 1`
  - Add `HeatScore` by `LUX*_DIALOGUE_HEAT_ADD` and mark dirty
  - Impl: `incrementDialogueCountAndMembershipHeat(characterId, luxLevel, language)`

Refs: `app/api/official-model/submit/route.ts`, `lib/server/character-repo.ts`

## 4) Frontend self-healing and errors
- If response has `deactivateBinding=true` (and success): set local `marketBinding.active=false`
- On 402 billing failures, workflow returns a readable billing text instead of throwing

## 5) Env
```
LUNACOIN_HOT_INC_MULTIPLIER=2
LUX0_DIALOGUE_HEAT_ADD=0
LUX1_DIALOGUE_HEAT_ADD=0
LUX2_DIALOGUE_HEAT_ADD=0
CHARACTER_MARKET_REFRESH_INTERVAL=600000
```

## 6) Typical sequences
1) Local import: `active=false` → pure local chat → no counters
2) Market import: `active=true` + community id → official model → concurrent billing and counters
3) Community card deleted: still return content with `deactivateBinding=true`; frontend flips `active=false`
