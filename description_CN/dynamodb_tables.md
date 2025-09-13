#+ DynamoDB 表结构设计与字段说明

本文档详细描述了项目中正在使用的 DynamoDB 表的结构、设计理念和字段含义。

## 概述

项目使用 DynamoDB 作为主要的数据存储服务，采用 PK/SK（分区键/排序键）的主键模型设计。所有表都通过环境变量配置表名，支持灵活的部署环境配置。

## 新增：liked_table - 点赞表

### 表设计
- 表名：通过 `LIKED_TABLE_NAME` 配置（默认 `liked_table`）
- 主键：
  - PK：`USER#{userId}`
  - SK：`LIKED#{characterId}`
- 字段：
  - `userId`: string
  - `characterId`: string
  - `createdAt`: ISO string
  - `expiresAt?`: number（Epoch 秒，用于 TTL）

### TTL
- 由环境变量 `LIKED_TTL_SECONDS` 控制过期秒数。若设置，例如 30 天，则写入时会设置 `expiresAt = nowSeconds + TTL_SECONDS`。

### 与角色热度联动
- 点赞成功后，后端会原子更新对应角色卡：
  - `Metadata.LikedCount += 1`
  - 若 `CHARACTER_LIKE_HEAT_ADD > 0`，则同步 `HeatScore += CHARACTER_LIKE_HEAT_ADD`，并设置 `DIRTY_PK/SK`，进入热榜批处理流程（同对话触发逻辑）。

## 1. user_table - 用户表

### 表设计
- **表名**: `user_table`（通过环境变量 `USER_TABLE_NAME` 或 `NEXT_PUBLIC_USER_TABLE_NAME` 配置）
- **主键模型**: PK/SK 复合主键
- **用途**: 存储用户基本资料和用户名唯一性管理

### 主键结构

#### 用户资料记录
- **PK**: `USER#{userId}` - 用户唯一标识
- **SK**: `PROFILE` - 固定值，标识用户资料
- **字段**:
  - `userId`: string - Cognito 用户 ID（sub）
  - `email`: string - 用户邮箱地址
  - `createdAt`: string - 账户创建时间（ISO 格式）
  - `username?`: string - 用户自定义用户名（可选）
  - `usernameLower?`: string - 用户名小写形式（可选，用于唯一性检查）

#### 用户名占用记录
- **PK**: `USERNAME#{usernameLower}` - 用户名小写形式
- **SK**: `OWNER` - 固定值，标识用户名所有权
- **字段**:
  - `username`: string - 原始用户名
  - `usernameLower`: string - 用户名小写形式
  - `userId`: string - 拥有该用户名的用户 ID

### 设计特点
- **唯一性保障**: 不依赖 GSI，通过"用户名占位项 + 事务"实现原子性与唯一性
- **大小写处理**: 支持用户名大小写变更，自动同步占位项
- **原子操作**: 用户名设置和修改使用 TransactWrite 确保数据一致性

### 典型操作
- **初始化用户**: `Put USER#{id} / PROFILE`（条件：不存在）
- **设置用户名**: 使用 TransactWrite 原子操作：
  1. 预占新用户名：`Put USERNAME#{newLower} / OWNER`
  2. 更新用户资料：`Update USER#{id} / PROFILE`
  3. 释放旧用户名：`Delete USERNAME#{oldLower} / OWNER`（仅当 userId 匹配时）

### 可选 GSI
- 如需按用户名进行非权威性的快速查询，可考虑添加 `GSI1`：
  - 分区键：`usernameLower`
  - 排序键：`SK`（可选）
- 注意：用户名唯一性仍由“用户名占位项 + 条件事务写入”保证，GSI 仅用于读取便捷。

---

## 2. user_wallet_table - 用户钱包表

### 表设计
- **表名**: `user_wallet_table`（通过环境变量 `USER_WALLET_TABLE_NAME` 或 `NEXT_PUBLIC_USER_WALLET_TABLE_NAME` 配置）
- **主键模型**: PK/SK 复合主键
- **用途**: 管理用户会员等级和虚拟货币余额

### 主键结构
- **PK**: `USER#{userId}` - 用户唯一标识
- **SK**: `WALLET` - 固定值，标识钱包记录

### 字段说明
- `userId`: string - 用户唯一标识
- `luxLevel`: number - 会员等级
  - `0`: Lux（普通用户）
  - `1`: Super Lux（超级会员）
  - `2`: Ultra Lux（至尊会员）
