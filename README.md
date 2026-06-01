# Research Wiki — Obsidian + Claude Code 个人知识库模板

用 Obsidian 做界面，Claude Code 做维护，构建一个会自己生长的研究知识库。

## 核心思路

**AI 维护结构，人类负责判断。**

你只做两件事：把原始资料扔进 `raw/`，然后跟 Claude Code 说"ingest"。AI 负责提取概念、建立交叉引用、维护索引、发现你没注意到的连接。你读 wiki、提问、做研究决策。

灵感来源：[Karpathy 的 LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## 快速开始

### 1. 克隆到你的 Obsidian vault

```bash
cd /path/to/your/obsidian-vault
git clone https://github.com/your-username/research-wiki.git
```

### 2. 编辑 `claude.md`

- 把 "Who I Am" 改成你自己的研究领域
- 填入你的 Core Research Questions（3-5 个你最关心的驱动性问题）
- 调整 Style Notes

### 3. 编辑 `learner_profile.md`

- 填入你的知识背景、学习目标、语言偏好

### 4. 开始使用

```bash
cd research-wiki
claude
```

然后说：`请 ingest raw/papers/ 里的 xxx 论文`

## 目录结构

```
research-wiki/
├── claude.md                  # Claude Code 的指令文件（必须自定义）
├── raw/                       # 原始资料（人类维护，不可变）
│   ├── papers/                # 论文 PDF/markdown
│   ├── books/                 # 教材章节
│   ├── courses/               # 课件、课程笔记
│   ├── clips/                 # 网页剪藏（推荐用 Obsidian Web Clipper）
│   └── assets/                # 图片附件
└── wiki/                      # AI 生成并维护的知识库
    ├── concepts/              # 概念页面（一个概念一个文件）
    ├── papers/                # 论文摘要页面（一篇论文一个文件）
    ├── connections/           # 跨概念连接（idea 产生的地方）
    ├── questions/             # 开放研究问题（未来方向的种子）
    ├── index.md               # 所有页面的目录（AI 维护）
    ├── log.md                 # 操作日志（AI 维护）
    ├── system.md              # 苏格拉底教学系统的规则
    ├── learner_profile.md     # 学习者背景和偏好（必须自定义）
    ├── progress.md            # 学习进度追踪（AI 维护）
    └── revision_notes.md     # 薄弱点和复习笔记（AI 维护）
```

## 三个核心操作

| 操作         | 说明                  | 示例                                       |
| ---------- | ------------------- | ---------------------------------------- |
| **Ingest** | 把 raw/ 里的资料编译进 wiki | `ingest raw/papers/xxx.pdf`              |
| **Query**  | 基于 wiki 内容回答问题      | `rate reduction 和 score function 有什么联系？` |
| **Lint**   | 检查 wiki 健康状况        | `lint 一下 wiki`                           |

## 知识循环

```
papers 喂进来 → 长出 concepts → concepts 碰撞出 connections
    ↑                                          ↓
  找新论文 ← questions 驱动方向 ← 想不清楚的变成 questions
```

## 苏格拉底学习系统

wiki 内置了一套苏格拉底式教学系统。当你想深入理解某个概念时：

```
用 Socrates 模式解释 [[coding-rate]]
```

Claude Code 会读取你的 `learner_profile.md` 和 `progress.md`，从你已知的知识出发，用追问的方式引导你自己推导出结论。对话结束后自动更新进度。

**核心规则：AI 永远不直接给答案，而是通过提问引导你自己得出结论。**

## 自定义指南

### Core Research Questions

这是整个系统最重要的配置。不要按学科分类，按你真正关心的问题组织：

```markdown
## Core Research Questions

1. What defines a "good" representation?
2. ...
```

这些问题会随着 wiki 生长而演化。定期更新，然后跑一遍 `lint` 让 AI 重新对齐。

### 与现有 Obsidian vault 共存

不需要迁移你现有的笔记。research-wiki 作为 vault 的一个子目录独立运行，Obsidian 的 graph view 会自动连接两边同名的概念。

## 需要什么

- [Obsidian](https://obsidian.md/)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) （或其他终端 AI agent）
- 推荐：[Obsidian Web Clipper](https://obsidian.md/clipper) 浏览器插件

## License

MIT