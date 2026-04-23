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
  - raw/arxiv-org-html-2501-09136v4.md
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/code-research-claude-code-2026-04-15.md
  - raw/code-research-karpathy-autoresearch-2026-04-15.md
  - raw/code-research-openclaw-openclaw-2026-04-15.md
  - raw/code-research-all-hands-ai-openhands-2026-04-15.md
  - raw/code-research-anomalyco-opencode-2026-04-15.md
  - raw/code-research-666ghj-mirofish-2026-04-15.md
  - raw/code-research-claude-code-2026-04-14.md
  - raw/code-research-openclaw-openclaw-2026-04-14.md
  - raw/akshay_pachaar-2043745099792953508.md
  - raw/hwchase17-2042978500567609738.md
  - raw/garrytan-2043198780800197025.md
source_count: 17
status: draft
last_compiled: 2026-04-23
---

# Agent Memory and Context Management

Memory is not an optional plugin for an agent harness -- it is the harness itself. How an agent manages what enters context, what gets evicted, what persists across sessions, and what gets retrieved determines its ceiling on any non-trivial task. The field is actively debating whether simple filesystem tools or structured databases are the right substrate for agent memory, but all sides agree that the harness must own memory decisions rather than delegating them to external services.

## Memory Is Core, Not a Plugin

Sarah Wooders (Letta CTO) frames memory's role bluntly: "Asking to plug memory into an agent harness is like asking to plug driving into a car." [Source: raw/sarahwooders-2040121230473457921.md] Memory is not an integration to bolt on after the fact. It is the mechanism by which an agent maintains coherence across turns, sessions, and tasks. Wooders argues that the harness makes many invisible decisions that an external plugin cannot control: how AGENTS.md or CLAUDE.md files are loaded into context, what survives compaction, whether interactions are stored and made queryable, and how filesystem information is exposed. [Source: raw/sarahwooders-2040121230473457921.md]

This framing has practical consequences. If memory is core, then the harness must own decisions about what enters context, what gets evicted, and what gets persisted -- rather than delegating those decisions to a third-party memory service with its own abstractions and failure modes.

### Chase: "Your Harness, Your Memory"

Harrison Chase's April 2026 essay "Your harness, your memory" extended Wooders's point into a political argument about lock-in. Memory is just a form of context, and harnesses control context. If you use a closed harness — especially one behind a proprietary API — you are yielding ownership of your agent's memory to a third party. Chase's three degrees of lock-in, worst to best:

- **Worst:** Whole harness (including long-term memory) behind an API. Anthropic's Claude Managed Agents puts literally everything behind an API.
- **Bad:** Closed harness (Claude Agent SDK uses Claude Code under the hood, which is not open source). Memory interactions are unknown and non-transferable.
- **Mildly bad:** Stateful APIs (OpenAI Responses API, Anthropic server-side compaction). Can't swap models mid-thread. [Source: raw/hwchase17-2042978500567609738.md]

Chase identifies the invisible decisions Wooders flagged and adds more: Can the agent modify its own system instructions? What survives compaction, and what's lost? How is memory metadata presented to the agent? Even Codex, though open-source, generates encrypted compaction summaries unusable outside OpenAI's ecosystem. "Model providers are incredibly incentivized [to move memory behind APIs], and they are starting to." [Source: raw/hwchase17-2042978500567609738.md]

### Tan's Counterpoint: "Memory Is Markdown"

Garry Tan responded the same day: **"If your memory dies when your harness dies, you built the harness too thick. Memory is markdown. Skills are markdown. Brain is a git repo. The harness is a thin conductor — it reads the files, it doesn't own them."** [Source: raw/garrytan-2043198780800197025.md] Tan's position: the substrate for open memory already exists (AGENTS.md, CLAUDE.md, skills files, brain-as-git-repo), and any harness that hoards state in opaque internal structures is structurally wrong. See [Thin Harness, Fat Skills](thin-harness-fat-skills.md) for the full debate including Chase's "directionally correct" concession a day later.

## The MemGPT Memory Hierarchy

The MemGPT design, developed by the Letta team, introduces a memory hierarchy inspired by operating system memory management. Agents actively manage what remains in their immediate context versus what gets stored in external layers that can be retrieved as needed. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md] The tiers are:

- **Core memory (in-context):** The information currently sitting inside the model's context window -- the agent's working memory, fast to access, limited in size, and the only memory the model can directly reason over.
- **Conversational memory:** Past conversation turns that have been evicted from the context window but remain retrievable, analogous to data paged out to disk in an OS.
- **Archival memory:** Long-term storage for facts, preferences, and accumulated knowledge that persists across sessions. The agent can write to and query this store explicitly.
- **External files:** Documents, codebases, and other resources the agent can access through tools.

