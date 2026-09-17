# AI SRE 经验归档仓库结构设计

- 日期：2026-09-17
- 状态：已批准（方案 A）

## 1. 背景与目标

把 holmesgpt 每次问题定位的结论沉淀为可复用的「经验」，形成自我进化闭环：

```
问题发生 → holmesgpt 排查并定位 → 生成 SKILL.md → 提 PR 到本仓 → 合入
        → holmesgpt 每 5 分钟自动拉取本仓 → 下次同类问题直接命中经验
```

本仓即 holmesgpt 的 **Skill 仓库**（holmesgpt 旧称 runbook，0.26.0+ 改名 Skill）。

## 2. 关键约束（来自 holmesgpt 官方文档）

- 每个 skill = 一个目录 + 一个 `SKILL.md`。
- `SKILL.md` = YAML frontmatter（`name`、`description` 必填、`last_updated` 可选）+ markdown 正文。
- holmesgpt 用 `description` 做 LLM 语义匹配；颗粒度越细、描述越具体，匹配越准、越省 token。
- holmesgpt 通过 `skillRepos` 直接 git clone 本仓，每 5 分钟自动拉取。
- **硬约束：holmesgpt 只扫描 subPath 下最多 2 层深度的 `SKILL.md`**，第 3 层不会被加载。

## 3. 目录方案（A，已采纳）

两层结构，服务编码进 skill 名：

```
skills/<layer>/<service-or-component>-<issue>/SKILL.md
```

- `layer` ∈ {infra, middleware, app, access}
- 第 2 层目录名 = `<service-or-component>-<issue>`，如 `order-service-oom-kill`
- 一个 issue 一个 SKILL.md，匹配最精准
- 每层 `INDEX.md` 提供按服务聚合的人工导航，补足"按服务"的视觉需求
- 富元数据（服务/层/关键词/来源工单…）放正文 `## 元数据` 表格，不放 frontmatter，规避 holmes 解析未知字段的风险

被否决的方案：B（service 作第 2 层、多问题合一 → 匹配颗粒度粗）；C（多 subPath 换 3 层 → 配置复杂、同仓多次 clone）。

## 4. SKILL.md 模板结构

- frontmatter：`name`、`description`（极度具体）、`last_updated`
- 正文：`## 元数据` → `## 目标` → `## 症状/已知故障模式` → `## 排查步骤` → `## 根因` → `## 修复方案` → `## 验证` → `## 负责人与升级路径` → `## 参考`

## 5. 交付物

- `CLAUDE.md`：AI agent 在本仓工作的规范（格式/约束/命名/PR 流程/质量 checklist/安全红线）
- `templates/SKILL.template.md`：经验归档模板
- `README.md`：仓库说明 + 接入 holmesgpt 的 `skillRepos` 配置示例
- `skills/INDEX.md` + 四层 `INDEX.md`
- 4 个示例 skill：infra/k8s-crashloopbackoff、middleware/redis-memory-spike、app/order-service-oom-kill、access/gateway-502

## 6. 安全红线

Skill 会被 holmesgpt 送进 LLM 上下文，严禁写入任何密钥/Token/密码/证书/kubeconfig/数据库凭据。
