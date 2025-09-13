## 性能优化与传输策略（减少数据传输、加快加载）

本文汇总本项目为减少数据传输、降低请求次数与加快加载所采取的策略，来源于本目录各模块文档、`resume/优化总结与效果预测.md` 与 `user_flow/` 用户路径说明，并与代码实现核对。

### 1. 鉴权与会话
- 后端集中对接 Cognito（`app/api/auth/**`），前端仅使用 REST 接口。
- 单活登录：`lib/server/session-registry.ts` 基于内存注册表；新设备登录令旧设备失效（统一返回 460）。
- 会话自修复：`app/api/_utils/session.ts` 可基于 Cookie 重签 SessionToken，避免误过期。
- 刷新校验端点：`GET /api/session/validate` 仅做头校验，不访问数据库。

效果：
- 登录后页面切换无需重复拉取资料；鉴权相关多余请求预计减少 60–80%。

### 2. 本地优先同步（资料/钱包/汇总/邀请/模型配置）
- 统一模块：`lib/data/user-profile-sync.ts`。
- 登录后一次性同步 Profile、Wallet、LedgerSummary、InviteProfile、ModelConfigs 并写入本地。
- 刷新与跨页：优先读本地缓存，后台轻量校验会话。
- 改名：使用 `POST /api/user/username` 返回体直接更新本地，不再额外 GET。

效果：
- 同类数据重复拉取显著减少；单会话资料类读取请求预计下降 70–90%。

### 3. 官方模型计费并发调用
- 路由：`POST /api/official-model/submit`。
- 审批计费与调用并发（Promise.all）：返回内容与最新钱包；累计 `DialogueCount`，必要时累加 `TotalLunaCoinUsed`。

效果：
- 端到端往返减少 1 次；P95 延迟预计提升 15–25%。

### 4. 角色市场列表缓存与本地搜索
- 页面：`app/character-market/page.tsx`。
- 本地缓存键：`character_market_cache_<lang>_<sort>` 与过期时间；语言与排序维度隔离。
- 首屏与懒加载：`/api/character/hot-first-page|hot-more|time-first-page|time-more`；后端返回 `cacheTtlMillis`。
- 关键词/Tag/NSFW/原创过滤：全部在前端完成，不产生额外请求。

效果：
- 多次进入或语言切换基本命中本地缓存；搜索/筛选阶段后端请求减少 90–99%。

### 5. S3 直传与回退
- 默认：`/api/character/presign-upload` 获取预签名 URL，前端直传。
- 本地/多域 CORS 受限：回退到 `/api/character/upload-direct` 由服务端中转。
- 发布完成：`/api/character/finalize-publish` 生成缩略图并写表。

效果：
- 避免服务端带宽放大，缩短发布耗时；同时保障本地开发的可用性。

### 6. 热度/时间批处理与最小化更新
- 对话/点赞/下载只做 O(1) 自增与脏标，批处理重排（`lib/server/process-dirty-hot-batch.ts`、`process-dirty-time-batch.ts`）。
- 下载在本会话内使用 `sessionStorage` 去重（可携带 `skipCount=1`）。

效果：
- 在线接口轻量稳定；重排成本集中到可控的批任务。

### 7. 对话工作流异步化
- `function/dialogue/chat.ts` 的后处理入库（对话树）转为异步，不阻塞主链路。

效果：
- 首次可见时间预计提升 20–25%。

---

### 参考与路径
- 代码：`app/api/**`、`lib/server/**`、`lib/data/user-profile-sync.ts`、`app/character-market/page.tsx`、`function/character/publish.ts`。
- 文档：`description/*`、`resume/优化总结与效果预测.md`、`user_flow/*`。


