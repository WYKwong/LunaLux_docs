# DynamoDB Tables Design (Overview)

This doc summarizes the primary tables used by the project. See code under `lib/server/**` for exact key builders and repos.

## liked_table
- PK: `USER#{userId}`; SK: `LIKED#{characterId}`
- TTL via `LIKED_TTL_SECONDS`
- On like: increment character `Metadata.LikedCount` and optional heat

## user_table
- Profile: `PK=USER#{userId}`, `SK=PROFILE`
- Username ownership: `PK=USERNAME#{lower}`, `SK=OWNER`
- Unique rename via transaction

## user_wallet_table
- `PK=USER#{userId}`, `SK=WALLET`
- `luxLevel`, `starCoin`, `lunaCoin`

## invite_profile_table
- `PK=USER#{userId}`, `SK=INVITE_PROFILE`
- Code owner: `PK=INVITE_CODE#{code}`, `SK=OWNER`

## model_config_table
- `PK=MODEL#{modelId}`, `SK=MODEL`
- Display and billing settings; read-only by API

## ledger_summary_table
- `PK=USER#{userId}`, `SK=LEDGER_SUMMARY`
- Totals for star/luna gain/used

## coupons_table
- `PK=COUPON#{coupon}`, `SK=COUPON`
- One-time or multi-claim with set and decrement

## character_table_EN / character_table_CN
- `PK=CHAR#<uuid>`, `SK=PROFILE`
- Fields include `HeatScore`, `Intro`, `Tags`, `Metadata` (Liked/Download/Dialogue/TotalLunaCoinUsed)
- GSIs/dirty keys support hot/time sorting and batching
