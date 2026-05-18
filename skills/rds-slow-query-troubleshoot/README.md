# RDS 慢查询诊断与优化 Skill

本 Skill 用于快速诊断 RDS MySQL / Aurora MySQL 慢查询问题。

## 触发方式

说 **"RDS 慢查询"** 或 **"这条 SQL 很慢"** 即可激活。

## 功能

- 🔍 通过 AWS DevOps Agent 自动分析慢查询日志、索引情况和 CloudWatch 指标
- 📋 输出分级修复方案（P1 索引/改写 → P2 参数 → P3 架构）
- 📢 可选同步结果到 Slack 线程 + 邮件运维报告
- 🛠️ 可选通过 Kiro 生成可执行的 SQL 和 Shell 脚本

## 典型案例

### 案例 1：JOIN 查询缺少索引
- **SQL**: `SELECT ... FROM orders o JOIN users u ON o.user_id = u.id GROUP BY u.city`
- **根因**: `orders.user_id` 和 `users.city` 缺少索引，全表扫描 200 万行
- **修复**: `CREATE INDEX idx_orders_user_id ON orders(user_id)` + `CREATE INDEX idx_users_city ON users(city)`
- **效果**: 4-7s → < 100ms

### 案例 2：否定条件导致索引失效
- **SQL**: `SELECT * FROM orders WHERE status != 'cancelled' AND total_amount > 5000 ORDER BY total_amount DESC LIMIT 10000`
- **根因**: `!=` 否定条件使 B-tree 索引失效 + 无 `total_amount` 索引导致 filesort
- **修复**: 改写为 `IN(...)` + `ALTER TABLE orders ADD INDEX idx_status_amount (status, total_amount)`
- **效果**: 3-8s → < 100ms

## 相关文件

- [`SKILL.md`](./SKILL.md) — 完整 Skill 指令
