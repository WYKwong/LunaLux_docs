# Lunalux

Lunalux is a character‑centric AI chat application with a production‑ready architecture.

## Architecture Overview
- Frontend (Next.js App Router): `/character` (chat), `/character-market` (market), `/create-character`, `/user`, `/settings`
- Dialogue workflow: preset → context → worldbook → llm → regex → plugin
- Backend (API Routes): auth, official model gateway, character community, user/wallet/ledger/invite/coupons
- Data (DynamoDB + S3): O(1) reads, conditional atomic writes, dirty + batch processing for hot/time ranking

## Modules (with deep links)

- Authentication & Session
  - Cognito + DynamoDB user profiles; single‑active session; header adoption & cookie‑based repair
  - Docs: [Auth + user_table](description_EN/auth_dynamodb_integration.md), [User info sync](description_EN/user_info_sync_and_localization.md)

- Official Model Billing
  - Approval order: free → star → luna; concurrent LLM call + wallet deduction; ledger summary updates
  - Docs: [Billing flow](description_EN/official_model_billing_flow.md), [Model config table](description_EN/model_config_schema_and_integration.md)

- Character Community & Market
  - Presign/Direct upload, finalize with thumbnail, hot/time lists, downloads, likes; local per‑language cache
  - Docs: [S3 + DynamoDB workflow](description_EN/character_table_and_s3_workflow.md), [Market caching](description_EN/character_market_caching.md), [S3 CORS fallback](description_EN/s3_cors_and_upload_fallback.md), [SK migration](description_EN/character_table_sk_migration.md)

- Dialogue Workflow & Binding
  - Node‑based pipeline; async post‑processing; optional community binding for official models
  - Docs: [Chat workflow](description_EN/character_chat_workflow.md), [Binding & usage](description_EN/character_binding_and_usage.md)

- User Data & Operations
  - Wallet (lux/star/luna), Ledger summary, Invite profile, Coupons
  - Docs: [Wallet](description_EN/user_wallet_integration.md), [Ledger](description_EN/ledger_summary_integration.md), [Invite](description_EN/invite_profile_integration.md), [Coupons](description_EN/coupons_table_and_redeem_flow.md)

- Data Model
  - Tables, keys, fields, and access patterns
  - Docs: [DynamoDB tables](description_EN/dynamodb_tables.md)

- Performance & Heat Policies
  - Local‑first sync, request reduction, presigned URLs, dirty+batch, session repair
  - Docs: [Performance optimization](description_EN/performance_optimization.md), [RangeHeat policy](description_EN/range_heat_policy.md)

## Documentation
- English: `description_EN/`
- Chinese: [README_CN.md](./README_CN.md), `description_CN/`