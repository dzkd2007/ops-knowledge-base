---
name: 20260517_demo-app-5xx_SA-fengepei
display_name: EC2 Flask App 5xx Error Troubleshooting
description: |
  基于真实事件调查总结的排障技能。系统化诊断 EC2 上 Flask 应用通过 ALB 暴露时产生 5xx 错误的根因分析方法，
  覆盖从 CloudWatch 报警发现、ALB 指标分析、应用日志排查、CloudTrail 验证到修复执行的完整流程。
author: fengepei
date: 2026-05-17
incident_id: f55ab4ad-6426-481a-8b4e-b3f8b8c36eea
account: "652875617673"
region: ap-northeast-1
tags:
  - ec2
  - alb
  - 5xx
  - flask
  - gunicorn
  - aurora
  - cloudwatch
  - cloudtrail
  - troubleshooting
  - devops-agent
severity: HIGH
---

# EC2 Flask App 5xx Error Troubleshooting Skill

## 事件概要

| 字段 | 值 |
|------|----|
| 事件时间 | 2026-05-17 07:41-07:44 UTC |
| 区域 | ap-northeast-1 (东京) |
| 影响 | 25 个 5xx 错误，持续约 2 分钟 |
| 应用 | demo-app (Flask order-api v2.1.0) |
| 架构 | EC2 (m5.large) → Nginx → Flask → Aurora PostgreSQL |
| 报警 | demo-app-5xx-errors (阈值: 5 个 5xx/分钟) |

## 适用场景

- CloudWatch 报警 `HTTPCode_Target_5XX_Count` 触发
- ALB 目标响应时间突然飙升
- HealthyHostCount 降为 0
- 应用返回 500/502/503 错误
- EC2 实例重启后应用不可用

---

## 调查方法论（5 步法）

### Step 1: 确认报警范围与时间窗口

**目标**: 确定报警触发/恢复时间，了解影响范围。

```bash
# 查看报警当前状态
aws cloudwatch describe-alarms \
  --alarm-names "demo-app-5xx-errors" \
  --region ap-northeast-1

# 获取报警状态变更历史
aws cloudwatch describe-alarm-history \
  --alarm-name "demo-app-5xx-errors" \
  --history-item-type StateUpdate \
  --start-date 2026-05-17T07:00:00Z \
  --end-date 2026-05-17T08:00:00Z \
  --region ap-northeast-1
```

**关键问题**:
- 报警何时触发、何时恢复？
- 阈值是多少，实际值是多少？
- 是单实例还是多实例问题？

---

### Step 2: 分析 ALB 指标

**目标**: 区分 ELB 层错误还是目标（应用）层错误。

```bash
# 目标 5XX 计数（每分钟）
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/demo-app-alb/2116213145 \
  --start-time 2026-05-17T07:35:00Z \
  --end-time 2026-05-17T07:50:00Z \
  --period 60 --statistics Sum \
  --region ap-northeast-1

# 目标响应时间
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/demo-app-alb/2116213145 \
  --start-time 2026-05-17T07:35:00Z \
  --end-time 2026-05-17T07:50:00Z \
  --period 60 --statistics Average Maximum \
  --region ap-northeast-1

# 健康/不健康主机数
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HealthyHostCount \
  --dimensions Name=TargetGroup,Value=targetgroup/demo-app-tg/635c246e257c26be \
    Name=LoadBalancer,Value=app/demo-app-alb/2116213145 \
  --start-time 2026-05-17T07:35:00Z \
  --end-time 2026-05-17T07:50:00Z \
  --period 60 --statistics Average \
  --region ap-northeast-1
```

**分析模式**:
| 指标组合 | 含义 |
|---------|------|
| ELB 5XX 有值 | ALB 层问题（无健康目标、超时） |
| Target 5XX 有值 | 应用层问题（应用返回错误） |
| 响应时间飙升 + 5XX | 应用过载或依赖故障 |
| HealthyHost = 0 | 所有目标不健康 |

---

### Step 3: 检查应用日志 (CloudWatch Logs)

