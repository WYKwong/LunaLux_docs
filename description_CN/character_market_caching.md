# 角色市场列表缓存与语言切换行为

本文档说明 `/character-market` 页面角色列表的前端缓存策略、语言切换行为，以及与后端 Presigned URL 的协同。

## 环境变量

```
CHARACTER_MARKET_REFRESH_INTERVAL=600000  # 统一：毫秒 TTL（前端缓存窗口 + 后端 presign 参考）
HOT_TOP_LIMIT=100                         # 首屏 TopHot 数量
GSI_HOT_PAGE_LIMIT=100                    # 懒加载每批数量（GSI_HOT）
# 新增：时间排序（最新）
TIME_TOP_LIMIT=100
GSI_TIME_PAGE_LIMIT=100
```

- 前端：使用接口返回的 `cacheTtlMillis` 作为本地缓存过期时间（服务端即基于该变量返回）。
- 后端：`/api/character/list` 与 `/api/character/hot-first-page` 使用 `CHARACTER_MARKET_REFRESH_INTERVAL` 计算 `cacheTtlMillis`，并据此生成 S3 Presigned URL 有效期（≈ 0.9）。

## 前端缓存（按语言隔离；每窗口内每语言最多拉取一次）

- 使用 `localStorage`，按语言 + 排序分别缓存：
  - `character_market_cache_<lang>_<sort>`：角色数组（`id`, `name`, `thumbnailUrl`, `language`, `likedCount`, `downloadCount`, `totalCoinUsed`, `intro?`）
  - `character_market_cache_exp_<lang>_<sort>`：到期毫秒时间戳
  - `character_market_sort_<lang>`：用户选择的排序（`hot`/`new`）
  - `character_market_sort_exp_<lang>`：排序选项记忆到期时间（与角色缓存一致）
- 页面加载或语言切换时：
  1. 若当前语言缓存未过期，直接读取本地缓存显示；
  2. 若未命中或已过期，清空列表以避免语言闪烁：
     - 最热：`GET /api/character/hot-first-page?lang=<lang>`
     - 最新：`GET /api/character/time-first-page?lang=<lang>`
     写入新缓存。

### 前端关键词搜索（纯前端）

- `/character-market` 页面新增前端搜索框（组件：`components/character-market/SearchPanel.tsx`），在本地已加载的角色列表中按 `name` 字段做不区分大小写的模糊匹配。
- 该搜索不产生额外网络请求，不影响本地缓存，仅作为展示层过滤。
- 过滤优先级：
  1) NSFW 开关、原创开关、Tag 过滤；
  2) 关键词匹配（`includes`，忽略大小写）。
- 过滤管道实现位置：`components/character-market/CardDisplay.tsx`。

### UI 补充：市场卡片收藏、统计栏、简介与标签

- `MarketCharacterCard` 右上角新增“收藏”圆形按钮：
  - 默认：70% 不透明度黑色圆圈，内为白色镂空爱心图标。
  - 悬浮：仅当鼠标置于圆圈上时微放大（scale≈1.05）并变为红色；移出后恢复黑色与原尺寸。缩放动效使用平滑缓出曲线 `cubic-bezier(0.22, 1, 0.36, 1)`，时长约 300ms。
  - 点击：圆圈固定为红色，爱心变为白色实心。再次点击可切换回默认。
  - 该交互仅为前端视觉状态，不影响列表缓存或后端数据。
  - Icon 组件：`components/icons/HeartIcon.tsx`，在 `components/icons/index.ts` 导出。

- 角色名下方新增统计栏组件 `components/CharacterStatsBar.tsx`：
  - 展示顺序与示例样式：
    - 爱心（LikedCount）+ 数字
    - 下载（DownloadCount）+ 数字
    - LunaCoin 用量（TotalLunaCoinUsed）+ 数字
  - 使用的图标：`HeartIcon`、`DownloadIcon`、`StarCoinIcon`（均已在 `components/icons` 下）。
  - `MarketCharacterCard` 通过 props 传入上述三个数字并渲染。

