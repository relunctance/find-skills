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

### 第二步：多源并行检索（9大来源）

```bash
# 1. Hermes Skills Hub（官方市场，79个官方skill）
hermes skills search "<query>" 2>/dev/null

# 2. OpenClaw ClawHub（语义检索，需网络）
curl -s "https://clawhub.ai/api/v1/search?q=<url-encoded-query>&limit=10" 2>/dev/null | jq '.[] | {slug, displayName, summary, score}'

# 3. gql-skills 索引（内部skill总览）
grep -i "<keyword>" ~/repos/gql-skills/README.md ~/repos/gql-skills/SKILL.md 2>/dev/null

# 4. Antigravity awesome-skills（1459个skill的结构化目录）
curl -s "https://raw.githubusercontent.com/sickn33/antigravity-awesome-skills/main/skills_index.json" 2>/dev/null | jq ".[] | select(.name | test(\"<query>\"; \"i\")) | {name, description, tags}" 2>/dev/null

# 5. ComposioHQ awesome-claude-skills（1000+skill）
curl -s "https://raw.githubusercontent.com/ComposioHQ/awesome-claude-skills/main/README.md" 2>/dev/null | grep -A3 "<query>"

# 6. obra/superpowers（14个核心skills，质量最高）
# https://github.com/obra/superpowers/tree/main/skills
# brainstorm/subagent-driven-dev/test-driven-development/systematic-debugging/writing-plans/...
curl -s "https://raw.githubusercontent.com/obra/superpowers/main/skills/<skill-name>/SKILL.md" 2>/dev/null

# 7. Fission-AI/OpenSpec（43个spec-driven开发技能）
# https://github.com/Fission-AI/OpenSpec/tree/main/openspec/specs
# cli-spec/schema-validation/workspace-foundation/telemetry/opsx-archive-skill/...
curl -s "https://raw.githubusercontent.com/Fission-AI/OpenSpec/main/openspec/specs/<spec-name>/SKILL.md" 2>/dev/null

# 8. affaan-m/everything-claude-code（228个skill + 60个agent）
# https://github.com/affaan-m/everything-claude-code/tree/main/skills
# python-reviewer/go-reviewer/tdd-guide/security-reviewer/...
curl -s "https://raw.githubusercontent.com/affaan-m/everything-claude-code/main/skills/<skill-name>/SKILL.md" 2>/dev/null

# 9. skill-market 本地索引（427个SKILL.md，基于 marketplace.zip）
# 自动安装：检测到未安装时自动 clone + 下载完整 zip + 解压
MARKETPLACE_REPO="$HOME/repos/skill-market"
MARKETPLACE_DIR="$HOME/repos/skill-market/data"
MARKETPLACE_ZIP="$MARKETPLACE_DIR/marketplace.zip"
MARKETPLACE_EXTRACT="$MARKETPLACE_DIR/marketplace"

if [ ! -d "$MARKETPLACE_REPO" ]; then
    echo "[skill-market] 首次使用，正在自动安装..."
    git clone https://github.com/relunctance/skill-market.git "$MARKETPLACE_REPO" 2>/dev/null && \
    bash "$MARKETPLACE_REPO/scripts/setup.sh" > /dev/null 2>&1
fi

if [ ! -f "$MARKETPLACE_ZIP" ]; then
    echo "[skill-market] 正在下载 marketplace.zip（15MB）..."
    mkdir -p "$MARKETPLACE_DIR"
    curl -sL "https://download.codebuddy.cn/plugin-marketplace/codebuddy-plugins-official.zip" -o "$MARKETPLACE_ZIP"
fi

if [ ! -d "$MARKETPLACE_EXTRACT" ]; then
    echo "[skill-market] 正在解压..."
    unzip -q "$MARKETPLACE_ZIP" -d "$MARKETPLACE_EXTRACT"
fi

# 解压后用 grep -rni 搜索 + 解析 frontmatter + 标注匹配位置
QUERY="<query>"
echo ""
echo "## skill-market 检索结果: $QUERY"
echo ""

# 搜索（grep -rni 返回 文件:行号:上下文）
TMPRESULTS=$(mktemp)
> "$TMPRESULTS"

grep -rni "$QUERY" "$MARKETPLACE_EXTRACT" --include="SKILL.md" 2>/dev/null | while IFS=: read -r file linenum context; do
    # 按文件去重（只处理一次）
    if ! grep -q "^FILE:$file" "$TMPRESULTS" 2>/dev/null; then
        echo "FILE:$file" >> "$TMPRESULTS"
        
        # 解析 frontmatter
        NAME=$(grep "^name:" "$file" 2>/dev/null | head -1 | cut -d: -f2- | sed 's/^ *//' | sed 's/^"//' | sed 's/"$//')
        DESC=$(grep "^description:" "$file" 2>/dev/null | head -1 | cut -d: -f2- | sed 's/^ *//' | sed 's/^"//' | sed 's/"$//')
        
        # 分类
        case "$file" in
            *superpowers/skills/*) SOURCE="superpowers" ;;
            *scientific-skills/*) SOURCE="scientific-skills" ;;
            *plugins/*) SOURCE="official" ;;
            *) SOURCE="other" ;;
        esac
        
        # 标注匹配位置（name/description/正文）
        if echo "$NAME" | grep -qi "$QUERY"; then
            MATCH="name"
        elif echo "$DESC" | grep -qi "$QUERY"; then
            MATCH="description"
        else
            MATCH="正文"
        fi
        
        # 简化路径
        DISPLAY=$(echo "$file" | sed "s|$MARKETPLACE_EXTRACT/||" | sed 's|/SKILL.md||')
        echo "$SOURCE|$NAME|$MATCH|${DESC:0:70}...|$DISPLAY" >> "$TMPRESULTS"
    fi
done

# 搜索结果输出给 LLM 翻译和总结
# 格式：原始 JSON，便于 LLM 解析
echo ""
echo "## skill-market 检索结果: $QUERY"
echo ""

# 生成 JSON 格式结果
TMPRESULTS=$(mktemp)
echo "[" > "$TMPRESULTS"

FIRST=true
grep -rni "$QUERY" "$MARKETPLACE_EXTRACT" --include="SKILL.md" 2>/dev/null | while IFS=: read -r file linenum context; do
    if ! grep -q "\"file\":\"$file\"" "$TMPRESULTS" 2>/dev/null; then
        NAME=$(grep "^name:" "$file" 2>/dev/null | head -1 | cut -d: -f2- | sed 's/^ *//' | sed 's/^"//' | sed 's/"$//')
        DESC=$(grep "^description:" "$file" 2>/dev/null | head -1 | cut -d: -f2- | sed 's/^ *//' | sed 's/^"//' | sed 's/"$//')
        
        case "$file" in
            *superpowers/skills/*) SOURCE="superpowers" ;;
            *scientific-skills/*) SOURCE="scientific-skills" ;;
            *plugins/*) SOURCE="official" ;;
            *) SOURCE="other" ;;
        esac
        
        if echo "$NAME" | grep -qi "$QUERY"; then
            MATCH="name"
        elif echo "$DESC" | grep -qi "$QUERY"; then
            MATCH="description"
        else
            MATCH="正文"
        fi
        
        DISPLAY=$(echo "$file" | sed "s|$MARKETPLACE_EXTRACT/||" | sed 's|/SKILL.md||')
        
        # 计算 quality_score
        case "$SOURCE" in
            superpowers) SCORE=100 ;;
            scientific-skills) SCORE=80 ;;
            official) SCORE=60 ;;
            *) SCORE=40 ;;
        esac
        
        [ "$FIRST" = true ] && FIRST=false || echo "," >> "$TMPRESULTS"
        echo "  {" >> "$TMPRESULTS"
        echo "    \"name\": \"$NAME\"," >> "$TMPRESULTS"
        echo "    \"description\": \"$DESC\"," >> "$TMPRESULTS"
        echo "    \"source\": \"$SOURCE\"," >> "$TMPRESULTS"
        echo "    \"match\": \"$MATCH\"," >> "$TMPRESULTS"
        echo "    \"path\": \"$DISPLAY\"," >> "$TMPRESULTS"
        echo "    \"quality_score\": $SCORE" >> "$TMPRESULTS"
        echo "  }" >> "$TMPRESULTS"
    fi
done

echo "]" >> "$TMPRESULTS"

# 读取结果用于后续处理
RESULTS=$(cat "$TMPRESULTS")

# 按 quality_score 排序输出 Markdown 表格
echo "### 检索结果（按质量排序）"
echo ""
echo "| 质量 | 名称 | 匹配 | 描述 | 来源 |"
echo "|-----|------|------|------|------|"

echo "$RESULTS" | python3 -c "
import json, sys
try:
    data = json.load(sys.stdin)
    # 按 quality_score 降序，match 优先级排序
    def sort_key(x):
        match_priority = {'name': 0, 'description': 1, '正文': 2}
        return (-x.get('quality_score', 0), match_priority.get(x.get('match', '正文'), 2))
    data.sort(key=sort_key)
    for item in data:
        score = '⭐' * (item.get('quality_score', 0) // 25)
        name = item.get('name', '')
        match = item.get('match', '')
        desc = item.get('description', '')[:60]
        source = item.get('path', '').split('/')[0]
        print(f'| {score} | \`{name}\` | {match} | {desc}... | {source} |')
except:
    pass
"

echo ""
echo "💡 **推荐**: superpowers 来源质量最高（obra 出品，PR 94% 拒收率）"

# 提供 LLM 翻译提示
echo ""
echo "---"
echo "**LLM 翻译任务**：请将以上结果翻译为中文，并说明："
echo "1. 每个 skill 的**功能**（能做什么）"
echo "2. 每个 skill 的**价值**（解决什么问题）"
echo "3. 多个结果时的**对比**和**选择建议**"
echo ""
echo "如需查看完整描述，可用："
echo "```bash"
echo "cat << 'EOF' | python3 -c \"import json,sys; [print(f'## {x[""name""]}\n{f\"\"来源: {x['"'"'source'\"']}\"}\n{f\"\"描述: {x['"'"'description'\"']}\"}\n\") for x in json.load(sys.stdin)]\" "
cat "$TMPRESULTS"
echo "EOF"
echo "```"

