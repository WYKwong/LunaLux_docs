# User Info Sync and Localization Strategy

## Goals
- Avoid duplicate profile fetch on refresh
- After login, initialize on server and write local caches (profile/wallet/ledger/invite)
- Only refresh when data changes or re-login

## Sources
- Authority: DynamoDB `user_table`
- Auth: Cognito (tokens only)

## Module: `lib/data/user-profile-sync.ts`
- Profile: `read/write/clear`, `initUserProfileOnServer`, `fetchUserProfileFromServer`, `sync*`
- Wallet: `read/write`, `fetch`, `sync`
- Invite: `read/write`, `fetch`, `sync`
- Ledger: `read/write`, `fetch`, `sync`
- Model configs: `read/write`, `fetch`, `sync`

## Session validation endpoint
- `GET /api/session/validate` checks headers only; returns 401/460 on invalid

## Lifecycle
- Login:
  1) Persist `sessionToken/userId`
  2) `syncProfileAfterLogin(email)` (idempotent init + fetch)
  3) Parallel sync wallet/ledger/invite/model-configs
  4) If username empty, dispatch `require-set-username`
- Refresh: prefer local cache and validate via `/api/session/validate`
- Rename: use response to update cache and context directly

## Local keys
- `userId`, `email`, `username`, `isLoggedIn`, `sessionToken`, UI keys for display

## Interceptor
- `utils/session-client.ts` injects SSO headers for same-origin non-auth APIs; dispatches `session-expired` on 460