The key insight is that the agent itself manages transitions between these tiers. It decides what to commit to archival memory, what to retrieve, and what to let go. The harness provides the mechanisms; the agent drives the policy. [Source: raw/letta-com-blog-benchmarking-ai-agent-memory.md]

## Claude Code's 5-Layer Memory System

Source code analysis of Claude Code reveals the most complex memory architecture in any open agent harness: five distinct memory systems operating in parallel. These are: (1) a CLAUDE.md hierarchy with 6 priority levels from managed to local, providing layered instructions the agent cannot modify; (2) a memdir auto-memory system with a MEMORY.md index and individual topic files in YAML frontmatter that the agent CAN write to; (3) an extractMemories background forked agent that proactively runs after each turn, using a side-query with Sonnet to rank relevance of existing memories and write/edit topic files; (4) SessionMemory, which maintains structured mid-session running notes across 8 sections; and (5) prompt history persisted as JSONL files that the agent can grep for past interactions. Each layer has different scope, persistence, and write permissions. [Source: raw/code-research-claude-code-2026-04-15.md]

Claude Code manages context overflow through a 4-layer compaction cascade: (1) snip (drop oldest messages), (2) microcompact (clear old tool_result bodies), (3) API-level context management (server-side clearing), and (4) autocompact (a forked agent that produces a structured 9-section summary). All four layers can fire in the same iteration. Critically, after autocompact, up to 5 recently-read files (within a 50K token budget, 5K per file) are re-injected as attachments, and skill content intentionally survives compaction. This post-compact file restore ensures the agent does not lose active working context during long coding sessions. [Source: raw/code-research-claude-code-2026-04-15.md]

## Claude Code Memory Design Details

### Memory Aging: Staleness-as-Prose

Claude Code's `memdir/memoryAge.ts` injects staleness warnings as pre-computed human-readable prose rather than raw timestamps. The rationale is explicit in the source: "the model is poor at date arithmetic." Warnings produced by this module say things like "claims about code behavior or file:line citations may be outdated." By translating a timestamp into a pre-evaluated natural-language warning, the harness operationalizes a known model weakness at the layer where it can be addressed -- the harness -- rather than relying on the model to correctly compute elapsed time at reasoning time. [Source: raw/code-research-claude-code-2026-04-15.md]

### Formal Memory Type Taxonomy

Claude Code defines four memory content types for memdir topic files -- `user`, `feedback`, `project`, and `reference` -- each with XML-structured guidance in the system prompt for when and how to write them. A notable design constraint on the `feedback` type: it must record both confirmations AND corrections. Recording only corrections causes the model to "grow overly cautious" over time, as the stored signal is negatively skewed. The `project` type requires converting relative dates to absolute dates at write time, making memories time-anchored rather than relative to an unstated "now." [Source: raw/code-research-claude-code-2026-04-15.md]

## Autoresearch: Git as Memory and Minimal-Signal Context Management

Karpathy's autoresearch demonstrates a radically different memory architecture: git branches serve as an experiment database, git commits as checkpoints, and git reset as rollback, with branch HEAD always pointing to the current best-performing hypothesis. Outcome-memory is deliberately separated from code-state -- results.tsv is gitignored so that the clean git history contains only kept experiments while the untracked ledger records all attempts including failures. This provides complete, versioned, diffable experiment history with zero additional infrastructure. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

Autoresearch also demonstrates a minimal-signal extraction pattern as a form of context management. All training output is redirected to a log file, and only 2 scalar values (val_bpb and peak_vram_mb) are extracted via grep into the agent's context per experiment. This adds roughly 3-5 lines to context per iteration instead of 630 lines of raw training output. The agent's context window is treated as a scarce resource with explicit architectural protection -- the system prompt explicitly warns "do NOT use tee or let output flood your context." [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

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

## Agentic RAG: From Static Retrieval to Agent-Driven Memory

The evolution from static RAG (Retrieval-Augmented Generation) to agentic RAG represents a fundamental shift in how agents manage their relationship with external knowledge. In traditional RAG, the pipeline is fixed: embed a query, retrieve the top-k documents, generate a response. The agent has no agency over the retrieval process itself. [Source: raw/arxiv-org-html-2501-09136v4.md]

Agentic RAG embeds autonomous agents into the retrieval pipeline. The agent dynamically decides what to retrieve, evaluates whether retrieved information is sufficient, reformulates queries when results are poor, and iteratively refines its understanding before generating output. This transforms retrieval from a one-shot lookup into an active memory management process -- the agent is not just consuming memory but curating it in real-time. [Source: raw/arxiv-org-html-2501-09136v4.md]

This connects directly to the MemGPT hierarchy described above: agentic RAG agents are effectively performing the same tier-management operations (deciding what to pull into working memory, what to query from archival storage, when to search for new information) but doing so within a single turn rather than across sessions. The practical implication: harness engineers should think of RAG not as a static component but as another memory management surface the agent controls.

## External Storage as Non-Negotiable for Long-Horizon Agents

Anthropic's multi-agent Research feature provides concrete evidence that external memory storage is not optional for long-horizon agent work. The lead research agent saves its plan to persistent Memory (Anthropic's external storage system) because context may be truncated at 200K tokens. If the agent's context is compacted or the session is interrupted, the plan survives in external storage and can be re-loaded. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

