#+ RangeHeat 策略与环境变量说明

本文件说明 `RangeHeat` 字段的含义，以及用于控制热度统计窗口与更新频率的环境变量。当前仓库中已包含热度增量与脏标处理（`lib/server/character-repo.ts` 及 `lib/server/process-dirty-hot-batch.ts`、`process-dirty-time-batch.ts`），`RangeHeat` 聚合实现可在后续批处理作业中逐步接入。

## 字段定义

- `RangeHeat`: 指定时间窗口内的综合热度值。用于前端排序或展示“近期热度”。

## 环境变量

- `CHARACTER_RANGE_HEAT_WINDOW`: 热度统计窗口长度。
  - 示例：`7d`（7天）、`30d`（30天）、`1h`（1小时）、`1m`（1分钟）等。
  - 建议：统一采用 `{number}{unit}` 表示法，`unit` 支持 `d`（天）、`h`（小时）、`m`（分钟）、`s`（秒）。

- `CHARACTER_RANGE_HEAT_UPDATE_INTERVAL`: 热度更新频率。
  - 示例：`1d`（每天一次）、`2d`（两天一次）、`1h`（每小时）、`1m`（每分钟）、`30s`（30秒）等。
  - 更新频率通常由定时任务（如 CloudWatch Event/Lambda 定时器）或增量事件流驱动。

- `CHARACTER_MARKET_REFRESH_INTERVAL`: 角色市场列表刷新间隔。
  - 用于控制前端重新拉取 `/api/character/list` 的频率，同时影响后端生成的缩略图 Presigned URL 有效期（取该间隔的约 90%）。
  - 支持单位：`s`、`m`、`h`、`d`、`mo`（按 30 天近似）。

## 实施建议

1. 后端批处理：根据 `CHARACTER_RANGE_HEAT_WINDOW` 聚合近窗内的互动指标（浏览、下载、收藏等），写回 `RangeHeat`。
2. 更新调度：使用 `CHARACTER_RANGE_HEAT_UPDATE_INTERVAL` 控制批处理运行频率，权衡实时性与成本。
3. 前端排序：前端按 `HeatScore` 主排序，必要时使用 `RangeHeat` 做二次排序或标签展示。
4. 可观测性：记录每次计算的时间范围与统计来源，便于回溯与调优。


