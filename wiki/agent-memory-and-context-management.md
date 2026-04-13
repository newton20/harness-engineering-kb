---
title: Agent Memory and Context Management
type: wiki
tags:
  - memory
  - context-management
  - memgpt
  - letta
  - context-window
  - agent-architecture
  - filesystem
  - context-engineering
sources:
  - raw/sarahwooders-2040121230473457921.md
  - raw/letta-com-blog-benchmarking-ai-agent-memory.md
  - raw/systematicls-2028814227004395561.md
  - raw/kevingu-2031889622385729730.md
source_count: 4
status: draft
last_compiled: 2026-04-13
---

# Agent Memory and Context Management

Memory is not an optional plugin for an agent harness -- it is the harness itself. How an agent manages what enters context, what gets evicted, what persists across sessions, and what gets retrieved determines its ceiling on any non-trivial task. The field is actively debating whether simple filesystem tools or structured databases are the right substrate for agent memory, but all sides agree that the harness must own memory decisions rather than delegating them to external services.

## Memory Is Core, Not a Plugin

Sarah Wooders (Letta CTO) frames memory's role bluntly: "Asking to plug memory into an agent harness is like asking to plug driving into a car." [Source: raw/sarahwooders-2040121230473457921.md] Memory is not an integration to bolt on after the fact. It is the mechanism by which an agent maintains coherence across turns, sessions, and tasks. Wooders argues that the harness makes many invisible decisions that an external plugin cannot control: how AGENTS.md or CLAUDE.md files are loaded into context, what survives compaction, whether interactions are stored and made queryable, and how filesystem information is exposed. [Source: raw/sarahwooders-2040121230473457921.md]

This framing has practical consequences. If memory is core, then the harness must own decisions about what enters context, what gets evicted, and what gets persisted -- rather than delegating those decisions to a third-party memory service with its own abstractions and failure modes.

## The MemGPT Memory Hierarchy

The MemGPT design, developed by the Letta team, introduces a memory hierarchy inspired by operating system memory management. Agents actively manage what remains in their immediate context versus what gets stored in external layers that can be retrieved as needed. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md] The tiers are:

- **Core memory (in-context):** The information currently sitting inside the model's context window -- the agent's working memory, fast to access, limited in size, and the only memory the model can directly reason over.
- **Conversational memory:** Past conversation turns that have been evicted from the context window but remain retrievable, analogous to data paged out to disk in an OS.
- **Archival memory:** Long-term storage for facts, preferences, and accumulated knowledge that persists across sessions. The agent can write to and query this store explicitly.
- **External files:** Documents, codebases, and other resources the agent can access through tools.

The key insight is that the agent itself manages transitions between these tiers. It decides what to commit to archival memory, what to retrieve, and what to let go. The harness provides the mechanisms; the agent drives the policy. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md]

## Filesystem Tools Beat Specialized Memory Tools

One of the most surprising findings from Letta's benchmarking work is that simple filesystem tools outperform purpose-built memory systems. On the LoCoMo benchmark (which tests long-context memory retrieval), filesystem tools paired with gpt-4o-mini achieved **74.0%** accuracy -- beating Mem0's specialized memory tools at **68.5%** for their top-performing graph variant. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md]

The explanation: agents today are highly effective at using tools likely to have been in their training data, such as filesystem operations (grep, search_files, open, close). Specialized memory tools that may have originally been designed for single-hop retrieval are less effective than simply allowing the agent to autonomously search through data with iterative querying. Agents can generate their own queries rather than simply searching the original questions, and they can continue searching until the right data is found. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md]

The practical takeaway: **simpler tools beat complex knowledge graphs for memory.** Before building a bespoke memory system with embeddings and graph databases, try giving the agent a filesystem and standard search tools. You may find it performs better with tools it already understands deeply.

## Context Constitution and Context Repositories

Letta's "Context Constitution" captures a set of principles for managing what goes into the context window. The core idea is that context is a scarce resource and must be curated deliberately, not filled passively. [Source: raw/sarahwooders-2040121230473457921.md]

For coding agents specifically, Letta introduced **Context Repositories** -- git-backed memory where agent memory is projected to a filesystem that can be concurrently modified by background memory subagents specializing in prompt rewriting and active memory management. [Source: raw/sarahwooders-2040121230473457921.md] This approach provides a full history of what the agent knew and when, integrates naturally with developer workflows, and leverages tools (git, grep, file read/write) that models handle well.

## Context Is Everything

@systematicls articulates a complementary principle: "Context is everything." The recommendation is to strip away dependencies and give agents only what they need -- no more, no less. [Source: raw/systematicls-2028814227004395561.md]

