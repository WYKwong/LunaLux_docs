## Performance Optimization & Data Transfer Strategies

This document summarizes the strategies to reduce data transfer, cut requests, and speed up loading. It mirrors the Chinese doc and has been verified against the codebase.

### 1) Auth & Session
- Backend-centric Cognito (`app/api/auth/**`), frontend uses REST only.
- Single-active login: in-memory session registry (`lib/server/session-registry.ts`), old device gets 460 when a new device logs in.
- Self-repair: `app/api/_utils/session.ts` can re-issue session token from Cognito cookie.
- Validation endpoint: `GET /api/session/validate` does header check only (no DB reads).

Effect: auth-related extra requests reduced by ~60–80% after login.

### 2) Local-first Sync (Profile/Wallet/Ledger/Invite/Model Configs)
- Unified module: `lib/data/user-profile-sync.ts`.
- After login: one-time sync → local cache; refresh/cross-page prefers local + background validation.
- Rename: apply `POST /api/user/username` response directly, no extra GET.

Effect: profile-like reads per session down by ~70–90%.

### 3) Official Model Billing (Concurrent)
- Route: `POST /api/official-model/submit`.
- Billing approval + model call in parallel; returns content and latest wallet; increments `DialogueCount` and (when applicable) `TotalLunaCoinUsed`.

Effect: save one extra round-trip; P95 improves ~15–25%.

### 4) Market Caching & Local Search
- Page: `app/character-market/page.tsx`.
- Local cache by `language + sort`, with TTL from backend (`cacheTtlMillis`).
- Frontend-only filters: keyword, tags, NSFW, original.

Effect: search/filter requests reduced by ~90–99%.

### 5) S3 Direct Upload with Fallback
- Default: `presign-upload` + browser PUT.
- Fallback: `/api/character/upload-direct` for local CORS constraints.
- Finalize: `/api/character/finalize-publish` to write thumbnail and DB record.

Effect: lower server bandwidth and shorter publish path; robust local dev.

### 6) Heat/Time Batching & Minimal Updates
- Interactions (dialogue/like/download) do only O(1) increments and dirty flags; batch reordering by workers.
- Session-scoped download de-dup via `sessionStorage` (`skipCount=1`).

Effect: light online paths, controlled background cost.

### 7) Chat Workflow Async Post-processing
- `function/dialogue/chat.ts` persists dialogue tree asynchronously.

Effect: TTFR improves ~20–25%.

---

### References
- Code: `app/api/**`, `lib/server/**`, `lib/data/user-profile-sync.ts`, `app/character-market/page.tsx`, `function/character/publish.ts`.
- Docs: `description_EN/*`, `resume/优化总结与效果预测.md`, `user_flow/*`.