- `starCoin`: number - 星元余额（虚拟货币）
- `lunaCoin`: number - 月华余额（虚拟货币）
- `createdAt`: string - 钱包创建时间（ISO 格式）
- `updatedAt`: string - 最后更新时间（ISO 格式）

### 设计特点
- **简单结构**: 每个用户只有一条钱包记录
- **幂等初始化**: 支持重复初始化，默认余额 100/100，等级 0
- **原子更新**: 支持余额和等级的原子更新操作

### 典型操作
- **初始化钱包**: `Put USER#{id} / WALLET`（条件：不存在）
- **查询钱包**: `Get USER#{id} / WALLET`
- **更新余额**: `Update` 操作，支持部分字段更新

---

## 3. invite_profile_table - 邀请资料表（新增）

### 表设计
- 表名：`invite_profile_table`（通过环境变量 `INVITE_PROFILE_TABLE_NAME` 或 `NEXT_PUBLIC_INVITE_PROFILE_TABLE_NAME` 配置）
- 主键模型：PK/SK 复合主键
- 用途：管理每位用户的邀请码和邀请进度

### 主键结构
- 用户邀请档案记录：
  - PK: `USER#{userId}`
  - SK: `INVITE_PROFILE`
- 邀请码占位记录（用于 O(1) 反查）：
  - PK: `INVITE_CODE#{inviteCode}`
  - SK: `OWNER`

### 字段说明
- `userId`: string - 用户ID
- `inviteCode`: string - 邀请码（注册完成时生成并写入）
- `totalInvites`: number - 总邀请人数（默认0）
- `unissuedRewardCount`: number - 未发放奖励人数（默认0）
- `createdAt`, `updatedAt`

### 典型操作
- 初始化邀请档案：事务写入 `USER#id/INVITE_PROFILE` 与 `INVITE_CODE#code/OWNER`（双项都使用条件表达式避免重复）
- 使用邀请码：O(1) 查找 `INVITE_CODE#code/OWNER`，原子自增邀请人的 `totalInvites` 与 `unissuedRewardCount`
- 里程碑结算：按 `.env.local` 配置的阈值与单次奖励，批量结算 `unissuedRewardCount` 并对钱包与汇总入账

---

## 4. model_config_table - 模型配置表

### 表设计
- **表名**: `model_config_table`（通过环境变量 `MODEL_CONFIG_TABLE_NAME` 或 `NEXT_PUBLIC_MODEL_CONFIG_TABLE_NAME` 配置）
- **主键模型**: 主键设计，使用 PK/SK 结构（PK=`MODEL#{modelId}`，SK=`MODEL`）
- **用途**: 集中管理可用模型的展示与计费配置

### 字段说明
- `displayname`: string - 对用户显示的名称
- `modelType`: string - 模型提供商类型
  - `openai`: OpenAI 模型
  - `gemini`: Google Gemini 模型
- `modelName`: string - 实际模型名称
  - 例如：`gpt-5`、`gemini-2.5-flash`、`gemini-2.5-pro`
- `modelId`: string - 模型唯一标识
  - 例如：`001`、`002` 等
- `freeLevel`: number - 会员免费等级
  - `-1`: 不免费
  - `0`: 普通用户永久免费
  - `1+`: 从对应等级起免费
- `allowStar`: boolean - 是否允许使用星元支付
- `allowLuna`: boolean - 是否允许使用月华支付
- `starCost`: number - 星元消耗数量
  - `-1`: 无效/不支持
  - `>=0`: 具体消耗数量
- `lunaCost`: number - 月华消耗数量
  - `-1`: 无效/不支持
  - `>=0`: 具体消耗数量
- `sortOrder`: number - 排序顺序
  - 用于控制模型在界面中的显示顺序
- `[key: string]: any` - 允许控制台自由添加其他字段

### 设计特点
- **配置驱动**: 完全由表中数据驱动，取代硬编码的模型按钮
- **灵活计费**: 支持多种支付方式和会员等级策略
- **只读设计**: 服务端不提供写接口，仅支持控制台手动配置
- **批量读取**: 使用 Scan 操作一次性拉取所有配置

### 典型操作
- **查询所有配置**: `Scan` 操作获取完整配置列表
- **按ID查询配置**: `Get MODEL#{modelId} / MODEL`，O(1) 复杂度
- **字段验证**: 服务端过滤缺少关键字段的配置项
- **前端缓存**: 登录后一次性拉取并写入本地缓存
- **计费流程**: 详见 `description/official_model_billing_flow.md`

