# llm-wiki Skill 拆分重做 — 设计文档 v2

> 日期：2026-04-12（替代 2026-04-11 v1）

## 背景

v1 拆分（commit 36fbee3）将 702 行的根 SKILL.md 拆为 8 个子 skill，但子 skill 加总只有 315 行，丢失了 55% 的内容。丢失的不是冗余内容，而是实质性的工作流指导：

- 所有用户输出示例（zh/en）
- 输出语言规则 + 术语对照表
- 依赖检查 + setup.sh 引用
- 外挂状态模型（仅 wiki-ingest 保留，其他 7 个 skill 丢失）
- markdown 模板引用（source-template、entity-template 等）
- wiki-digest 的报告模板、wiki-graph 的 Mermaid 生成规则
- 回归测试从 ~26 个缩减到 ~10 个

同时 Codex adversarial review 指出了 4 个问题，其中 frontmatter 收窄和 digest 手动触发是符合预期的设计决策，但内容丢失和测试缩水需要修复。

## 目标

- 从 36fbee3 的原始 SKILL.md 重新拆分，确保内容 100% 覆盖
- 共享内容抽取到 `skills/_shared/context.md`，子 skill 通过引用获取
- 每个子 skill 包含完整的工作流内容（步骤、输出示例、模板引用）
- 保留 source-registry / adapter-state / legacy 兼容测试
- frontmatter 复用 v1 拆分的版本（已收窄，符合预期）

## 策略

混合策略：reset working tree 回到 36fbee3，拿回完整的原始 SKILL.md，然后参考 v1 的子 skill 目录结构重新拆分。

## 文件结构

```
skills/
├── _shared/
│   └── context.md          # 共享上下文（~120行）
├── wiki-init/SKILL.md
├── wiki-ingest/SKILL.md
├── wiki-batch-ingest/SKILL.md
├── wiki-query/SKILL.md
├── wiki-lint/SKILL.md
├── wiki-status/SKILL.md
├── wiki-digest/SKILL.md
└── wiki-graph/SKILL.md
```

## 内容分配方案

原始 SKILL.md 的每一行都必须有明确去向，不允许默认丢弃。

### → `skills/_shared/context.md`（约 120 行）

从原始 SKILL.md 搬运以下段落：

| 原始段落 | 原始行范围 | 说明 |
|---|---|---|
| H1 标题 + tagline | 12-14 | `# llm-wiki — 个人知识库构建系统` + 引言，作为共享上下文的标题 |
| "这个 skill 做什么" + 核心理念 | 16-27 | 整体定位，运行时 agent 需要理解 |
| Script Directory + Path Resolution | 37-45 | `WIKI_SKILL_ROOT` = `~/.agents/skills/llm-wiki/` |
| 外挂状态模型 | 62-104 | 5 种状态 + adapter-state.sh check/classify-run |
| 通用前置检查 | 125-139 | CWD → ~/.llm-wiki-path → .wiki-schema.md → WIKI_LANG |
| 输出语言规则 + 术语对照 | 141-160 | WIKI_LANG 切换规则 + 中英术语表 |

关键变更：
- `WIKI_SKILL_ROOT` 不再用相对路径推导，固定为 `~/.agents/skills/llm-wiki/`。需同步更新 AGENTS.md 中的相关描述。
- 子 skill 不再各自包含 Path Resolution 段落，统一由 `_shared/context.md` 提供。子 skill 只保留引用行。

### → 各子 skill（完整搬运对应工作流段落）

