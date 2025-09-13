# Character Community Publish (S3 + DynamoDB)

## 1) Upload and initial storage
- Prefer: `/api/character/presign-upload` → PUT to `s3://{CHARACTER_S3_BUCKET}/{CHARACTER_S3_ORIGINAL_PREFIX}/{filename}`
- Fallback: `/api/character/upload-direct` when local CORS blocks browser PUT
- Thumbnail: `/api/character/finalize-publish` calls `lib/server/thumbnail.ts` (sharp) to write WebP 400x400 to thumbnail prefix and stores URL

## 2) DynamoDB tables (multi-language)
- Tables: `character_table_EN` (`CHARACTER_TABLE_NAME_EN`), `character_table_CN` (`CHARACTER_TABLE_NAME_CN`)
- Keys: `PK=CHAR#<uuid>`, `SK=PROFILE`
- Fields: `CharacterID`, `CharacterName`, `Language`, `HeatScore`, `RangeHeat?`, `ThumbnailURL`, `FullImageURL`, `Intro?`, `Tags?`, `OwnerUserId?`, timestamps

## 3) Finalize
- Write entity with `HeatScore=0`, preserve PNG metadata in object
- Mark time-dirty: `DIRTY_TIME_PK/ DIRTY_TIME_SK` for time batching

## 4) Env (excerpt)
```
CHARACTER_TABLE_NAME_EN=character_table_EN
CHARACTER_TABLE_NAME_CN=character_table_CN
CHARACTER_S3_BUCKET=character-cards
CHARACTER_S3_ORIGINAL_PREFIX=characters/original/
CHARACTER_S3_THUMBNAIL_PREFIX=characters/thumbs/
GSI_DIRTY_HOT=GSI_DIRTY_HOT
GSI_HOT=GSI_HOT
GSI_DIRTY_TIME=GSI_DIRTY_TIME
GSI_TIME=GSI_TIME
```

## 5) Market list APIs
- hot: `GET /api/character/hot-first-page`, `POST /api/character/hot-more`
- new: `GET /api/character/time-first-page`, `POST /api/character/time-more`
- list (compat): `GET /api/character/list?lang=zh|en`
- download:
  - `GET /api/character/presign-original?id=...`
  - `GET /api/character/download-original?id=...`
  - `skipCount=1` to skip counters when needed

## 6) Session-scoped download de-dup
- Track in `sessionStorage.character_market_downloads`; set `skipCount=1` if already downloaded this session or already imported
