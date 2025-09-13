# Invite System (invite_profile_table) Integration

## Overview
- New table `invite_profile_table` manages invite code and progress per user
- PK/SK design; owner record and code placeholder

## Server
- `lib/server/dynamodb.ts`: `getInviteProfileTableName()`, `buildInviteProfilePk()`, `buildInviteCodePk()`
- `lib/server/invite-profile-repo.ts`: init profile (transaction), get by userId/code, increment inviter counters, settle milestones
- APIs:
  - `POST /api/user/init`: init invite profile idempotently; if referralCode present, increment inviter counters
  - `GET /api/invite/me`: return current profile (init if missing)
  - `POST /api/invite/settle/me`: settle by threshold, credit wallet and ledger

## Frontend
- `lib/data/user-profile-sync.ts`: read/write local invite profile and sync after login
- `components/LoginModal.tsx`: optional referral code field; temporarily stored and set as cookie on signup
- `/user`: display invite code, total invites, unissued reward count