每个子 skill 开头加引用行：
```
> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`
```

| 子 skill | 原始段落 | 原始行范围 | 预估行数 | 关键内容 |
|---|---|---|---|---|
| wiki-init | 工作流 1 + init 前置检查 + 依赖检查 | 47-60, 162-221 | ~85 | 依赖检查（setup.sh + install.sh + 3 个依赖列表）、询问主题/语言/路径、init-wiki.sh、语言配置、输出引导（zh/en） |
| wiki-ingest | 工作流 2（素材路由 + 分级处理 + 完整/简化流程） | 223-367 | ~155 | source-registry.sh 路由、Chrome 提示、内容分级、完整处理 9 步、简化处理 6 步、输出示例（zh/en） |
| wiki-batch-ingest | 工作流 3 | 369-431 | ~70 | 文件列表确认、逐个 ingest、每 5 个暂停、完成报告（zh/en） |
| wiki-query | 工作流 4 | 433-453 | ~30 | 读 index、Grep 搜索、综合回答、建议回写 |
| wiki-lint | 工作流 5 | 455-516 | ~70 | 触发时机、检查范围、5 项检查（孤立页面/断链/矛盾/交叉引用/index 一致性）、报告模板（zh/en） |
| wiki-status | 工作流 6 | 518-571 | ~60 | source-registry.sh list、按来源统计、adapter-state.sh summary-human、报告模板（zh/en） |
| wiki-digest | 工作流 7 | 573-646 | ~80 | 触发关键词、搜索相关页面、深度报告 markdown 模板、输出示例（zh/en） |
| wiki-graph | 工作流 8 | 648-702 | ~60 | 触发关键词、扫描双向链接、Mermaid 生成规则（节点命名/关系上限）、输出示例（zh/en） |

### → 不进入子 skill 的内容

| 原始段落 | 原始行范围 | 去向 |
|---|---|---|
| YAML frontmatter（`---` 块） | 1-10 | 各子 skill 各自定义（复用 v1 版本） |
| 快速开始 | 28-35 | 不保留（用户引导语，agent 不需要） |
| 工作流路由表 | 106-123 | 不保留（各子 skill 的 frontmatter description 承担路由） |

### 预估总量

共享 ~120 + 子 skill 合计 ~610 + frontmatter/引用行 ~80 = ~810 行 > 原始 702 行 ✓

## Frontmatter（复用 v1）

各子 skill 的 frontmatter 保持 v1 拆分后的版本不变：

```yaml
# wiki-init
name: wiki-init
description: Use when the user wants to create or initialize a new personal knowledge base, says "初始化知识库", "新建 wiki", or "创建知识库"

# wiki-ingest
name: wiki-ingest
description: Use when the user explicitly asks to add material to their llm-wiki knowledge base, says "消化这篇到知识库", "添加素材", "整理到 wiki", or "wiki ingest". Do NOT trigger for generic URL sharing, summarization, or file reading without wiki intent.

# wiki-batch-ingest
name: wiki-batch-ingest
description: Use when the user explicitly asks to batch-add materials to their llm-wiki knowledge base, says "批量消化到知识库", "wiki batch ingest", or "把这些都整理到 wiki". Do NOT trigger for generic folder operations without wiki intent.

# wiki-query
name: wiki-query
description: Use when the user explicitly asks to query their llm-wiki knowledge base, says "查询知识库", "从 wiki 里找", "wiki query", or "知识库里有没有". Do NOT trigger for general questions unrelated to the wiki.

# wiki-lint
name: wiki-lint
description: Use when the user wants a health check on their wiki knowledge base, says "检查知识库", "健康检查", or "wiki lint". Do NOT trigger for generic linting requests (ESLint, pylint, etc.).

# wiki-status
name: wiki-status
description: Use when the user asks about their wiki knowledge base status, says "知识库状态", "wiki 有什么", or "知识库有多少素材". Do NOT trigger for generic status queries unrelated to the wiki.

# wiki-digest
name: wiki-digest
description: Use ONLY when the user explicitly invokes this skill via "wiki digest", "/wiki-digest", or "知识库 digest". This is a manual-only workflow — never auto-trigger from conversational phrases like "给我讲讲" or "总结一下".

