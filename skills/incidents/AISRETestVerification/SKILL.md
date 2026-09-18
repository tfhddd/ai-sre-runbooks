# AISRETestVerification 告警分析

## 何时使用

当收到 `AISRETestVerification` 告警�集群 `kind-dev-cluster`)时使用本 skill。该告警是人为创建的测试告警,用于验证 AlertManager → Keep → Holmes → 邮件 → GitHub PR 链路是否正常工作,非真实故障。

## 根因模式

- **PrometheusRule**: `aisre-test-rule`(namespace: `monitoring`)
- **告警表达式**: `vector(1)` — Prometheus 常量,恒等于 1(永远为真)
- **`for: 0s`** — 无等待时间,立即触发
- **评估间隔**: 15s,每 15 秒重新触发一次
- **创建时间**: 2026-09-18T04:10:24Z
- **持续触发**: 自 2026-09-18T04:11:15Z 起持续 firing

告警注解明确说明用途:"用于验证 AlertManager → Robusta → Holmes → 邮件 → GitHub PR 链路。vector(1) 恒为真,删除此 PrometheusRule 即可停止。"

## 排查步骤(含 grafana/loki LogQL)

1. **确认告警规则**:检查 PrometheusRule `aisre-test-rule` 是否存在且表达式为 `vector(1)`:
   ```bash
   kubectl get prometheusrule aisre-test-rule -n monitoring -o yaml
   ```

2. **确认告警状态**:通过 Prometheus API 或 AlertManager 确认告警正在 firing。

3. **Loki 日志查询 — 确认链路流转**:查询最近 1 小时内包含告警名称的日志,确认告警已成功流转至下游各环节:
   ```logql
   {cluster="kind-dev-cluster"} |= "AISRETestVerification"
   ```

4. **验证各环节**:
   - **Prometheus**:规则健康(`health: ok`),每 15s 求值,`vector(1)` 恒为真
   - **AlertManager**:告警已推送(`"pushed": true`),记录 fingerprint
   - **Keep(告警平台)**:pod `keep-backend-*` 收到事件,运行 workflow,执行 extraction rules
   - **Holmes**:pod `holmes-*` 收到请求并开始分析

5. **Loki 日志查询 — Keep 环节**:
   ```logql
   {cluster="kind-dev-cluster", pod=~"keep-backend-.*"} |= "AISRETestVerification"
   ```

6. **Loki 日志查询 — Holmes 环节**:
   ```logql
   {cluster="kind-dev-cluster", pod=~"holmes-.*"} |= "AISRETestVerification"
   ```

## 修复

**方案一(推荐Y— 删除测试规则,停止告警:**
```bash
kubectl delete prometheusrule aisre-test-rule -n monitoring
```

**方案二 — 保留规则但禁用(便于将来复测):**
将 `expr` 从 `vector(1)` 改为 `vector(0)`,告警将不再触发:
```bash
kubectl patch prometheusrule aisre-test-rule -n monitoring --type='json' \
  -p='[{"op":"replace","path":"/spec/groups/0/rules/0/expr","value":"vector(0)"}]'
```
需要再次验证链路时改回 `vector(1)` 即可。

**方案三 — 添加抑制规则(inhibit rule):** 在 AlertManager 配置中对该告警做静默处理,仅在需要手动触发时临时解除。

## 预防

1. **测试告警生命周期管理**:测试告警应在验证完成后立即删除或禁用,避免长期 firing 产生噪音和资源消耗。
2. **命名规范**:测试告警规则应使用统一前缀(如 `aisre-test-`),便于识别和批量清理。
3. **添加注解**:测试告警规则应包含注解说明用途和清理方式,方便后续维护者快速处理。
4. **定期巡检**:定期检查 `monitoring` namespace 下的 PrometheusRule,清理遗留的测试规则。
5. **避免 `vector(1)` 长期运行**:`vector(1)` 恒为真,若长期运行会持续触发 Keep workflow 和 Holmes 分析,消耗计算资源。