---

## 5. ledger_summary_table - 用户资产汇总表

### 表设计
- 表名：`ledger_summary_table`（通过环境变量 `LEDGER_SUMMARY_TABLE_NAME` 或 `NEXT_PUBLIC_LEDGER_SUMMARY_TABLE_NAME` 配置）
- 主键模型：PK/SK 复合主键
- 用途：累计统计用户资产的获得与消耗

### 主键结构
- PK: `USER#{userId}`
- SK: `LEDGER_SUMMARY`

### 字段说明
- `userId`: string
- `totalStarCoinUsed`: number
- `totalStarCoinGain`: number
- `totalLunaCoinUsed`: number
- `totalLunaCoinGain`: number
- `createdAt`, `updatedAt`: string

### 典型操作
- 初始化：幂等 `Put USER#{id} / LEDGER_SUMMARY`
- 读取：`Get USER#{id} / LEDGER_SUMMARY`
- 自增：`ADD` 原子自增上述累计字段并更新 `updatedAt`

详见 `description/ledger_summary_integration.md`。

---

## 6. coupons_table - 兑换码表

### 表设计
- 表名：`coupons_table`（通过环境变量 `COUPONS_TABLE_NAME` 配置）
- 主键模型：PK/SK 复合主键
- 用途：兑换码的兜底存储与原子核销，支持一次性与多次领取模式

### 主键结构
- PK: `COUPON#{coupon}`
- SK: `COUPON`

### 字段说明
- `coupon`: string
- `rewardType`: number（`0=星元`，`1=月华`）
- `amount`: number
- `remainingClaims?`: number（可选；存在则表示多次领取剩余次数）
- `claimedBy?`: string set（可选；多次领取模式下的已领取用户集合，用于单用户一次校验）

### 典型操作
- 一次性兑换：`Delete` + 条件表达式，返回 `ALL_OLD` 获取奖励信息
- 多次领取：`Update` 条件更新，`ADD remainingClaims -1` 与 `ADD claimedBy :uidSet`，并在耗尽后 best-effort 条件删除

详见 `description/coupons_table_and_redeem_flow.md`。

---

## 7. character_table_EN / character_table_CN - 角色社区表（多语言分表）

### 表设计
- 表名：`character_table_EN`（`CHARACTER_TABLE_NAME_EN`）、`character_table_CN`（`CHARACTER_TABLE_NAME_CN`）
- 用途：存储发布到社区的角色卡信息，支持按热度与按更新时间排序

### 主键结构
- `PK`: string - 形如 `CHAR#<uuid>` 的分区键
- `SK`: string - 固定值 `PROFILE`

### 字段说明
- `CharacterID`: string - 逻辑ID（与 `PK` 的 uuid 对应）
- `HeatScore`: number - 热度值（展示/排行用，不作为 SK）
- `CharacterName`: string - 角色名（可作为GSI）
- `Language`: string - 角色卡语言，如 `en`/`zh`
- `ThumbnailURL`: string - 缩略图对象URL（S3路径）
- `FullImageURL`: string - 原始PNG对象URL（S3路径）
- `Intro?`: string - 简介（在发布到社区前由用户输入，非空且不可全空格）
- `Tags?`: string set - 标签集合（最多10个，支持中英；中文标签建议≤7字符，英文标签建议≤20字符）
- `Metadata`: json - 储存 TotalLunaCoinUsed，LikedCount，DownloadCount，DialogueCount 等数据（兼容旧字段 TotalCoinUsed 作为回退读取）

#### 点赞与热度变更（新增）
- API `/api/liked/add` 将通过 `incrementLikedCountAndMaybeHeat` 实现一次性原子更新：
  - 自增 `Metadata.LikedCount`；
  - 若 `CHARACTER_LIKE_HEAT_ADD > 0`，则同时自增 `HeatScore` 并置脏。
- `CreatedAt`, `UpdatedAt`: string - ISO时间
- `OwnerUserId?`: string - 发布者用户ID

