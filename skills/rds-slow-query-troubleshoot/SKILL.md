---
name: rds-slow-query-troubleshoot
display_name: RDS 慢查询诊断与优化
icon: "🔍"
description: |
  当 RDS MySQL/Aurora 实例出现 SQL 查询缓慢、全表扫描、filesort 等性能问题时使用本 skill。
  触发信号：用户报告某条 SQL 执行缓慢、慢查询日志告警、CloudWatch CPUUtilization 升高、
  应用响应超时。使用 AWS DevOps Agent 自动调查根因，输出索引优化和查询改写方案。
trigger: "RDS 慢查询"
inputs:
  - name: sql_query
    description: "需要诊断的 SQL 语句"
    type: string
    required: true
  - name: db_instance
    description: "RDS 实例 ID（如 order-service-prod）"
    type: string
    required: true
  - name: aws_account
    description: "AWS 账号 ID（12 位数字）"
    type: string
    required: true
  - name: region
    description: "AWS Region（如 ap-northeast-1）"
    type: string
    required: true
    default: "ap-northeast-1"
  - name: slack_channel
    description: "需要同步结果的 Slack 频道（可选）"
    type: string
    required: false
  - name: notify_email
    description: "发送运维报告的邮件地址（可选）"
    type: string
    required: false
depends-on: []
---

## Overview

本 Skill 专用于 RDS MySQL / Aurora MySQL 的慢查询根因分析与优化建议生成。
当收到慢查询告警或用户报告 SQL 执行缓慢时，自动调用 AWS DevOps Agent 深度调查
（慢查询日志 + CloudWatch 指标 + 索引结构分析），输出分级修复方案，并可选同步到 Slack 和邮件。

## Workflow

### Step 1: 信息收集
- **Mode**: `agentic`
- **Input**: `{{sql_query}}`, `{{db_instance}}`, `{{aws_account}}`, `{{region}}`
- **Output**: 确认实例 ARN、明确问题 SQL、了解基本背景
- **Validate**: 确认 AWS 账号和 Region 可访问，实例存在
- **On failure**: 提示用户确认账号权限或实例 ID 是否正确

收集以下信息：
- 完整 SQL 语句（含 WHERE/JOIN/ORDER BY/LIMIT）
- 表的大致数据量（便于判断全表扫描影响程度）
- 是否已知相关索引情况（可选）
- 期望执行时间 SLA（如 < 500ms）

### Step 2: 创建 DevOps Agent 调查任务
- **Mode**: `deterministic`
- **Tool**: `aws_devops_agent__create_investigation`
- **Input**: `{{sql_query}}`, `{{db_instance}}`, `{{aws_account}}`, `{{region}}`
- **Output**: `task_id`, `execution_id`，任务状态变为 IN_PROGRESS
- **Validate**: 返回有效 task_id，status 为 PENDING_START 或 IN_PROGRESS
- **On failure**: 检查 agent_space_id 是否正确（user space: 93577a78-cea4-4b44-9fe9-97f195065ff9）

调查描述应包含：
```
实例: {{db_instance}}（账号: {{aws_account}}, Region: {{region}}）
问题 SQL: {{sql_query}}
请分析：慢查询日志记录、相关字段索引情况、执行计划问题、优化建议
```

### Step 3: 轮询等待调查完成
- **Mode**: `deterministic`
- **Tool**: `aws_devops_agent__get_task`
- **Input**: `task_id` from Step 2
- **Output**: 任务状态变为 COMPLETED
- **Validate**: `status == 'COMPLETED'`
- **On failure**: 最多等待 5 分钟；若 FAILED 则读取错误日志后降级为手动分析

每 30 秒轮询一次，或使用后台任务（start_task）等待完成后回调。

### Step 4: 读取调查日志提取根因
- **Mode**: `deterministic`
- **Tool**: `aws_devops_agent__list_journal_records`
- **Input**: `execution_id` from Step 2, `order=ASC`
- **Output**: 完整调查日志，含慢查询确认、指标数据、根因分析
- **Validate**: 日志中包含 role=user 的子任务结果消息
- **On failure**: 若日志为空，检查任务是否真正 COMPLETED；必要时直接调用 CloudWatch / RDS API 手动验证

重点提取：
- 慢查询日志中的 Query_time / Rows_examined / Rows_sent
- CloudWatch 指标（CPU、连接数、IOPS）
- 缺失索引列表
- 查询改写建议（`!=` → `IN`、`SELECT *` → 明确列名等）

