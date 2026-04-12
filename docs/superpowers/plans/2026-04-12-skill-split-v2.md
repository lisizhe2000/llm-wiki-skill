# llm-wiki Skill 拆分 v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-split the original 702-line root SKILL.md into 8 sub-skills + shared context with 100% content coverage, fixing the v1 split that lost 55% of content.

**Architecture:** Reset working tree to commit 36fbee3 (original SKILL.md intact), then re-split using a shared `skills/_shared/context.md` for common content (state model, preflight checks, language rules, terminology) and 8 sub-skills each containing complete workflow documentation with output examples. Frontmatter reuses v1 definitions. Tests updated from the 36fbee3 regression.sh baseline.

**Tech Stack:** Bash scripts, Markdown, shell-based regression tests

**Spec:** `docs/superpowers/specs/2026-04-11-skill-split-cleanup-design.md`

---

## File Structure

```
skills/
├── _shared/
│   └── context.md              # CREATE ~120 lines — shared state model, preflight, language rules
├── wiki-init/
│   ├── SKILL.md                # CREATE ~85 lines — init workflow + dependency checks
│   ├── install.sh              # MOVE from repo root
│   └── setup.sh                # MOVE from repo root
├── wiki-ingest/SKILL.md        # CREATE ~155 lines — full ingest workflow
├── wiki-batch-ingest/SKILL.md  # CREATE ~70 lines
├── wiki-query/SKILL.md         # CREATE ~30 lines
├── wiki-lint/SKILL.md          # CREATE ~70 lines
├── wiki-status/SKILL.md        # CREATE ~60 lines
├── wiki-digest/SKILL.md        # CREATE ~80 lines
└── wiki-graph/SKILL.md         # CREATE ~60 lines

# Root level changes:
SKILL.md                        # DELETE (replaced by sub-skills)
install.sh                      # MOVE → skills/wiki-init/install.sh
setup.sh                        # MOVE → skills/wiki-init/setup.sh
platforms/                      # DELETE (obsolete)
AGENTS.md                       # REWRITE as unified dev guide
CLAUDE.md                       # RECREATE as symlink → AGENTS.md
README.md                       # UPDATE (keep original, remove platforms/ refs)
docs/references/karpathy-llm-wiki.md  # MOVE from repo root
tests/regression.sh             # REWRITE from 36fbee3 baseline
tests/adapter-state.sh          # UPDATE paths only
```

---

### Task 1: Reset working tree and backup design doc

**Files:**
- Backup: `docs/superpowers/specs/2026-04-11-skill-split-cleanup-design.md`
- Backup: `docs/references/karpathy-llm-wiki.md`
- Reset: entire working tree to 36fbee3

- [ ] **Step 1: Backup files that must survive the reset**

```bash
cp docs/superpowers/specs/2026-04-11-skill-split-cleanup-design.md /tmp/skill-split-design.md
cp docs/references/karpathy-llm-wiki.md /tmp/karpathy-llm-wiki.md
cp docs/superpowers/plans/2026-04-12-skill-split-v2.md /tmp/skill-split-v2-plan.md
```

- [ ] **Step 2: Reset working tree AND index to 36fbee3 clean state**

Note: `git checkout -- .` would only restore working tree to match the index (which has staged deletions/additions). We need `git checkout HEAD -- .` to reset both index and working tree to the last commit.

```bash
git checkout HEAD -- .
```

This restores: root SKILL.md (702 lines), install.sh, setup.sh, platforms/, original AGENTS.md, README.md, CLAUDE.md, original tests/regression.sh (521 lines), original tests/adapter-state.sh. It also unstages all changes.

- [ ] **Step 3: Verify reset succeeded**

```bash
test -f SKILL.md && echo "OK: SKILL.md restored ($(wc -l < SKILL.md) lines)"
test -f install.sh && echo "OK: install.sh restored"
test -f setup.sh && echo "OK: setup.sh restored"
test -d platforms && echo "OK: platforms/ restored"
wc -l < tests/regression.sh  # expect 521
```

- [ ] **Step 4: Restore backed-up files**

```bash
mkdir -p docs/superpowers/specs docs/superpowers/plans docs/references
cp /tmp/skill-split-design.md docs/superpowers/specs/2026-04-11-skill-split-cleanup-design.md
cp /tmp/karpathy-llm-wiki.md docs/references/karpathy-llm-wiki.md
cp /tmp/skill-split-v2-plan.md docs/superpowers/plans/2026-04-12-skill-split-v2.md
```

- [ ] **Step 5: Commit baseline**

```bash
git add docs/superpowers/ docs/references/
git commit -m "chore: backup design doc and plan before v2 split"
```

---

### Task 2: Create `skills/_shared/context.md`

**Files:**
- Create: `skills/_shared/context.md`
- Source: `SKILL.md` lines 12-14, 16-27, 37-45, 62-104, 125-139, 141-160

This file extracts all shared content that every sub-skill needs. Content is copied verbatim from the original SKILL.md with one change: `WIKI_SKILL_ROOT` is fixed to `~/.agents/skills/llm-wiki/` instead of relative path resolution.

- [ ] **Step 1: Create the shared context file**

Copy the following sections from SKILL.md into `skills/_shared/context.md`:

1. H1 title + tagline (lines 12-14): `# llm-wiki — 个人知识库构建系统` + intro paragraph
2. "这个 skill 做什么" + 核心理念 (lines 16-27): full section
3. Script Directory + Path Resolution (lines 37-45): rewrite `SKILL_DIR` to fixed `WIKI_SKILL_ROOT = ~/.agents/skills/llm-wiki/`
4. 外挂状态模型 (lines 62-104): full section with all 5 states, `source-registry.sh` and `adapter-state.sh` usage
5. 通用前置检查 (lines 125-139): full CWD → ~/.llm-wiki-path → .wiki-schema.md → WIKI_LANG logic
6. 输出语言规则 + 术语对照 (lines 141-160): full section with WIKI_LANG rules and zh/en terminology table

The file header should be:
```markdown
# llm-wiki — 共享上下文

> 所有子 skill 在执行前先读取本文件。
```

