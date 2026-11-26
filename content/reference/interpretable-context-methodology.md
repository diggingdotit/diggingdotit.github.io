---
title: "Interpretable Context Methodology (ICM)"
slug: "interpretable-context-methodology"
date: 2026-11-26T01:55:00Z
draft: false
categories: ["Reference"]
tags: ["icm", "agents-md", "ai-workflows", "context-engineering", "architecture"]
summary: "Quick reference to Jake Van Clief and David McDermott's Interpretable Context Methodology (ICM) — using filesystem structure and plain Markdown instead of multi-agent frameworks."
---

**Interpretable Context Methodology (ICM)** — introduced by Jake Van Clief and David McDermott alongside the MIT-licensed *Model Workspace Protocol (MWP)* — is a lightweight approach to structuring AI agent workflows using plain directory hierarchies and Markdown files instead of programmatic multi-agent orchestration frameworks.

## Core Idea

Multi-agent frameworks earn their keep in complex, concurrent systems, but for sequential workflows where a human reviews output between steps, framework glue adds unnecessary overhead.

ICM replaces code-level orchestration with filesystem structure: if prompts, rules, and intermediate state live as plain Markdown files in well-organised folders, one general-purpose agent reading the right files at the right moment can do the work of a multi-agent framework. The methodology draws directly on Unix pipeline design, modular decomposition, multi-pass compilation, and literate programming.

## Key Building Blocks

- **Lean root routing file (`AGENTS.md` / `CLAUDE.md`):** A short Layer 0 map (typically under ~60 lines) that tells the agent where it is and where each stage lives — pure routing, no bloated business logic.
- **Numbered stage folders:** Sequential directories (for example `01-research/`, `02-draft/`, `03-review/`) encode execution order in the filesystem itself. Inspecting or reordering a workflow is just opening or renaming folders.
- **Scoped stage contracts (`CONTEXT.md`):** Each stage directory holds a small brief specifying inputs, rules, outputs, and human review checks, keeping context windows clean and preventing token pollution.
- **Factory vs. product separation:** Durable reference rules, templates, and schemas stay separate from working outputs and ephemeral session state (such as a local, unversioned `CHECKPOINT.md`).
- **Local scripts for mechanical work:** Deterministic tasks stay in ordinary scripts rather than delegating mechanical chores to an AI model.

## Cross-Agent Inward Pointers

Because different coding agents auto-discover different default files (`~/.gemini/AGENTS.md` for Antigravity, `~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` for Codex), ICM favours plain-Markdown redirects over symlinks or duplicated files — with every tool-specific entrypoint pointing **inward** to one canonical file:

```text
~/.gemini/AGENTS.md   ──▶  \
~/.claude/CLAUDE.md   ──▶   ├──▶  ~/.agents/AGENTS.md  (single source of truth)
~/.codex/AGENTS.md    ──▶  /
```

Each agent lands on its expected entrypoint at startup, follows the one-line redirect, and you only ever maintain one canonical `AGENTS.md`.

## Primary Links

- **Original Paper (arXiv):** [Interpretable Context Methodology: Folder Structure as Agentic Architecture](https://arxiv.org/abs/2603.16021) — Jake Van Clief & David McDermott, March 2026 ([Direct PDF](https://arxiv.org/pdf/2603.16021))
- **YouTube Channel:** [Jake Van Clief (@JEVanClief)](https://www.youtube.com/@JEVanClief)