#### 索引与分区（热度/时间）
- 热度稀疏索引（拉取脏卡）：`GSI_DIRTY_HOT`（分区键 `DIRTY_PK='DIRTY#HOT'`，排序键 `DIRTY_SK='TS#<epochMillis>#CHAR#<id>'`）
- 热度全量排序索引：`GSI_HOT`（分区键 `GSI_HOT_PK='ALL#HOT'`，排序键 `GSI_HOT_SK='NEGHEAT#<zeroPad(MAX_HEAT - HeatScore)>#CHAR#<id>'`）
- 热度首屏分区：`PK='TOP#HOT'`，`SK` 为倒序热度键
- 时间稀疏索引（拉取更新时间脏卡）：`GSI_DIRTY_TIME`（分区键 `DIRTY_TIME_PK='DIRTY#TIME'`，排序键 `DIRTY_TIME_SK='TS#<epochMillis>#CHAR#<id>'`）
- 时间全量排序索引：`GSI_TIME`（分区键 `GSI_TIME_PK='ALL#TIME'`，排序键 `GSI_TIME_SK='NEGTIME#<zeroPad(MAX_TIME - UpdatedAtMs)>#CHAR#<id>'`）
- 时间首屏分区：`PK='TOP#TIME'`，`SK` 为倒序时间键

### 环境变量
```bash
CHARACTER_TABLE_NAME_EN=character_table_EN
CHARACTER_TABLE_NAME_CN=character_table_CN
CHARACTER_S3_BUCKET=character-cards
CHARACTER_S3_ORIGINAL_PREFIX=characters/original/
CHARACTER_S3_THUMBNAIL_PREFIX=characters/thumbs/
# 热榜
GSI_DIRTY_HOT=GSI_DIRTY_HOT
GSI_HOT=GSI_HOT
HOT_TOP_LIMIT=100
HOT_TOP_BUFFER=150
MAX_HEAT=999999999999
LUNACOIN_HOT_INC_MULTIPLIER=2
LUX0_DIALOGUE_HEAT_ADD=0
LUX1_DIALOGUE_HEAT_ADD=0
LUX2_DIALOGUE_HEAT_ADD=0

# 批处理调度（毫秒）
BATCH_LIMIT=1000
BATCH_WINDOW_MS_EN=5000
BATCH_WINDOW_MS_CN=5000
# 时间排序批处理调度（毫秒）
TIME_BATCH_WINDOW_MS_EN=5000
TIME_BATCH_WINDOW_MS_CN=5000
```

### 典型写入流程（发布角色）
1. 前端请求服务端获取 S3 预签名URL（`/api/character/presign-upload`）
2. 前端 PUT 上传PNG到S3原始路径
3. 服务端在 `/api/character/finalize-publish` 内生成缩略图并写入缩略图前缀
4. 服务端写入 `character_table` 记录（仅名称、语言、缩略图URL、原图URL等；不保存元数据）


## 环境变量配置

### 必需的环境变量
```bash
# 用户表
USER_TABLE_NAME=user_table
NEXT_PUBLIC_USER_TABLE_NAME=user_table

# 用户钱包表
USER_WALLET_TABLE_NAME=user_wallet_table
NEXT_PUBLIC_USER_WALLET_TABLE_NAME=user_wallet_table

# 模型配置表
MODEL_CONFIG_TABLE_NAME=model_config_table
NEXT_PUBLIC_MODEL_CONFIG_TABLE_NAME=model_config_table

# 兑换码表
COUPONS_TABLE_NAME=coupons_table

# 用户资产汇总表
LEDGER_SUMMARY_TABLE_NAME=ledger_summary_table

# 邀请资料表
INVITE_PROFILE_TABLE_NAME=invite_profile_table

# 区域配置
CLOUD_REGION=us-east-1
COGNITO_REGION=us-east-1
```

## 数据访问模式

### 读取路径
- **用户资料**: O(1) 复杂度，直接通过 `userId` 查询
- **用户名查找**: 通过用户名占位项快速定位用户
- **钱包信息**: O(1) 复杂度，直接通过 `userId` 查询
- **模型配置**: 一次性 Scan 获取，适合配置数据

### 写入模式
- **用户创建**: 幂等操作，支持重复初始化
- **用户名设置**: 事务操作，确保唯一性和一致性
- **钱包更新**: 原子更新，支持部分字段修改
- **配置管理**: 仅支持控制台操作，无服务端写接口

<!-- 扩展性与优化建议已迁移至 improve_description/dynamodb_optimization_ideas.md -->

## 注意事项

1. **用户名唯一性**: 通过占位项和事务实现，不依赖数据库约束
2. **大小写敏感**: 用户名存储支持大小写，但唯一性检查基于小写形式
3. **幂等操作**: 用户和钱包初始化支持重复调用
4. **配置管理**: 模型配置表仅支持控制台操作，无 API 写接口
5. **事务限制**: DynamoDB 事务限制为 25 个项目，当前设计符合要求