All `${SKILL_DIR}/scripts/` references become `${WIKI_SKILL_ROOT}/scripts/`. All `${SKILL_DIR}/templates/` become `${WIKI_SKILL_ROOT}/templates/`.

- [ ] **Step 2: Verify shared context covers all required sections**

```bash
grep -c "外挂状态模型" skills/_shared/context.md    # expect 1
grep -c "通用前置检查" skills/_shared/context.md    # expect 1
grep -c "输出语言规则" skills/_shared/context.md    # expect 1
grep -c "术语对照" skills/_shared/context.md        # expect 1
grep -c "WIKI_SKILL_ROOT" skills/_shared/context.md # expect >= 4
wc -l < skills/_shared/context.md                   # expect ~120
```

- [ ] **Step 3: Commit**

```bash
git add skills/_shared/context.md
git commit -m "feat: create shared context for sub-skills (~120 lines from original SKILL.md)"
```

---

### Task 3: Create wiki-init sub-skill + move install.sh/setup.sh

**Files:**
- Create: `skills/wiki-init/SKILL.md`
- Move: `install.sh` → `skills/wiki-init/install.sh`
- Move: `setup.sh` → `skills/wiki-init/setup.sh`
- Source: `SKILL.md` lines 47-60 (dependency checks), 162-221 (init workflow)

- [ ] **Step 1: Create skills/wiki-init/ directory**

```bash
mkdir -p skills/wiki-init
```

- [ ] **Step 2: Create wiki-init/SKILL.md**

Write `skills/wiki-init/SKILL.md` with:

1. Frontmatter (reuse v1):
```yaml
---
name: wiki-init
description: Use when the user wants to create or initialize a new personal knowledge base, says "初始化知识库", "新建 wiki", or "创建知识库"
---
```

2. Shared context reference:
```markdown
> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`
```

3. Full content from SKILL.md lines 47-60 (dependency checks section):
   - "首次使用时，检查以下依赖是否已安装" paragraph
   - `bash ${WIKI_SKILL_ROOT}/setup.sh` reference (update path to `${WIKI_SKILL_ROOT}/skills/wiki-init/setup.sh`)
   - Three dependency skills list (baoyu-url-to-markdown, wechat-article-to-markdown, youtube-transcript)
   - "即使部分依赖缺失，skill 仍可工作" note

4. Full content from SKILL.md lines 162-221 (init workflow):
   - 前置检查（含多知识库 CWD 检查）: 3-step check (CWD → ~/.llm-wiki-path → init)
   - 步骤 1: 询问知识库主题 (with default "我的知识库")
   - 步骤 2: 询问知识库语言 (zh/en, default zh)
   - 步骤 3: 询问保存位置 (with defaults for zh/en)
   - 步骤 4: 运行 `bash ${WIKI_SKILL_ROOT}/scripts/init-wiki.sh "<路径>" "<主题>"`
   - 步骤 5: 写入语言配置 + 本地化种子文件 (English template overwrite logic)
   - 步骤 6: 记录路径到 `~/.llm-wiki-path`
   - 步骤 7: 输出引导 — full zh example block AND en note

Expected: ~85 lines total.

- [ ] **Step 3: Move install.sh and setup.sh**

```bash
git mv install.sh skills/wiki-init/install.sh
git mv setup.sh skills/wiki-init/setup.sh
```

- [ ] **Step 4: Verify**

```bash
wc -l < skills/wiki-init/SKILL.md  # expect ~85
test -f skills/wiki-init/install.sh && echo "OK"
test -f skills/wiki-init/setup.sh && echo "OK"
grep -c "_shared/context.md" skills/wiki-init/SKILL.md  # expect 1
grep -c "setup.sh" skills/wiki-init/SKILL.md  # expect >= 1
```

- [ ] **Step 5: Commit**

```bash
git add skills/wiki-init/
git commit -m "feat: create wiki-init sub-skill with dependency checks, move install.sh/setup.sh"
```

---

### Task 4: Create wiki-ingest sub-skill

**Files:**
- Create: `skills/wiki-ingest/SKILL.md`
- Source: `SKILL.md` lines 223-367 (ingest workflow)

- [ ] **Step 1: Create skills/wiki-ingest/ directory**

```bash
mkdir -p skills/wiki-ingest
```

- [ ] **Step 2: Create wiki-ingest/SKILL.md**

Write `skills/wiki-ingest/SKILL.md` with:

1. Frontmatter (reuse v1):
```yaml
---
name: wiki-ingest
description: Use when the user explicitly asks to add material to their llm-wiki knowledge base, says "消化这篇到知识库", "添加素材", "整理到 wiki", or "wiki ingest". Do NOT trigger for generic URL sharing, summarization, or file reading without wiki intent.
---
```

2. Shared context reference:
```markdown
> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`
```

3. Full content from SKILL.md lines 223-367:
   - H1 + intro: "这是最核心的工作流。用户给一个素材进来，AI 做所有的整理工作。"
   - 前置检查: "执行**通用前置检查**（见共享上下文）"
   - 素材提取路由 — FULL section including:
     - 外挂前置判断 (source-registry.sh match-url/match-file/get, adapter-state.sh check, 10-column and 8-column descriptions)
     - Chrome 提示 (full block about lsof, env_unavailable, baoyu-url-to-markdown auto-handling)
     - URL 类素材 routing (manual_only, wechat, youtube, baoyu)
     - 本地文件 routing
     - 纯文本粘贴 routing
     - 统一回退规则 (classify-run, runtime_failed, empty_result handling)
   - 内容分级处理: >1000字 完整, <=1000字 简化
   - 完整处理流程 (9 steps with full descriptions)
   - 完整处理输出示例 (zh block + en note)
   - 简化处理流程 (6 steps)
   - 简化处理输出示例 (zh block + en note)

Expected: ~155 lines total.

- [ ] **Step 3: Verify**

