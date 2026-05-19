---
name: dynamodb-throttling-5xx-troubleshoot
display_name: DynamoDB 限流导致 5xx 告警排查与修复
description: |
  排查应用层 5xx 告警持续抖动、根因为 DynamoDB WriteThrottleEvents 的完整工作流。
  触发信号：「5xx 告警持续抖动」「DynamoDB 限流」「st3-order-app 503」「order app 5xx」
icon: "🔴"
trigger: DynamoDB 限流 5xx 排查
inputs:
  - name: alarm_name
    description: "CloudWatch 告警名称（如 st3-order-app-5xx-errors）"
    type: string
    required: true
  - name: region
    description: "AWS 区域（如 ap-northeast-1）"
    type: string
    required: true
  - name: dynamodb_table
    description: "DynamoDB 表名（如 st3-order-app-orders）"
    type: string
    required: true
  - name: ec2_instance_id
    description: "应用所在 EC2 实例 ID"
    type: string
    required: false
tools: [user_mcp__aws_devops_agent, user_mcp__aws_mcp_d633e0f3, acp_agents]
---

## Overview

当 CloudWatch 5xx 告警持续抖动（ALARM/OK 周期性切换，或长期处于 ALARM 状态）时，按本 Skill 的工作流依次排查 DynamoDB 限流、应用 SDK 配置、ALB 目标组健康状态三层原因，并给出可执行的修复方案。

典型模式：告警在 2-3 分钟内 ALARM→OK→ALARM→OK 反复切换，或 5xx 量级在 ~3000/分钟高位维持超过 10 分钟。

## Workflow

### Step 1: 查询告警当前状态与指标时间线
- **Mode**: `deterministic`
- **Tool**: `user_mcp__aws_mcp_d633e0f3`（aws___call_aws）
- **Input**: `{{alarm_name}}`、`{{region}}`
- **Output**: 告警 StateValue、StateReason、最近数据点
- **命令**:
  ```bash
  # 查告警状态
  aws cloudwatch describe-alarms \
    --alarm-names {{alarm_name}} \
    --region {{region}}
  
  # 查最近 2 小时 5xx 指标（1分钟粒度）
  aws cloudwatch get-metric-statistics \
    --namespace <Namespace> \
    --metric-name <MetricName> \
    --start-time <2h ago ISO> --end-time <now ISO> \
    --period 60 --statistics Sum \
    --region {{region}}
  ```
- **Validate**: 能读出 StateValue（ALARM/OK）和至少一个数据点
- **解读**：
  - 数据点 > 阈值：告警应触发
  - 数据点存在极高尖峰（>1000）后骤降：典型限流 + 零重试模式
  - 数据点在 ALARM 状态但未更新：CloudWatch 评估周期未收到新数据

### Step 2: 启动 DevOps Agent 调查
- **Mode**: `deterministic`
- **Tool**: `user_mcp__aws_devops_agent`（create_investigation）
- **Input**: 告警名、区域、指标时间线摘要
- **Output**: task_id、execution_id
- **调用方式**:
  ```
  create_investigation(
    title="{{alarm_name}} 告警持续抖动（{{region}}）",
    priority="HIGH",
    description="事件摘要 + 抖动时间线 + 需重点排查：1）5xx path分布 2）目标组健康 3）DynamoDB性能 4）部署变更 5）告警配置灵敏度"
  )
  ```
- **轮询**: 每 30 秒调用 get_task 检查状态，完成后调用 list_journal_records 读取调查日志
- **Validate**: status = COMPLETED，journal records 包含根因分析段落

### Step 3: 执行紧急修复
- **Mode**: `agentic`
- **Tool**: `acp_agents`（send_message_to_acp_agent → Kiro）
- **Input**: 调查确认的根因、受影响资源
- **Output**: 修复操作执行结果

**根因1：DynamoDB WCU 不足**
```bash
# 临时提升 WCU（保守方案）
aws dynamodb update-table \
  --table-name {{dynamodb_table}} \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=50 \
  --region {{region}}

# 或切换为按需模式（推荐，注意 24h 内不可切回）
aws dynamodb update-table \
  --table-name {{dynamodb_table}} \
  --billing-mode PAY_PER_REQUEST \
  --region {{region}}
```

**根因2：应用 max_retries=0**
通过 SSM 连接实例，修改 DynamoDB SDK 配置：
```python
from botocore.config import Config
config = Config(
    retries={'max_attempts': 10, 'mode': 'adaptive'},
    max_pool_connections=50
)
```
重新部署应用后验证告警恢复。

- **Validate**: DescribeTable 确认配置生效；2 分钟后 CloudWatch 数据点 ≤ 阈值

### Step 4: 监控恢复状态
- **Mode**: `deterministic`
- **Tool**: `user_mcp__aws_mcp_d633e0f3`
- **Input**: `{{alarm_name}}`、`{{region}}`
- **Output**: StateValue = OK
- **轮询间隔**: 60 秒，最多 5 次
- **恢复判断**: StateValue 从 ALARM 变为 OK
- **超时处理**: 5 分钟未恢复 → 执行 Step 3 中下一条修复项

### Step 5: 生成运维报告
- **Mode**: `deterministic`
- **Skill**: `ops-incident-report`
- **Input**: 告警名、根因、时间线、修复记录
- **Output**: Word 格式事故报告，通过 Outlook 发送

## Output

- CloudWatch 告警恢复为 OK
- DynamoDB 表 WCU 已提升或切换为按需模式
- 应用 max_retries 已修复
- 运维事故报告（OPS-YYYYMMDDNN）已生成并发送

## Lessons Learned

### Do
- 先用 DevOps Agent 做全面调查（约 8 分钟），避免漏掉多根因场景
- WCU 提升后等待 2-3 分钟让 CloudWatch 收到新数据点再判断是否恢复
- 5xx 大幅下降但仍超阈值时，说明还有残留根因（通常是 max_retries=0）
- 可视化指标时间线（Highcharts 直方图）有助于快速识别爆发时间点和根因关联

### Don't
- 不要仅提升 WCU 就认为问题解决——如果 max_retries=0 未修复，偶发限流仍会产生 503
- 不要将 DynamoDB 从 On-Demand 手动切回低 WCU 的 PROVISIONED，这是本次事故的直接触发操作
- 告警 StateValue 停滞（长时间未更新）不代表问题已解决，可能是该时段无新数据点

### Common Failures
- **DevOps Agent 调查超时**: 调查通常需要 5-10 分钟，需要后台任务轮询，不要在主线程等待
- **Kiro SSM 修复需要确认**: Kiro 在执行破坏性操作前会暂停等待用户确认，及时回复
- **告警 24h 内无法切换计费模式**: DynamoDB PAY_PER_REQUEST 切换后 24h 内不能切回 PROVISIONED

### When to Ask the User
- DynamoDB 是否允许切换为 On-Demand 模式（涉及计费变化）
- 是否有计划中的大促/流量高峰需要提前扩容而非按需
- 应用重新部署时是否需要审批流程
