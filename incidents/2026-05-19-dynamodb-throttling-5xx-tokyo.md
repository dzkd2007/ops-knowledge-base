# 事故记录：DynamoDB 限流导致 st3-order-app 5xx 告警持续抖动

- **日期**: 2026-05-19
- **区域**: ap-northeast-1（东京）
- **严重程度**: HIGH
- **持续时长**: ~2 小时（11:36 – 13:40+ CST）
- **报告编号**: OPS-2026051901

## 影响

- 告警 `st3-order-app-5xx-errors` 持续 ALARM，峰值 3,055 次 5xx/分钟
- 估计产生 84,000+ 次 503 错误（主要集中在 12:55–13:23 CST）
- 订单写入服务受影响，读取服务不受影响

## 时间线摘要

| CST | 事件 |
|-----|------|
| 09:43 | SSM 部署引入 max_retries=0、连接池=10 |
| 11:36 | 首批异常 5xx 出现 |
| 12:08–12:10 | **第一波**：5xx 最高 1,060/分钟 |
| 12:47 | DynamoDB 被切回 PROVISIONED WCU=1 |
| 12:55–13:23 | **第二波**：5xx 持续 ~3,000/分钟 |
| 13:23 | WCU 提升至 50 |
| 13:24 | 5xx 开始急剧下降 |
| 13:39 | 5xx 首次降至 2（低于阈值） |

## 根因

| 层次 | 根因 |
|------|------|
| 直接 | DynamoDB WCU=1，实际需求 ~14 WCU/s |
| 间接 | 应用 max_retries=0，限流直接返回 503 |
| 根本 | 变更管控缺失：部署未验证 SDK 参数；运维手动切回低容量 PROVISIONED |

## 修复方案

| 优先级 | 操作 | 状态 |
|--------|------|------|
| P0 | 提升 DynamoDB WCU 至 50 | ✅ 已完成 |
| P0 | 修复应用 max_retries=0→10，连接池→50 | ⏳ 待执行 |
| P1 | 启用 DynamoDB Auto Scaling（WCU 20-200） | 计划中 |
| P1 | 优化 CloudWatch 告警灵敏度 | 计划中 |
| P2 | 添加 WriteThrottleEvents 独立告警 | 计划中 |

## 工具使用

- **AWS DevOps Agent**（调查）：Task `cc4f7271-724f-4818-b2d0-dad1e3e26a0f`，8 分钟完成根因分析
- **Kiro Agent**（修复）：执行 WCU 提升 AWS CLI 命令
- **Amazon Quick**：告警监控、指标可视化、运维报告生成

## 经验教训

1. `max_retries=0` 是高危反模式，生产环境必须强制 ≥ 3
2. CloudTrail + CloudWatch 时间关联是快速定位变更引入根因的关键
3. AI 辅助运维（DevOps Agent + Kiro）显著压缩 MTTD/MTTR
4. WCU 提升后需等待 2-3 个 CloudWatch 评估周期才能确认恢复
