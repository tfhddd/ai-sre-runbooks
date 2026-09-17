# ai-sre-runbooks

AI SRE 经验归档仓库 —— 作为 [holmesgpt](https://github.com/HolmesGPT/holmesgpt) 的 Skill 源，把每次问题定位的结论沉淀为可复用的经验，形成自我进化闭环。

```
问题发生 → holmesgpt 排查定位 → 按模板生成 SKILL.md → 提 PR → 合入
        → holmesgpt 每 5 分钟自动拉取本仓 → 下次同类问题命中经验
```

## 目录结构

```
skills/<layer>/<service-or-component>-<issue>/SKILL.md
```

- `layer` ∈ `infra`（基础设施）/ `middleware`（中间件）/ `app`（业务应用）/ `access`（接入层）
- 第 2 层目录名 = `<service-or-component>-<issue>`，如 `order-service-oom-kill`
- 一个 issue 一个 SKILL.md，颗粒度细、匹配准
- holmesgpt 只扫描 subPath 下最多 2 层 `SKILL.md`，因此**禁止第 3 层嵌套**

完整骨架见下方"目录总览"。所有 skill 索引见 `skills/INDEX.md`。

## 新增一条经验

1. 阅读 `CLAUDE.md`（强制规范）。
2. 复制 `templates/SKILL.template.md` 到 `skills/<layer>/<service-or-component>-<issue>/SKILL.md`。
3. 填 frontmatter（`name` / `description` 必填且具体 / `last_updated`）与正文各节。
4. 在对应层 `INDEX.md` 增一行链接。
5. 自检 `CLAUDE.md` 第 5 节质量 checklist（含安全红线）。
6. 分支 `skill/<layer>-<service>-<issue>`，提 PR。

## 接入 holmesgpt

CLI `~/.holmes/config.yaml`：

```yaml
skill_repos:
  - url: https://github.com/tfhddd/ai-sre-runbooks.git
    branch: main
    sub_path: skills
    # 公开仓库无需 token；私有仓库加 token_env: GITHUB_SKILLS_TOKEN
```

合入 `main` 后，holmesgpt 最长 5 分钟自动拉取生效，无需重启。详见 `CLAUDE.md` 第 8 节。

## 目录总览

```
ai-sre-runbooks/
├─ CLAUDE.md                    # AI agent 协作规范（必读）
├─ README.md
├─ templates/
│  └─ SKILL.template.md         # 经验归档模板
├─ docs/superpowers/specs/      # 设计稿
└─ skills/                      # ← holmesgpt subPath 指向这里
   ├─ INDEX.md
   ├─ infra/
   │  ├─ INDEX.md
   │  └─ k8s-crashloopbackoff/SKILL.md
   ├─ middleware/
   │  ├─ INDEX.md
   │  └─ redis-memory-spike/SKILL.md
   ├─ app/
   │  ├─ INDEX.md
   │  └─ order-service-oom-kill/SKILL.md
   └─ access/
      ├─ INDEX.md
      └─ gateway-502/SKILL.md
```

## 安全红线

Skill 会被 holmesgpt 送进 LLM 上下文，严禁写入任何密钥/Token/密码/证书/kubeconfig/数据库凭据/PII。详见 `CLAUDE.md` 第 6 节。
