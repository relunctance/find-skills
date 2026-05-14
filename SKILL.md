---
name: find-skills
description: 全平台 AI Skill 智能检索 — 去重 + 精准匹配 + Health Score 排序，支持 Hermes/Claude/OpenClaw 所有生态
triggers:
  - 找 skill
  - 检索 skill
  - 有没有 XXX 的 skill
  - 搜索 skill
  - 安装 skill
  - skill 推荐
  - 哪个 skill 能做 XXX
  - find a skill
  - search for skill
  - skill for XXX
category: Infrastructure
author: relunctance
created: 2026-05-14
updated: 2026-05-14
tags:
  - skill
  - search
  - discovery
  - hermes
  - claude
  - openclaw
  - cross-platform
---

# find-skills

> 全平台 AI Skill 智能检索引擎 — 去重 · 精准匹配 · Health Score 排序

多生态（Hermes/Claude/OpenClaw）Skill 一站检索，自动去重，评分排序，安装命令一键生成。

## 何时触发

用户说：
- `找/搜索/有没有 XXX 的 skill`
- `安装 skill`
- `哪个 skill 能做 XXX`
- `skill 推荐`
- `find a skill for XXX`

**自动触发，无需用户明确说"用 find-skills"**

## 检索策略

### 第一步：理解需求

- 提取用户需求的核心功能词（英译中、中译英都要试）
- 判断类别：开发工具 / 办公效率 / AI/ML / 系统 / 创意
- 判断平台偏好：Hermes / Claude / OpenClaw / 通用
- 非英语查询：同时搜英文原文 + 中文

### 第二步：多源并行检索

```bash
# Hermes Skills Hub（官方市场，79个官方skill）
hermes skills search "<query>" 2>/dev/null

# OpenClaw ClawHub（语义检索，需网络）
curl -s "https://clawhub.ai/api/v1/search?q=<url-encoded-query>&limit=10" 2>/dev/null | jq '.[] | {slug, displayName, summary, score}'

# gql-skills 索引（内部skill总览）
grep -i "<keyword>" ~/repos/gql-skills/README.md ~/repos/gql-skills/SKILL.md 2>/dev/null

# Antigravity awesome-skills（1459个skill的结构化目录）
curl -s "https://raw.githubusercontent.com/sickn33/antigravity-awesome-skills/main/skills_index.json" 2>/dev/null | jq ".[] | select(.name | test(\"<query>\"; \"i\")) | {name, description, tags}" 2>/dev/null

# ComposioHQ awesome-claude-skills（1000+官方skill目录）
curl -s "https://raw.githubusercontent.com/ComposioHQ/awesome-claude-skills/main/README.md" 2>/dev/null | grep -A3 "<query>"

# VoltAgent awesome-agent-skills
curl -s "https://raw.githubusercontent.com/VoltAgent/awesome-agent-skills/main/README.md" 2>/dev/null | grep -A2 "<query>"
```

> **clawhub.sh 已下线**：之前用 clawhub mirror 的方式已废弃，直接用 `https://clawhub.ai/api/v1/search` 的公开 API（无需 token，120 reads/min）。

### 第三步：去重与合并

来自不同平台的同一 skill，按以下优先级保留最新最准的记录：

| 平台 | 优先级 | 说明 |
|------|--------|------|
| Hermes 官方 | ⭐⭐⭐ | Nous Research 出品，质量最稳定 |
| gql-skills | ⭐⭐⭐ | 内部精选，真实可用 |
| ClawHub（活跃度高）| ⭐⭐ | 有统计数据 |
| Antigravity | ⭐⭐ | 数量最大，质量参差 |
| 其他 awesome 列表 | ⭐ | 需人工核实 |

去重规则：
- 名称完全相同 → 保留优先级高的
- 名称相似（编辑距离 < 3）→ 保留优先级高的
- 同一作者同一类目的多个版本 → 只保留最新版

### 第四步：Health Score 评分

每个候选 skill 打分（0–100）：

| 指标 | 计分 |
|------|------|
| 官方推荐/精选 | +30 |
| 下载/安装数高（>10k/+30，>1k/+15，>100/+8） | +8~30 |
| Stars 高（>10/+20，>1/+10） | +10~20 |
| 有安装命令/使用文档 | +10 |
| 近期更新（30天内/+10，90天内/+5） | +5~10 |
| 有中文说明 | +5 |
| 安全扫描通过（无危险权限） | +5 |
| 需要凭证/高权限 | -20 |
| 零安装零下载 | -20 |

评分等级：
- 🟢 **70–100**：强烈推荐
- 🟡 **40–69**：可用，建议人工核实
- 🔴 **< 40**：不推荐

### 第五步：安全扫描（评分 ≥ 40 必做）

读取 SKILL.md，检查危险信号：

```bash
curl -s "https://clawhub.ai/api/v1/skills/<slug>/<slug>/file?path=SKILL.md" 2>/dev/null
```

标记 ⚠️ 如包含：
- 硬编码的未知域名 curl 请求
- 读取 `~/.env`、`~/.ssh`、`~/.aws` 等凭证路径
- `rm -rf` 或大规模破坏性命令
- base64 编码后 pipe 到 shell
- 静默覆盖 system prompt 或外传数据

## 输出格式

```
🔍 **Skill 检索：「<需求>」**

---

**精选推荐** 🟢

1. `<skill-name>` ⭐⭐⭐ 85/100
   Hermes 官方 | ⭐ 45 | 📥 115 安装 | ↓ 14.3k 下载
   > 一句话描述
   触发词：<triggers>
   🔗 https://clawhub.ai/<owner>/<slug>
   安装：`hermes skills install <id>` 或 `clawhub install <slug>`

---

**其他候选** 🟡

2. `<skill-name>` 62/100
   > 描述
   ...

---

**安装命令**

\`\`\`bash
hermes skills install <identifier>
\`\`\`

**找不到？** → 考虑用 `skill-created` 创建自定义 skill
```

## 平台安装命令速查

| 平台 | 安装命令 |
|------|---------|
| Hermes | `hermes skills install <id>` |
| OpenClaw/ClawHub | `clawhub install <slug>` 或 `npx clawhub install <slug>` |
| Claude Code | `claude --skill-dir ./skills <skill-name>` |
| 通用（Git URL） | `npx skills add <github-url> --skill <name> --yes` |

## 平台链接格式

| 平台 | 链接格式 |
|------|---------|
| ClawHub | `https://clawhub.ai/<owner>/<slug>` |
| Hermes Skills Hub | `https://hermes-agent.nousresearch.com/docs/reference/skills-catalog` |
| gql-skills | `https://github.com/relunctance/gql-skills` |
| Antigravity | `https://github.com/sickn33/antigravity-awesome-skills` |
| Composio | `https://github.com/ComposioHQ/awesome-claude-skills` |
