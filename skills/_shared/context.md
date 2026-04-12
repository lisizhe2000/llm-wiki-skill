# llm-wiki — 共享上下文

> 所有子 skill 在执行前先读取本文件。

---

# llm-wiki — 个人知识库构建系统

> 把碎片化的信息变成持续积累、互相链接的知识库。你只需要提供素材，AI 做所有的整理工作。

## 这个 skill 做什么

llm-wiki 帮你构建一个**持续增长的个人知识库**。它不是传统的笔记软件，而是一个让 AI 帮你维护的 wiki 系统：

- 你给素材（链接、文件、文本），AI 提取核心知识并整理成互相链接的 wiki 页面
- 知识库随着每次使用变得越来越丰富，而不是每次重新开始
- 所有内容都是本地 markdown 文件，用 Obsidian 或任何编辑器都能查看

## 核心理念

传统方式（RAG/聊天记录）的问题：每次问问题，AI 都要从头阅读原始文件，没有积累。知识库的价值在于**知识被编译一次，然后持续维护**，而不是每次重新推导。

---

## Script Directory

Scripts located in `scripts/` subdirectory.

**Path Resolution**:
1. `WIKI_SKILL_ROOT` = `~/.agents/skills/llm-wiki/`
2. Script path = `${WIKI_SKILL_ROOT}/scripts/<script-name>`

---

## 外挂状态模型

外挂失败统一分成 `not_installed / env_unavailable / runtime_failed / unsupported / empty_result` 五类。

所有需要枚举来源、读取 `source_label`、`raw_dir`、`adapter_name`、`fallback_hint` 的地方，都先读来源总表：

```bash
bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh list
```

需要拿单个来源的定义时，用：

```bash
bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh get <source_id>
```

对 URL 类来源，先运行：

```bash
bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh check <source_id>
```

`adapter-state.sh check` 返回 8 列：

```text
source_id	source_label	state	state_label	detail	recovery_action	install_hint	fallback_hint
```

- `not_installed`：提示用户可补安装，同时允许改走手动入口
- `env_unavailable`：说明缺少的环境条件，同时允许改走手动入口
- `runtime_failed`：说明本次提取执行失败，允许重试一次，再改走手动入口
- `unsupported`：直接给出手动入口，不尝试自动提取
- `empty_result`：说明自动提取没拿到有效内容，请用户手动补全文本

当自动提取实际执行后，再运行：

```bash
bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh classify-run <source_id> <exit_code> <output_path>
```

用返回的 `detail`、`recovery_action`、`install_hint`、`fallback_hint` 生成提示。核心主线不因外挂失败而中断。

---

## 通用前置检查

除 `init` 外，其他工作流默认先执行这段检查：

1. 先检查**当前工作目录**是否包含 `.wiki-schema.md`
   - 如果包含 → 用当前目录作为知识库根路径
   - 如果不包含 → 回退到读取 `~/.llm-wiki-path`
2. 如果两者都没有：
   - `ingest` / `batch-ingest` → 先运行 `init`
   - `query` / `lint` / `status` / `digest` / `graph` → 提示用户先初始化知识库
3. 读取知识库根目录下的 `.wiki-schema.md`
4. 从 `.wiki-schema.md` 的"语言"字段判断 `WIKI_LANG`
   - `语言：中文` → `WIKI_LANG=zh`
   - `语言：English` → `WIKI_LANG=en`
   - 字段缺失 → 默认 `WIKI_LANG=zh`

## 输出语言规则

所有面向用户的输出和新写入的 wiki 内容，都按 `WIKI_LANG` 生成：

- `WIKI_LANG=zh` → 使用下文中文示例
- `WIKI_LANG=en` → 保持与中文示例相同的结构、信息量和顺序，仅改为自然英文措辞
- 文件路径、wiki 链接、目录名保持现有约定，不因为语言切换而改动

**术语对照**：
- 素材 → Source
- 实体 → Entity
- 主题 → Topic
- 摘要 → Summary
- 综合 → Synthesis
- 消化 → Ingest
- 对比 → Comparison
- 深度报告 → Deep Dive Report
- 知识图谱 → Knowledge Graph
