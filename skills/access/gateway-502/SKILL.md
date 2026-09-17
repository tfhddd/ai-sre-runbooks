---
name: gateway-502
description: Diagnose intermittent 502 Bad Gateway from the API gateway caused by upstream order-service Pods being mid-rollout and not yet ready, returning connection refused to the gateway
last_updated: 2026-09-17
---

## 元数据

| 字段 | 值 |
|---|---|
| service | gateway |
| layer | access |
| component | api-gateway / upstream order-service |
| issue_type | 502 |
| severity | P1 |
| keywords | gateway,502,bad-gateway,rollout,readiness,connection-refused,nginx |
| source_ticket | holmes-2026-09-17-004 |

## 目标

定位网关间歇性 502 的根因，区分「上游 Pod 未就绪」「上游 Pod 崩溃」「网关到上游网络/配置错误」三类。

## 症状或已知故障模式

- 网关访问日志出现间歇 `502 Bad Gateway`，集中在某 upstream（如 order-service）。
- 时间窗与 order-service 滚动发布重合；发布结束后 502 消失。
- 上游日志伴随短暂 `connection refused`。

## 排查步骤

1. **网关侧定位 502 的 upstream**
   ```bash
   # 网关日志示例（按你的实际日志栈调整）
   grep " 502 " /var/log/gateway/access.log | tail -50
   ```
   - 记录 502 对应的 upstream host/port 与时间。
2. **核 upstream 就绪状态与 rollout**
   ```bash
   kubectl -n <ns> get pod -l app=order-service -o wide
   kubectl -n <ns> rollout status deployment/order-service
   ```
   - 502 窗口内存在 `0/1`、`Terminating`、`Running` 但未 Ready 的 Pod → 滚动期流量打到未就绪 Pod。
3. **核对网关上游配置**
   - 确认 upstream 指向的是 Service（带 readiness）而非直接 Pod IP。
   - 确认 readinessProbe 与网关健康检查一致。

## 根因

- 直接原因：滚动发布期间，新 Pod 尚未通过 readinessProbe 即被网关路由，或旧 Pod 已 Terminating 仍被选中，连接被拒。
- 触发条件：`maxSurge`/`maxUnavailable` 配置激进 + readiness 起慢 + 网关未尊重 endpoints 就绪。
- 影响范围：发布窗口内的该 upstream 流量，间歇 502。

## 修复方案

### 短期止血
1. 暂停激进的 rollout，缩小 `maxSurge` / 放大 `maxUnavailable` 至 0（保证可用实例不降）：
   ```bash
   kubectl -n <ns> patch deployment order-service --type=json -p '[{"op":"replace","path":"/spec/strategy/rollingUpdate/maxSurge","value":"25%"},{"op":"replace","path":"/spec/strategy/rollingUpdate/maxUnavailable","value":0}]'
   ```
2. 必要时回滚到上一 healthy revision。

### 长期根治
1. 网关上游只认 Service + readiness endpoints，不直连 Pod IP。
2. 上游配置 `connection drained` / `preStop` + 优雅下线，给 SIGTERM 后留足排空时间。
3. 上游加 readiness 起到"真正能接流量"才过；网关侧加 502 重试 + 熔断。

## 验证

```bash
kubectl -n <ns> rollout status deployment/order-service
# 发布期间观察网关 502 计数
```
- 通过标准：一次完整 rollout 期间 0 条 502、新副本全部 Ready 后流量平滑迁移。

## 负责人与升级路径

- **服务 owner**：接入网关团队（Slack: #access-gateway）
- **on-call**：access on-call（PagerDuty: access-oncall）
- **升级路径**：502 持续超 5 分钟 → 升级网关主管；根因属上游未就绪 → 转对应 service owner。

## 参考

- holmesgpt 会话：holmes-2026-09-17-004
- 外部文档：https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