```bash
wc -l < skills/wiki-ingest/SKILL.md  # expect ~155
grep -c "_shared/context.md" skills/wiki-ingest/SKILL.md  # expect 1
grep -c "source-registry.sh match-url" skills/wiki-ingest/SKILL.md  # expect 1
grep -c "adapter-state.sh check" skills/wiki-ingest/SKILL.md  # expect >= 1
grep -c "adapter-state.sh classify-run" skills/wiki-ingest/SKILL.md  # expect >= 1
grep -c "Chrome CDP" skills/wiki-ingest/SKILL.md  # expect >= 1
grep -c "remote-debugging-port=9222" skills/wiki-ingest/SKILL.md  # expect >= 1
grep -c "已消化" skills/wiki-ingest/SKILL.md  # expect >= 1 (zh output example)
```

- [ ] **Step 4: Commit**

```bash
git add skills/wiki-ingest/
git commit -m "feat: create wiki-ingest sub-skill with full routing, grading, and output examples"
```

---

### Task 5: Create wiki-batch-ingest sub-skill

**Files:**
- Create: `skills/wiki-batch-ingest/SKILL.md`
- Source: `SKILL.md` lines 369-431

- [ ] **Step 1: Create wiki-batch-ingest/SKILL.md**

```bash
mkdir -p skills/wiki-batch-ingest
```

Write `skills/wiki-batch-ingest/SKILL.md` with:

1. Frontmatter (reuse v1):
```yaml
---
name: wiki-batch-ingest
description: Use when the user explicitly asks to batch-add materials to their llm-wiki knowledge base, says "批量消化到知识库", "wiki batch ingest", or "把这些都整理到 wiki". Do NOT trigger for generic folder operations without wiki intent.
---
```

2. Shared context reference
3. Full content from SKILL.md lines 369-431:
   - 步骤 1: 确认知识库路径 (通用前置检查 reference)
   - 步骤 2: 列出所有可处理文件 (.md, .txt, .pdf, .html; ignore hidden/.git/node_modules)
   - 步骤 3: 展示文件列表 (zh output example + en note)
   - 步骤 4: 逐个处理 (ingest workflow per file, auto grade by length)
   - 步骤 5: 每 5 个文件后暂停 (zh progress example + en note)
   - 步骤 6: 全部完成后 (index.md update, zh summary report + en note)

Expected: ~70 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-batch-ingest/SKILL.md  # expect ~70
grep -c "每 5 个" skills/wiki-batch-ingest/SKILL.md  # expect 1
grep -c "发现.*个文件" skills/wiki-batch-ingest/SKILL.md  # expect 1
git add skills/wiki-batch-ingest/
git commit -m "feat: create wiki-batch-ingest sub-skill with progress reporting"
```

---

### Task 6: Create wiki-query sub-skill

**Files:**
- Create: `skills/wiki-query/SKILL.md`
- Source: `SKILL.md` lines 433-453

- [ ] **Step 1: Create wiki-query/SKILL.md**

```bash
mkdir -p skills/wiki-query
```

Write with frontmatter + shared context reference + full content:

```yaml
---
name: wiki-query
description: Use when the user explicitly asks to query their llm-wiki knowledge base, says "查询知识库", "从 wiki 里找", "wiki query", or "知识库里有没有". Do NOT trigger for general questions unrelated to the wiki.
---
```

- 步骤 1: 确认知识库路径
- 步骤 2: 读取 index.md
- 步骤 3: 搜索相关页面 (index.md → Grep wiki/ → read 3-5 pages)
- 步骤 4: 综合回答 (WIKI_LANG, [[页面名]] citations, multiple viewpoints)
- 步骤 5: 建议回写 (ask user, save to wiki/synthesis/ or wiki/comparisons/, update index.md + log.md)

Expected: ~30 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-query/SKILL.md  # expect ~30
grep -c "建议回写" skills/wiki-query/SKILL.md  # expect 1
git add skills/wiki-query/
git commit -m "feat: create wiki-query sub-skill"
```

---

### Task 7: Create wiki-lint sub-skill

**Files:**
- Create: `skills/wiki-lint/SKILL.md`
- Source: `SKILL.md` lines 455-516

- [ ] **Step 1: Create wiki-lint/SKILL.md**

Write with frontmatter + shared context reference + full content:

```yaml
---
name: wiki-lint
description: Use when the user wants a health check on their wiki knowledge base, says "检查知识库", "健康检查", or "wiki lint". Do NOT trigger for generic linting requests (ESLint, pylint, etc.).
---
```

- 触发时机: user request + every 10th ingest auto-suggest
- 前置检查: 通用前置检查 reference
- 确定检查范围: recent 10 + random 10, or all if <=20
- 5 检查项 with full descriptions:
  1. 孤立页面 (Grep [[页面名]], list orphans, suggest links)
  2. 缺失概念页 (find [[某概念]] links to non-existent pages)
  3. 矛盾信息 (cross-page contradiction detection)
  4. 交叉引用缺失 (related topics should link each other)
  5. index 一致性 (index.md vs actual wiki files)
- 输出报告 (zh full example block + en note)
- 询问用户是否自动修复

Expected: ~70 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-lint/SKILL.md  # expect ~70
grep -c "孤立页面" skills/wiki-lint/SKILL.md  # expect >= 1
grep -c "断链" skills/wiki-lint/SKILL.md  # expect >= 1
grep -c "矛盾信息" skills/wiki-lint/SKILL.md  # expect >= 1
grep -c "知识库健康检查报告" skills/wiki-lint/SKILL.md  # expect 1
git add skills/wiki-lint/
git commit -m "feat: create wiki-lint sub-skill with 5 check items and report template"
```

---

### Task 8: Create wiki-status sub-skill

**Files:**
- Create: `skills/wiki-status/SKILL.md`
- Source: `SKILL.md` lines 518-571

- [ ] **Step 1: Create wiki-status/SKILL.md**

Write with frontmatter + shared context reference + full content:

```yaml
---
name: wiki-status
description: Use when the user asks about their wiki knowledge base status, says "知识库状态", "wiki 有什么", or "知识库有多少素材". Do NOT trigger for generic status queries unrelated to the wiki.
---
```

- 前置检查: 通用前置检查 reference
- 步骤 1: `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh list` 读取来源总表
- 步骤 2: 获取知识库路径
- 步骤 3: 统计 (按 source_label + raw_dir 逐项统计 raw/, entities/, topics/, sources/, comparisons/, synthesis/)
- 步骤 4: 读取 log.md 最后 5 条
- 步骤 5: 读取 index.md 主题概览
- 步骤 6: `bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh summary-human`
- 输出报告 (zh full example with 素材分布/Wiki 页面/最近活动/外挂状态/建议 + en note)
- Note: "外挂状态直接使用 adapter-state.sh summary-human 的输出，不要自己再重写一套来源清单"

Expected: ~60 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-status/SKILL.md  # expect ~60
grep -c "source-registry.sh list" skills/wiki-status/SKILL.md  # expect 1
grep -c "adapter-state.sh summary-human" skills/wiki-status/SKILL.md  # expect >= 1
grep -c "按来源总表中的" skills/wiki-status/SKILL.md  # expect 1
git add skills/wiki-status/
git commit -m "feat: create wiki-status sub-skill with source-registry integration"
```

