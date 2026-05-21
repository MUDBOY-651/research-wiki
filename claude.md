# Research Wiki — CLAUDE.md

## Who I Am

This is Roshin's personal research knowledge base. The domain is computer science, focus on C/C++ Programming, operating System, AI Agent, AI infra and other infrastructure knowledge.

## Architecture

```
research-wiki/
├── CLAUDE.md          ← you are here
├── raw/               ← immutable source documents (papers, notes, clips)
│   └── assets/        ← images downloaded from clipped articles
├── wiki/
│   ├── index.md       ← catalog of all wiki pages with one-line summaries
│   ├── log.md         ← chronological record of ingests, queries, lints
│   ├── concepts/      ← concept pages (one per key idea)
│   ├── papers/        ← paper summary pages (one per paper)
│   ├── questions/     ← open research questions and explorations
│   └── connections/   ← cross-cutting pages linking multiple concepts
└── templates/
    ├── paper.md       ← template for paper summaries
    └── concept.md     ← template for concept pages
```

**Raw** is immutable. Never modify files in raw/. This is the source of truth.

**Wiki** is LLM-owned. The LLM writes, updates, and maintains everything here. Roshin reads and browses; the LLM does the bookkeeping.

## Core Research Questions

Organize the wiki around these driving questions (not by subfield):
1. 如何掌握现代C++编程，第一优先级是如何掌握常见场景，第二优先级是如何应对面试？
2. 如何掌握计算机系统基础，例如操作系统等基础计算机基础概念。
3. 如何学习某一特定领域知识，例如机器学习系统（MLSYS）
4. 如何学习一个开源项目，梳理其架构和实现目的，并能够合入PR解决对应的issue？

When ingesting a paper or note, always ask: which of these questions does this source speak to?

## Conventions

### Page Format

All wiki pages use YAML frontmatter:

```yaml
---
title: "Page Title"
type: concept | paper | question | connection
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: ["raw/filename.md"]
questions: [1, 3]  # which core questions this relates to
tags: [representation-learning, information-theory]
---
```

### Linking

- Use Obsidian wikilinks: `[[concept-name]]`
- Every concept page should link to related concepts and papers
- Every paper page should link to concepts it contributes to
- Backlinks are implicit in Obsidian; forward links must be explicit

### Naming

- Concept pages: lowercase-with-hyphens (e.g., `rate-reduction.md`, `equilibrium-propagation.md`)
- Paper pages: `AuthorYear-short-title.md` (e.g., `Ramsauer2020-hopfield-attention.md`)
- Question pages: `q-short-description.md` (e.g., `q-rate-reduction-as-energy.md`)
- Connection pages: `conn-topic.md` (e.g., `conn-mcr2-meets-eqprop.md`)

## Operations

### Ingest

When told to ingest a source:

1. Read the full source in `raw/`
2. Discuss key takeaways with Erber if in interactive mode
3. Create or update a paper summary page in `wiki/papers/`
4. For each key concept mentioned:
    - If concept page exists → update it with new information, note the source
    - If concept page doesn't exist → create it from template
5. Check if this source creates new connections between existing concepts → create connection pages if so
6. Tag which core research questions (1-5) this source speaks to
7. Update `wiki/index.md`
8. Append entry to `wiki/log.md` with format: `## [YYYY-MM-DD] ingest | Source Title`

### Query

When asked a question:

1. Read `wiki/index.md` to find relevant pages
2. Load and read those pages
3. Synthesize an answer with `[[wikilinks]]` to source pages
4. If the answer reveals a genuinely new connection or insight, offer to save it as a new page in `wiki/questions/` or `wiki/connections/`

### Lint

When asked to lint:

1. Check for orphan pages (no inbound links)
2. Check for concepts mentioned in text but lacking their own page
3. Check for stale claims that newer sources contradict
4. Check for core questions (1-5) that have thin coverage
5. Suggest specific papers or topics to investigate for gaps
6. Report in a structured format

## Paper Summary Template

```markdown
---
title: "Paper Title"
type: paper
authors: [Author1, Author2]
year: YYYY
venue: NeurIPS/ICML/ICLR/arXiv/etc.
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: ["raw/filename.md"]
questions: []
tags: []
---

## Core Contribution
[1-2 sentences: what is the single main claim or result?]

## Method
[How they achieve it. Key equations/algorithms if relevant.]

## Key Results
[What did they show? Quantitative if possible.]

## Limitations & Open Questions
[What doesn't it do? What assumptions does it make?]

## Connections
- [[concept-1]]: how this paper relates
- [[concept-2]]: how this paper relates

## Raw Source
- [[raw/filename.md]]
```

## Concept Page Template

```markdown
---
title: "Concept Name"
type: concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: []
questions: []
tags: []
---

## Definition
[What is this concept? Be precise.]

## Key Ideas
[Core mechanics, equations, intuitions.]

## Connections
- [[related-concept]]: relationship
- [[related-paper]]: contribution

## Open Questions
[What's unresolved about this concept?]
```

## Style Notes

- Write for a reader with strong math/CS background (Roshin himself)
- Don't dumb things down; use proper notation
- When in doubt, preserve the mathematical formulation, not just the prose description
- Flag genuine contradictions between sources explicitly rather than smoothing them over
- 中英文混用 is fine — use whatever is clearest for the concept
- 如果可以生成一些简单的示例代码，尽量生成代码进行讲解

## Socrates System

wiki/ 下有四个 Socrates 相关文件：
- `wiki/learner_profile.md`: Roshin 的知识背景和学习风格
- `wiki/progress.md`: 各概念的掌握程度追踪
- `wiki/revision_notes.md`: 复习笔记和薄弱点
- `wiki/system.md`: Socrates 教学模式的行为规则

进入 Socrates 模式时，先读这四个文件了解学习者状态，
对话结束后更新 progress.md 和 revision_notes.md。
Socrates 对话中产生的概念理解，同步更新到对应的 wiki/concepts/ 页面。
