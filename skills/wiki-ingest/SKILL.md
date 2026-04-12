---
name: wiki-ingest
description: Use when the user explicitly asks to add material to their llm-wiki knowledge base, says "消化这篇到知识库", "添加素材", "整理到 wiki", or "wiki ingest". Do NOT trigger for generic URL sharing, summarization, or file reading without wiki intent.
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：ingest（消化素材）

这是最核心的工作流。用户给一个素材进来，AI 做所有的整理工作。

### 前置检查

执行**通用前置检查**（见共享上下文）。

### 素材提取路由

根据素材类型自动路由到最佳提取方式：

**外挂前置判断**：

- URL 先调用 `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh match-url "<url>"`
- 本地文件先调用 `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh match-file "<path>"`
- 纯文本粘贴直接调用 `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh get plain_text`
- `source-registry.sh` 返回 10 列：`source_id`、`source_label`、`source_category`、`input_mode`、`match_rule`、`raw_dir`、`adapter_name`、`dependency_name`、`dependency_type`、`fallback_hint`
- 调用 `bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh check <source_id>`
- 从 `adapter-state.sh check` 的 8 列结果里读取 `state`、`detail`、`recovery_action`、`install_hint`、`fallback_hint`
- 如果 `state=not_installed` / `env_unavailable` / `unsupported` → 不调用外挂，直接按 `detail`、`recovery_action`、`install_hint`、`fallback_hint` 告诉用户下一步
- 只有返回 `available` 时，才继续自动提取

**URL 类素材**（统一走来源总表，不手写域名表）：

> **Chrome 提示**（仅当 `adapter_name=baoyu-url-to-markdown` 时）：
> adapter-state.sh check 已通过 `lsof -i :9222 -sTCP:LISTEN` 确认 Chrome 调试端口状态。
> 如果 check 返回 `env_unavailable`，直接按 `fallback_hint` 引导用户，不要自行检测 Chrome。
> 如果 check 返回 `available`，正常调用外挂。baoyu-url-to-markdown 会自己处理 Chrome 启动，**继续执行，不要等待用户确认**。
> 如果提取仍然失败，提示用户：`open -na "Google Chrome" --args --remote-debugging-port=9222`

- 如果 `source_category=manual_only` → 不调用外挂，直接使用 `fallback_hint`
- 如果 `adapter_name=wechat-article-to-markdown` → 执行 `wechat-article-to-markdown "<URL>"`
- 如果 `adapter_name=youtube-transcript` → 调用 `youtube-transcript`
- 如果 `adapter_name=baoyu-url-to-markdown` → 调用 `baoyu-url-to-markdown`

**本地文件**：
- 统一走 `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh match-file "<path>"`
- 命中后直接读取，不调用外挂

**纯文本粘贴**：
- 统一视为 `plain_text`
- 直接使用用户提供的文本

**统一回退规则**：

- 对自动提取结果，统一运行 `bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh classify-run <source_id> <exit_code> <output_path>`
- 从 `classify-run` 返回的 8 列结果里读取 `state`、`detail`、`recovery_action`、`fallback_hint`
- 如果返回 `runtime_failed` → 按 `detail`、`recovery_action`、`fallback_hint` 告诉用户"这次自动提取失败，可以先重试一次；如果还不行，就改走手动入口"
- 如果返回 `empty_result` → 按 `detail`、`recovery_action`、`fallback_hint` 告诉用户"自动提取没有拿到有效正文，请手动补全文本后继续"
- 其他状态也使用同一份返回结果，不再手写第二套回退文案

### 内容分级处理

根据素材长度和信息密度自动选择处理级别：

**判断标准**：
- 素材内容 > 1000 字 → **完整处理**
- 素材内容 <= 1000 字（短推文、小红书笔记等）→ **简化处理**

### 完整处理流程（长素材 > 1000 字）

1. **提取素材内容**：按上面的路由获取素材文本

2. **保存原始素材**到 `raw/` 对应目录：
   - 根据素材类型保存到对应目录（articles/、tweets/、wechat/、xiaohongshu/、zhihu/ 等）
   - 文件名格式：`{日期}-{短标题}.md`
   - 如果是 URL 类素材，在文件头部记录原始 URL

3. **阅读素材，提取关键信息**：
   - 核心观点（3-5 个要点）
   - 关键概念（3-5 个，需要创建或更新的实体）
   - 与已有素材的关联

4. **生成素材摘要页**（`wiki/sources/{日期}-{短标题}.md`）：
   - 参考 `templates/source-template.md` 的格式
   - 包含：基本信息、核心观点、关键概念、与其他素材的关联、原文精彩摘录

5. **更新或创建实体页**（`wiki/entities/`）：
   - 对每个关键概念，检查 `wiki/entities/` 下是否已有对应页面
   - 如果已有 → 追加新信息，更新"不同素材中的观点"部分
   - 如果没有 → 创建新实体页，参考 `templates/entity-template.md`
   - 使用 `[[实体名]]` 语法做双向链接

6. **更新或创建主题页**（`wiki/topics/`）：
   - 识别素材涉及的主要研究主题
   - 如果已有对应主题页 → 更新素材汇总表和核心观点
   - 如果没有 → 创建新主题页，参考 `templates/topic-template.md`

7. **更新 index.md**：
   - 在对应分类下添加新条目
   - 更新概览统计数字

8. **更新 log.md**：
   - 追加格式：`## {日期} ingest | {素材标题}`
   - 记录新增和更新的页面列表

9. **向用户展示结果**（按 `WIKI_LANG` 切换语言）：

   **中文（zh）**：
   ```
   已消化：{素材标题}

   新增页面：
   - {素材摘要页}
   - {新实体页1}
   - {新主题页1}

   更新页面：
   - {已有实体页2}（追加了新信息）

   发现关联：
   - 这篇素材和 [[已有素材]] 在 {某概念} 上有联系
   ```
   （英文版按「输出语言规则」生成，结构相同。）

### 简化处理流程（短素材 <= 1000 字）

适用于短推文、小红书笔记、简短评论等。

1. **保存原始素材**到对应 `raw/` 目录
2. **生成简化摘要页**（`wiki/sources/`）：
   - 只包含基本信息和核心观点
   - 不写"原文精彩摘录"部分
3. **提取 1-3 个关键概念**：
   - 如果对应实体页已存在 → 追加一句话说明
   - 如果不存在 → 在摘要页中用 `[待创建: [[概念名]]]` 标记
4. **更新 index.md 和 log.md**
5. **跳过**：主题页创建/更新、overview 更新

6. **向用户展示简化结果**（按 `WIKI_LANG` 切换语言）：

   **中文（zh）**：
   ```
   已消化：{素材标题}（短内容，简化处理）

   新增：
   - 素材摘要页

   待完善：
   - [待创建: [[概念名]]]（积累更多素材后整理）
   ```
   （英文版按「输出语言规则」生成，结构相同。）
