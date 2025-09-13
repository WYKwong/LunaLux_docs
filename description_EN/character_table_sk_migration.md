# character_table SK migration (fixed PROFILE)

## Background
- Before: `PK=CHAR#<uuid>`, `SK=0`
- Now: `PK=CHAR#<uuid>`, `SK=PROFILE`
- Reason: simplify O(1) reads and decouple `HeatScore` from primary key design.

## Code compatibility
- `lib/server/character-repo.ts`
  - Writes with `SK=PROFILE`
  - Reads try `PROFILE` then fallback to legacy `0`
  - Counters update try `PROFILE` then fallback

## Optional offline migration
- Put new key if not exists, then delete old key when new exists (batched, throttled)

## Frontend/Doc impact
- Docs updated; sorting relies on attributes + GSIs, not SK