---

### Task 9: Create wiki-digest sub-skill

**Files:**
- Create: `skills/wiki-digest/SKILL.md`
- Source: `SKILL.md` lines 573-646

- [ ] **Step 1: Create wiki-digest/SKILL.md**

Write with frontmatter + shared context reference + full content:

```yaml
---
name: wiki-digest
description: Use ONLY when the user explicitly invokes this skill via "wiki digest", "/wiki-digest", or "知识库 digest". This is a manual-only workflow — never auto-trigger from conversational phrases like "给我讲讲" or "总结一下".
---
```

- Intro: "区别于 query：query 是快速问答，不生成新页面；digest 是跨素材深度综合，生成持久化报告。"
- 触发关键词: "给我讲讲 XX"、"深度分析 XX"、"综述 XX"、"digest XX"、"全面总结一下 XX"
- 前置检查: 通用前置检查 reference
- 步骤 1: 搜索相关页面 (Grep wiki/, list coverage)
- 步骤 2: 深度阅读所有相关页面 (sources/, entities/, topics/)
- 步骤 3: 生成结构化深度报告 — full markdown template:
  ```
  # {主题} 深度报告
  > 综合自 {N} 篇素材 | 生成日期：{日期}
  ## 背景概述
  ## 核心观点 (with source citations)
  ## 不同视角对比 (table format)
  ## 知识脉络
  ## 尚待解决的问题
  ## 相关页面
  ```
  Save to `wiki/synthesis/{主题}-深度报告.md`
- 步骤 4: 更新 index.md + log.md
- 步骤 5: 向用户展示结果 (zh full example + en note)

Expected: ~80 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-digest/SKILL.md  # expect ~80
grep -c "深度报告" skills/wiki-digest/SKILL.md  # expect >= 2
grep -c "背景概述" skills/wiki-digest/SKILL.md  # expect 1
grep -c "不同视角对比" skills/wiki-digest/SKILL.md  # expect 1
grep -c "尚待解决的问题" skills/wiki-digest/SKILL.md  # expect 1
git add skills/wiki-digest/
git commit -m "feat: create wiki-digest sub-skill with deep report template"
```

---

### Task 10: Create wiki-graph sub-skill

**Files:**
- Create: `skills/wiki-graph/SKILL.md`
- Source: `SKILL.md` lines 648-702

- [ ] **Step 1: Create wiki-graph/SKILL.md**

Write with frontmatter + shared context reference + full content:

```yaml
---
name: wiki-graph
description: Use when the user asks for a knowledge graph of their llm-wiki, says "知识库图谱", "wiki graph", or "画个知识库关联图". Do NOT trigger for generic "graph" or diagram requests unrelated to the wiki.
---
```

- 触发关键词: "画个知识图谱"、"看看关联图"、"graph"、"知识库地图"、"展示知识关联"
- 前置检查: 通用前置检查 reference
- 步骤 1: 扫描双向链接 (遍历 wiki/*.md, extract [[链接]], build A → B list)
- 步骤 2: 生成 Mermaid 图表文件 `wiki/knowledge-graph.md` — full template:
  ````
  # 知识图谱
  > 自动生成 | {日期} | 共 {N} 个节点，{M} 条关联
  ```mermaid
  graph LR
    A[概念1] --> B[概念2]
  ```
  ````
  生成规则:
  - 节点名用中括号 `[名称]`，名称太长则截断到 10 字
  - 只展示有双向链接关系的节点（孤立节点不纳入图谱）
  - 关系超过 50 条 → 只保留被引用次数最多的 30 个节点
- 步骤 3: 向用户展示结果 (zh full example with 节点数/关联数/查看方式/孤立页面 + en note)

Expected: ~60 lines.

- [ ] **Step 2: Verify and commit**

```bash
wc -l < skills/wiki-graph/SKILL.md  # expect ~60
grep -c "Mermaid" skills/wiki-graph/SKILL.md  # expect >= 1
grep -c "graph LR" skills/wiki-graph/SKILL.md  # expect 1
grep -c "截断到 10 字" skills/wiki-graph/SKILL.md  # expect 1
grep -c "30 个节点" skills/wiki-graph/SKILL.md  # expect 1
git add skills/wiki-graph/
git commit -m "feat: create wiki-graph sub-skill with Mermaid generation rules"
```

---

### Task 11: Content completeness verification

**Files:**
- Read: `SKILL.md` (original 702 lines)
- Read: `skills/_shared/context.md` + all 8 sub-skills

This task verifies that every non-blank, non-frontmatter line from the original SKILL.md has a home.

- [ ] **Step 1: Count total lines across all new files**

```bash
echo "Shared context:"
wc -l < skills/_shared/context.md

