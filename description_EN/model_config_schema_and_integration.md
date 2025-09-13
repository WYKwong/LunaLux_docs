# Model Config (model_config_table) and Integration

## Overview
- Central table `model_config_table` manages model options and billing configuration.
- Read once after login and cache locally; Sidebar renders dynamically.

## Server
- `lib/server/dynamodb.ts`: `getModelConfigTableName()`
- `lib/server/model-config-repo.ts`: `listAllModelConfigs()`, `getModelConfigById(modelId)`
- API: `GET /api/model-config/list` (session-protected)

## Frontend
- `lib/data/user-profile-sync.ts`: types + `fetch/sync/read/write` helpers for model configs
- `components/ModelSidebar.tsx`: render from cached configs; show free level and costs
- Enable official model mode and store selection in localStorage

See also: `description_EN/official_model_billing_flow.md`.
