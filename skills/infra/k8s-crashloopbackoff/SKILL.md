---
name: k8s-crashloopbackoff
description: Diagnose Kubernetes Pod CrashLoopBackOff caused by application panic on startup, failed readiness/liveness probes, or bad config/env vars injected by the latest Deployment rollout
last_updated: 2026-09-17
---

## 元数据

| 字段 | 值 |
|---|---|
| service | kubernetes |
| layer | infra |
| component | kubelet / pod |
| issue_type | crashloop |
| severity | P1 |
| keywords | kubernetes,crashloopbackoff,pod,probe,deployment |
| source_ticket | holmes-2026-09-17-001 |

## 目标

定位 Pod 进入 CrashLoopBackOff 的根因，区分「应用启动崩溃」「探针配置错误」「配置/环境变量缺失」三类，给出对应修复。

## 症状或已知故障模式

- `kubectl get pod` 显示 `CrashLoopBackOff`，RESTARTS 持续增长。
- 多见于最近一次 Deployment rollout 之后。
- `kubectl describe pod` 的 Events 末尾出现 `Back-off restarting failed container`。

## 排查步骤

1. **看当前容器日志与上次崩溃日志**
   ```bash
   kubectl -n <ns> logs <pod> --tail=200
   kubectl -n <ns> logs <pod> --previous --tail=200
   ```
   - `--previous` 报 panic / `panic:` / `nil pointer` → 应用启动崩溃。
   - `connection refused` / `i/o timeout` 退出 → 依赖未就绪。
2. **看 Events 与探针**
   ```bash
   kubectl -n <ns> describe pod <pod> | Select-String -Pattern "Liveness|Readiness|Warning|Back-off|Unhealthy"
   ```
   - `Liveness probe failed: HTTP probe ... status=500` → 探针路径/端口错或应用未起。
   - `Readiness probe failed` 但容器在跑 → readiness 路径错。
3. **比对最近 rollout**
   ```bash
   kubectl -n <ns> rollout history deployment/<dep>
   kubectl -n <ns> rollout status deployment/<dep>
   ```
   - 问题出现在 revision N → 对比 N 与 N-1 的镜像 / env / config。

## 根因

- 直接原因：容器启动后退出（非 0 退出码）或被探针反复杀死后由 kubelet 重启。
- 触发条件：最近发布引入 bad config / 缺失 env / 探针端口路径变更 / 镜像 tag 写错。
- 影响范围：对应 Deployment 的所有副本，流量无法接续。

## 修复方案

### 短期止血
1. 回滚到上一个 healthy revision：
   ```bash
   kubectl -n <ns> rollout undo deployment/<dep> --to-revision=$((N-1))
   ```
2. 确认副本恢复：
   ```bash
   kubectl -n <ns> rollout status deployment/<dep>
   ```

### 长期根治
1. 修复根因（补 env / 修探针路径 / 修配置 / 修镜像 tag）后重新发布。
2. 在 CI 增加启动冒烟：发布后 30s 内 `rollout status` + 探针通过才判定成功，否则自动回滚。

## 验证

```bash
kubectl -n <ns> get pod -l app=<dep> --show-labels
kubectl -n <ns> rollout status deployment/<dep>
```
- 通过标准：所有 Pod `Running`、RESTARTS 稳定不再增长、`rollout status` 显示 `successfully rolled out`。

## 负责人与升级路径

- **服务 owner**：平台 SRE 团队（Slack: #sre-platform）
- **on-call**：按平台 on-call 排班 PagerDuty
- **升级路径**：回滚后 10 分钟仍未恢复 → 升级给 SRE 主管；属应用代码崩溃 → 转 owner 团队。

## 参考

- holmesgpt 会话：holmes-2026-09-17-001
- 外部文档：https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/