echo "Sub-skills:"
total=0
for f in skills/*/SKILL.md; do
  lines=$(wc -l < "$f")
  echo "  $f: $lines"
  total=$((total + lines))
done
echo "Sub-skill total: $total"

echo "Original SKILL.md:"
wc -l < SKILL.md
```

Expected: shared (~120) + sub-skills (~610) + frontmatter/references (~80) = ~810 ≥ 702.

- [ ] **Step 2: Verify key content from each original section exists somewhere**

Check that these distinctive strings from the original SKILL.md appear in the new files:

```bash
# From 外挂状态模型 (should be in _shared/context.md)
grep -rl "not_installed / env_unavailable / runtime_failed / unsupported / empty_result" skills/

# From 通用前置检查 (should be in _shared/context.md)
grep -rl "当前工作目录.*wiki-schema.md" skills/

# From 输出语言规则 (should be in _shared/context.md)
grep -rl "素材 → Source" skills/

# From init 依赖检查 (should be in wiki-init)
grep -rl "setup.sh" skills/wiki-init/

# From ingest Chrome 提示 (should be in wiki-ingest)
grep -rl "lsof -i :9222" skills/wiki-ingest/ || grep -rl "remote-debugging-port=9222" skills/wiki-ingest/

# From ingest 完整处理 output example (should be in wiki-ingest)
grep -rl "已消化：{素材标题}" skills/wiki-ingest/

# From batch-ingest 每5个暂停 (should be in wiki-batch-ingest)
grep -rl "每 5 个" skills/wiki-batch-ingest/

# From lint 报告模板 (should be in wiki-lint)
grep -rl "知识库健康检查报告" skills/wiki-lint/

# From status adapter-state (should be in wiki-status)
grep -rl "adapter-state.sh summary-human" skills/wiki-status/

# From digest 报告模板 (should be in wiki-digest)
grep -rl "尚待解决的问题" skills/wiki-digest/

# From graph Mermaid 规则 (should be in wiki-graph)
grep -rl "截断到 10 字" skills/wiki-graph/
```

All should return matches. If any fails, go back and fix the corresponding sub-skill.

- [ ] **Step 3: Verify all sub-skills reference shared context**

```bash
for f in skills/*/SKILL.md; do
  if ! grep -q "_shared/context.md" "$f"; then
    echo "MISSING shared context reference: $f"
  fi
done
```

Expected: no output (all reference shared context).

- [ ] **Step 4: Verify "不保留" content is correctly excluded**

The design doc says these sections are intentionally not preserved:
- YAML frontmatter (lines 1-10) → each sub-skill has its own
- 快速开始 (lines 28-35) → user guidance, agent doesn't need
- 工作流路由表 (lines 106-123) → replaced by frontmatter descriptions

```bash
# These should NOT appear in any sub-skill
grep -rl "## 快速开始" skills/ && echo "UNEXPECTED: 快速开始 found" || echo "OK: 快速开始 correctly excluded"
grep -rl "## 工作流路由" skills/ && echo "UNEXPECTED: 工作流路由 found" || echo "OK: 工作流路由 correctly excluded"
```

---

### Task 12: Delete root SKILL.md and platforms/

**Files:**
- Delete: `SKILL.md`
- Delete: `platforms/` directory

- [ ] **Step 1: Delete obsolete files**

```bash
git rm SKILL.md
git rm -r platforms/
```

- [ ] **Step 2: Verify**

```bash
test ! -f SKILL.md && echo "OK: SKILL.md removed"
test ! -d platforms && echo "OK: platforms/ removed"
```

- [ ] **Step 3: Commit**

```bash
git commit -m "chore: remove root SKILL.md and platforms/ (replaced by sub-skills)"
```

---

### Task 13: Move karpathy-llm-wiki.md to docs/references/

**Files:**
- Move: `karpathy-llm-wiki.md` → `docs/references/karpathy-llm-wiki.md`

- [ ] **Step 1: Move the file**

```bash
# Check if already moved (from Task 1 backup restore)
if [ -f docs/references/karpathy-llm-wiki.md ] && [ -f karpathy-llm-wiki.md ]; then
  git rm karpathy-llm-wiki.md
elif [ -f karpathy-llm-wiki.md ]; then
  mkdir -p docs/references
  git mv karpathy-llm-wiki.md docs/references/karpathy-llm-wiki.md
fi
```

- [ ] **Step 2: Commit**

```bash
git add docs/references/
git commit -m "chore: move karpathy-llm-wiki.md to docs/references/"
```

---

### Task 14: Rewrite AGENTS.md, update README.md, create CLAUDE.md symlink

**Files:**
- Rewrite: `AGENTS.md`
- Update: `README.md` (keep original content, remove obsolete platforms/ references)
- Create: `CLAUDE.md` → `AGENTS.md` symlink

README.md is the user-facing project README (157 lines at 36fbee3). It stays as a real file — not a symlink. Only CLAUDE.md becomes a symlink to AGENTS.md.

- [ ] **Step 1: Rewrite AGENTS.md**

Rewrite `AGENTS.md` as the unified development guide. It must contain these sections (per design doc):

```markdown
# llm-wiki — 多平台知识库构建 Skill

> 基于 Karpathy 的 llm-wiki 方法论，为 Claude Code、Codex、OpenClaw 这类 agent 提供统一的个人知识库构建系统。

## 前置条件
(Chrome CDP, uv, bun/npm requirements)

## 架构
(8 sub-skills table + shared resources: templates/, scripts/, deps/)
(Note: WIKI_SKILL_ROOT = ~/.agents/skills/llm-wiki/)

## 来源边界
(source-registry.tsv as authority, 3-category table with adapter mappings)
(Must reference: scripts/source-registry.tsv)

## 功能
(Feature list: zero-config init, smart routing, content grading, batch, structured wiki, health check, Obsidian compatible)

## 目录结构
(Wiki directory structure: raw/, wiki/, index.md, log.md, .wiki-schema.md)

## 常见问题
(FAQ: multi-platform, Chrome/CDP failures, WeChat failures)

## 开发约定
(Sub-skill naming, shared context reference convention, WIKI_SKILL_ROOT usage)