Overloading context with irrelevant information is not just wasteful; it actively degrades performance. The analogy is vivid: if you ask an agent to build a hangman game in Python but its context is polluted with memory notes from 26 sessions ago and screen-hanging incidents from 71 sessions ago, the agent gets confused about what is relevant. [Source: raw/systematicls-2028814227004395561.md] The discipline is in curation: assembling precisely the context the agent needs for each step, rather than providing a firehose of information and hoping the model sorts it out.

@systematicls recommends treating CLAUDE.md as a "logical, nested directory of where to find context given a scenario and an outcome" -- as barebones as possible, containing only the if-else logic of where to seek context. [Source: raw/systematicls-2028814227004395561.md]

## The Critique of Markdown-as-Database

Kevin Gu (CTO of Dex) pushes back on the "filesystem for everything" position, arguing that the ongoing romanticization of putting everything in markdown files is "reinventing the database in the worst possible substrate." [Source: raw/kevingu-2031889622385729730.md]

His core arguments:

- **The data modeling problem:** The moment you need to query across your graph (e.g., "find all decisions that contradict the current roadmap"), you are doing full-text search across thousands of files. A real database already gives you joins, indexes, and constraints. Markdown files just give you grep. [Source: raw/kevingu-2031889622385729730.md]
- **Maintenance and drift:** If your source of truth is a Google Doc and you have extracted claims into a markdown note, you now have two copies. Two copies drift. A database with a reference to the original source (a Drive file ID, a Slack message URL) can cascade updates when the source changes. Dependency tracked, provenance preserved, zero drift by design. [Source: raw/kevingu-2031889622385729730.md]
- **The flattening problem:** A note might contain a decision, a guess, a stale belief, or someone's personal framing. All of it becomes text, all of it becomes traversable, all of it looks more structurally similar than it really is. A markdown graph makes many things readable; it does not make them governable. [Source: raw/kevingu-2031889622385729730.md]
- **Schema is context too:** When data lives in a typed schema, the agent knows how it was generated, what kind of thing it is reading, what other records depend on it, and what it depends on. It can reason about provenance, not just content. [Source: raw/kevingu-2031889622385729730.md]

Gu's proposed architecture has three layers: work apps as the source of truth, a file layer for hot access metadata and base instructions, and a database as the graph storing derived information, structure, relationships, and dependency chains. [Source: raw/kevingu-2031889622385729730.md]

## What Survives Compaction Matters

When a context window fills up, something has to give. @systematicls notes that agents are "still atrocious at connecting the dots, filling in the gaps, or making assumptions" -- and that compaction decisions are where this weakness becomes critical. [Source: raw/systematicls-2028814227004395561.md] One of the most important rules to include in CLAUDE.md is a rule on how to deal with grabbing context after compaction: re-reading the task plan and re-reading the relevant files before continuing. [Source: raw/systematicls-2028814227004395561.md]

If the harness summarizes away a critical constraint ("the client requires Python 3.8 compatibility"), the agent will violate that constraint in future turns with no awareness that it ever knew otherwise.

## Related

- [Tool Design Patterns](tool-design-patterns.md) -- Filesystem tools as memory tools, primitives over integrations
- [Practical Best Practices](practical-best-practices.md) -- Progressive disclosure, CLAUDE.md as directory, context curation
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- Context resets vs. compaction, progress files as memory
- [Claude Code Architecture](claude-code-architecture.md) -- Multi-level memory hierarchy built directly into the harness

## Open Questions

- Where is the crossover point where filesystem-based memory breaks down and a structured database becomes necessary? Letta's benchmarks show filesystem wins on LoCoMo, but Kevin Gu argues this fails at organizational scale with drift and governance problems.
- How should compaction policies be designed? No standard approach exists for deciding what information is load-bearing versus expendable during compaction.
- Can the MemGPT agent-driven memory management approach work when the memory policy itself requires domain-specific knowledge the agent may not have?

## Sources

- [raw/sarahwooders-2040121230473457921.md](../raw/sarahwooders-2040121230473457921.md) -- Sarah Wooders (Letta CTO) on memory as core harness function, not plugin. Context Constitution and Context Repositories.
- [raw/letta-com-blog-benchmarking-ai-agent-memory.md](../raw/letta-com-blog-benchmarking-ai-agent-memory.md) -- Letta blog on benchmarking agent memory. Filesystem tools (74.0%) beat Mem0 (68.5%) on LoCoMo. MemGPT memory hierarchy.
- [raw/systematicls-2028814227004395561.md](../raw/systematicls-2028814227004395561.md) -- @systematicls on "context is everything," stripping dependencies, CLAUDE.md as directory, compaction handling.
- [raw/kevingu-2031889622385729730.md](../raw/kevingu-2031889622385729730.md) -- Kevin Gu critique of markdown-as-database. Arguments for typed schemas, provenance tracking, and structured databases over filesystem-only memory.
