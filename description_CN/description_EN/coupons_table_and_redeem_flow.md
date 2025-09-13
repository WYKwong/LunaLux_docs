# Coupons Table and Redeem Flow

## Overview
- `coupons_table` supports one-time and multi-claim modes
- O(1) primary-key access (see `description_EN/dynamodb_tables.md`)

## Server
- `lib/server/dynamodb.ts`: `getCouponsTableName()`, `buildCouponPk()`
- `lib/server/coupon-repo.ts`: `getCoupon()`, atomic claim (delete or update with set + decrement)
- API: `POST /api/coupon/redeem`
  - Body: `{ coupon }`
  - Flow: atomic claim → credit wallet → increment ledger summary → return updated wallet/ledger

## Response example
```
{
  "success": true,
  "currency": "star",
  "amount": 100,
  "wallet": { "userId": "u-1", "luxLevel": 0, "starCoin": 200, "lunaCoin": 100 },
  "ledger": { "userId": "u-1", "totalStarCoinUsed": 0, "totalStarCoinGain": 200, "totalLunaCoinUsed": 0, "totalLunaCoinGain": 100 }
}
```

## Frontend (/user)
- Keep the redeem panel
- On success: write local wallet/ledger, show i18n toast (amount + currency)
- On failure (`COUPON_INVALID_OR_USED`): show i18n error
