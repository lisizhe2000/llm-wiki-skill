---
name: wiki-graph
description: Use when the user asks for a knowledge graph of their llm-wiki, says "知识库图谱", "wiki graph", or "画个知识库关联图". Do NOT trigger for generic "graph" or diagram requests unrelated to the wiki.
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：graph（Mermaid 知识图谱）

### 触发关键词

"画个知识图谱"、"看看关联图"、"graph"、"知识库地图"、"展示知识关联"

### 前置检查

执行**通用前置检查**（见共享上下文）。如果没有可用知识库，提示用户先初始化。

1. **扫描双向链接**：
   - 遍历 `wiki/` 下所有 `.md` 文件
   - 提取每个文件中的 `[[链接]]` 语法，建立关系列表：`页面A → 页面B`

2. **生成 Mermaid 图表文件** `wiki/knowledge-graph.md`：
   ````markdown
   # 知识图谱

   > 自动生成 | {日期} | 共 {N} 个节点，{M} 条关联

   ```mermaid
   graph LR
     A[概念1] --> B[概念2]
     A --> C[素材1]
     D[主题1] --> A
     D --> E[概念3]
   ```

   查看方式：用 Typora、VS Code（Markdown Preview Enhanced）、或直接在 GitHub 上查看。
   ````

   **生成规则**：
   - 节点名用中括号 `[名称]`，名称太长则截断到 10 字
   - 只展示有双向链接关系的节点（孤立节点不纳入图谱）
   - 如果关系超过 50 条，只保留被引用次数最多的 30 个节点，避免图谱过于密集

3. **向用户展示结果**（按 `WIKI_LANG` 切换语言）：

   **zh**：
   \`\`\`
   知识图谱已生成！

   共 {N} 个节点，{M} 条关联
   文件：wiki/knowledge-graph.md

   查看方式：
   - Obsidian：直接打开即可渲染
   - VS Code：安装 Markdown Preview Enhanced 插件
   - GitHub：上传后自动渲染
   - Typora：直接打开

   孤立页面（未纳入图谱）：
   - [[某页面]]（建议添加到相关实体页或主题页）
   \`\`\`
   （英文版按「输出语言规则」生成，结构相同。）
