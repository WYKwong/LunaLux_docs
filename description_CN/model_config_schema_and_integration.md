# 模型配置（model_config_table）表与集成说明

## 概述
- 新增 DynamoDB 表 `model_config_table` 用于集中管理可用模型的展示与计费配置（表名由环境变量 `MODEL_CONFIG_TABLE_NAME` 配置）。
- 登录后从服务端一次性拉取所有模型配置，写入本地缓存，供模型设置侧边栏动态展示。
- 取代此前写死的 gpt-5、gemini-2.5-flash、gemini-2.5-pro 按钮，完全由表中数据驱动。

## 表结构
- 统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `model_config_table` 小节）。
> 提示：该表的元素由控制台手动添加。服务端不提供写接口，仅提供只读查询。

## 环境变量
- 统一维护于《DynamoDB 表结构设计》的“环境变量配置”章节。

## 服务端实现
- `lib/server/dynamodb.ts`
  - 新增：`getModelConfigTableName()` 读取表名
- `lib/server/model-config-repo.ts`
  - `listAllModelConfigs()`：使用 Scan 拉取所有配置，并做基本字段校验
  - `getModelConfigById(modelId)`：使用 O(1) Get 按主键读取单个配置（主键设计见《DynamoDB 表结构设计》）
- API：
  - `GET /api/model-config/list`：受会话保护，返回 `{ success, configs }`

## 前端实现
- `lib/data/user-profile-sync.ts`
  - 新增类型：`ModelConfigItem`、`ModelConfigList`
  - 本地缓存读写：`readLocalModelConfigs()`、`writeLocalModelConfigs()`
  - 服务端拉取：`fetchModelConfigsFromServer()`
  - 登录后同步：`syncModelConfigsAfterLogin()`
- `contexts/AuthContext.tsx`
  - 登录成功后：`syncProfileAfterLogin` → `syncWalletAfterLogin` → `syncModelConfigsAfterLogin()`（失败不阻断）
- `components/ModelSidebar.tsx`
  - 移除硬编码官方模型按钮，改为读取本地 `modelConfigs` 动态渲染
  - 点击模型即开启官方模型模式并写入：`localStorage.officialModelEnabled=true`、`officialModelName`、`officialModelType`、`officialModelId`
  - 成本展示逻辑：
    - 若 `allowStar && starCost >= 0` 显示 星元 图标与数字
    - 若 `allowLuna && lunaCost >= 0` 显示 月华 图标与数字
    - 若 `freeLevel >= 0` 显示"免费等级"徽标（i18n：`modelSettings.freeLevel`）
- `app/user/page.tsx`
  - 提取 星元/月华 图标组件到 `components/icons`，在多个位置复用

## 国际化
- `app/i18n/locales/en.json` / `zh.json`
  - 新增：
    - `modelSettings.costStar`、`modelSettings.costLuna`（如需文字标签）
    - `modelSettings.freeLevel`：英文 `Free L{level}`，中文 `对L{level}免费`

## 与聊天调用的衔接
- 前端工作流 `LLMNodeTools.invokeLLM`：
  - 读取 `localStorage.officialModelEnabled/name`
  - 自动推断提供商：`gemini-` → `gemini`，包含 `gpt` → `openai`
  - 通过 `/api/official-model/submit` 使用服务端密钥转发调用；在官方模式下前端随 body 携带 `modelId` 与 `userId`，后端将按以下顺序验证计费并并发执行：会员免费 → 星元 → 月华；若非免费则异步原子扣费（DynamoDB ADD 负数）并与模型内容一并返回余额

## 测试要点
- 登录后打开模型设置侧边栏，能看到 `model_config` 中的全部项；
- 选择某个项后，后续对话将按其 `modelType/modelName` 转发调用；
- 当 `allowStar` 或 `allowLuna` 为 `false` 时，不显示对应图标与数字；
- 当 `freeLevel >= 0` 时显示对应免费徽标；
- 刷新页面后仍保留已选项（读取 localStorage）。

---

## 进一步阅读
- 参见《官方模型扣费与调用流程》：`description/official_model_billing_flow.md`