# llm-wiki - 多平台知识库构建 Skill

> 基于 [Karpathy 的 llm-wiki 方法论](docs/references/karpathy-llm-wiki.md)，为 Claude Code、Codex、OpenClaw 这类 agent 提供统一的个人知识库构建系统。

## 它做什么

把碎片化的信息变成持续积累、互相链接的知识库。你只需要提供素材，agent 会把链接、文件和文本整理成 wiki 页面。

核心区别：知识被**编译一次，持续维护**，而不是每次查询都从原始文档重新推导。

## 你怎么用

最省事的方式是把这个仓库链接直接扔给你正在用的 agent，让它自己完成安装。

llm-wiki 已拆分为多个独立子 skill（`skills/wiki-init/`、`skills/wiki-ingest/` 等），agent 会根据你的意图自动路由到对应工作流。

## 前置条件

- 你的 agent 能执行 shell 命令
- 如果你要自动提取网页或公众号，Chrome 需要以调试模式启动
- 如果你要自动提取微信公众号或 YouTube 字幕，机器上需要有 `uv`
- `bun` 或 `npm` 二选一即可，安装网页提取依赖时会自动择一使用

## 来源边界

当前来源定义以 `scripts/source-registry.tsv` 为唯一权威来源，URL 和文件路由统一通过 `scripts/source-registry.sh` 读取。

| 分类 | 当前来源 | 处理方式 |
|------|----------|----------|
| 核心主线 | `PDF / 本地 PDF`、`Markdown/文本/HTML`、`纯文本粘贴` | 不依赖外挂，直接进入主线 |
| 可选外挂 | `网页文章`、`X/Twitter`、`微信公众号`、`YouTube`、`知乎` | 先自动提取；失败时按回退提示改走手动入口 |
| 手动入口 | `小红书` | 当前只支持用户手动粘贴 |

当前外挂对应关系：

- `网页文章`、`X/Twitter`、`知乎`：`baoyu-url-to-markdown`
- `微信公众号`：`wechat-article-to-markdown`
- `YouTube`：`youtube-transcript`

## 功能

- **零配置初始化**：一句话创建知识库，自动生成目录结构和模板
- **智能素材路由**：根据来源注册表自动选择提取方式
- **内容分级处理**：长文章完整整理，短内容简化处理，避免浪费
- **批量消化**：给一个文件夹路径，批量处理所有文件
- **结构化 Wiki**：自动生成素材摘要、实体页、主题页，用 `[[双向链接]]` 互相关联
- **知识库健康检查**：自动检测孤立页面、断链、矛盾信息
- **Obsidian 兼容**：所有内容都是本地 markdown，直接用 Obsidian 打开查看

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

## 目录结构

```
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

## 致谢

本项目复用和集成了以下开源项目，感谢它们的作者：

- **[baoyu-url-to-markdown](https://github.com/JimLiu/baoyu-skills#baoyu-url-to-markdown)** - by [JimLiu](https://github.com/JimLiu)
  网页文章、X/Twitter 等内容提取，通过 Chrome CDP 渲染并转换为 markdown

- **youtube-transcript** - YouTube 视频字幕/逐字稿提取

- **[wechat-article-to-markdown](https://github.com/jackwener/wechat-article-to-markdown)** - 微信公众号文章提取

核心方法论来自：

- **[Andrej Karpathy](https://karpathy.ai/)** - [llm-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## License

MIT
