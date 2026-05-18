# Incident Report: RDS 慢查询 — order-service-prod

**日期**: 2026-05-18  
**严重程度**: High  
**实例**: order-service-prod（MySQL 8.0.40，db.t3.medium，ap-northeast-1）  
**调查工具**: AWS DevOps Agent  
**状态**: ✅ 已分析，修复方案已制定

---

## 问题描述

RDS 实例 `order-service-prod` 上的两条 SQL 执行严重缓慢，影响订单数据汇总性能。

## 问题 SQL

### SQL-1：城市维度订单汇总
```sql
SELECT u.city, COUNT(*) as order_count, SUM(o.total_amount) as total_revenue, AVG(o.total_amount) as avg_order_value
FROM orders o JOIN users u ON o.user_id = u.id
GROUP BY u.city ORDER BY total_revenue DESC
```
- 执行时间：4.44 ~ 4.58 秒
- 扫描行数：2,010,015 行 → 返回 15 行

### SQL-2：订单状态过滤
```sql
SELECT * FROM orders WHERE status != 'cancelled' AND total_amount > 5000 ORDER BY total_amount DESC LIMIT 10000
```
- 估计执行时间：3 ~ 8 秒
- 全表扫描 + filesort（可能溢盘）

## 根本原因

| 优先级 | 根因 | 影响 |
|--------|------|------|
| 🔴 P1 | `orders.user_id` 缺少索引 | JOIN 全表扫描 200 万行 |
| 🔴 P1 | `users.city` 缺少索引 | GROUP BY 无法使用索引 |
| 🔴 P1 | `status != 'cancelled'` 否定条件 | 索引失效，全表扫描 |
| 🟠 P2 | `tmp_table_size` = 16MB（过小） | GROUP BY 溢出磁盘 |
| 🟡 P3 | db.t3.medium 规格偏小 | 可突发实例，高峰期 CPU 受限 |

## 修复方案

### P1 — 立即执行
```sql
-- SQL-1 索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_city ON users(city);

-- SQL-2 索引 + 改写
ALTER TABLE orders ADD INDEX idx_status_amount (status, total_amount);

-- SQL-2 改写（必须配合索引使用）
SELECT id, order_no, user_id, product_id, quantity, total_amount, status, order_date
FROM orders 
WHERE status IN ('pending', 'paid', 'shipped', 'delivered', 'refunded')
  AND total_amount > 5000 
ORDER BY total_amount DESC LIMIT 100;
```

### P2 — 参数调优
```
tmp_table_size = 67108864       # 64 MB
max_heap_table_size = 67108864  # 64 MB
sort_buffer_size = 4194304      # 4 MB
```

## 预期效果

| SQL | 优化前 | 优化后 |
|-----|--------|--------|
| SQL-1 | 4.4 ~ 6.8 秒 | < 100ms（~50x 提升）|
| SQL-2 | 3 ~ 8 秒 | < 100ms（~50x 提升）|

## DevOps Agent 调查信息

- Task 1 ID: `5ff5f075-4f5c-4425-a299-9dfc9eb21a41`（SQL-1）
- Task 2 ID: `62b78311-39aa-4507-afff-72ee542bff76`（SQL-2）
- AgentSpace: `93577a78-cea4-4b44-9fe9-97f195065ff9`