# wiki-graph
name: wiki-graph
description: Use when the user asks for a knowledge graph of their llm-wiki, says "知识库图谱", "wiki graph", or "画个知识库关联图". Do NOT trigger for generic "graph" or diagram requests unrelated to the wiki.
```

## 其他清理项

### 1. 删除根 SKILL.md

拆分完成后删除。工作流路由表不再需要，各子 skill 的 frontmatter description 承担路由。

### 2. 删除 platforms/ 目录

三个平台入口文件过时，Skill 加载时不会自动读取。

### 3. install.sh / setup.sh 移入 wiki-init

install.sh 和 setup.sh 保留，移动到 `skills/wiki-init/` 目录下。依赖检查逻辑（原始 SKILL.md 47-60 行）也移入 wiki-init/SKILL.md。wiki-init 负责首次使用时的环境准备。

### 4. 统一仓库入口文件

`AGENTS.md` 作为唯一源文件（仓库开发指南），包含：
- 目录结构说明
- 子 skill 列表和职责
- 共享资源位置（scripts/、templates/、deps/）
- 开发约定
- 测试方法
- 前置条件和来源边界表
- 常见问题

注意：AGENTS.md 是给开发 skill 的 coding agent 看的，运行时调用 skill 时不会被读取。

`CLAUDE.md` → `AGENTS.md` 软链接
`README.md` → `AGENTS.md` 软链接

### 5. 移动 karpathy-llm-wiki.md

```bash
mkdir -p docs/references
git mv karpathy-llm-wiki.md docs/references/
```

## 测试策略

regression.sh 将从 36fbee3 版本出发修改（不是从 v1 拆分后的版本），确保每个测试都有明确的保留/删除/更新/新增状态。

### 保留的测试（从 36fbee3 的 regression.sh，不做修改）

| 测试类别 | 测试名 | 理由 |
|---|---|---|
| source-registry 契约 | test_source_registry_contract_is_frozen | 验证 scripts/ 的字段契约，和拆分无关 |
| source-registry 契约 | test_source_registry_groups_core_optional_and_manual_sources | 分类验证 |
| source-registry 契约 | test_source_registry_exposes_install_dependency_groups | 依赖分组 |
| source-registry 契约 | test_source_registry_validation_passes | 校验通过 |
| source-registry 契约 | test_source_registry_matches_urls_and_files_from_shared_table | URL/文件匹配 |
| legacy 兼容 | test_legacy_wiki_defaults_missing_fields_without_forcing_migration | 向后兼容 |
| legacy 兼容 | test_legacy_wiki_lazily_creates_new_source_dirs_without_moving_old_materials | 懒创建 |
| 模板 | test_english_templates_exist_and_have_placeholders | 英文模板完整性 |
| 模板 | test_english_templates_have_no_empty_links | 模板质量 |
| init | test_init_fills_language_placeholder | init 脚本 |
| adapter-state | 从 regression.sh 调用 `bash tests/adapter-state.sh` | 适配器状态测试（独立文件，由 regression.sh 调用）。注意：adapter-state.sh 内部引用根 SKILL.md 和 install.sh 的测试需要更新路径（见下方更新表） |

### 删除的测试

| 测试名 | 理由 |
|---|---|
| test_route_table_references_match_subskills | 根 SKILL.md 已删除（被 test_all_subskills_exist 替代） |
| test_skill_md_routes_wechat_to_new_tool | 根 SKILL.md 已删除 |
| test_skill_md_has_shared_preflight_and_language_rules | 根 SKILL.md 已删除（被 test_shared_context_exists 替代） |
| test_skill_md_uses_external_english_templates_and_no_english_output_blocks | 根 SKILL.md 已删除 |
| test_batch_ingest_has_step_two | 根 SKILL.md 已删除（内容移入子 skill） |

### 更新的测试

| 测试名 | 变更 |
|---|---|
| test_setup_runs_on_bash_3_2 | 更新路径：setup.sh 移到 skills/wiki-init/ |
| test_install_dry_run_for_claude | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_install_auto_refuses_ambiguous_platforms | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_install_openclaw_copies_bundle | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_install_warns_when_managed_source_is_missing | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_install_prints_source_boundary_from_registry | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_uv_tool_install_failure_is_graceful | 更新路径：install.sh 移到 skills/wiki-init/ |
| test_setup_wrapper_is_marked_deprecated | 更新路径：setup.sh 移到 skills/wiki-init/ |
| test_readme_sections | 更新断言（README 现在是 AGENTS.md 软链接） |
| test_readme_aligns_source_boundary_to_registry | 改为检查 AGENTS.md 中的来源边界表 |
| test_schema_template_aligns_source_boundary_to_registry | 保留，不需要路径更新（templates/ 不变） |
| test_skill_status_and_ingest_align_to_registry | 改为检查 wiki-status 和 wiki-ingest 子 skill |
| tests/adapter-state.sh 内部测试 | 改写引用根 SKILL.md 的测试（→ 改为检查子 skill）；改写引用 install.sh 的测试（→ 更新路径到 skills/wiki-init/） |

### 新增的测试

| 测试名 | 验证内容 |
|---|---|
| test_shared_context_exists | `_shared/context.md` 存在且包含关键段落标记（外挂状态模型、通用前置检查、输出语言规则、术语对照表） |
| test_all_subskills_exist | 8 个子 skill 目录都有 SKILL.md |
| test_all_subskills_reference_shared_context | 每个子 skill 都包含 `_shared/context.md` 引用行 |
| test_subskill_frontmatter_has_name | 每个子 skill 的 frontmatter 有 name 字段 |
| test_subskill_frontmatter_requires_wiki_intent | frontmatter description 包含 wiki 相关关键词 |
| test_legacy_entrypoints_removed | 根 SKILL.md、platforms/ 不存在；install.sh 和 setup.sh 已移到 skills/wiki-init/ |
| test_karpathy_reference_moved | docs/references/karpathy-llm-wiki.md 存在 |

## 执行顺序

1. 备份需要保留的文件到 /tmp：
   - `docs/superpowers/specs/2026-04-11-skill-split-cleanup-design.md`（本设计文档）
   - `docs/references/karpathy-llm-wiki.md`（如果已存在）
   - 注意：AGENTS.md、CLAUDE.md、README.md、tests/regression.sh 不需要备份，因为它们会在后续步骤中从 36fbee3 基础上重新修改/创建
2. `git checkout -- .` 回到 36fbee3 干净状态（所有 working tree 修改被还原）
3. 恢复备份的文件
4. 创建 `skills/_shared/context.md`，从原始 SKILL.md 搬运共享内容（含改写的依赖检查段落和默认路由规则）
5. 创建 8 个子 skill 目录，每个包含：frontmatter（复用 v1）+ 共享上下文引用 + 完整工作流内容
6. 内容完整性验证：对比原始 SKILL.md，确认每个非空行都已分配到 _shared/context.md 或某个子 skill
7. 删除根 SKILL.md、platforms/；移动 install.sh 和 setup.sh 到 skills/wiki-init/
8. 更新 AGENTS.md 为开发指南（含 WIKI_SKILL_ROOT 固定路径说明）
9. 创建 CLAUDE.md / README.md → AGENTS.md 软链接
10. 移动 karpathy-llm-wiki.md 到 docs/references/
11. 更新 tests/regression.sh（从 36fbee3 版本出发修改，按测试策略表操作）
12. 运行 `bash tests/regression.sh` 验证
13. 运行 `bash tests/adapter-state.sh` 验证

## 验证清单

- [ ] 原始 SKILL.md 每一行都有明确去向（共享上下文 / 子 skill / 不保留并注明理由）
- [ ] 子 skill 加总行数 ≥ 原始 702 行
- [ ] 所有子 skill 引用共享上下文
- [ ] 所有子 skill 包含完整的输出示例（zh/en）
- [ ] `_shared/context.md` 包含：外挂状态模型、通用前置检查、输出语言规则、术语对照表
- [ ] wiki-init 包含：依赖检查、install.sh、setup.sh
- [ ] install.sh / setup.sh 相关测试已更新路径到 skills/wiki-init/
- [ ] source-registry / adapter-state / legacy 兼容测试保留
- [ ] AGENTS.md 中 WIKI_SKILL_ROOT 描述已更新为固定路径
- [ ] `bash tests/regression.sh` 全部通过
- [ ] `bash tests/adapter-state.sh` 全部通过

## 不变的部分

- `scripts/`、`templates/`、`deps/` 目录结构不变
- 子 skill 的 frontmatter（name + description）不变（复用 v1）
- `CHANGELOG.md`、`PLAN.md`、`TODOS.md` 不动
