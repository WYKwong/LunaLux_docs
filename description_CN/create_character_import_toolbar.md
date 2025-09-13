## 创建角色 - 导入工具栏（Import Toolbar）

本文档描述 `/create-character` 页面的导入工具栏实现与行为，并与代码实现核对。

### 组件位置
- 路径：`components/create-character/ImportToolbar.tsx`
- UI：与页面风格一致的 session-card 背景，包含两个按钮：
  - 导入文件：打开文件选择器（支持 `.png` 或 `.json`）
  - 清空：重置当前 JSON 并清除已上传图片缓存

### 行为
- 导入 PNG：
  - 使用 `utils/character-parser.parseCharacterCard(file)` 从 PNG 的 `tEXt` chunk（`ccv3`/`chara`）提取嵌入 JSON
  - 更新页面状态中的 JSON
  - 将上传图片写入本地 Blob 存储键 `create_character_uploaded.png`，并更新所选图片
- 导入 JSON：
  - 通过 `File.text()` + `JSON.parse` 更新页面状态 JSON
- 清空：
  - 恢复默认 JSON（`lib/defaults/character-card.json`）
  - 删除缓存图片 Blob，并清空所选图片状态

### 集成
- 页面：`app/create-character/page.tsx`
- 工具栏位于页面顶部，向上传入的处理器回调：
  - 解析文件并更新 `characterJson` 状态
  - 通过 `setBlob`/`deleteBlob` 管理 `selectedImage` 与本地 Blob
- 导出仍由 `utils/character-parser.writeCharacterToPng` 实现。

