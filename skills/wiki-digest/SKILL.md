---
name: wiki-digest
description: Use ONLY when the user explicitly invokes this skill via "wiki digest", "/wiki-digest", or "知识库 digest". This is a manual-only workflow — never auto-trigger from conversational phrases like "给我讲讲" or "总结一下".
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：digest（深度综合报告）

**区别于 query**：query 是快速问答，不生成新页面；digest 是跨素材深度综合，生成持久化报告。

### 触发关键词

"给我讲讲 XX"、"深度分析 XX"、"综述 XX"、"digest XX"、"全面总结一下 XX"

### 前置检查

执行**通用前置检查**（见共享上下文）。如果没有可用知识库，提示用户先初始化。

1. **搜索相关页面**：
   - 用 Grep 在 `wiki/` 下搜索主题关键词
   - 列出将要综合的页面（让用户了解报告覆盖范围）

2. **深度阅读所有相关页面**：
   - 读取找到的所有相关 wiki 页面（sources/、entities/、topics/）
   - 归纳每个页面的核心观点和来源信息

3. **生成结构化深度报告**，保存到 `wiki/synthesis/{主题}-深度报告.md`（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```markdown
   # {主题} 深度报告

   > 综合自 {N} 篇素材 | 生成日期：{日期}

   ## 背景概述
   （简要说明这个主题的背景和重要性）

   ## 核心观点
   （按重要性排列，每个观点标注来源）
   - 观点一（来源：[[素材A]]、[[素材B]]）
   - 观点二（来源：[[素材C]]）

   ## 不同视角对比
   （如有多个素材观点不同，在此对比）
   | 维度 | 来源A的观点 | 来源B的观点 |
   |------|------------|------------|

   ## 知识脉络
   （按时间或逻辑顺序梳理该主题的发展）

   ## 尚待解决的问题
   （现有素材中尚未回答的问题，可作为下次搜集素材的方向）

   ## 相关页面
   （列出所有综合来源的链接）
   ```
   （英文版按「输出语言规则」生成，结构相同。）

4. **更新 index.md 和 log.md**：
   - index.md 的"综合分析"分类下添加新报告条目
   - log.md 追加：`## {日期} digest | {主题}`

5. **向用户展示结果**（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```
   已生成深度报告：{主题}

   综合了 {N} 篇素材：
   - [[素材1]]、[[素材2]]...

   报告已保存：wiki/synthesis/{主题}-深度报告.md

   发现这些待解决问题，可以继续搜集素材：
   - {问题1}
   - {问题2}
   ```
   （英文版按「输出语言规则」生成，结构相同。）