This validates the MemGPT design principle -- agents must actively manage what gets persisted externally -- but goes further: for any agent session that might exceed the context window, external storage is a prerequisite, not an optimization.

### Subagent Output to Filesystem

A related pattern from the same system: subagents write their research outputs directly to the filesystem rather than passing results back through the orchestrator's context window. This avoids what Anthropic calls a "game of telephone" -- each relay through an intermediate agent's context risks information loss, summarization artifacts, and token overhead. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

The filesystem-as-communication-channel pattern has broader implications for multi-agent memory. When agents need to share information, writing to a shared filesystem is often more reliable than passing data through context windows. The orchestrator reads subagent outputs when it needs them, rather than having all outputs injected into its context simultaneously. This keeps context lean and gives the orchestrator control over when and how much information to ingest -- the same "context is everything" principle applied to inter-agent communication.

### Long-Horizon Conversation Management

For agents running research sessions that consume 15x the tokens of a standard chat turn, a pattern has emerged: summarize completed phases into external storage, then spawn fresh subagents with clean contexts for subsequent phases. This prevents the progressive degradation that occurs as a context window fills with completed work that is no longer relevant to the current phase. Each subagent starts with a clean context containing only the plan and the specific task, not the full history of all prior work. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

## The Cognitive-Science Frame: Why "More Context" Doesn't Solve Memory

Akshay Pachaar's "Build Agents that never forget" (April 2026) makes the case against treating raw context length as a memory substitute. The 128K/200K/1M token windows feel like they should solve everything. They don't. **Accuracy drops over 30% when relevant information sits in the middle of a long context** (the "lost in the middle" effect). Context is a shared budget: system prompts, retrieved docs, conversation history, and output all fight for the same tokens. "Memory isn't about cramming more text into the prompt. It's about structuring what the agent remembers so it can find what matters." [Source: raw/akshay_pachaar-2043745099792953508.md]

Pachaar catalogs seven failure modes that appear the instant you skip real memory: context amnesia, zero personalization, multi-step task failure, repeated mistakes, no knowledge accumulation, hallucination from gaps, identity collapse. [Source: raw/akshay_pachaar-2043745099792953508.md]