### Step 5: 生成修复方案
- **Mode**: `agentic`
- **Input**: 调查日志中的根因分析
- **Output**: 分级修复方案（P1/P2/P3）

按以下优先级输出：

**P1（立即执行 — 最高收益）**
- 缺失索引的 `CREATE INDEX` / `ALTER TABLE ADD INDEX` DDL
- 需要改写的 SQL（如 `!=` → `IN`，`SELECT *` → 明确列名）
- 预期效果：查询时间从秒级 → 毫秒级

**P2（近期执行 — 参数调优）**
- `tmp_table_size`, `max_heap_table_size` → 64 MB+
- `sort_buffer_size` → 2-4 MB

**P3（中长期 — 架构优化）**
- 实例规格升级（t3.medium → m6g.large）
- 启用 Performance Insights
- 读写分离（只读副本分担分析查询）

### Step 6: 同步结果（可选）
- **Mode**: `agentic`
- **Input**: 修复方案 from Step 5, `{{slack_channel}}`, `{{notify_email}}`
- **Output**: Slack 线程回复 + 运维报告邮件
- **Validate**: Slack API 返回 ok:true；邮件发送成功
- **On failure**: 若 Slack channel_not_found，尝试搜索频道名；若邮件失败检查地址格式

若指定了 `{{slack_channel}}`，发送包含根因摘要和 P1 修复 SQL 的 Slack 消息。
若指定了 `{{notify_email}}`，发送完整 HTML 格式运维报告邮件。

### Step 7: 生成执行脚本（可选，使用 Kiro）
- **Mode**: `agentic`
- **Tool**: `send_message_to_acp_agent` (acp-kiro)
- **Input**: 修复方案, 目标保存路径
- **Output**: SQL 脚本文件 + Shell 脚本文件
- **Validate**: 文件成功写入目标路径
- **On failure**: 指定完整绝对路径重新发送（Kiro 需要明确路径，否则会报 permission error）

## Output

- **调查摘要**：根因列表（优先级 + 说明 + 影响范围）
- **修复 SQL**：可直接执行的 DDL 和查询改写版本
- **效果预估**：优化前后执行时间对比（预期）
- **执行脚本**（可选）：`optimize_rds.sql` + `apply_rds_params.sh`
- **运维报告**（可选）：HTML 格式邮件 + Slack 线程同步

## Lessons Learned

### Do
- 先用 `aws_devops_agent__create_chat` 做快速预分析，确认问题方向后再创建正式 Investigation
- 调查描述中包含具体 SQL、账号、Region、已知背景，DevOps Agent 分析质量更高
- 等待 Investigation 完成时，用后台任务轮询，避免阻塞主线程
- `list_journal_records` 用 `order=ASC` 读取，找 role=user 的大段文本消息（子任务结果）
- 向 Slack 发消息时，使用 `thread_ts` 在原始 Ticket 线程中回复，保持上下文连贯

### Don't
- 不要将 `!=`（NOT EQUAL）语义的 SQL 直接当作已确认全表扫描 — 要用 EXPLAIN 或慢查询日志确认
- 不要在生产高峰期执行 `ALTER TABLE ADD INDEX`（200 万行大表加索引会锁表或影响性能）
- 不要只看 CPU 低就断定没问题 — DatabaseConnections=0 时 CPU 低是因为没有负载，不代表实例健康
- 不要让 Kiro 写文件时省略目标路径，必须指定完整绝对路径（如 `/Users/xxx/Documents/2026/`）

### Common Failures
- **channel_not_found**: Slack 频道 ID 与工作区不匹配。解决方案：先 `channels_list` 搜索频道名找到正确 ID
- **Kiro 文件写入失败**: 未指定完整路径。解决方案：重新发送任务并明确 `保存到 /Users/.../目录`
- **Investigation 无 Recommendations**: 分析类调查（非 CloudWatch 告警触发）不自动生成 Recommendations，结论在 journal_records 里
- **DevOps Agent 轮询超时**: Investigation 通常 2-3 分钟完成，若 5 分钟后仍 IN_PROGRESS，检查账号权限（需要 CloudWatch Logs + RDS DescribeDB 权限）

### When to Ask the User
- orders.status 的完整取值范围（将 `!= 'cancelled'` 改写为 `IN(...)` 时需要确认所有有效状态值）
- 建索引是否需要在维护窗口执行（生产环境大表）
- LIMIT 数量是否可以减小（业务上是否真的需要 10000 条）
- 是否需要同时升级实例规格（需要评估成本影响）
