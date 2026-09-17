# CLAUDE.md — AI SRE 经验归档仓库协作规范

本文件是任何 AI agent（Claude / opencode / Copilot 等）在本仓工作时的强制规范。改动本仓前必读。

## 1. 仓库定位与闭环

本仓是 **holmesgpt 的 Skill 仓库**（holmesgpt 0.26.0+ 用 Skill 取代旧 runbook）。闭环：

```
问题发生 → holmesgpt 排查定位 → 按模板生成 SKILL.md → 提 PR → 合入
        → holmesgpt 每 5 分钟自动拉取本仓 → 下次同类问题命中经验
```

你的核心任务：把一次问题定位的结论，归档成一个**高质量、可被 holmesgpt 精准匹配**的 Skill，并提 PR。

## 2. Skill 格式（必须遵守）

来源：holmesgpt 官方文档 `docs/reference/skills.md`。

- 一个 skill = 一个目录，目录内一个 `SKILL.md`。
- `SKILL.md` = YAML frontmatter + markdown 正文。
- **frontmatter 只允许三个字段**，不要加别的（避免 holmes 解析未知字段）：
  - `name`：小写连字符，默认等于父目录名，可省略。
  - `description`：**必填**。holmesgpt 只靠它做语义匹配，必须极度具体、问题导向。
  - `last_updated`：`YYYY-MM-DD`。
- 推荐正文小节：`## 目标` / `## 排查步骤` / `## 综合判断` / `## 修复建议`。
- 本仓在此基础上扩展为：`## 元数据` / `## 目标` / `## 症状或已知故障模式` / `## 排查步骤` / `## 根因` / `## 修复方案` / `## 验证` / `## 负责人与升级路径` / `## 参考`。

### description 怎么写

- 好：「Diagnose Redis memory spikes every 6 hours caused by missing TTL on cache keys from cache-refresh service」
- 坏：「Redis 排查」「常见问题」「debug」——太泛，会被误匹配、白烧 token。

## 3. 目录结构（方案 A，两层）

```
skills/<layer>/<service-or-component>-<issue>/SKILL.md
```

- **硬约束：holmesgpt 只扫描 subPath 下最多 2 层 `SKILL.md`**。`skills/<layer>/<skill>/SKILL.md` 正好 2 层，**禁止再嵌套第 3 层**（会被漏掉）。
- 技术层 `<layer>` 固定四选一：
  - `infra`：基础设施（K8s/节点/网络/存储/容器运行时）
  - `middleware`：中间件（Redis/MySQL/Kafka/ES/RabbitMQ/MQ）
  - `app`：业务应用微服务
  - `access`：接入层（网关/LB/Ingress/CDN/DNS）
- 第 2 层目录名 = `<service-or-component>-<issue>`，全小写连字符。例：
  - `app/order-service-oom-kill`
  - `middleware/redis-memory-spike`
  - `infra/k8s-crashloopbackoff`
  - `access/gateway-502`
- 若某 service 问题多，用前缀天然聚拢：`order-service-oom-kill`、`order-service-latency-spike`… 不要把它们塞进同一个 SKILL.md。
- 富元数据放正文 `## 元数据` 表格（service/layer/component/issue_type/severity/keywords/source_ticket），**不放 frontmatter**。

## 4. 归档流程（提 PR）

1. 复制 `templates/SKILL.template.md` 为 `skills/<layer>/<service-or-component>-<issue>/SKILL.md`。
2. 填 frontmatter：`name` = 目录名；`description` 极度具体；`last_updated` = 今天。
3. 填正文各节（参考 `skills/` 下已有示例）。
4. 在对应 `skills/<layer>/INDEX.md` 增一行链接。
5. 本地自检（见第 5 节 checklist）。
6. 分支命名：`skill/<layer>-<service>-<issue>`，如 `skill/app-order-service-oom-kill`。
7. commit message：`docs(skill): add <layer>/<service-or-component>-<issue>`。
8. PR 标题同上；PR 描述写：定位的问题、根因一句话、来源（holmesgpt 会话/工单号）、是否含敏感信息自检结论。

## 5. 质量 checklist（PR 前逐条确认）

- [ ] 目录在 `skills/<layer>/<2层>/SKILL.md`，未超 2 层。
- [ ] `description` 具体到「服务 + 故障现象 + 触发条件」，不泛。
- [ ] 排查步骤可复现、可执行（有命令/查询/检查点），不止"检查一下"。
- [ ] 有明确根因，不止症状。
- [ ] 有修复方案与验证方式。
- [ ] 有 `## 负责人与升级路径`（owner/Slack/on-call）。
- [ ] `last_updated` 已填。
- [ ] 对应层 `INDEX.md` 已加链接。
- [ ] **无任何密钥/Token/密码/证书/kubeconfig/数据库凭据/PII**。

## 6. 安全红线（不可破）

Skill 会被 holmesgpt 送进 LLM 上下文，可能出现在回答里。**严禁写入**：API key、token、密码、私钥、证书、kubeconfig、数据库连接串、生产敏感 IP/账号、用户 PII。需要凭据时只写"联系 XXX 团队获取"，不写值本身。

## 7. 维护与过期清理

- 服务改名/迁移/下线、依赖变更、步骤失效时更新或删除对应 skill。过期 skill 会误导排查，宁缺毋滥。
- 季度复盘：按 `last_updated` 找超过 180 天的 skill，复核是否仍准确。
- 删除优于保留失效内容。

## 8. holmesgpt 接入（仓库维护者参考）

本仓作为 holmesgpt skill 源，`skillRepos` 配置示例（CLI `~/.holmes/config.yaml`）：

```yaml
skill_repos:
  - url: https://github.com/tfhddd/ai-sre-runbooks.git
    branch: main
    sub_path: skills          # 指向本仓 skills/ 目录
    # 公开仓库无需 token；私有仓库加 token_env: GITHUB_SKILLS_TOKEN
```

合入 main 后，holmesgpt 最长 5 分钟自动拉取生效，无需重启。
