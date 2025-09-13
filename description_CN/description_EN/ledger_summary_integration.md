# Ledger Summary (ledger_summary_table) Integration

## Overview
- O(1) ledger summary per user: totals for star/luna gain/used
- Cached locally after login; `/user` shows i18n stats
- Official model path increments used; coupons/invites increment gain

## Server
- `lib/server/dynamodb.ts`: `getLedgerSummaryTableName()`, `buildLedgerSummaryPk(userId)`
- `lib/server/ledger-summary-repo.ts`: `init/get/increment`
- APIs:
  - `GET /api/ledger/summary/me`
  - `POST /api/user/init`
  - `POST /api/official-model/submit`
  - `POST /api/coupon/redeem`

## Frontend
- `lib/data/user-profile-sync.ts`: types and `fetch/sync/read/write` helpers
- `contexts/AuthContext.tsx`: sync after login along with wallet/model configs

Notes: free path does not update ledger used; conditional updates ensure consistency.
