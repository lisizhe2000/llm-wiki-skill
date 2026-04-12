---
name: wiki-init
description: Use when the user wants to create or initialize a new personal knowledge base, says "初始化知识库", "新建 wiki", or "创建知识库"
---

> 前置：先读取 `${WIKI_SKILL_ROOT}/skills/_shared/context.md`

## 依赖检查

首次使用时，检查以下依赖是否已安装。如果缺失，提示用户运行安装：

```bash
bash ${WIKI_SKILL_ROOT}/skills/wiki-init/setup.sh
```

依赖 skill / 工具：
- `baoyu-url-to-markdown` — 普通网页、X/Twitter、部分知乎提取
- `wechat-article-to-markdown` — 微信公众号提取
- `youtube-transcript` — YouTube 字幕提取

即使部分依赖缺失，skill 仍可工作（用户可以手动粘贴文本内容）。

## 工作流：init（初始化知识库）

### 前置检查（含多知识库 CWD 检查）

1. 先检查**当前工作目录**是否包含 `.wiki-schema.md`
   - 如果包含 → 当前目录已经是一个知识库，提示用户已存在并询问是否要重新初始化
2. 如果当前目录没有 → 读取 `~/.llm-wiki-path` 文件
   - 如果存在 → 提示用户已有一个知识库（显示路径），询问是要新建还是切换到那个
3. 两个都没有 → 进入初始化流程

### 步骤

1. **询问知识库主题**（先向用户提问）：
   - "你的知识库要围绕什么主题？比如'AI 学习笔记'、'产品竞品分析'、'读书笔记'"
   - 如果用户没想法，默认用"我的知识库"

2. **询问知识库语言**（先向用户提问）：
   - "知识库内容用什么语言？中文 / English（默认中文）"
   - 选项：`zh`（中文）或 `en`（English）
   - 如果用户没有明确说，默认 `zh`
   - 将选择记录为 `WIKI_LANG`（`zh` 或 `en`）

3. **询问保存位置**（先向用户提问）：
   - 默认：`~/Documents/我的知识库/`（zh）或 `~/Documents/my-wiki/`（en）
   - 用户可以自定义路径

4. **运行初始化脚本**：
   ```bash
   bash ${WIKI_SKILL_ROOT}/scripts/init-wiki.sh "<路径>" "<主题>"
   ```

5. **写入语言配置并本地化种子文件**：
   - 将 `.wiki-schema.md` 中的 `语言：{{LANGUAGE}}` 替换为：
     - `zh` → `语言：中文`（种子文件保持中文，无需额外处理）
     - `en` → `语言：English`，**同时**覆写以下种子文件为英文版：
   - 如果 `WIKI_LANG=en`，读取 `${WIKI_SKILL_ROOT}/templates/index-en-template.md`、`${WIKI_SKILL_ROOT}/templates/overview-en-template.md`、`${WIKI_SKILL_ROOT}/templates/log-en-template.md`，将 `{{DATE}}` 和 `{{TOPIC}}` 替换为实际值后，分别写入 `index.md`、`wiki/overview.md`、`log.md`

6. **记录路径**到 `~/.llm-wiki-path`：
   ```bash
   echo "<路径>" > ~/.llm-wiki-path
   ```

7. **输出引导**（根据 `WIKI_LANG` 切换语言）：

   **中文（zh）**：
   ```
   知识库已创建！路径：<路径>

   接下来你可以：
   - 给我一个链接，我会自动提取并整理（网页、X/Twitter、公众号、知乎等）
   - 小红书内容请直接粘贴文本给我（暂不支持自动提取）
   - 给我一个本地文件路径（PDF、Markdown 等）
   - 直接粘贴文本内容
   - 批量消化：给我一个文件夹路径

   推荐：用 Obsidian 打开这个文件夹，可以实时看到知识库的构建效果。
   ```
   （英文版按「输出语言规则」生成，结构相同。）