rm -f "$TMPRESULTS"
```

> **clawhub.sh 已下线**：改用 `https://clawhub.ai/api/v1/search` 公开 API（无需 token，120 reads/min）。

### 第三步：去重与合并

来自不同平台的同一 skill，按以下优先级保留最新最准的记录：

| 平台 | 优先级 | 说明 |
|------|--------|------|
| Hermes 官方 | ⭐⭐⭐ | Nous Research 出品，质量最稳定 |
| gql-skills | ⭐⭐⭐ | 内部精选，真实可用 |
| obra/superpowers | ⭐⭐⭐ | 核心方法论，PR 94%拒收率，质量最高 |
| skill-market | ⭐⭐ | 本地 ZIP，427个 SKILL.md，快速检索 |
| affaan-m/everything-claude-code | ⭐⭐ | 228 skill + 60 agent，规模最大 |
| Fission-AI/OpenSpec | ⭐⭐ | spec-driven 开发，架构严谨 |
| ClawHub（活跃度高）| ⭐⭐ | 有统计数据 |
| Antigravity | ⭐ | 1459 skill，数量最大，质量参差 |
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
| Hermes Skills Hub | `https://hermes-agent.nousresearch.com/docs/reference/skills-catalog` |
| ClawHub | `https://clawhub.ai/<owner>/<slug>` |
| gql-skills | `https://github.com/relunctance/gql-skills` |
| obra/superpowers | `https://github.com/obra/superpowers/tree/main/skills` |
| Fission-AI/OpenSpec | `https://github.com/Fission-AI/OpenSpec/tree/main/openspec/specs` |
| affaan-m/everything-claude-code | `https://github.com/affaan-m/everything-claude-code/tree/main/skills` |
| Antigravity | https://github.com/sickn33/antigravity-awesome-skills |
| Composio | https://github.com/ComposioHQ/awesome-claude-skills |
| skill-market | https://github.com/relunctance/skill-market |
