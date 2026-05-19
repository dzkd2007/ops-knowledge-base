# DynamoDB 限流导致 5xx 告警排查与修复

## 功能概述

本 Skill 面向以下场景：CloudWatch 5xx 告警持续抖动或长期 ALARM，根因为 DynamoDB WriteThrottleEvents 超限。Skill 覆盖从指标分析、DevOps Agent 自动调查、Kiro 执行修复、到告警恢复验证的完整闭环。

**适用服务**: DynamoDB (PROVISIONED 模式) + ALB + EC2/ECS

**平均处理时间**: 调查 ~8 分钟 + 修复 ~5 分钟

---

## 典型案例

### 案例 1：WCU=1 + max_retries=0 双重根因（2026-05-19，东京区域）

**问题**: `st3-order-app-5xx-errors` 告警今日多次触发，峰值 3,055 次/分钟 5xx 错误，告警在 ALARM/OK 间持续抖动。

**根因**:
1. DynamoDB `st3-order-app-orders` 表使用 PROVISIONED 模式，WCU=1，实际需求 ~14 WCU/s，导致大量 PutItem 请求被限流
2. SSM 部署（09:43 CST）将应用 DynamoDB SDK `max_retries` 设为 0，连接池=10，任何限流直接转化为 503

**修复**:
- 提升 WCU：`aws dynamodb update-table --provisioned-throughput WriteCapacityUnits=50`
- 后续计划：修复 `max_retries=10`，启用 Auto Scaling，评估切换 On-Demand

**效果**: WCU 提升后 5xx 从 ~3000/分钟降至 5-30/分钟，告警数据点首次降至阈值以下

---

## 关键诊断信号

| 信号 | 可能根因 |
|------|----------|
| 5xx 尖峰后急剧恢复（数分钟内） | 限流 + 无重试，流量波动触发 |
| 5xx 持续高位（~3000/分钟，无恢复） | 限流 + 无重试，持续高负载 |
| ALARM → OK → ALARM 2-3分钟周期 | 限流 + 告警阈值过低（EvaluationPeriods=1） |
| 告警触发时间与部署时间差 <5 分钟 | 部署引入问题配置 |

---

## 预防措施

1. **DynamoDB 推荐使用 On-Demand 模式**，避免容量规划失误
2. **SDK 强制要求 max_retries ≥ 3**，生产环境建议 adaptive 模式
3. **CloudWatch 告警配置**：EvaluationPeriods ≥ 3，避免单点抖动触发
4. **部署前检查清单**：自动验证 DynamoDB retry/pool 参数
5. **添加 WriteThrottleEvents 独立告警**，比 5xx 提前 1-2 分钟预警
