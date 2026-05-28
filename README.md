<p align="center">
  <h1 align="center">🌱 Mind Garden</h1>
  <p align="center"><strong>AI-Curated Personal Knowledge Base</strong></p>
  <p align="center">Based on Karpathy's LLM Wiki Pattern</p>
</p>

---

## Overview

Mind Garden is an opinionated template for building a **self-maintaining personal
knowledge base** — a collection of interconnected Markdown files where an LLM
acts as your curator, librarian, and gardener.

Instead of manually organizing notes, you feed raw source materials (articles,
papers, transcripts) into the `raw/` directory. The LLM then reads them,
extracts insights, creates structured summaries, maintains a concept graph,
cross-links related pages, and keeps an activity log — all following the schema
defined in `AGENTS.md`.

> *"Obsidian is the IDE, the LLM is the programmer, the wiki is the codebase."*
> — Andrej Karpathy

## Why This Exists

Personal knowledge bases fail when maintenance costs exceed their utility.
By delegating summarization, linking, deduplication, and archival to an LLM,
the cost of keeping your wiki organized drops to near zero.

**Mind Garden is not RAG.** It does not rely on vector databases or embedding
searches. Instead, the LLM actively builds a persistent, compounding artifact of
interconnected Markdown files — a knowledge graph you can browse, search, and
version-control.

## Quick Start

### Prerequisites

- [Obsidian](https://obsidian.md) (for browsing your wiki)
- [Obsidian Web Clipper](https://obsidian.md/clipper) browser extension
- An LLM-powered coding assistant that reads `AGENTS.md` (Claude Code,
  Qoder CLI, Codex CLI, or any agent that respects AGENTS.md conventions)

### Setup

```bash
# 1. Clone the repository
git clone https://github.com/Muzixin/mind-garden.git
cd mind-garden

# 2. Open as an Obsidian vault
#    File → Open Vault → Select the mind-garden directory

# 3. Start clipping
#    Use the Obsidian Web Clipper to save articles to raw/articles/

# 4. Let the LLM work
#    In your coding assistant, say:
#      ingest raw/articles/<filename>.md
```

Your first `ingest` will populate the `wiki/` directory with source summaries,
concept pages, and index files. The root `index.md` becomes your dashboard.

## Architecture

```
mind-garden/
├── index.md              # Dashboard (hub-and-spoke, < 35 lines)
├── AGENTS.md             # Schema — the constitution
├── raw/                  # Immutable source layer (read-only)
│   ├── .ingested         # Ingestion tracker (auto-maintained)
│   └── articles/
│       papers/
│       books/
│       transcripts/
│       inbox/
├── wiki/                 # AI-curated layer (LLM writes here)
│   ├── 概念/ (Concepts)
│   ├── 实体/ (Entities)
│   ├── 素材/ (Source Summaries)
│   ├── 对比/ (Comparisons)
│   └── 合成/ (Synthesis)
└── skills/               # Optional automation skills
```

### Three Layers

| Layer | Purpose | Mutable? |
|-------|---------|----------|
| `raw/` | Original source materials | No (immutable) |
| `wiki/` | AI-generated structured knowledge | Yes (LLM writes) |
| `AGENTS.md` | Rules, workflows, conventions | Yes (co-evolves with you) |

### Two Special Files

- **`index.md`** — A categorized table of contents. Lightweight, scans in one
  glance. Details live in each category's `_index.md`.
- **`log.md`** — An append-only activity log in reverse chronological order.
  Every ingest, query, lint, and refactor is recorded.

## Core Workflows

### Ingest

```
ingest raw/articles/attention-is-all-you-need.md
```

The LLM reads the source file, creates a source summary, updates or creates
relevant concept and entity pages, maintains indexes, checks for cross-links,
and appends a log entry. **A single ingest typically touches 5-15 files.**

### Query

Ask questions naturally. The LLM scans `index.md` to locate relevant pages,
reads them in full, then answers with `[[wiki-link]]` citations. High-value
answers are offered for archival into `合成/`.

### Lint

```
lint wiki
```

The LLM scans for contradictions, orphan pages, missing concept entries, stale
claims, and unmet comparison/synthesis triggers. Returns a prioritized fix list.

## Page Conventions

Every wiki page follows this frontmatter template:

```yaml
---
title: Page Title
type: 概念 | 实体 | 素材摘要 | 对比 | 合成 | 目录
entity_kind: 产品 | 框架 | 人物 | 组织   # Only for 实体
sources: [raw/articles/example.md]
related: [[other-page]]
created: 2026-05-28
updated: 2026-05-28
confidence: 高 | 中 | 低
tags: [Chinese tags, international terms in English]
---
```

### Naming

- **Wiki folders and filenames**: Chinese by default (`概念/检索增强生成/架构与演进.md`).
  `index.md` and `_index.md` keep their English names as convention identifiers.
- **Brand/product names**: May keep original names (`Dify.md`, `Neo4j.md`).

## Ingestion Tracking

The `raw/.ingested` file tracks every processed source file. This enables
batch processing — clip multiple articles, then run the `process-inbox` skill
to ingest all unprocessed files at once.

## Optional Skills

The `skills/` directory contains three optional automation skills for supported
LLM environments:

| Skill | Command | Description |
|-------|---------|-------------|
| `url-ingest` | `/url-ingest <url>` | Fetch a URL, save to raw/, run full ingest |
| `extract-knowledge` | `/extract-knowledge` | Extract knowledge from conversation, save to wiki |
| `process-inbox` | `/process-inbox` | Scan raw/ for unprocessed files, batch ingest |

## Obsidian Graph Compatibility

This schema is designed for Obsidian's graph view:

- `[[wiki-link]]` double brackets → graph edges
- `tags:` frontmatter → tag clusters
- Folder structure (`概念/`, `实体/`, `素材/`) → color-groupable by path
- `_index.md` pages → natural hub nodes with high indegree

**Key rule**: `raw/` files do not participate in interlinking. All links live
within `wiki/`. Filter the graph view to path `wiki/` for a clean visualization.

## Philosophy

Knowledge bases die from administrative overhead. Mind Garden keeps them alive
by making maintenance cost zero. You provide the raw materials and make the
high-level decisions. The LLM does the rest.

This pattern mirrors historical visions of associative memory (Memex, Hypertext),
but with an AI that actually does the linking.

## Credits

Inspired by [Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

## License

MIT
