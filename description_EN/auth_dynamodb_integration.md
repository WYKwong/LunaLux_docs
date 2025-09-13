# Auth: Cognito + DynamoDB user_table Integration

## Overview
- Keep Cognito for auth, add DynamoDB `user_table` as the source of truth for user profile (username, email, createdAt).
- After email verification and login, initialize the profile and require a unique username.
- Frontend reads user info from `user_table` (O(1) by userId).

## APIs & Services
- DynamoDB client: `lib/server/dynamodb.ts`
- User repo: `lib/server/user-repo.ts`
  - `initUserProfile(userId, email)` (idempotent)
  - `getUserProfile(userId)`
  - `setUsername(userId, username)` (transactional ownership + case-insensitive lower)
- API routes:
  - `POST /api/user/init` (headers include `x-user-id`/`x-session-token`)
  - `POST /api/user/username`
  - `GET /api/user/me`
  - `GET /api/wallet/me`, `GET /api/invite/me`, `POST /api/invite/settle/me`
  - `GET /api/model-config/list`

## Frontend
- `contexts/AuthContext.tsx`: persist `sessionToken/userId` on login, then call `syncProfileAfterLogin(email)` from `lib/data/user-profile-sync.ts`.
- Refresh uses local cache + `GET /api/session/validate` (no DB access) to validate session.
- `updateUsername` applies server response directly without extra fetch.
- If `username` is empty on login, dispatch `require-set-username` to force UI.

## Local-first & Repair
- Continue using SSO headers `x-user-id`/`x-session-token` with `utils/session-client.ts`.
- Server can re-issue session token from Cognito cookie when registry is cold.

## Cross References
- Table design: `description_EN/dynamodb_tables.md`
- Billing flow: `description_EN/official_model_billing_flow.md`
