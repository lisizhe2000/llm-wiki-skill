---
name: llm-wiki
description: >
  Use when the user wants to work with their personal knowledge base (wiki):
  add material, initialize, query, check health, generate graph, or produce
  synthesis reports. Also trigger when given a URL/file/text with intent to
  save or organize, even without mentioning "wiki".
---

# llm-wiki — 个人知识库路由器

> 路由器，不含工作流逻辑。确定子 skill 后读取其 SKILL.md 并严格执行。

```
WIKI_SKILL_ROOT=~/.agents/skills/llm-wiki
```

## 路由表

首个命中胜出，无命中则走默认 wiki-ingest。

| 子 skill | 何时使用 |
|---|---|
| wiki-init | 用户要创建 / 初始化一个新知识库 |
| wiki-query | 用户要从已有知识库里查找或提问，不是添加素材 |
| wiki-digest | 用户**明确**说 digest；不要把"总结一下"当作 digest |
| wiki-status | 用户想看知识库统计、内容概览 |
| wiki-lint | 用户要检查知识库健康度、断链、一致性 |
| wiki-graph | 用户要生成或查看知识库关联图谱 |
| wiki-batch-ingest | 用户给了多个文件、一个文件夹、或明确说批量处理 |
| **wiki-ingest**（默认） | 以上都不命中：给了单个 URL / 文件 / 粘贴文本，或意图不明确 |

→ 读取 `${WIKI_SKILL_ROOT}/skills/<子skill>/SKILL.md` 并执行。

## 执行协议

1. 选中子 skill 后，用 Read 读取其 SKILL.md **全文**
2. 严格按子 skill 指令执行，不跳步骤、不混合多个子 skill
3. 不要在路由阶段读 `skills/_shared/context.md`，让子 skill 自己处理
4. 意图不明确时问一句：是添加素材还是查询已有内容？
