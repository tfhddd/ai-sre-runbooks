---
name: <service-or-component>-<issue>
description: <一句话：服务 + 故障现象 + 触发条件。极度具体，holmesgpt 靠它做语义匹配。例：Diagnose Redis memory spikes every 6 hours caused by missing TTL on cache keys from cache-refresh service>
last_updated: YYYY-MM-DD
---

<!--
本文件是「经验归档模板」。复制本文件到 skills/<layer>/<service-or-component>-<issue>/SKILL.md 后填空。
填写前必读仓库根目录 CLAUDE.md。删除所有 <!-- --> 注释后再提 PR。
frontmatter 只允许 name / description / last_updated 三个字段，富元数据放正文 ## 元数据。
-->

## 元数据

| 字段 | 值 |
|---|---|
| service | <服务/组件名，如 order-service / redis> |
| layer | <infra / middleware / app / access> |
| component | <更细组件，如 redis-cache-master / 无则填 -> |
| issue_type | <oom / latency / crashloop / disk-full / connection-exhaustion / ...> |
| severity | <P0 / P1 / P2 / P3> |
| keywords | <逗号分隔，便于检索，如 redis,memory,ttl,cache-refresh> |
| source_ticket | <来源 holmesgpt 会话 ID / 工单号 / 事件链接> |

## 目标

<这个 skill 要解决什么问题、覆盖什么场景。2-4 句。>

## 症状或已知故障模式

<告警/日志/指标的表象。写清楚"什么时候、看到什么、反复出现的规律"。
例：Redis used_memory 每 6 小时爬升一次，随后骤降；与 cache-refresh 重建时间吻合。>

- 现象 1：
- 现象 2：

## 排查步骤

<可执行、可复现。每步给出命令/查询/检查点与"符合什么说明是这个问题"。>

1. **<步骤标题>**
   ```bash
   <命令或查询>
   ```
   - 期望输出 / 判断标准：
2. **<步骤标题>**
   - 操作：
   - 判断标准：
3. **<步骤标题>**
   - 操作：
   - 判断标准：

## 根因

<一句话根因 + 必要的链路解释。不止描述症状，要写到底层原因。>
- 直接原因：
- 触发条件：
- 影响范围：

## 修复方案

<分短期止血与长期根治。给出操作步骤，命令可执行；涉及凭据写"联系 XXX 团队获取"。>

### 短期止血
1.
2.

### 长期根治
1.
2.

## 验证

<修复后如何确认恢复。给可执行命令与期望结果。>
```bash
<验证命令>
```
- 通过标准：

## 负责人与升级路径

- **服务 owner**：<团队>（Slack: #<channel>）
- **on-call**：<排班/PagerDuty 说明>
- **升级路径**：<持续多久未恢复 / 哪类问题升级给谁>

## 参考

- holmesgpt 会话/工单：<链接或 ID>
- 相关 PR：<本仓 PR 链接>
- 外部文档：<官方文档/ Runbook 链接>