**目标**: 找到具体错误信息和根因线索。

```bash
# 列出相关日志组
aws logs describe-log-groups --region ap-northeast-1

# 搜索错误日志
aws logs filter-log-events \
  --log-group-name "/demo-app/application" \
  --start-time 1779003660000 \
  --end-time 1779003900000 \
  --filter-pattern "?ERROR ?CRITICAL ?Exception ?500 ?502 ?503" \
  --region ap-northeast-1

# 检查 Nginx 访问日志中的 5xx
aws logs filter-log-events \
  --log-group-name "/demo-app/nginx-access" \
  --start-time 1779003660000 \
  --end-time 1779003900000 \
  --filter-pattern '"50"' \
  --region ap-northeast-1
```

**常见根因日志特征**:
| 日志内容 | 根因 |
|---------|------|
| `Connection refused: ...rds...:5433` | 数据库端口配置错误 |
| `Connection pool exhausted` | 连接池耗尽/连接泄漏 |
| `WARNING: This is a development server` | Flask werkzeug 在生产环境运行 |
| `MemoryError` / `OOM` | 实例规格不足 |
| `Timeout` | 依赖方慢响应 |

---

### Step 4: 验证基础设施变更 (CloudTrail)

**目标**: 确认是否有人为或自动化的基础设施变更导致问题。

```bash
# 检查 EC2 状态变更
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=StopInstances \
  --start-time 2026-05-17T07:00:00Z \
  --end-time 2026-05-17T08:00:00Z \
  --region ap-northeast-1

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=StartInstances \
  --start-time 2026-05-17T07:00:00Z \
  --end-time 2026-05-17T08:00:00Z \
  --region ap-northeast-1

# 检查 ALB/目标组变更
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticloadbalancing.amazonaws.com \
  --start-time 2026-05-17T07:00:00Z \
  --end-time 2026-05-17T08:00:00Z \
  --region ap-northeast-1
```

**关注点**:
- StopInstances / StartInstances / TerminateInstances
- RegisterTargets / DeregisterTargets
- 操作者是谁（IAM user/role）
- 是手动操作（CLI/Console）还是自动化（CloudFormation, ASG）

---

### Step 5: 检查实例状态

**目标**: 确认实例运行状态和应用进程。

```bash
# 验证实例状态
aws ec2 describe-instances \
  --instance-ids i-0b58c40354af345e2 \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType,LaunchTime]' \
  --region ap-northeast-1

# 检查目标组中的健康状态
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-northeast-1:652875617673:targetgroup/demo-app-tg/635c246e257c26be \
  --region ap-northeast-1

# 通过 SSM 检查实际运行进程
aws ssm send-command \
  --instance-ids i-0b58c40354af345e2 \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["ps aux | grep -E gunicorn|flask|werkzeug"]' \
  --region ap-northeast-1
```

---

## 本次事件根因

### 因果链

```
workshop 用户 Stop/Start 同一实例 (07:32)
→ 启动失败一次 (07:33, IncorrectInstanceState)
→ 第二次成功启动 (07:37)
→ 健康检查通过 (07:37:44, 低流量下)
→ 真实流量到达 → Flask werkzeug 单线程阻塞
→ 同时 DB 端口 5433 连接被拒（正确端口 5432）
→ 25 个 5xx (07:41-07:43)
→ 约 2 分钟后自动恢复
```

### 三重根因

| 因素 | 详情 |
|------|------|
| **触发因素** | workshop 用户 CLI 执行 Stop/Start (非实例替换) |
| **直接原因** | Flask werkzeug 开发服务器冷启动 + DB 端口 5433 配置错误 |
| **放大因素** | 单实例架构，ALB 仅 1 个目标 |

---

## 常见根因与修复方案

### 1. Flask 开发服务器在生产环境

**症状**: 单线程阻塞，日志中 `WARNING: This is a development server`

