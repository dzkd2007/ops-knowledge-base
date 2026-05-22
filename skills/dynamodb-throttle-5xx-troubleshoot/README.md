# DynamoDB 限流导致 5xx 错误排查

## 功能概述

排查因 DynamoDB 预置容量（WCU/RCU）被错误降低导致的写入限流和应用 5xx 错误。涵盖从告警发现、根因定位（CloudTrail 追溯）到修复验证的完整流程。

## 典型案例

### 案例 1：dev-engineer 手动降低 WCU 导致订单平台 33 小时持续故障

**问题描述：**
- 告警 `st3-order-app-5xx-errors` 持续触发，错误量 ~3,000 次/分钟
- 阈值 50 次/分钟，超标约 60 倍
- 持续时间超过 33 小时

**根因：**
- IAM 用户 `dev-engineer` 通过 AWS CLI 手动将 DynamoDB 表 `st3-order-app-orders` 的 WriteCapacityUnits 从 50 降至 1
- 导致 WriteThrottleEvents ~1,500 次/分钟
- 供需失衡比例约 26 倍

**修复方案：**
```bash
aws dynamodb update-table \
  --table-name st3-order-app-orders \
  --provisioned-throughput ReadCapacityUnits=25,WriteCapacityUnits=50 \
  --region ap-northeast-1
```

**修复效果：**
- 执行修复后 2-3 分钟内 5xx 从 ~3,000/min 降至个位数
- WriteThrottleEvents 归零
- 告警自动恢复 OK

**防护措施：**
1. 启用 DynamoDB Auto Scaling（WCU min=10, max=200）
2. 配置 IAM Deny 策略限制手动 UpdateTable
3. 添加 WriteThrottleEvents 独立告警
4. 通过 EventBridge 监控容量变更事件
5. 应用层增加指数退避重试和熔断器
