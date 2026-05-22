---
name: dynamodb-throttle-5xx-troubleshoot
display_name: DynamoDB 限流导致 5xx 错误排查
description: |
  排查因 DynamoDB 容量配置错误（WCU/RCU 被手动降低）导致写入限流和应用 5xx 错误的问题。
  触发信号：「DynamoDB 限流」「WriteThrottleEvents」「5xx 错误 + DynamoDB」「容量不足」「WCU 降低」
icon: "🔧"
trigger: DynamoDB 限流排查
inputs:
  - name: table_name
    description: "DynamoDB 表名"
    type: string
    required: true
  - name: region
    description: "AWS 区域（如 ap-northeast-1）"
    type: string
    required: true
  - name: alarm_name
    description: "CloudWatch 告警名称（可选）"
    type: string
    required: false
tools: [aws_mcp, aws_devops_agent, cloudwatch, cloudtrail]
---

## Overview

当 DynamoDB 表的预置容量（WCU/RCU）被手动或错误地降低时，会导致大量 WriteThrottleEvents/ReadThrottleEvents，进而使应用层产生 5xx 错误。本 Skill 提供从告警发现到根因定位再到修复验证的完整排查流程。

## Workflow

### Step 1: 确认告警和 Metric 状态
- **Mode**: `deterministic`
- **Tool**: CloudWatch GetMetricStatistics / DescribeAlarms
- **Input**: alarm_name, table_name, region
- **Output**: 当前告警状态、5xx 错误量趋势、是否持续
- **Validate**: 确认告警处于 ALARM 状态，获取错误量级

### Step 2: 检查 DynamoDB 限流指标
- **Mode**: `deterministic`
- **Tool**: CloudWatch GetMetricStatistics
- **Input**: Namespace=AWS/DynamoDB, MetricName=WriteThrottleEvents/ReadThrottleEvents, TableName
- **Output**: 限流事件数量和趋势
- **Validate**: WriteThrottleEvents > 0 确认存在限流

### Step 3: 检查当前表容量配置
- **Mode**: `deterministic`
- **Tool**: DynamoDB DescribeTable
- **Input**: table_name, region
- **Output**: 当前 WCU/RCU 配置、BillingMode、表状态
- **Validate**: 对比 WCU 与实际写入需求是否匹配

### Step 4: 追溯变更操作（CloudTrail）
- **Mode**: `agentic`
- **Tool**: CloudTrail LookupEvents / DevOps Agent Investigation
- **Input**: EventName=UpdateTable, ResourceName=table_name
- **Output**: 操作人、操作时间、变更内容、来源 IP
- **Validate**: 确认是人为操作还是 Auto Scaling 行为

### Step 5: 排除其他故障可能
- **Mode**: `agentic`
- **Tool**: ALB/ELB metrics, ECS/EC2 health check
- **Input**: 关联的负载均衡器和计算资源
- **Output**: 排除网络、实例健康等问题
- **Validate**: 确认问题仅由 DynamoDB 限流引起

### Step 6: 执行修复
- **Mode**: `deterministic`
- **Tool**: DynamoDB UpdateTable
- **Input**: table_name, 目标 WCU/RCU（或切换 PAY_PER_REQUEST）
- **Output**: 表状态变更确认
- **Validate**: 表状态回到 ACTIVE，WCU 恢复正常值
- **Command**:
  ```bash
  aws dynamodb update-table \
    --table-name {{table_name}} \
    --provisioned-throughput ReadCapacityUnits=<RCU>,WriteCapacityUnits=<WCU> \
    --region {{region}}
  ```

### Step 7: 验证恢复
- **Mode**: `deterministic`
- **Tool**: CloudWatch DescribeAlarms + GetMetricStatistics
- **Input**: alarm_name, 等待 3-5 分钟
- **Output**: 告警恢复 OK、5xx 降至正常、ThrottleEvents 归零
- **Validate**: StateValue=OK, Sum(5xx) < threshold, Sum(ThrottleEvents)=0

## Output

- 根因分析报告（因果链 + 操作人 + 时间线）
- 修复命令和验证结果
- 防护建议列表

## Lessons Learned

### Do
- 优先检查 DynamoDB WriteThrottleEvents/ReadThrottleEvents 指标，这是最直接的限流证据
- 使用 CloudTrail 追溯 UpdateTable 事件，确认操作者身份
- 修复后等待至少一个评估周期（通常 1-5 分钟）再验证告警状态
- 建议同时配置 Auto Scaling 和 IAM Deny 策略防止再次发生

### Don't
- 不要只看应用层 5xx 错误就下结论——先排除 ALB、实例、网络等问题
- 不要假设限流一定是手动操作——可能是 Auto Scaling 配置错误或限额问题
- 修复后不要立即判断告警未恢复为失败——CloudWatch 有评估延迟

### Common Failures
- **DescribeTable 返回 WCU=1 但 BillingMode=PAY_PER_REQUEST**: 按需模式下 WCU 显示值无意义
- **CloudTrail 查不到 UpdateTable**: 可能超过 90 天保留期，需查 S3 归档
- **修复后告警仍 ALARM**: 等待时间不足，CloudWatch 需要新的低错误率数据点

### When to Ask the User
- 目标 WCU/RCU 值（恢复到多少合适）
- 是否切换为按需模式（PAY_PER_REQUEST）
- 是否需要联系操作者确认变更意图
