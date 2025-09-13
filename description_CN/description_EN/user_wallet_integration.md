# User Wallet (user_wallet_table) Integration

## Overview
- Table manages user `luxLevel`, `starCoin`, `lunaCoin`
- Idempotent init after login; local-first caching strategy

## Server
- `lib/server/dynamodb.ts`: `getUserWalletTableName()`, `buildWalletPk(userId)`
- `lib/server/user-wallet-repo.ts`: `init/get/update/deduct/credit`
- APIs: `POST /api/user/init`, `GET /api/wallet/me`

## Frontend
- `lib/data/user-profile-sync.ts`: types and read/write/sync helpers
- `contexts/AuthContext.tsx`: sync wallet once after login
- UI `/user`: badges and balances with icons

## i18n
- `account.wallet`, `account.lux/superLux/ultraLux`, `account.starCoin`, `account.lunaCoin`
