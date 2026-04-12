---
name: wiki-query
description: Use when the user explicitly asks to query their llm-wiki knowledge base, says "查询知识库", "从 wiki 里找", "wiki query", or "知识库里有没有". Do NOT trigger for general questions unrelated to the wiki.
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：query（查询知识库）

### 步骤

1. **确认知识库路径**：
   - 执行**通用前置检查**（见共享上下文），获取知识库根路径和 `WIKI_LANG`
   - 如果没有可用知识库，提示用户先初始化

2. **读取 index.md** 了解知识库全貌

3. **搜索相关页面**：
   - 先在 index.md 中定位相关分类和条目
   - 再用 Grep 在 `wiki/` 目录下搜索关键词
   - 读取最相关的 3-5 个页面

4. **综合回答**：
   - 按 `WIKI_LANG` 用对应语言回答用户的问题
   - 标注信息来源（引用 wiki 页面，用 `[[页面名]]` 格式）
   - 如果多个素材有不同观点，分别列出并标注来源

5. **建议回写**：
   - 如果这个回答产生了有价值的分析或综合，询问用户是否保存为新的 wiki 页面
   - 如果保存 → 创建新页面（`wiki/synthesis/` 或 `wiki/comparisons/`），更新 index.md 和 log.md
