# 事件记录：DynamoDB 限流导致订单平台 5xx 错误

## 基本信息

| 项目 | 值 |
|------|----|
| **日期** | 2026-05-21 ~ 2026-05-22 |
| **严重程度** | HIGH |
| **影响服务** | Order Platform (st3-order-app) |
| **影响区域** | ap-northeast-1 |
| **持续时间** | ~33 小时 |
| **状态** | ✅ 已恢复 |

## 问题描述

CloudWatch 告警 `st3-order-app-5xx-errors` 持续触发，应用层产生约 3,000 次/分钟的 5xx 错误，远超阈值（50 次/分钟）。

## 根因

| 层级 | 原因 |
|------|------|
| 直接原因 | DynamoDB 表 `st3-order-app-orders` WriteThrottleEvents ~1,500/min |
| 根本原因 | IAM 用户 `dev-engineer` 手动将 WCU 从 50 降至 1 |
| 操作时间 | 2026-05-21 03:44 UTC |
| 操作工具 | aws-cli/2.28.19 (macOS arm64) |
| 来源 | 内网 VPC 端点 |

## 时间线

| 时间 (UTC) | 事件 |
|------------|------|
| 05-21 03:44 | dev-engineer 将 WCU 降至 1 |
| 05-21 05:25 | 告警进入 ALARM 并持续 |
| 05-22 14:22 | DevOps Agent 调查启动 |
| 05-22 14:26 | 根因确认 |
| 05-22 14:35 | 执行修复（WCU 恢复至 50） |
| 05-22 14:38 | 告警恢复 OK |

## 修复方案

```bash
aws dynamodb update-table \
  --table-name st3-order-app-orders \
  --provisioned-throughput ReadCapacityUnits=25,WriteCapacityUnits=50 \
  --region ap-northeast-1
```

## 后续行动

- [ ] P1: 启用 DynamoDB Auto Scaling
- [ ] P1: 添加 WriteThrottleEvents 独立告警
- [ ] P2: 配置 IAM Deny 策略
- [ ] P3: 评估切换 PAY_PER_REQUEST 模式
- [ ] P3: 应用层韧性增强（重试 + 熔断）

## 关联

- DevOps Agent 调查 ID: bb1f6904-812b-47ef-9a0c-7c5b83ec8d99
- 运维报告: OPS-2026052201
- Skill: dynamodb-throttle-5xx-troubleshoot
