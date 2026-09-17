---
name: redis-memory-spike
description: Diagnose Redis used_memory spikes every 6 hours caused by missing TTL on cache keys written by cache-refresh service during its scheduled redeploy
last_updated: 2026-09-17
---

## 元数据

| 字段 | 值 |
|---|---|
| service | redis |
| layer | middleware |
| component | redis-cache |
| issue_type | memory-spike |
| severity | P1 |
| keywords | redis,memory,ttl,cache-refresh,eviction,used_memory |
| source_ticket | holmes-2026-09-17-002 |

## 目标

定位 Redis `used_memory` 周期性飙升并触发 eviction 的根因，确认是否由 cache-refresh 服务写入无 TTL key 引起。

## 症状或已知故障模式

- Redis `used_memory` 每 6 小时持续爬升，随后骤降（eviction/maxmemory 触发）。
- `evicted_keys` 在飙升窗口同步增长；客户端出现短暂 `OOM command not allowed`。
- 时间点与 `cache-refresh` 服务按 PDB 滚动重建高度吻合。

## 排查步骤

1. **确认飙升时间点与重建相关性**
   ```bash
   redis-cli -h <host> -p <port> info memory | findstr "used_memory used_memory_peak evicted_keys"
   ```
   - 记录 used_memory 上升起点，与 cache-refresh rollout 时间对比。
2. **统计 key 数与无 TTL key 占比**
   ```bash
   redis-cli -h <host> -p <port> --scan --pattern "*" | Measure-Object | Select-Object -ExpandProperty Count
   redis-cli -h <host> -p <port> --bigkeys
   ```
   - 在飙升窗口前/后各采样一次，无 TTL key 占比应明显升高。
3. **抽样确认无 TTL key 来源**
   ```bash
   redis-cli -h <host> -p <port> --scan --pattern "cache-refresh:*" | Select-Object -First 5 | ForEach-Object { redis-cli -h <host> -p <port> ttl $_ }
   ```
   - `TTL` 返回 `-1`（无过期）即坐实。
4. **看 cache-refresh 服务日志**
   - 确认其在重启时是否 SET 但未 EXPIRE。

## 根因

- 直接原因：`cache-refresh` 服务（或其重启路径）写入 cache key 未设置 TTL。
- 触发条件：每 6 小时 PDB 触发滚动重建，重启时 key 被重写但无过期，旧 key 累积直到 maxmemory eviction。
- 影响范围：整个 Redis 实例，极端情况挤压其他业务 key。

## 修复方案

### 短期止血
1. 临时扩容 maxmemory 或加节点，避免 eviction 影响其他业务：
   ```bash
   redis-cli -h <host> -p <port> config set maxmemory <bytes>
   ```
2. 手动清理已知无 TTL 前缀 key（先小批量验证）。

### 长期根治
1. 修复 `cache-refresh`：所有 SET 必须带 EXPIRE（或用 `SET k v EX <ttl>`）。
2. 加守护：发布后用 `--bigkeys` 与无 TTL key 占比校验，回归阈值未达标则回滚。
3. Redis 侧加 `maxmemory-policy allkeys-lru` 兜底（按容量规划评估）。

## 验证

```bash
redis-cli -h <host> -p <port> info memory | findstr "used_memory evicted_keys"
redis-cli -h <host> -p <port> --scan --pattern "cache-refresh:*" | ForEach-Object { redis-cli -h <host> -p <port> ttl $_ }
```
- 通过标准：修复发布后一个 6 小时周期内 used_memory 无突增、`evicted_keys` 不再增长、抽样 key `TTL > 0`。

## 负责人与升级路径

- **服务 owner**：缓存中间件团队（Slack: #middleware-cache）
- **on-call**：middleware on-call（PagerDuty: middleware-cache-oncall）
- **升级路径**：止血后 30 分钟仍在 eviction → 升级中间件主管；根因属应用 → 转 cache-refresh owner 团队。

## 参考

- holmesgpt 会话：holmes-2026-09-17-002
- 外部文档：https://redis.io/docs/management/optimization/memory-management/