- 统计栏下方展示简介组件 `components/CharacterIntroText.tsx`：
  - 字体：白色（与标题一致）、字号略小于角色名（text-xs）
  - 行数：最多显示两行，超出使用省略号（`line-clamp-2`）
  - 文本来源：角色卡 `intro` 字段（可为空；为空则不渲染）

- 简介下方展示标签组件 `components/CharacterTags.tsx`：
  - 数据来源：接口字段 `tags: string[]`（最多 10 个，见表结构说明）
  - 展示规则：
    - 每个标签前自动添加 `#` 前缀
    - 最多完整显示 2 个标签（可配置 `maxVisible`，默认 2）
    - 若有更多标签，额外显示一个形如 `+N` 的气泡；鼠标悬停在该气泡上，通过原生 `title` 提示展示剩余标签列表
  - 样式：不透明度 80% 的黑色背景（`bg-black/80`），浅灰描边（`border-[#534741]`），浅色文字（`#eae6db`），圆角气泡
  - 组件使用位置：`MarketCharacterCard` 中 `CharacterIntroText` 下方

### 点赞数据（liked_table）集成

- 环境变量：
  - `LIKED_TABLE_NAME`：DynamoDB 表名
  - `LIKED_TTL_SECONDS`：点赞记录 TTL（秒）
- API：
  - `GET /api/liked/me`：返回当前用户点赞过的 `characterId[]`
  - `POST /api/liked/add`：Body `{ characterId }`，写入用户点赞记录，使用 `LIKED_TTL_SECONDS` 计算 `expiresAt`；同时对对应角色卡执行原子自增：
    - `Metadata.LikedCount += 1`
    - 若 `CHARACTER_LIKE_HEAT_ADD > 0`，则 `HeatScore += CHARACTER_LIKE_HEAT_ADD` 并设置 `DIRTY_PK/SK` 标记，进入与对话热度相同的批处理重排流程。
- 前端：
  - 登录成功后调用 `syncLikesAfterLogin()` 拉取并写入本地 `userLikedCharacterIds`
  - `MarketCharacterCard` 初始根据本地列表决定按钮状态；若已点赞，按钮固定红色且不可取消；
  - 点赞时不重新拉取，写库成功后直接在本地列表追加该 `characterId`。

### 环境变量补充

```
LIKED_TABLE_NAME=liked_table           # 点赞表名
LIKED_TTL_SECONDS=2592000              # 点赞记录 TTL（秒）
CHARACTER_LIKE_HEAT_ADD=0              # 单次点赞增加的热度，0 表示不改动热度
```

## 语言切换特殊情况（多语言分表）

- 缓存按语言隔离；切换语言时不会相互污染：
  - 英文缓存存在且未过期 → 切换到英文直接使用缓存；
  - 中文缓存存在且未过期 → 切换到中文直接使用缓存；
  - 若某语言缓存不存在或过期 → 仅对该语言重新拉取。

## 后端配合（多语言分表）

- 接口：
  - 最热：`GET /api/character/hot-first-page`, `POST /api/character/hot-more`
  - 最新：`GET /api/character/time-first-page`, `POST /api/character/time-more`
  - 兼容接口：`GET /api/character/list`
- 后端会：
  - 读取 `CHARACTER_MARKET_REFRESH_INTERVAL`，计算 `presignTtl ≈ 0.9 * interval`；
  - 按 `lang` 参数选择对应表（`CHARACTER_TABLE_NAME_EN` 或 `CHARACTER_TABLE_NAME_CN`），并返回该语言数据；
  - 对缩略图 `s3://bucket/key` 生成 Presigned GET URL，过期时间为 `presignTtl`。
  - 角色表主键为 `PK=CHAR#<uuid>`, `SK=PROFILE`（固定值），不再依赖 `SK` 排序。
  - 列表接口和 `hot-first-page`/`hot-more` 接口返回 `Metadata` 中的 `LikedCount`、`DownloadCount`、`TotalLunaCoinUsed`（兼容旧字段 `TotalCoinUsed` 作为回退；前端拍平成 `totalCoinUsed` 字段），并包含 `Intro` 字段（若存在）。前端在缓存中一并保存，便于复用与快速渲染。


