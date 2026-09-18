# AISREKeepTest6 告警分析

## 何时使用

当收到 `AISREKeepTest6` 告警（集群 `kind-dev-cluster`）时使用本 skill。该告警是人工通过 Keep API 推送的测试告警，用于验证 AI-SRE v4 pipeline（告警→Holmes 分析→邮件→skill 归档→PR）。如果告警名称以 `AISREKeepTest` 开头且 `fingerprint` 为手动指定值、`pushed: true`、`value`/`instance`/`job` 字段为空，则适用本 skill。

## 根因模式

### 模式 1：人工推送测试告警（直接根因）

人工通过 Keep API 推送测试告警，触发了 `alert-to-holmes-to-mail` workflow。告警特征：
- Prometheus rules 中无对应规则定义；`ALERTS{alertname="AISREKeepTest6"}` 查询返回空
- 告警 `value`、`instance`、`job` 字段均为空（无真实指标数据）
- `fingerprint` 为手动指定（如 `keepaisre6`），非自动生成
- `pushed: true`（通过 API 推送，非 Alertmanager webhook 接收）
- `description` 为测试描述（如 "v4 workflow B 触发用"）
- `startsAt` 为未来时间戳，非真实告警时间
- 这是 AISREKeepTest 系列测试告警的第 6 个（Test1~Test5 已在 03:39~03:55 间依次触发）

### 模式 2：Keep workflowstore parser bug

`workflowstore.py` 中提取 providers 的逻辑对 dict 类型对象错误调用了字符串方法 `.split()`，报错 `'dict' object has no attribute 'split'`（日志中拼写错误为 "providerts" 应予"providers"）。䨤个 workflow 均受影响，在 03:39~03:58 间至尤报错 8 次。

### 模式 3：K8s Secret 未酭移或丹�:

workflow 的 K8s Secrew（如 `keep-cf5af687-...-secrets`（不存在或丹空，导致 `Could not load secrets for workflow` 警告，secrets 无法加载。

### 模式 4：级联执行失败

`confirmed-to-skill-pr` workflow 丯 `format-as-skill` 步骨引用 `alert.enriched.holmes_analysis`，佇该孛容在 alert acknowledged 时分支已存在或 b=token 权限不足返回 422。

### 模式 5：硬编码凭据泄露（严藍）
- SMTP 密码明文硬编码在 `alert-to-holmes-to-mail` workflow YAML 中
- GitHub token 明文硬编码在 `confirmed-to-skill-pr` workflow YAML 中
- 两者均以明文出现在 Loki 日志中，任何有日志读取权限的人均可获取

## 排查步骤

### 步骤 1：确认告警是否为测试告警

查询 Prometheus 确认无对应规则：

```promql
ALERTS{alertname="AISREKeepTest6"}
```

如果返回空且告警字段（value/instance/job）为空、`fingerprint` 为手动指定、`pushed: true`，则为人工推送的测试告警，无真实基础设施故障。

### 步骤 2：查看 Keep backend 日志确认 workflow 执行

使用 Loki 查询 Keep backend 日志，确认告警接收和 workflow 触发：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "AISREKeepTest6"
```

查询 workflow 执行状态：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "alert-to-holmes-to-mail"
```

确认 workflow `cf5af687-941a-49f1-be9b-925b129862f2` 的触发条件 `source.contains("prometheus")` 是否匹配，以及 ask-holmes（POST `http://holmes-holmes.holmes.svc.cluster.local:80/api/chat`）和 send-mail（SMTP `smtp.exmail.qq.com:465` → `2272751277@qq.com`）两步执行情况。

### 步骤 3：排查 workflowstore parser bug

查询 Keep backend 日志中的 parser 错误：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "providerts" |= "split"
```

或：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "'dict' object has no attribute 'split'"
```

确认错误是否反复出现（03:39~03:58 间至少 8 次），两个 workflow 是否均受影响。

### 步骤 4：排查 secrets 加载失败

查询 secrets 相关警告：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "Could not load secrets for workflow"
```

检查 K8s Secret 是否存在：

```bash
kubectl get secret -n keep | grep keep-cf5af687
```

### 步骤 5：排查级联失败

查询 `confirmed-to-skill-pr` workflow 的错误：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "format-as-skill"
```

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "create-pr" |= "422"
```

确认 `format-as-skill` 是否因 `alert.enriched.holmes_analysis` 未就绪而失败，`create-pr` 是否因分支已存在或 token 权限不足返回 422。

### 步骤 6：检查硬编码凭据泄露

查询日志中是否出现明文凭据：

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "smtp" |= "password"
```

```logql
{namespace="keep", pod="keep-backend-6d86585d96-dv22z"} |= "ghp_"
```

## 修复

### 1. 修复 Keep workflowstore parser bug（高优先级）

检查 Keep 源码 `keep/workflowmanager/workflowstore.py` 中 provider 提取逻辑，增加类型判断或使用正确的 dict 访问方式，避免对 dict 对象调用 `.split()`。同时修正日志消息中的拼写错误 "providerts" → "providers"。

### 2. 将硬编码凭据迁移到 Kubernetes Secret（高优先级）

SMTP 密码和 GitHub token 不应出现在 workflow YAML 明文中。使用 Keep 的 secret manager 创建对应的 K8s Secret（如 `keep-cf5af687-...-secrets`），在 workflow 中通过 `apiKeyRef` 引用。当前 `Could not load secrets for workflow` 警告也说明 secret 未正确配置。

### 3. 修复 `confirmed-to-skill-pr` workflow 的级联失败

在 `format-as-skill` 步骤增加条件检查，或调整 workflow 执行顺序确保 enrich 数据可用。`create-pr` 的 422 错误需检查分支是否已存在（重复执行时分支不会自动覆盖），考虑复用现有分支或生成唯一分支名。

### 4. 测试告警管理

AISREKeepTest 系列告警用于验证 AI-SRE v4 pipeline，测试目的已达到。建议在测试完成后在 Keep 中 dismiss/resolve 这些测试告警，避免干扰生产告警看板。

## 预防

- **测试告警隔离**：为测试告警使用独立 fingerprint 前缀（如 `keepaisre*`），并在 Keep 中设置过滤规则，避免测试告警混入生产告警看板
- **凭据管理**：所有敏感凭据（SMTP 密码、GitHub token）必须通过 Kubernetes Secret 或 Keep secret manager 管理，禁止明文硬编码在 workflow YAML 中；定期扫描日志中是否泄露凭据
- **workflow 代码审查**：对 Keep workflowstore parser 等核心模块增加类型检查和单元测试，防止 dict/str 类型混用导致的运行时错误
- **级联失败防护**：workflow 步骤间依赖数据（如 `alert.enriched.holmes_analysis`）应增加就绪检查或重试机制，避免数据未就绪时的级联失败
- **PR 创建幂等性**：`create-pr` 步ꪤ应处理分支已存在的情况（复用现有分支或生成唯一分支名），避免 422 错误

