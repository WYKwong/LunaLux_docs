# Coupons Table 与兑换流程

## 概述
- 表 `coupons_table` 用于承载兑换码，支持一次性与“可多次领取”的新模式。
- 采用 O(1) 主键查询与兑换（主键结构见《DynamoDB 表结构设计》：`description/dynamodb_tables.md`）。

## 表结构
- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `coupons_table` 小节）。

## 环境变量
- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节。

## 服务端实现
- `lib/server/dynamodb.ts`
  - `getCouponsTableName()`、`buildCouponPk(coupon)`
- `lib/server/coupon-repo.ts`
  - `getCoupon(coupon)`：O(1) 查询
  - `claimCouponAtomic(coupon, userId)`：支持两种模式
    - 一次性模式（无 `remainingClaims` 字段）：`Delete` 条件删除，返回 `ALL_OLD`
    - 多次领取模式（含 `remainingClaims: number` 与 `claimedBy: string set`）：
      - 条件：`remainingClaims >= 1 AND (attribute_not_exists(claimedBy) OR NOT contains(claimedBy, :uid))`
      - 原子更新：`ADD remainingClaims -1, ADD claimedBy :uidSet`，并更新 `updatedAt`
      - 若更新后 `remainingClaims <= 0`，尝试条件删除 item（best-effort）
- API：`POST /api/coupon/redeem`
  - Header：需携带会话头（由拦截器与 `ensureSessionOrReissue` 校验）
  - Body：`{ coupon }`
  - 流程：
    1) 原子领取（多次/一次性）与“单用户一次领取”校验
    2) 根据 `rewardType/amount` 调用 `creditWalletBalance()` 入账钱包
    3) 调用 `incrementLedgerSummary()` 累计对应 `Gain`
    4) 返回更新后的 `wallet` 与 `ledger`

### 返回示例
```json
{
  "success": true,
  "currency": "star",
  "amount": 100,
  "wallet": { "userId": "u-1", "luxLevel": 0, "starCoin": 200, "lunaCoin": 100 },
  "ledger": { "userId": "u-1", "totalStarCoinUsed": 0, "totalStarCoinGain": 200, "totalLunaCoinUsed": 0, "totalLunaCoinGain": 100 }
}
```

## 前端实现（/user 页面）
- `app/user/page.tsx` 的“兑换码”面板保持不变。
- 成功后：
  - 调用 `writeLocalUserWallet()/writeLocalLedgerSummary()` 同步本地缓存
  - 根据返回的 `wallet/ledger` 直接刷新 UI，无需额外拉取
  - 根据 `currency/amount` 显示国际化成功消息：如“100星元已添加到您的账户”
  - 失败返回 `COUPON_INVALID_OR_USED` 时，提示“兑换码无效或已被使用/已领取”

## 国际化
- `app/i18n/locales/zh.json` / `en.json` 新增 `coupon.*` 键位：
  - `redeemTitle`、`placeholder`、`redeemButton`、`redeeming`
  - `successStar`、`successLuna`、`invalidOrUsed`、`empty`


