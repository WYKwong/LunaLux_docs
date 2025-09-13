#+ character_table 主键变更（SK 固定为 PROFILE）迁移说明

本说明文档记录将 `character_table` 的排序键（SK）从数值热度（0/HeatScore）调整为固定字符串 `PROFILE` 的背景、步骤与兼容性策略。

## 背景与动机

- 之前：`PK=CHAR#<uuid>`, `SK=0`（或数值型热度）
- 现在：`PK=CHAR#<uuid>`, `SK=PROFILE`（固定值）

变更原因：
- 使用固定 SK 简化 O(1) 读取逻辑，避免热度字段与主键设计耦合。
- 热度/排行相关字段（`HeatScore`/`RangeHeat`）继续保留为属性，由前端或二级索引/GSI 实现排序与筛选。

## 代码层面的兼容策略

- `lib/server/character-repo.ts`
  - 写入：新写入使用 `SK=PROFILE`。
  - 读取：`getCharacterByIdRaw` 先读 `SK=PROFILE`，若未命中再回退读 `SK=0`（兼容旧数据）。
  - 计数更新：`addToCharacterTotalLunaCoinUsed` 优先更新 `SK=PROFILE`，失败再尝试 `SK=0`。
  - 新增：`Intro` 字段作为简介（字符串，可选）被一并写入实体行与 TopHot 行。

- API 路由：
  - `app/api/character/presign-original/route.ts`、`download-original/route.ts`：读取时先查 `PROFILE`，未命中回退到 `0`。

## 数据迁移（可选的离线批处理）

如需完全去除旧键，可离线执行一次迁移，将旧记录的 `SK=0` 重写为 `SK=PROFILE`：

1. 扫描 `character_table` 中 `PK` 形如 `CHAR#*` 的项目，过滤出 `SK=0` 的条目。
2. 对每条旧记录执行以下事务（伪代码）：
   - `Put` 新键：`{ PK, SK: "PROFILE", ...oldWithoutKeys }`（条件：不存在）
   - 可选：`Delete` 旧键：`{ PK, SK: 0 }`（条件：新键存在）

> 注意：建议分批执行，控制吞吐量与重试；或直接保留旧键，依赖代码的回退逻辑维持兼容。

## 对前端/文档的影响

- 文档已更新为固定 SK 设计，不再说明“SK 为热度”。
- 市场列表与计费/使用计数流程不受影响。

## 环境变量与其他

- 无新增环境变量。
- 相关：`TotalLunaCoinUsed` 仅在 LunaCoin 扣费路径累计；免费与星元路径不再累计。


