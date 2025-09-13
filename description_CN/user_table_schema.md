# user_table 设计

完整表结构、字段与环境变量统一维护于《DynamoDB 表结构设计》：`description/dynamodb_tables.md`（参见 `user_table` 小节）。

本文仅补充典型操作与注意事项：

## 典型操作

- 初始化用户：`Put USER#<id> / PROFILE`（条件：不存在）
- 设置用户名（含改名）：`TransactWrite`：
  1) 预占新用户名：`Put USERNAME#<newLower> / OWNER`（条件：不存在）
  2) 更新用户资料：`Update USER#<id> / PROFILE` 设置 `username/usernameLower`
  3) 如存在旧用户名：`Delete USERNAME#<oldLower> / OWNER`（仅当 `userId` 匹配时）

## 注意

- 用户名唯一性通过“用户名占位 + 条件写入事务”保证，不依赖数据库唯一约束。
- 仅大小写变更（`lower` 不变）时，同步占位与展示名，不新增/删除占位。
 - 前端策略：用户登录或注册确认成功后，如 `username` 为空，强制打开设置用户名面板（不可手动关闭），直至设置成功。
