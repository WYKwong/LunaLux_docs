# User Page 实现文档

## 概述

本文档描述了本项目中新增的 User 页面实现细节。该页面替代了原有的 `AccountModal` 模态框，为用户提供了更完整的用户资料管理体验。

## 实现目标

1. **页面化体验**：将原本的模态框改为独立页面，提供更好的用户体验
2. **背景样式统一**：采用与character-cards页面相同的背景逻辑和样式
3. **功能完整性**：保持所有原有的用户管理功能
4. **响应式设计**：支持桌面和移动设备的访问

## 技术实现

### 页面结构

- **路径**：`/user`
- **文件位置**：`app/user/page.tsx`
- **组件类型**：Next.js App Router页面组件

### 核心功能

1. **用户信息显示**
   - 用户名（支持编辑）
   - 邮箱地址
   - 用户ID（支持复制）
   - 账户状态标识
   - 钱包信息（新增）：会员等级徽章与余额展示（本地读取）

2. **用户操作**
   - 用户名编辑和保存
   - 登出功能
   - 自动重定向（未登录用户）

3. **背景样式**
   - 动态背景图片加载
   - 渐变效果和动画
   - 与character-cards页面保持一致的视觉风格

4. **钱包展示（新增）**
   - 从本地缓存读取钱包（登录后已同步并持久化，刷新不重复拉取）
   - 显示会员徽章：`Lux`/`Super Lux`/`Ultra Lux`
   - 显示余额：星元与月华，图标 + 数字
   - 国际化键位：`account.lux/superLux/ultraLux/starCoin/lunaCoin`
5. **邀请信息展示（新增）**
   - 本地读取并展示邀请信息：邀请码、总邀请人数、未发放奖励人数
   - 国际化键位：`account.inviteCode/totalInvites/unissuedRewardCount`
6. **兑换码面板（新增）**
   - 新增兑换码输入框 + 按钮，调用 `POST /api/coupon/redeem` 进行 O(1) 兑换
   - 成功后服务端返回 `wallet` 与 `ledger`，前端写入本地缓存并更新UI
   - 根据奖励类型显示国际化提示（如：`100星元已添加到您的账户`）

### 技术特性

- **客户端渲染**：使用"use client"指令
- **状态管理**：React hooks管理组件状态
- **动画效果**：Framer Motion提供流畅的页面动画
- **国际化支持**：完整的多语言支持
- **响应式设计**：适配不同屏幕尺寸

## 组件依赖

### 核心依赖
- `useLanguage`：国际化支持
- `useAuth`：用户认证状态管理
- `useRouter`：页面导航
- `Toast`：错误提示组件

### 样式依赖
- `fantasy-ui.css`：奇幻主题样式
- `framer-motion`：动画效果
- 自定义背景图片：`background_yellow.png`、`background_red.png`

## 路由集成

### Sidebar集成
- 修改 Sidebar 架构后由 `components/sidebar/sidebarConfig.tsx` 驱动菜单项
- 新增“设置（Settings）”菜单项：指向 `/settings`，选中时 SettingsIcon 旋转 90°，与下拉设置按钮视觉一致
- “打开账户”按钮保持跳转 `/user`（底部区），不影响新增设置项

### 移动端集成
- 更新MobileBottomNav组件，移除模态框依赖
- 直接导航到`/user`页面
- 保持响应式设计

### MainLayout更新
- 移除AccountModal组件的渲染
- 简化Sidebar和MobileBottomNav的props传递
- 保持整体布局结构

## 国际化支持

### 新增文本
- `userPage.title`：页面标题
- 英文：`"User Profile"`
- 中文：`"用户资料"`
- `coupon.*`：兑换码相关文案键位
  - `redeemTitle`、`placeholder`、`redeemButton`、`redeeming`、`successStar`、`successLuna`、`invalidOrUsed`、`empty`

### 复用文本
- 用户管理相关文本复用现有的`account`命名空间
- 保持翻译的一致性和完整性

## 用户体验改进

### 优势
1. **更好的导航体验**：独立页面提供更清晰的导航路径
2. **更丰富的交互**：页面化设计支持更复杂的用户操作
3. **一致的视觉风格**：与项目整体设计保持一致
4. **更好的移动端支持**：响应式设计优化移动端体验

### 功能保持
- 用户名编辑功能完全保留
- 用户信息显示更加清晰
- 登出功能正常工作
- 错误处理和提示保持完整

## 安全考虑

### 访问控制
- 未登录用户自动重定向到首页
- 使用`useAuth`钩子验证用户状态
- 客户端和服务器端双重验证

### 数据保护
- 敏感信息（如用户ID）支持复制但不直接暴露
- 用户名编辑有长度和格式验证
- 登出后清理本地状态

## 进一步阅读

更多关于性能优化、兼容性与未来规划，参见：
- `improve_description/user_page_optimization_and_future.md`

## 总结

User页面的实现成功地将原有的模态框体验升级为完整的页面体验，在保持功能完整性的同时，提供了更好的用户体验和更一致的视觉设计。该实现为项目的用户管理功能奠定了良好的基础，为未来的功能扩展提供了可能性。

---

*本文档将随着项目发展持续更新，如有问题或建议，请通过项目渠道反馈。*
