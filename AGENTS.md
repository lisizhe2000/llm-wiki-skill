# llm-wiki — 多平台知识库构建 Skill

> 基于 [Karpathy 的 llm-wiki 方法论](docs/references/karpathy-llm-wiki.md)，为 Claude Code、Codex、OpenClaw 这类 agent 提供统一的个人知识库构建系统。

把碎片化的信息编译成可持续维护、互相链接的本地 markdown wiki。你只需要提供素材，agent 负责提取、整理、关联和持续维护。

## 前置条件

- 当前 agent 能执行 shell 命令
- 网页类自动提取依赖 Chrome CDP；在 macOS 上使用 Google Chrome 调试端口 `9222`
- 如果要自动提取微信公众号或 YouTube 字幕，机器上需要有 `uv`
- `bun` 或 `npm` 二选一即可，供 `baoyu-url-to-markdown` 运行脚本时使用

## 架构

llm-wiki 已拆分为多个独立子 skill，各自负责单一工作流：

| 子 skill | 说明 |
|---|---|
| `skills/wiki-init/` | 初始化知识库 |
| `skills/wiki-ingest/` | 消化单个素材 |
| `skills/wiki-batch-ingest/` | 批量消化 |
| `skills/wiki-query/` | 查询知识库 |
| `skills/wiki-digest/` | 深度综合报告（仅手动触发） |
| `skills/wiki-lint/` | 健康检查 |
| `skills/wiki-status/` | 查看状态 |
| `skills/wiki-graph/` | 知识图谱 |

共享资源统一保留在仓库根目录：

- `templates/`：知识库页面模板
- `scripts/`：来源注册表、状态检查、初始化等共享脚本
- `deps/`：打包在仓库里的依赖 skill

所有子 skill 都通过相对路径解析 `${WIKI_SKILL_ROOT}`，不依赖固定安装目录。

## 来源边界

当前来源定义以 `scripts/source-registry.tsv` 为唯一权威来源，URL 和文件路由统一通过 `scripts/source-registry.sh` 读取。

| 分类 | 当前来源 | 处理方式 |
|------|----------|----------|
| 核心主线 | `PDF / 本地 PDF`、`Markdown/文本/HTML`、`纯文本粘贴` | 不依赖外挂，直接进入主线 |
| 可选外挂 | `网页文章`、`X/Twitter`、`微信公众号`、`YouTube`、`知乎` | 先自动提取；失败时按统一回退提示改走手动入口 |
| 手动入口 | `小红书` | 当前只支持用户手动粘贴 |

当前外挂对应关系：

- `网页文章`、`X/Twitter`、`知乎`：`baoyu-url-to-markdown`
- `微信公众号`：`wechat-article-to-markdown`
- `YouTube`：`youtube-transcript`

## 功能

- **零配置初始化**：一句话创建知识库，自动生成目录结构和模板
- **智能素材路由**：根据来源注册表自动选择提取方式
- **内容分级处理**：长内容完整整理，短内容简化处理，避免过度加工
- **批量消化**：给一个文件夹路径，按文件逐个进入 ingest 工作流
- **结构化 Wiki**：自动生成素材摘要、实体页、主题页，用 `[[双向链接]]` 互相关联
- **知识库健康检查**：检测孤立页面、断链、矛盾信息和索引不一致
- **Obsidian 兼容**：所有内容都是本地 markdown，可直接用 Obsidian 打开查看

## 目录结构

```text
你的知识库/
├── raw/                    # 原始素材（不可变）
│   ├── articles/           # 网页文章
│   ├── tweets/             # X/Twitter
│   ├── wechat/             # 微信公众号
│   ├── xiaohongshu/        # 小红书
│   ├── zhihu/              # 知乎
│   ├── pdfs/               # PDF
│   ├── notes/              # 笔记
│   └── assets/             # 图片等附件
├── wiki/                   # AI 生成的知识库
│   ├── entities/           # 实体页（人物、概念、工具）
│   ├── topics/             # 主题页
│   ├── sources/            # 素材摘要
│   ├── comparisons/        # 对比分析
│   └── synthesis/          # 综合分析
├── index.md                # 索引
├── log.md                  # 操作日志
└── .wiki-schema.md         # 配置
```

## 常见问题

### 这个仓库只给某个 agent 用吗？

不是。Claude Code、Codex、OpenClaw 共用同一套核心内容。这个仓库本身就是通用技能包，不再维护根路由器 `SKILL.md` 和 `platforms/` 入口树。

### 为什么网页 / X / 知乎提取失败？

这几类来源统一走 `baoyu-url-to-markdown`，依赖 Chrome CDP。先在 macOS 上执行：

```bash
open -na "Google Chrome" --args --remote-debugging-port=9222
```

如果 Chrome 已启动但仍失败，优先检查当前 Chrome 会话是否已登录目标站点；再不行就改走手动粘贴入口。

### 为什么公众号提取失败？

公众号当前使用 `wechat-article-to-markdown`。如果机器上没有 `uv`，先安装；如果自动提取仍失败，也可以直接把正文粘贴给 agent 继续走知识库主线。

## 开发约定

- 子 skill 命名：`wiki-<动作>`，放在 `skills/wiki-<动作>/SKILL.md`
- 每个子 skill 的 SKILL.md 开头必须引用共享上下文：`> 前置：先读取 ${WIKI_SKILL_ROOT}/skills/_shared/context.md`
- 共享脚本和模板保留在仓库根目录的 `scripts/` 和 `templates/`
- 路径解析统一使用 `${WIKI_SKILL_ROOT}`，不硬编码安装目录

## 测试方法

```bash
# 回归测试
bash tests/regression.sh

# 外挂状态测试
bash tests/adapter-state.sh
```