The cognitive-science framing he adopts (originally Lilian Weng's 2023 formulation) maps directly onto agent architecture:

- **Sensory memory** (raw perceptual input, sub-second) → tool output streams the agent may or may not attend to.
- **Working memory** (7±2 items, active thinking, Miller 1956) → the context window itself.
- **Long-term memory** (durable, no practical capacity limit, retrieval is the bottleneck) → external stores.

Long-term memory further splits into **episodic** (specific past events: "on Tuesday, the PostgreSQL cluster went down"), **semantic** (facts and concepts: "PostgreSQL is a relational database"), and **procedural** (skills and workflows: "when a user asks for a refund, first check the purchase date"). The bridge between episodic and semantic is **memory consolidation** — repeated specific events distilling into general knowledge. Without consolidation, an agent replays individual events rather than learning from them. [Source: raw/akshay_pachaar-2043745099792953508.md]

## The Four-Layer Memory Evolution

Pachaar walks through the evolution of agent memory by showing what breaks at each layer:

1. **No memory (stateless)** — "I have 4 apples... I ate one, how many left?" fails. Each call exists in isolation.
2. **Python list (in-memory conversation history)** — multi-turn works because history re-ships with every call. Breaks when the list hits the context ceiling (~turn 200) or the process dies.
3. **Markdown files (persistence)** — Claude Code's CLAUDE.md/MEMORY.md pattern. Restart-safe. Editable. Breaks at scale: 2,000 extracted facts across 200 conversation logs = 500K+ tokens on disk, 128K window. Keyword search is too brittle for synonyms/paraphrases.
4. **Vector search (embeddings)** — solves synonym problem. Breaks on multi-hop questions: "Was Alice's project affected by Tuesday's outage?" requires bridging three facts ("Alice owns Project Atlas," "Project Atlas uses PostgreSQL," "PostgreSQL outage Tuesday"). Vector search can't see the connective tissue because the bridge fact mentions neither Alice nor Tuesday. "Each fact is an isolated point in embedding space." [Source: raw/akshay_pachaar-2043745099792953508.md]

Pachaar notes that OpenClaw has reportedly seen this failure mode in practice: markdown checkpoint files work at small scale but earlier facts "quietly slip away as context accumulates and gets compacted. The storage is there. The retrieval isn't." [Source: raw/akshay_pachaar-2043745099792953508.md]

## Cognee: The Three-Store Architecture

Cognee is an open-source knowledge engine (pip install cognee) built as a unified solution for the four-layer problem. The design combines three storage paradigms under one engine with four async API calls:

- **Relational store (default SQLite, prod Postgres)** captures *provenance* — where data came from, when it was ingested, access control.
- **Vector store (default LanceDB, prod Qdrant/Pinecone/pgvector)** captures *semantics* — what content means, what it's similar to.
- **Graph store (default Kuzu, prod Neo4j/FalkorDB/Neptune)** captures *relationships* — how entities connect, what causes what, who reports to whom. [Source: raw/akshay_pachaar-2043745099792953508.md]

"Flatten any of these and you lose information that matters for retrieval accuracy." The core trick: every graph node has a corresponding embedding. You enter through vectors (find semantically similar content) and exit through the graph (follow relationships to connected entities), or vice versa. This is what makes multi-hop queries work without sacrificing semantic search. [Source: raw/akshay_pachaar-2043745099792953508.md]

### `cognify()`: The Ingestion Pipeline

Cognee's `cognee.cognify()` runs a six-stage pipeline: document classification, permission checking, chunk extraction (paragraph-respecting, not fixed-size), entity and relationship extraction via LLM with automatic deduplication through content hashing, summary generation, and dual indexing into both the vector store and graph store. The deduplication step is load-bearing: "If the same entity shows up across 50 documents, Cognee merges it into a single graph node with 50 inbound edges. Your agent no longer sees 'Alice' as 50 different strangers." The pipeline is incremental — only new or updated files get reprocessed. [Source: raw/akshay_pachaar-2043745099792953508.md]

### `memify()`: RL-Inspired Optimization

`memify()` is what Pachaar argues separates Cognee from "ingest and search" tools. It runs an RL-inspired optimization pass over the graph: strengthens useful paths that led to good retrieval, prunes stale nodes that haven't been touched, auto-tunes edge weights based on real usage, and adds derived facts by identifying implicit relationships. "A customer support agent's graph naturally strengthens paths through product docs and refund policies while letting rarely-queried HR edges decay. The graph develops its own sense of relevance over time." [Source: raw/akshay_pachaar-2043745099792953508.md]

Cognee ships **14 search modes** (not enumerated in the source) and supports multi-tenancy at the graph level — per-dataset permissions for read/write/delete/share, rather than namespace separation. [Source: raw/akshay_pachaar-2043745099792953508.md]

### When to Use Cognee vs. Flat Memory

Pachaar's decision rule: "If your queries only need similarity search ('find conversations like this one'), vector-only memory works. The moment queries cross entity boundaries ('Was Alice's project affected by Tuesday's outage?'), you need graph traversal." Teams that wire together separate vector + graph + relational stores themselves typically burn weeks on infrastructure for a memory layer that still doesn't learn from its own usage. [Source: raw/akshay_pachaar-2043745099792953508.md]

This provides an explicit counterpoint to Tan's markdown-is-enough position: Pachaar argues markdown works fine until the first multi-hop business question, at which point you need the graph layer. Tan's implicit counter is that skills and resolvers encode enough relational structure for most workflows; the graph problem only dominates when you're building a general-purpose brain over thousands of documents. Both views likely hold in their native regimes.

## The Critique of Markdown-as-Database

Kevin Gu (CTO of Dex) pushes back on the "filesystem for everything" position, arguing that the ongoing romanticization of putting everything in markdown files is "reinventing the database in the worst possible substrate." [Source: raw/kevingu-2031889622385729730.md]

His core arguments:

- **The data modeling problem:** The moment you need to query across your graph (e.g., "find all decisions that contradict the current roadmap"), you are doing full-text search across thousands of files. A real database already gives you joins, indexes, and constraints. Markdown files just give you grep. [Source: raw/kevingu-2031889622385729730.md]
- **Maintenance and drift:** If your source of truth is a Google Doc and you have extracted claims into a markdown note, you now have two copies. Two copies drift. A database with a reference to the original source (a Drive file ID, a Slack message URL) can cascade updates when the source changes. Dependency tracked, provenance preserved, zero drift by design. [Source: raw/kevingu-2031889622385729730.md]
- **The flattening problem:** A note might contain a decision, a guess, a stale belief, or someone's personal framing. All of it becomes text, all of it becomes traversable, all of it looks more structurally similar than it really is. A markdown graph makes many things readable; it does not make them governable. [Source: raw/kevingu-2031889622385729730.md]
- **Schema is context too:** When data lives in a typed schema, the agent knows how it was generated, what kind of thing it is reading, what other records depend on it, and what it depends on. It can reason about provenance, not just content. [Source: raw/kevingu-2031889622385729730.md]

Gu's proposed architecture has three layers: work apps as the source of truth, a file layer for hot access metadata and base instructions, and a database as the graph storing derived information, structure, relationships, and dependency chains. [Source: raw/kevingu-2031889622385729730.md]

## MiroFish: Temporal Fact Lifecycle with Graph Edges

MiroFish's memory architecture tracks fact validity explicitly at the graph-edge level. Each edge carries three timestamps: `valid_at` (when the fact became true), `invalid_at` (when it was superseded), and `expired_at` (when it was archived). The query result type `PanoramaResult` separates `active_facts` from `historical_facts`, allowing the agent to reason over both current state and past-state snapshots within the same response. This enables time-aware reasoning over an evolving world state -- a significant step beyond systems that store facts without provenance or lifespan. [Source: raw/code-research-666ghj-mirofish-2026-04-15.md]

## OpenClaw: Autonomous Memory Consolidation via "Dreaming"

OpenClaw implements the most sophisticated autonomous memory management system discovered in any open-source agent harness. Its memory architecture operates across 5 parallel systems: in-context workspace files (MEMORY.md, SOUL.md, daily files), a SQLite+sqlite-vec vector store for hybrid FTS+vector search, an external QMD binary as an alternative semantic search backend, session transcript JSONL files indexed into the vector store, and per-agent sessions.json metadata with compaction checkpoints. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

The most novel pattern is the **dreaming system** -- three cron-scheduled phases of autonomous background memory consolidation inspired by sleep neuroscience. Light dreaming (every 6 hours) performs recency-based deduplication of short-term memory fragments. Deep dreaming (nightly at 3am) promotes high-recall short-term fragments into durable long-term memory, using configurable thresholds (`minRecallCount: 3`, `minUniqueQueries: 3`, `recencyHalfLifeDays: 14`). REM dreaming (weekly) synthesizes patterns across all memory sources. A `short-term-recall.json` file tracks recall frequency candidates for promotion. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

This is a MemGPT-inspired hierarchy, but with a critical difference: promotion between tiers is fully autonomous and background-scheduled, not driven by explicit agent API calls. The agent is not aware of the tier boundary -- the harness manages it silently. This contrasts with MemGPT where the agent itself manages transitions between tiers.

### Deep Recovery Self-Healing

OpenClaw's deep dreaming phase includes a self-healing recovery mechanism that fires when memory health drops below 35%. This triggers a re-examination of the previous 30 days of memory candidates for promotion. If a candidate meets a 97% confidence threshold it is auto-promoted to long-term memory without further human or agent intervention. This loop prevents "promotion starvation" -- a failure mode where short-term fragments never reach long-term storage because the normal nightly promotion pass consistently falls short of threshold. The health metric itself serves as the circuit breaker; recovery only fires when systemic degradation is detected, not after every failed candidate. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

### Corpus Self-Ingestion Detection

OpenClaw's `dreaming-repair.ts` includes logic to detect a failure mode unique to cyclic LLM read/write systems: when the LLM's own narrative generation output has leaked back into the corpus as if it were a new memory. The detector identifies corrupted files where auto-generated narrative prose was written to a memory location. Rather than deleting these files, the harness archives them -- preserving a record of the contamination event for post-mortem analysis. This is a self-protective measure against the feedback loop where an LLM reads its own prior outputs and re-ingests them as facts. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

OpenClaw's **memory flush** is another distinctive pattern: rather than extracting key facts via a simple append operation, the harness spawns a complete embedded Pi agent turn with `trigger: "memory"` whose sole job is to compose memory content. The LLM decides *what* to remember, but the harness constrains *where* it writes (only `memory/YYYY-MM-DD.md`). This fires at a token threshold before compaction, ensuring important context is persisted before summarization discards it. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

After compaction, OpenClaw applies a **post-compaction context refresh** that re-reads `AGENTS.md` "Session Startup" and "Red Lines" sections and injects them as a system event with the current date substituted for `YYYY-MM-DD` placeholders. This means the agent will look for today's daily memory file even after a compaction, regardless of what date was in the old summary. The entire memory system is plugin-extensible via a single `registerMemoryCapability()` call that can replace the entire memory subsystem. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

## OpenHands: Event-Sourced Condenser Architecture

OpenHands implements the most extensive pluggable condenser system found in any open-source agent harness. Nine composable condenser implementations are available: NoOp, ObservationMasking, BrowserOutput, RecentEvents, ConversationWindow, AmortizedForgetting, LLMAttention, LLMSummarizing, and StructuredSummary -- plus a Pipeline condenser that chains them in any order. Each condenser operates on the agent's event view rather than mutating stored history, and any combination can be assembled declaratively. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

A distinctive design decision is the **condenser-returns-action pattern**: when condensation is needed, the agent's `step()` method returns a `CondensationAction` rather than a tool call or message. This makes condensation a first-class, auditable event in the agent's action stream instead of a silent background side-effect. Consumers can observe when condensation occurred, replay history including condensation events, and reason about the agent's context management decisions from the trace alone. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

The underlying persistence layer is an event store where each event is written as an individual JSON file to a FileStore backend (Local, S3, or GCS). Session restore works by replaying all event files in order to rebuild state -- condensation is a view-level operation, not a deletion from storage. This event-sourcing approach means that nothing is ever permanently lost from the record; a fuller view of history can always be reconstructed if needed. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

Condensation triggers follow a dual reactive-plus-proactive pattern. Proactive condensation fires when the agent's view size exceeds a configured `max_size`. Reactive condensation catches `ContextWindowExceededError` from the LLM API when a request is rejected mid-turn. A stuck-loop detector prevents infinite condensation cycles where repeated condensation fails to reduce view size below the threshold. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

One counterintuitive finding from OpenHands: there is no vector search anywhere in the memory pipeline. Microagent triggers use substring matching against file paths and content. The system achieved 77.6% on SWE-Bench without any semantic retrieval -- a practical data point for the "filesystem tools beat specialized memory" argument, at production scale. OpenHands also adds prompt caching support directly in `ConversationMemory`, marking the system message and the last user/tool message as `cache_prompt=True` to maximize Anthropic cache hit rates across turns. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

## OpenCode: Triple Storage and Snapshot Time-Travel

OpenCode uses three orthogonal storage layers with clearly separated concerns. SQLite with Drizzle ORM handles structured relational data: sessions, messages, message parts, and todos. A JSON filesystem layer stores large binary blobs and diffs that would be expensive to put in a relational table. A git bare repository records a snapshot of the working tree after every LLM step. Each layer is optimized for its access pattern, and the combination enables capabilities none of the three could provide alone. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

The git bare repo is the most novel component: it enables **snapshot-per-step time-travel**. After every step, the harness records git tree hashes alongside the session metadata. Any point in the session timeline can be restored via `SessionRevert`, giving the agent (and the user) a first-class undo system that operates at the granularity of individual LLM turns. This is a fundamentally different approach to session recovery than the event-sourcing model in OpenHands -- OpenCode reconstructs state by resetting git, not by replaying events. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

Context compaction uses a two-trigger design. Predictive compaction fires before the context is full, based on token counting, so the agent never hits a hard provider rejection in the normal case. Reactive compaction fires when a provider actually rejects a request. Within a compaction event, two-tier reduction applies: pruning erases old tool outputs in-place (cheaper, reversible within the session), and compaction rewrites history via an LLM summarization call (more aggressive, used when pruning is insufficient). One category of tool output is permanently protected from pruning: `PRUNE_PROTECTED_TOOLS = ["skill"]`, ensuring that skill tool results survive any amount of context reduction. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

Two additional memory patterns are worth noting. AGENTS.md instruction files are tracked per-message via a claims-based deduplication system that prevents the same instruction block from being injected more than once per conversation, keeping context lean without losing coverage. The `TodoWrite` tool is backed by SQL with atomic delete-and-reinsert semantics per session, giving the agent a structured, queryable short-term task list that survives compaction and is isolated per session by design. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Claude Code Memory Architecture Details (2026-04-14 Research)

### MEMORY.md 200-Line Index Cap

Claude Code's memdir design uses `MEMORY.md` as a bounded index file with a hard cap of 200 lines, alongside individual topic files with YAML frontmatter. This cap prevents unbounded growth of the primary index -- as new memories are written, the index is kept within this limit, and detail lives in per-topic files rather than accumulating in a single flat document. The separation of the bounded index from unbounded individual files is the key structural decision: the index stays navigable while the full memory corpus can grow. [Source: raw/code-research-claude-code-2026-04-14.md]

### No Project-Level autoMemoryDirectory Override

The memory path used by Claude Code's memdir system cannot be set or overridden from `.claude/settings.json` at the project level. This is an explicit defense-in-depth security decision: it prevents a malicious repository from redirecting the agent's memory writes to sensitive filesystem locations. A project-level `autoMemoryDirectory` override would allow an adversarial repo to set the memory path to `/etc/cron.d/`, `~/.ssh/`, or similar sensitive targets. By making the memory path non-configurable from project settings, the harness ensures that memory writes always land in a controlled location regardless of what a repository's config files specify. [Source: raw/code-research-claude-code-2026-04-14.md]

## OpenClaw Memory Architecture Details (2026-04-14 Research)

### Temporal Decay and MMR in Hybrid Search

OpenClaw's vector search implementation goes beyond naive cosine similarity in two important ways. First, results are scored with recency weighting -- memories decay in relevance as they age, so a recent memory with moderate semantic similarity can outrank an older memory with high semantic similarity. Second, results are reranked using Maximal Marginal Relevance (MMR), a diversity algorithm that penalizes retrieving near-duplicate memories. MMR selects each successive result to maximize both relevance to the query and diversity from already-selected results. The combination of temporal decay and MMR means the retrieval set is both fresh and non-redundant, which matters when a corpus accumulates many similar memories over time. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

### Plugin-Extensible Memory via registerMemoryCapability

OpenClaw's memory subsystem is not a hardcoded implementation but a swappable plugin registered via a single `registerMemoryCapability()` call. This call replaces the entire memory system: storage backend, retrieval logic, consolidation policies, and all. The harness provides the lifecycle hooks; the capability provides the implementation. This design makes it possible to swap from the default SQLite+sqlite-vec backend to an alternative without touching harness code. It also allows downstream consumers of the harness to inject custom memory implementations for domain-specific retention policies or storage requirements. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

### Daily Memory Files Labeled "Untrusted Context"

OpenClaw's system prompt explicitly marks injected daily memory files -- `memory/YYYY-MM-DD.md` -- as "untrusted context" when presenting them to the model. This label sets appropriate model expectations for the quality and reliability of recalled content: daily memory files are written by the agent itself and may contain errors, outdated information, or hallucinations from prior sessions. By framing them as untrusted, the system prompt instructs the model to treat recalled memories as fallible inputs requiring corroboration rather than ground truth. This is a calibration mechanism: the model's prior sessions inform its current reasoning without overriding it. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## What Survives Compaction Matters

When a context window fills up, something has to give. @systematicls notes that agents are "still atrocious at connecting the dots, filling in the gaps, or making assumptions" -- and that compaction decisions are where this weakness becomes critical. [Source: raw/systematicls-2028814227004395561.md] One of the most important rules to include in CLAUDE.md is a rule on how to deal with grabbing context after compaction: re-reading the task plan and re-reading the relevant files before continuing. [Source: raw/systematicls-2028814227004395561.md]

If the harness summarizes away a critical constraint ("the client requires Python 3.8 compatibility"), the agent will violate that constraint in future turns with no awareness that it ever knew otherwise.

## Related

- [Tool Design Patterns](tool-design-patterns.md) -- Filesystem tools as memory tools, primitives over integrations
- [Practical Best Practices](practical-best-practices.md) -- Progressive disclosure, CLAUDE.md as directory, context curation
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- Context resets vs. compaction, progress files as memory
- [Claude Code Architecture](claude-code-architecture.md) -- Multi-level memory hierarchy built directly into the harness
- [Deep Research Agents](deep-research-agents.md) -- external memory as non-negotiable for deep research, filesystem-based inter-agent communication
- [Agentic Design Patterns](agentic-design-patterns.md) -- Planning and Reflection patterns as they relate to memory management strategies
- [Multi-Agent Reliability](multi-agent-reliability.md) -- credibility scoring as a memory-layer concern: which retrieved information to trust
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) -- git-as-state-machine and results.tsv as memory patterns in autonomous research loops
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- OpenHands event-driven loop and OpenCode while(true) loop with DB as authority
- [Tool Design Patterns](tool-design-patterns.md) -- OpenCode fuzzy edit cascade and OpenHands security-risk-as-parameter
- [Thin Harness, Fat Skills](thin-harness-fat-skills.md) -- the Chase-Tan memory-ownership debate: closed-harness lock-in vs. "memory is markdown, brain is a git repo"
- [Self-Evolving Agents and Skillify](self-evolving-agents.md) -- memory as a first-class evolvable resource (Autogenesis RSPL); resolvers and skillify as self-healing memory patterns

