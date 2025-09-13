# Settings 页面实现概述

## 概述
- 新增独立的设置页面：`/settings`
- 目标：在页面中提供与 `components/SettingsDropdown.tsx` 相同的核心功能，同时保留原有下拉菜单，不做移除或替换。

## 路由与文件
- 路径：`app/settings/page.tsx`
- 渲染方式：客户端组件（`"use client"`）

## 功能清单（与 SettingsDropdown 对齐）
- 语言切换：`useLanguage`，切换 zh/en，并更新 `document.documentElement.lang`
- 模型设置（页面内展开）：在 `/settings` 点击“Model Settings”后，下拉展开内嵌表单（`components/ModelSettingsInline.tsx`）。该表单已移除“官方模型选择”与“点击启用”逻辑，仅用于管理用户自配置模型（增删改、拉取模型列表、测试）
- 声音开关：`useSoundContext`，持久化于 `localStorage.soundEnabled`
- 重置引导：`useTour.resetTour()` 后刷新页面
- 插件管理：保留 `PluginManagerModal` 依赖，但默认不在页面按钮中显式暴露（与现有 Dropdown 的“暂时隐藏”一致）

## 与布局的事件协作
- `MainLayout.tsx` 新增监听：
  - `openModelSidebar`：打开模型侧边栏
  - `toggleModelSidebar`：切换模型侧边栏
- `/settings` 页面通过 `window.dispatchEvent(new Event("openModelSidebar"))` 触发

## Sidebar 集成
- 在 `components/sidebar/sidebarConfig.tsx` 增加 `Settings` 菜单项，路径 `/settings`
- 图标采用 `components/icons/SettingsIcon.tsx`
- 选中时额外添加旋转效果：`className={\`transition-transform duration-300 ${isSettingsActive ? "rotate-90" : ""}\`}`

## 国际化
- 复用现有键位：
  - `common.settings`
  - `common.switchToEnglish` / `common.switchToChinese`
  - `modelSettings.title`
  - `common.soundOn` / `common.soundOff`
  - `tour.resetTour`

## 样式与交互
- 保持与 Dropdown 相同的深色奇幻主题：容器 `bg-[#1c1c1c]`、边框 `#333333`、悬停 `#252525`
- 与 Sidebar 菜单项风格一致，支持 hover 光晕与平滑过渡

## 依赖
- `useLanguage`、`useSoundContext`、`useTour`
- `PluginManagerModal`（按需显示）

## 测试要点
- 访问 `/settings`：
  - 语言在中英文间可正确切换
  - 点击“模型设置”按钮时，右侧模型设置侧边栏可打开/关闭
  - 声音开关持久化
  - 重置引导后刷新页面
- Sidebar 中“设置”项在当前路由下应高亮，并显示齿轮图标旋转


