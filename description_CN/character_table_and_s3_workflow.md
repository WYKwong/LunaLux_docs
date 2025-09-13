#+ 角色社区发布流程（S3 + DynamoDB）

本文档描述“发布到角色社区”的端到端流程与相关服务配置。

## 一、上传与初始存储

1. 前端上传角色卡PNG（内嵌JSON元数据）
   - 优先：通过 `/api/character/presign-upload` 获取 S3 预签名 PUT URL，前端直接 PUT 上传至 `s3://{CHARACTER_S3_BUCKET}/{CHARACTER_S3_ORIGINAL_PREFIX}/{filename}`
   - 回退：若本地开发环境遇到 S3 CORS 阻拦，自动回退到 `/api/character/upload-direct` 由服务端中转写入 S3
2. 缩略图生成（服务端）
   - 由 `/api/character/finalize-publish` 调用 `lib/server/thumbnail.ts` 使用 sharp 生成 400x400 WebP 缩略图
   - 写入 `s3://{CHARACTER_S3_BUCKET}/{CHARACTER_S3_THUMBNAIL_PREFIX}/{basename}.webp`
   - 然后写入 DynamoDB `thumbnailUrl` 为 `s3://.../{basename}.webp`

## 二、DynamoDB 表设计（多语言分表）

- 表：`character_table_EN`（`CHARACTER_TABLE_NAME_EN`）、`character_table_CN`（`CHARACTER_TABLE_NAME_CN`）
- 主键：
  - `PK`（分区键，形如 `CHAR#<uuid>`）
  - `SK`（固定字符串）=`PROFILE`
- 字段：
  - `CharacterID`（与 `PK` 对应的 uuid）、`CharacterName`、`Language`（`en`/`zh`）
  - `HeatScore`、`RangeHeat`（用于热度展示与排行，不再作为 SK）
  - `ThumbnailURL`、`FullImageURL`
  - `CreatedAt`、`UpdatedAt`、`OwnerUserId?`

## 三、发布流程（Finalize）

1. 前端在上传完成后，携带 `s3Key`、角色名（可缺省）、语言、以及必填的 `intro` 简介，调用 `/api/character/finalize-publish`（需有效会话；服务端会在重启后自动接纳现有 `x-user-id/x-session-token`）。若缺省名称，服务端将从文件名推导；`intro` 不可为空或纯空格。
2. 服务端根据文件名推导缩略图Key，拼接 `s3://` URL
3. 服务端写入 `character_table` 记录（`SK=PROFILE`，初始 `HeatScore=0`），并保存 `Intro` 字段；元数据仍保留在 PNG 中。
4. 同步标记“更新时间脏标”（仅发布时）：写入 `DIRTY_TIME_PK='DIRTY#TIME'` 与 `DIRTY_TIME_SK='TS#<epochMs>#CHAR#<id>'`，供时间排序批处理读取。未来如有“编辑角色卡”接口，可复用相同的脏标逻辑。

## 四、环境变量（更新）

```bash
CHARACTER_TABLE_NAME_EN=character_table_EN
CHARACTER_TABLE_NAME_CN=character_table_CN
CHARACTER_S3_BUCKET=character-cards
CHARACTER_S3_ORIGINAL_PREFIX=characters/original/
CHARACTER_S3_THUMBNAIL_PREFIX=characters/thumbs/
# 热榜
GSI_DIRTY_HOT=GSI_DIRTY_HOT
GSI_HOT=GSI_HOT
# 最新（时间排序）
GSI_DIRTY_TIME=GSI_DIRTY_TIME
GSI_TIME=GSI_TIME
TIME_TOP_LIMIT=100
TIME_TOP_BUFFER=150
GSI_TIME_PAGE_LIMIT=100
```

### 新增/保留

```bash
# LunaCoin 热度增量倍数（仅对 LunaCoin 消耗生效）
LUNACOIN_HOT_INC_MULTIPLIER=2
```

## 五、市场列表 API（前端渲染）

- 最热（热度排序）
  - 首屏：`GET /api/character/hot-first-page?limit=100`
  - 懒加载：`POST /api/character/hot-more`（Body 可携带 `exclusiveStartKey`，每批数量由 `GSI_HOT_PAGE_LIMIT` 控制）
- 最新（按更新时间排序）
  - 首屏：`GET /api/character/time-first-page?limit=100`
  - 懒加载：`POST /api/character/time-more`（Body 可携带 `exclusiveStartKey`，每批数量由 `GSI_TIME_PAGE_LIMIT` 控制）
- 兼容接口：`GET /api/character/list?lang=zh|en`（当前默认读取 TopHot）
- 下载接口：
  - 预签直连：`GET /api/character/presign-original?id=<CharacterID>`（成功后计入下载）
  - 服务端代理下载：`GET /api/character/download-original?id=<CharacterID>`（成功后计入下载）
  - 下载计数：对 `Metadata.DownloadCount` 自增 1；若 `CHARACTER_DOWNLOAD_HEAT_ADD > 0`，则对 `HeatScore` 增加同值，并打脏（与对话增量一致）。
  - 计数跳过：上述两个接口均支持可选 `skipCount=1`，服务端将跳过 DownloadCount 和 HeatScore 的更新。

### 前端（/character-market）会话内下载记录与跳过计数

- 客户端在本会话内使用 `sessionStorage.character_market_downloads` 记录已下载的 `CharacterID`；刷新不清除，登出或会话过期时清除（浏览器默认行为）。
- 当用户点击“立即对话”时：
  - 若本地已导入同一社区 `characterId` 的角色卡，或在本会话已下载过该 `characterId`，则附带 `skipCount=1` 调用下载接口，避免重复计数与热度增加；同时不阻止再次导入。
  - 下载成功后，将 `characterId` 写入 `sessionStorage.character_market_downloads`。
- 前端页面：`/character-market` 使用 `MarketCharacterCard` 组件网格展示，目前展示 `name + thumbnail + stats`，并在数据结构中包含 `intro`（可用于后续详情页）。

### 语言过滤与缓存

- 若传入 `lang`，后端会对 DynamoDB 结果进行语言过滤，仅返回对应语言的角色。
- 后端生成的缩略图 Presigned URL 有效期与 `CHARACTER_MARKET_REFRESH_INTERVAL`（毫秒）协调（约其 90%）。
- 前端基于 `localStorage` 进行按语言 + 排序缓存，键：
  - `character_market_cache_<lang>_<sort>`：角色数组
  - `character_market_cache_exp_<lang>_<sort>`：到期毫秒时间戳（与接口 `cacheTtlMillis` 对齐）
  - `character_market_sort_<lang>` 与 `character_market_sort_exp_<lang>`：在相同 TTL 窗口内记忆用户“最热/最新”的选择

## 五、前端交互（ImportCharacterModal）

- 支持“导入到本地”与“发布到角色社区”两种模式
- 选择发布时需要选择语言（默认英文），并填写简介 `intro`（必填，非空且不可全空格）
- 本地导入沿用既有逻辑（在前端解析PNG中的元数据并存入本地）；发布模式仅执行预签名上传 + Finalize API（不解析元数据）


