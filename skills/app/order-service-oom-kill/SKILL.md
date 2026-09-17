---
name: order-service-oom-kill
description: Diagnose order-service Pod OOMKilled on the big-promotion traffic spike because the order line-item list is unbounded and held fully in memory before batch insert
last_updated: 2026-09-17
---

## 元数据

| 字段 | 值 |
|---|---|
| service | order-service |
| layer | app |
| component | order-aggregator |
| issue_type | oom |
| severity | P0 |
| keywords | oom,oomkilled,order-service,memory,leak,big-promotion,java |
| source_ticket | holmes-2026-09-17-003 |

## 目标

定位 order-service Pod 被 OOMKilled 的根因，区分「单请求大对象」「无界集合持有」「内存泄漏」三类。

## 症状或已知故障模式

- `kubectl describe pod` 末尾 `Last State: Terminated Reason: OOMKilled Exit Code=137`。
- 发生在大促流量高峰；内存单调上升至 limit 后被 kill，重启后回落、再上升。
- 慢 GC、`Full GC` 频繁，接口 P99 抖动。

## 排查步骤

1. **确认 OOM 与内存趋势**
   ```bash
   kubectl -n <ns> describe pod <pod> | Select-String "OOM|Reason|Last State|Exit Code"
   kubectl -n <ns> top pod -l app=order-service --containers
   ```
   - 多副本同时上涨 → 与流量/数据相关；单副本上涨 → 可能泄漏。
2. **抓堆内存对象（限窗口、勿抓生产敏感数据）**
   - 联系应用 owner 团队触发 JVM dump（堆 dump 文件含业务数据，禁止入仓，仅本地分析）。
   - 关注大对象：`java.util.ArrayList` of `OrderLineItem`、`HashMap` 体积异常。
3. **看大对象接口的请求特征**
   - 在日志/Trace 中定位 OOM 前最近的慢请求：超大订单（行项目数 > N）走批量插入但先全量加载内存。

## 根因

- 直接原因：`/orders/batch` 接口把整张订单行项目无界加载进内存后再批量入库，大促大单触发峰值超 limit。
- 触发条件：大促 / 个别超大订单（行项目数远超常规）。
- 影响范围：order-service 全部副本，订单写入链路 P99 退化甚至不可用。

## 修复方案

### 短期止血
1. 临时调高 order-service 内存 limit（仅止血，治标）：
   ```bash
   kubectl -n <ns> set resources deployment/order-service --limits=memory=<新值>
   ```
2. 对超大订单限流 / 拆单，降低单请求内存峰值。

### 长期根治
1. 改为流式/分批处理：行项目分页加载、分批 insert，避免全量驻留。
2. 加单请求行项目数上限 + 降级（超限走异步队列）。
3. 加 JVM 堆与 RSS 告警，到 80% 提前预警而非直接 OOM。

## 验证

```bash
kubectl -n <ns> top pod -l app=order-service --containers
kubectl -n <ns> rollout status deployment/order-service
```
- 通过标准：大促峰值下内存稳定在 limit 80% 以下、无 OOMKilled、`rollout status` 正常。

## 负责人与升级路径

- **服务 owner**：订单交易团队（Slack: #trade-order）
- **on-call**：交易 on-call（PagerDuty: trade-oncall）
- **升级路径**：P0 事件 15 分钟未止血 → 升级交易主管；内存行为疑泄漏 → 转平台 SRE + JVM 分析。

## 参考

- holmesgpt 会话：holmes-2026-09-17-003
- 相关 PR：本仓 `skill/app-order-service-oom-kill` 的 PR
- 外部文档：https://kubernetes.io/docs/concepts/config/manage-resources-containers/
