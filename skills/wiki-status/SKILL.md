---
name: wiki-status
description: Use when the user asks about their wiki knowledge base status, says "知识库状态", "wiki 有什么", or "知识库有多少素材". Do NOT trigger for generic status queries unrelated to the wiki.
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 工作流：status（查看状态）

### 前置检查

执行**通用前置检查**（见共享上下文）。如果没有可用知识库，提示用户先初始化。

### 步骤

1. 先运行 `bash ${WIKI_SKILL_ROOT}/scripts/source-registry.sh list` 读取来源总表
2. 获取知识库路径（按通用前置检查的 CWD 检查逻辑）
3. 统计：
   - 按来源总表中的 `source_label` 和 `raw_dir` 逐项统计 `raw/` 文件数
   - `wiki/entities/` 下的页面数
   - `wiki/topics/` 下的页面数
   - `wiki/sources/` 下的页面数
   - `wiki/comparisons/` 和 `wiki/synthesis/` 下的页面数
4. 读取 `log.md` 最后 5 条记录
5. 读取 `index.md` 获取主题概览
6. 运行 `bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh summary-human` 获取外挂状态
7. **输出报告**（按 `WIKI_LANG` 切换语言）：

   **zh**：
   ```
   知识库状态：{主题}

   素材分布（按来源总表）：
   - {source_label}：{N}
   - {source_label}：{N}
   ...

   Wiki 页面：{总数} 页
     - 实体页：{N}
     - 主题页：{N}
     - 素材摘要：{N}
     - 对比分析：{N}
     - 综合分析：{N}

   最近活动：
   - {日期} ingest | {素材标题}
   - {日期} ingest | {素材标题}
   ...

   外挂状态：
   {summary-human 原文}

   建议：
   - 你可能想深入了解 {某主题}，已有 {N} 篇相关素材
   - {某实体} 被 {N} 篇素材提到，值得整理成独立页面
   ```
   （英文版按「输出语言规则」生成，结构相同。）

   外挂状态直接使用 `bash ${WIKI_SKILL_ROOT}/scripts/adapter-state.sh summary-human` 的输出，不要自己再重写一套来源清单。