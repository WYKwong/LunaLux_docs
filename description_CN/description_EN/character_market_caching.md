# Character Market Caching and Language Switching

This doc explains frontend caching, language switching, and presigned URL cooperation with backend.

## Env
```
CHARACTER_MARKET_REFRESH_INTERVAL=600000
HOT_TOP_LIMIT=100
GSI_HOT_PAGE_LIMIT=100
TIME_TOP_LIMIT=100
GSI_TIME_PAGE_LIMIT=100
```
- Frontend uses `cacheTtlMillis` from API as local TTL.
- Backend uses `CHARACTER_MARKET_REFRESH_INTERVAL` to compute `cacheTtlMillis` and presigned URL TTL (~90%).

## Frontend caching (per-language)
- LocalStorage keys per language + sort:
  - `character_market_cache_<lang>_<sort>` (items)
  - `character_market_cache_exp_<lang>_<sort>` (expire timestamp)
  - `character_market_sort_<lang>` + `character_market_sort_exp_<lang>`
- On load or language change:
  1) If cache valid, use it
  2) Else clear list to avoid flicker, then fetch
     - hot: `GET /api/character/hot-first-page?lang=<lang>`
     - new: `GET /api/character/time-first-page?lang=<lang>`

### Frontend keyword search
- `components/character-market/SearchPanel.tsx` does local, case-insensitive filter on `name`.
- Filtering order: NSFW/original/tags → keyword.
- Pipeline: `components/character-market/CardDisplay.tsx`.

### UI additions
- Favorite button (visual state only) and stats bar (liked/download/luna used) beneath title.
- Intro and tags rendered with clamp/tooltip.

### Likes integration
- Env: `LIKED_TABLE_NAME`, `LIKED_TTL_SECONDS`, optional `CHARACTER_LIKE_HEAT_ADD`.
- APIs: `GET /api/liked/me`, `POST /api/liked/add`.
- On add: atomic `LikedCount++` and optional heat add + dirty.
- Frontend syncs likes after login and updates local state without reread.

## Language switching
- Caches are isolated per language; missing/expired language triggers fetch only for that language.

## Backend endpoints (multi-language tables)
- hot: `GET /api/character/hot-first-page`, `POST /api/character/hot-more`
- new: `GET /api/character/time-first-page`, `POST /api/character/time-more`
- compatibility: `GET /api/character/list`
- Backend selects table by `lang`, generates presigned URLs, returns `Metadata` (LikedCount/DownloadCount/TotalLunaCoinUsed) and `Intro`.