**修复**:
```bash
# 替换为 Gunicorn
pip install gunicorn
gunicorn --workers 4 --bind 0.0.0.0:5000 --timeout 30 app:app

# 创建 systemd service
sudo tee /etc/systemd/system/demo-app.service << 'EOF'
[Unit]
Description=Demo App (Gunicorn)
After=network.target
[Service]
User=ec2-user
WorkingDirectory=/opt/demo-app
ExecStart=/usr/local/bin/gunicorn --workers 4 --bind 0.0.0.0:5000 --timeout 30 app:app
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now demo-app
```

### 2. 数据库端口配置错误

**症状**: `Connection refused` on port 5433 (正确为 5432)

**修复**:
```bash
grep -r "5433" /opt/demo-app/
sed -i 's/5433/5432/g' /opt/demo-app/config.json
sudo systemctl restart demo-app
curl -s http://localhost:5000/api/orders  # 验证返回 200
```

### 3. 实例 Stop/Start 后冷启动过载

**症状**: 重启后 3-5 分钟出现 5xx 爆发，之后自动恢复

**修复**:
```bash
# ALB Slow Start（60秒预热）
aws elbv2 modify-target-group-attributes \
  --target-group-arn <tg-arn> \
  --attributes Key=slow_start.duration_seconds,Value=60

# Connection Draining（30秒排空）
aws elbv2 modify-target-group-attributes \
  --target-group-arn <tg-arn> \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

### 4. 单点故障（仅 1 个实例）

**症状**: 任何实例问题 = 100% 不可用

**修复**:
```bash
# 创建 ASG (min=2, max=4)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name demo-app-asg \
  --launch-template LaunchTemplateName=demo-app-lt,Version='$Latest' \
  --min-size 2 --max-size 4 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-az-a,subnet-az-c" \
  --target-group-arns <tg-arn> \
  --health-check-type ELB \
  --health-check-grace-period 120
```

### 5. 故障注入残留

**症状**: 错误与触发文件（如 `/tmp/demo-app-fault`）存在相关

**修复**:
```bash
rm -f /tmp/demo-app-fault
# 建议：从代码中移除或用环境变量保护
# if os.getenv('ENABLE_FAULT_INJECTION') == 'true'
```

---

## 决策树

```
5xx 报警触发
├── 检查 ALB 指标 → ELB 5xx 还是 Target 5xx?
│   ├── ELB 5xx → 无健康目标 → 检查实例状态
│   └── Target 5xx → 应用错误
│       ├── 查应用日志 → DB 连接错误?
│       │   ├── 是 → 验证 DB 端点/端口/凭证
│       │   └── 否 → 检查是否开发服务器 / 资源耗尽
│       ├── 查 CloudTrail → 最近有基础设施变更?
│       │   ├── 是 → 关联变更时间与错误开始时间
│       │   └── 否 → 检查故障注入或代码 bug
│       └── 查 CPU/内存 → 资源耗尽?
│           ├── CPU 高 → 扩容
│           └── CPU 正常 → 应用层瓶颈（线程/连接池）
```

---

## 经验教训

1. **低 CPU ≠ 无问题** — 单线程应用可以在 <5% CPU 时返回 5xx
2. **健康检查通过 ≠ 可服务流量** — 低频健康检查通过但生产负载失败
3. **User Data 配置 ≠ 实际运行配置** — 始终验证实际进程匹配预期
4. **CloudTrail 是真相来源** — 日志时间戳可能近似，CloudTrail 给出精确 API 调用时间
5. **检查故障注入** — Demo/测试环境常有内置 chaos 机制
6. **单实例是最大风险** — 任何架构都应有 min=2 的冗余

---

## 使用的工具

| 工具 | 用途 |
|------|------|
| AWS DevOps Agent | 自动化调查、根因分析、生成修复建议 |
| AWS MCP (CloudWatch) | 查询 ALB 指标、EC2 CPU |
| AWS MCP (Logs) | 搜索应用日志和 Nginx 日志 |
| AWS MCP (CloudTrail) | 验证 EC2/ALB 变更事件 |
| AWS SSM | 远程执行修复命令 |
| Kiro Agent (ACP) | 自动化修复执行（DB 端口修改） |