## Open Questions

- Where is the crossover point where filesystem-based memory breaks down and a structured database becomes necessary? Letta's benchmarks show filesystem wins on LoCoMo, but Kevin Gu argues this fails at organizational scale with drift and governance problems.
- How should compaction policies be designed? No standard approach exists for deciding what information is load-bearing versus expendable during compaction.
- Can the MemGPT agent-driven memory management approach work when the memory policy itself requires domain-specific knowledge the agent may not have?

## Sources

- [raw/sarahwooders-2040121230473457921.md](../raw/sarahwooders-2040121230473457921.md) -- Sarah Wooders (Letta CTO) on memory as core harness function, not plugin. Context Constitution and Context Repositories.
- [raw/letta-com-blog-benchmarking-ai-agent-memory.md](../raw/letta-com-blog-benchmarking-ai-agent-memory.md) -- Letta blog on benchmarking agent memory. Filesystem tools (74.0%) beat Mem0 (68.5%) on LoCoMo. MemGPT memory hierarchy.
- [raw/systematicls-2028814227004395561.md](../raw/systematicls-2028814227004395561.md) -- @systematicls on "context is everything," stripping dependencies, CLAUDE.md as directory, compaction handling.
- [raw/kevingu-2031889622385729730.md](../raw/kevingu-2031889622385729730.md) -- Kevin Gu critique of markdown-as-database. Arguments for typed schemas, provenance tracking, and structured databases over filesystem-only memory.
- [raw/arxiv-org-html-2501-09136v4.md](../raw/arxiv-org-html-2501-09136v4.md) -- Singh et al., 2025. Agentic RAG survey: evolution from static RAG to agent-driven retrieval, transforming retrieval into active memory management.
- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) -- Anthropic, Apr 2026. Research plan saved to Memory for persistence beyond 200K token truncation. Subagent output to filesystem. Long-horizon conversation management via phase summarization and fresh subagent contexts.
- [raw/code-research-claude-code-2026-04-15.md](../raw/code-research-claude-code-2026-04-15.md) -- Code research, Apr 2026. Claude Code's 5-layer memory hierarchy, 4-layer compaction cascade, extractMemories background RAG agent, post-compact file re-injection.
- [raw/code-research-karpathy-autoresearch-2026-04-15.md](../raw/code-research-karpathy-autoresearch-2026-04-15.md) -- Code research, Apr 2026. Git-as-experiment-database, outcome-memory separated from code-state via gitignored results.tsv, minimal-signal extraction via grep.
- [raw/code-research-openclaw-openclaw-2026-04-15.md](../raw/code-research-openclaw-openclaw-2026-04-15.md) -- Code research, Apr 2026. OpenClaw's 5-system memory architecture, 3-tier hierarchy with dreaming consolidation (light/deep/REM), agentic memory flush, post-compaction context refresh, plugin-extensible memory via registerMemoryCapability().
- [raw/code-research-all-hands-ai-openhands-2026-04-15.md](../raw/code-research-all-hands-ai-openhands-2026-04-15.md) -- Code research, Apr 2026. OpenHands' 9-condenser pluggable pipeline, CondensationAction as a first-class stream event, event-sourced FileStore per-event JSON, dual proactive+reactive condensation triggers, no vector search (77.6% SWE-Bench via substring matching), prompt caching in ConversationMemory.
- [raw/code-research-anomalyco-opencode-2026-04-15.md](../raw/code-research-anomalyco-opencode-2026-04-15.md) -- Code research, Apr 2026. OpenCode's triple storage (SQLite/Drizzle + JSON filesystem + git bare repo), snapshot-per-step time-travel via SessionRevert, dual predictive+reactive compaction with two-tier pruning/LLM rewrite, PRUNE_PROTECTED_TOOLS=["skill"], claims-based AGENTS.md deduplication, SQL-backed TodoWrite.
- [raw/code-research-666ghj-mirofish-2026-04-15.md](../raw/code-research-666ghj-mirofish-2026-04-15.md) -- Code research, Apr 2026. MiroFish's Zep graph edges with valid_at/invalid_at/expired_at timestamps, PanoramaResult separating active_facts from historical_facts for time-aware reasoning.
- [raw/code-research-claude-code-2026-04-14.md](../raw/code-research-claude-code-2026-04-14.md) -- Code research, Apr 2026. MEMORY.md 200-line index cap in memdir design; no project-level autoMemoryDirectory override as defense-in-depth security for memory paths.
- [raw/code-research-openclaw-openclaw-2026-04-14.md](../raw/code-research-openclaw-openclaw-2026-04-14.md) -- Code research, Apr 2026. Temporal decay + MMR reranking in hybrid vector search; plugin-extensible memory via registerMemoryCapability(); daily memory files labeled as untrusted context.
- [raw/akshay_pachaar-2043745099792953508.md](../raw/akshay_pachaar-2043745099792953508.md) -- Akshay Pachaar, Apr 13, 2026. "Build Agents that never forget" -- seven failure modes without real memory, cognitive-science frame (sensory/working/long-term, episodic/semantic/procedural), four-layer evolution (none → list → markdown → vectors), introduction of Cognee's three-store architecture (relational + vector + graph), `memify()` RL-inspired graph optimization, 14 search modes.
- [raw/hwchase17-2042978500567609738.md](../raw/hwchase17-2042978500567609738.md) -- Harrison Chase, Apr 11, 2026. "Your harness, your memory" -- three degrees of memory lock-in (stateful API, closed harness, whole-harness-behind-API), Anthropic's Claude Managed Agents as worst case.
- [raw/garrytan-2043198780800197025.md](../raw/garrytan-2043198780800197025.md) -- Garry Tan, Apr 11, 2026. "If your memory dies when your harness dies, you built the harness too thick. Memory is markdown. Brain is a git repo."
