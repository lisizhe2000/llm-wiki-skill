---
name: wiki-batch-ingest
description: Use when the user explicitly asks to batch-add materials to their llm-wiki knowledge base, says “批量消化到知识库”, “wiki batch ingest”, or “把这些都整理到 wiki”. Do NOT trigger for generic folder operations without wiki intent.
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：batch-ingest（批量消化）

当用户给了一个文件夹路径，或者说”把这些都整理一下”。

### 步骤

1. **确认知识库路径**：
   - 执行**通用前置检查**（见共享上下文），获取知识库根路径和 `WIKI_LANG`

2. **列出所有可处理文件**：
   - 支持的格式：`.md`, `.txt`, `.pdf`, `.html`
   - 忽略：隐藏文件、`.git` 目录、`node_modules` 等

3. **展示文件列表**，确认处理范围（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```
   发现 {N} 个文件待处理：
   1. file1.pdf
   2. file2.md
   3. file3.txt

   预计需要 {N} 轮处理。是否开始？
   ```
   （英文版按「输出语言规则」生成，结构相同。）

4. **逐个处理**：对每个文件执行 ingest 工作流
   - 根据内容长度自动选择完整/简化处理

5. **每 5 个文件后暂停**，展示进度并询问是否继续（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```
   进度：5/{N} 已完成

   本批处理结果：
   - 新增素材摘要：5
   - 新增实体页：3
   - 更新已有页面：7

   继续处理剩余 {M} 个文件？
   ```
   （英文版按「输出语言规则」生成，结构相同。）

6. **全部完成后**：
   - 运行一次 index.md 全量更新
   - 输出总结报告（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```
   批量消化完成！

   处理了 {N} 个文件：
   - 成功：{S}
   - 跳过（内容为空/格式不支持）：{K}
   - 失败：{F}

   新增页面：{total_new}
   更新页面：{total_updated}
   ```
   （英文版按「输出语言规则」生成，结构相同。）