## 测试方法
(How to run regression.sh and adapter-state.sh)
```

Key assertions the tests will check on AGENTS.md:
- Contains "## 前置条件", "## 架构", "## 来源边界", "## 常见问题"
- Contains "baoyu-url-to-markdown", "wechat-article-to-markdown", "Chrome CDP", "remote-debugging-port=9222"
- Does NOT contain "x-article-extractor", "baoyu-danger-x-to-markdown", "headless Chromium", "install.sh --platform"
- Contains "scripts/source-registry.tsv", "核心主线", "可选外挂", "手动入口"
- All source labels from registry are present

- [ ] **Step 2: Update README.md**

README.md keeps its original structure but needs these updates:
- Remove "平台入口" links to `platforms/claude/CLAUDE.md`, `platforms/codex/AGENTS.md`, `platforms/openclaw/README.md` (platforms/ is deleted)
- Remove `bash install.sh --platform claude/codex/openclaw` references (install.sh moved)
- Update "安装方式" section to reflect new structure (sub-skills, no more platform-specific install)
- Keep: 前置条件, 来源边界, 功能, 目录结构, 常见问题, 致谢, License sections

Key assertions the tests will check on README.md:
- Contains "## 前置条件", "## 常见问题"
- Contains "wechat-article-to-markdown"
- Does NOT contain "x-article-extractor", "baoyu-danger-x-to-markdown"
- Does NOT contain "install.sh --platform" (removed)
- Contains "scripts/source-registry.tsv", "核心主线", "可选外挂", "手动入口"
- All source labels from registry are present
- Is a regular file (NOT a symlink)

- [ ] **Step 3: Create CLAUDE.md symlink**

```bash
rm -f CLAUDE.md
ln -s AGENTS.md CLAUDE.md
```

- [ ] **Step 4: Verify**

```bash
[ -L CLAUDE.md ] && echo "OK: CLAUDE.md is symlink"
readlink CLAUDE.md  # expect AGENTS.md
[ -f README.md ] && [ ! -L README.md ] && echo "OK: README.md is regular file"
```

- [ ] **Step 5: Commit**

```bash
git add AGENTS.md CLAUDE.md README.md
git commit -m "feat: rewrite AGENTS.md as dev guide, update README.md, symlink CLAUDE.md"
```

---

### Task 15: Update tests/regression.sh

**Files:**
- Rewrite: `tests/regression.sh`
- Source: 36fbee3 version (521 lines) as baseline

This is the most complex task. Start from the 36fbee3 regression.sh and apply changes. Key principles:
- No tests are deleted — all 5 previously "deleted" tests are adapted to check sub-skills instead of root SKILL.md
- 7 new structural checks (test_shared_context_exists, test_all_subskills_exist, etc.) are standalone verification only — run them once after the split to confirm structure, but do NOT add them to regression.sh
- Add 1 new regression test: `test_all_subskills_can_read_shared_context`

- [ ] **Step 1: Identify tests to keep, adapt, and update**

From the 36fbee3 baseline (26 tests + adapter-state.sh call):

**Keep unchanged (12 tests):**
- `test_source_registry_contract_is_frozen`
- `test_source_registry_groups_core_optional_and_manual_sources`
- `test_source_registry_exposes_install_dependency_groups`
- `test_source_registry_validation_passes`
- `test_source_registry_matches_urls_and_files_from_shared_table`
- `test_legacy_wiki_defaults_missing_fields_without_forcing_migration`
- `test_legacy_wiki_lazily_creates_new_source_dirs_without_moving_old_materials`
- `test_english_templates_exist_and_have_placeholders`
- `test_english_templates_have_no_empty_links`
- `test_init_fills_language_placeholder`
- `test_templates_have_no_empty_links`
- `test_schema_template_aligns_source_boundary_to_registry`

**Adapt (5 tests) — remap from root SKILL.md to sub-skills:**

1. `test_skill_md_routes_wechat_to_new_tool` → `test_ingest_routes_wechat_to_new_tool`
   - Was: checks `$REPO_ROOT/SKILL.md` for source-registry.sh match-url/match-file, adapter_name, no x-article-extractor
   - Now: checks `$REPO_ROOT/skills/wiki-ingest/SKILL.md` for the same assertions

2. `test_skill_md_has_shared_preflight_and_language_rules` → `test_shared_context_has_preflight_and_language_rules`
   - Was: checks `$REPO_ROOT/SKILL.md` for 通用前置检查, 输出语言规则, 素材 → Source, 知识图谱 → Knowledge Graph
   - Now: checks `$REPO_ROOT/skills/_shared/context.md` for the same assertions

3. `test_skill_md_uses_external_english_templates_and_no_english_output_blocks` → `test_init_uses_external_english_templates`
   - Was: checks `$REPO_ROOT/SKILL.md` for templates/index-en-template.md etc., no "**English（en）**："
   - Now: checks `$REPO_ROOT/skills/wiki-init/SKILL.md` for templates references (init is the skill that handles English template overwrite)

4. `test_batch_ingest_has_step_two` → adapted to check sub-skill
   - Was: extracts section from root SKILL.md between "## 工作流 3" and "## 工作流 4", checks for 3 numbered steps
   - Now: checks `$REPO_ROOT/skills/wiki-batch-ingest/SKILL.md` directly for the same step content ("列出所有可处理文件", "展示文件列表")

5. `test_skill_status_and_ingest_align_to_registry` → split into sub-skill checks
   - Was: checks `$REPO_ROOT/SKILL.md` for source-registry.sh list/get, source_id, recovery_action, install_hint, 按来源总表, 外挂状态直接使用
   - Now: checks `$REPO_ROOT/skills/wiki-ingest/SKILL.md` for source_id/recovery_action/install_hint AND `$REPO_ROOT/skills/wiki-status/SKILL.md` for source-registry.sh list, 按来源总表, adapter-state.sh summary-human

**Update (11 tests):**
- `test_setup_runs_on_bash_3_2`: `$REPO_ROOT/setup.sh` → `$REPO_ROOT/skills/wiki-init/setup.sh`
- `test_install_dry_run_for_claude`: `$REPO_ROOT/install.sh` → `$REPO_ROOT/skills/wiki-init/install.sh`
- `test_install_auto_refuses_ambiguous_platforms`: same path change
- `test_install_openclaw_copies_bundle`: path change + `SKILL.md` assertion → `skills/wiki-init/SKILL.md`; `install.sh` assertion → `skills/wiki-init/install.sh`
- `test_uv_tool_install_failure_is_graceful`: path change + `SKILL.md` assertion → `skills/wiki-init/SKILL.md`
- `test_install_prints_source_boundary_from_registry`: path change
- `test_install_warns_when_managed_source_is_missing`: path change to `skills/wiki-init/install.sh`
- `test_setup_wrapper_is_marked_deprecated`: path change to `skills/wiki-init/setup.sh`
- `test_readme_sections`: README.md is now a regular file (NOT a symlink). Update assertions:
  - Remove: symlink checks, `readlink` assertions
  - Remove: `bash install.sh --platform claude/codex/openclaw` assertions (install.sh moved)
  - Keep: `assert_file_contains "$REPO_ROOT/README.md" "## 前置条件"`, `"## 常见问题"`, `"wechat-article-to-markdown"`
  - Keep: `assert_file_not_contains` for x-article-extractor, baoyu-danger-x-to-markdown
  - Add: `assert_file_not_contains "$REPO_ROOT/README.md" "install.sh --platform"` (removed from README)
  - Add: `[ ! -L "$REPO_ROOT/README.md" ] || fail "README.md should not be a symlink"`
  - Add: `[ -L "$REPO_ROOT/CLAUDE.md" ] || fail "CLAUDE.md should be a symlink"`
  - Add: `readlink "$REPO_ROOT/CLAUDE.md"` check → AGENTS.md
  - Add: AGENTS.md section checks: `"## 前置条件"`, `"## 架构"`, `"## 来源边界"`, `"## 常见问题"`
- `test_readme_aligns_source_boundary_to_registry`: keep checking README.md (it's still a real file with source boundary content). Also add parallel checks on AGENTS.md.
- `test_skill_status_and_ingest_align_to_registry`: (covered in "Adapt" above)

**Add (1 new regression test):**
- `test_all_subskills_can_read_shared_context`: for each of the 8 sub-skills, verify that the `_shared/context.md` path referenced in the sub-skill actually resolves to an existing file. Implementation:
  ```bash
  test_all_subskills_can_read_shared_context() {
      local f ref_path
      for f in "$REPO_ROOT"/skills/*/SKILL.md; do
          grep -q "_shared/context.md" "$f" \
              || fail "$f does not reference shared context"
      done
      assert_path_exists "$REPO_ROOT/skills/_shared/context.md"
      # Verify shared context has the key sections agents need at runtime
      assert_file_contains "$REPO_ROOT/skills/_shared/context.md" "外挂状态模型"
      assert_file_contains "$REPO_ROOT/skills/_shared/context.md" "通用前置检查"
      assert_file_contains "$REPO_ROOT/skills/_shared/context.md" "输出语言规则"
  }
  ```

- [ ] **Step 2: Write the updated regression.sh**

Start from the 36fbee3 version. Apply all changes listed above. The test execution order at the bottom should list all surviving + adapted + new tests (total: 12 kept + 5 adapted + 11 updated + 1 new = 29 tests + adapter-state.sh call).

Key implementation details for adapted tests:

```bash
# Adapted from test_skill_md_routes_wechat_to_new_tool
test_ingest_routes_wechat_to_new_tool() {
    local file="$REPO_ROOT/skills/wiki-ingest/SKILL.md"
    assert_file_contains "$file" "source-registry.sh match-url"
    assert_file_contains "$file" "source-registry.sh match-file"
    assert_file_contains "$file" 'adapter_name'
    assert_file_not_contains "$file" "x-article-extractor"
}

# Adapted from test_skill_md_has_shared_preflight_and_language_rules
test_shared_context_has_preflight_and_language_rules() {
    local file="$REPO_ROOT/skills/_shared/context.md"
    assert_file_contains "$file" "## 通用前置检查"
    assert_file_contains "$file" "## 输出语言规则"
    assert_file_contains "$file" "素材 → Source"
    assert_file_contains "$file" "知识图谱 → Knowledge Graph"
}

# Adapted from test_skill_md_uses_external_english_templates_and_no_english_output_blocks
test_init_uses_external_english_templates() {
    local file="$REPO_ROOT/skills/wiki-init/SKILL.md"
    assert_file_contains "$file" "templates/index-en-template.md"
    assert_file_contains "$file" "templates/overview-en-template.md"
    assert_file_contains "$file" "templates/log-en-template.md"
}

# Adapted from test_batch_ingest_has_step_two
test_batch_ingest_has_steps() {
    local file="$REPO_ROOT/skills/wiki-batch-ingest/SKILL.md"
    assert_file_contains "$file" "列出所有可处理文件"
    assert_file_contains "$file" "展示文件列表"
    assert_file_contains "$file" "每 5 个"
}

# Adapted from test_skill_status_and_ingest_align_to_registry
test_status_and_ingest_subskills_align_to_registry() {
    local ingest="$REPO_ROOT/skills/wiki-ingest/SKILL.md"
    local status="$REPO_ROOT/skills/wiki-status/SKILL.md"
    assert_file_contains "$ingest" "source-registry.sh"
    assert_file_contains "$ingest" "source_id"
    assert_file_contains "$ingest" "recovery_action"
    assert_file_contains "$ingest" "install_hint"
    assert_file_contains "$status" "source-registry.sh list"
    assert_file_contains "$status" '按来源总表中的'
    assert_file_contains "$status" "adapter-state.sh summary-human"
}
```

Key implementation details for updated install/setup tests:
- All `$REPO_ROOT/install.sh` → `$REPO_ROOT/skills/wiki-init/install.sh`
- All `$REPO_ROOT/setup.sh` → `$REPO_ROOT/skills/wiki-init/setup.sh`
- `test_install_openclaw_copies_bundle`: `assert_path_exists "$tmp_dir/home/.openclaw/skills/llm-wiki/SKILL.md"` → `assert_path_exists "$tmp_dir/home/.openclaw/skills/llm-wiki/skills/wiki-init/SKILL.md"`; `"$tmp_dir/home/.openclaw/skills/llm-wiki/install.sh"` → `"$tmp_dir/home/.openclaw/skills/llm-wiki/skills/wiki-init/install.sh"`
- `test_uv_tool_install_failure_is_graceful`: `assert_path_exists "$tmp_dir/home/.claude/skills/llm-wiki/SKILL.md"` → `assert_path_exists "$tmp_dir/home/.claude/skills/llm-wiki/skills/wiki-init/SKILL.md"`

- [ ] **Step 3: Run regression tests**

```bash
bash tests/regression.sh
```

Expected: "All regression checks passed."

- [ ] **Step 4: Commit**

```bash
git add tests/regression.sh
git commit -m "test: update regression.sh for v2 split (adapt 5, update 11, add 1 shared-context test)"
```

---

### Task 16: Update tests/adapter-state.sh

**Files:**
- Modify: `tests/adapter-state.sh`

The adapter-state.sh tests reference root SKILL.md and install.sh in some places. Update those paths.

- [ ] **Step 1: Identify and update path references**

Search for references to root-level files:

```bash
grep -n "SKILL.md\|install.sh\|setup.sh" tests/adapter-state.sh
```

Update any references:
- `$REPO_ROOT/install.sh` → `$REPO_ROOT/skills/wiki-init/install.sh`
- `$REPO_ROOT/SKILL.md` → check if it should reference a sub-skill instead
- `$REPO_ROOT/setup.sh` → `$REPO_ROOT/skills/wiki-init/setup.sh`

Note: Most adapter-state.sh tests use `$REPO_ROOT/scripts/adapter-state.sh` which doesn't change. Only fix tests that reference the moved files.

- [ ] **Step 2: Run adapter-state tests**

```bash
bash tests/adapter-state.sh
```

Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git add tests/adapter-state.sh
git commit -m "test: update adapter-state.sh paths for v2 split"
```

---

### Task 17: Final verification

**Files:** None (read-only verification)

- [ ] **Step 1: Run full regression suite**

```bash
bash tests/regression.sh
```

Expected: "All regression checks passed."

- [ ] **Step 2: Run adapter-state tests**

```bash
bash tests/adapter-state.sh
```

Expected: all pass.

- [ ] **Step 3: Run standalone structural checks (not in regression.sh)**

These 7 checks verify the v2 split structure. They are one-time validation, not permanent regression tests:

```bash
echo "=== test_shared_context_exists ==="
test -f skills/_shared/context.md || echo "FAIL"
grep -q "外挂状态模型" skills/_shared/context.md || echo "FAIL: missing 外挂状态模型"
grep -q "通用前置检查" skills/_shared/context.md || echo "FAIL: missing 通用前置检查"
grep -q "输出语言规则" skills/_shared/context.md || echo "FAIL: missing 输出语言规则"
grep -q "术语对照" skills/_shared/context.md || echo "FAIL: missing 术语对照"
echo "OK"

echo "=== test_all_subskills_exist ==="
for s in wiki-init wiki-ingest wiki-batch-ingest wiki-query wiki-digest wiki-lint wiki-status wiki-graph; do
  test -f "skills/$s/SKILL.md" || echo "FAIL: missing skills/$s/SKILL.md"
done
echo "OK"

echo "=== test_all_subskills_reference_shared_context ==="
for f in skills/*/SKILL.md; do
  grep -q "_shared/context.md" "$f" || echo "FAIL: $f missing shared context ref"
done
echo "OK"

echo "=== test_subskill_frontmatter_has_name ==="
for f in skills/*/SKILL.md; do
  grep -q '^name:' "$f" || echo "FAIL: $f missing name:"
  grep -q '^description:' "$f" || echo "FAIL: $f missing description:"
done
echo "OK"

echo "=== test_subskill_frontmatter_requires_wiki_intent ==="
for f in skills/*/SKILL.md; do
  desc="$(sed -n '/^description:/,/^---$/p' "$f" | head -5)"
  echo "$desc" | grep -qE 'directly gives a link|关于 XX|XX 是什么' && echo "FAIL: $f has overly generic trigger"
done
echo "OK"

echo "=== test_legacy_entrypoints_removed ==="
test ! -f SKILL.md || echo "FAIL: root SKILL.md exists"
test ! -d platforms || echo "FAIL: platforms/ exists"
test ! -f install.sh || echo "FAIL: root install.sh exists"
test ! -f setup.sh || echo "FAIL: root setup.sh exists"
test -f skills/wiki-init/install.sh || echo "FAIL: install.sh not in wiki-init"
test -f skills/wiki-init/setup.sh || echo "FAIL: setup.sh not in wiki-init"
echo "OK"

echo "=== test_karpathy_reference_moved ==="
test -f docs/references/karpathy-llm-wiki.md || echo "FAIL"
echo "OK"
```

- [ ] **Step 4: Verify file structure matches design**

```bash
echo "=== Sub-skills ==="
ls -la skills/*/SKILL.md
echo ""
echo "=== Shared context ==="
ls -la skills/_shared/context.md
echo ""
echo "=== wiki-init extras ==="
ls -la skills/wiki-init/install.sh skills/wiki-init/setup.sh
echo ""
echo "=== Entry points ==="
ls -la CLAUDE.md README.md AGENTS.md
echo ""
echo "=== Removed ==="
test ! -f SKILL.md && echo "OK: no root SKILL.md"
test ! -d platforms && echo "OK: no platforms/"
test ! -f install.sh && echo "OK: no root install.sh"
test ! -f setup.sh && echo "OK: no root setup.sh"
echo ""
echo "=== Moved ==="
test -f docs/references/karpathy-llm-wiki.md && echo "OK: karpathy ref moved"
```

- [ ] **Step 5: Line count verification**

```bash
shared=$(wc -l < skills/_shared/context.md)
subskill_total=0
for f in skills/*/SKILL.md; do
  lines=$(wc -l < "$f")
  subskill_total=$((subskill_total + lines))
done
echo "Shared: $shared"
echo "Sub-skills total: $subskill_total"
echo "Grand total: $((shared + subskill_total))"
echo "Original SKILL.md was: 702 lines"
echo "Target: >= 702"
```

- [ ] **Step 6: Final commit (if any remaining changes)**

```bash
git status
# If clean, no commit needed
# If changes remain, stage and commit
```
