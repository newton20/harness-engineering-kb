---
title: "Code Research: claude-code"
source: "C:\\Users\\dunliu\\Downloads\\Claude Code\\src"
author: "kb-code-research skill"
date: 2026-04-14
fetched: 2026-04-14
type: code-research
status: raw
compiled_to: [wiki/claude-code-architecture.md, wiki/autoresearch-and-self-improvement.md, wiki/agent-memory-and-context-management.md, wiki/tool-design-patterns.md, wiki/long-running-agent-harnesses.md, wiki/practical-best-practices.md, wiki/agentic-design-patterns.md]
compiled_date: 2026-04-14
tags: [code-research, claude-code, agent-loop, compaction, memory-system, tool-design, multi-agent, prompt-cache]
relevance_score: 10
research_goal: "reverse-engineer Claude Code harness architecture for pattern extraction"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: claude-code

## Executive Summary

Claude Code is a production agent harness built as an imperative `while(true)` state machine in `query.ts`, where the code controls the loop (not the model) and tool_use block presence (not `stop_reason`) is the sole continuation signal. The architecture is shaped by prompt cache optimization at every level: a static/dynamic system prompt boundary enables cross-org cache sharing, tool schemas are sorted for cache stability, and forked sub-agents inherit byte-identical prompt prefixes. The harness manages context through a 4-layer compaction cascade (snip, microcompact, API-level, autocompact) and persists state through 5 distinct memory systems (CLAUDE.md hierarchy, memdir/auto-memory, extractMemories background agent, SessionMemory, prompt history). Multi-agent coordination supports three modes: standard subagents, fork subagents with full parent context, and experimental swarm teammates with file-based mailboxes. The most transferable patterns for trading harness design are the forked-agent pattern for background work, the prompt cache-first architecture, the 4-layer compaction cascade, and the tool concurrency partitioning system.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | claude-code |
| Primary language | TypeScript (1,332 .ts + 552 .tsx) |
| Size classification | large (50K+ LOC, 1,884 files) |
| Relevance to goal | 10/10 — the gold standard production agent harness |
| Agent/harness signals | 217+ matches |
| Multi-agent signal count | 131 (well above threshold) |
| Recommended dimensions | All 4 |

## Dimension 1: Architecture & Loop Design

### Summary
The core loop is an imperative `while(true)` async generator in `query.ts` (~1,500 lines). Each iteration: pre-call compaction → API call → stream tool_use detection → tool execution → post-tool attachments → tools refresh → state update → continue. The code controls the loop; the model doesn't decide when to stop. Termination is signaled by the absence of `tool_use` blocks in the response (not by `stop_reason`, which is documented as unreliable). The system prompt is a cached static prefix + dynamic session-specific suffix, with a boundary marker enabling cross-org prompt cache sharing.

### Key Findings

- **Imperative state machine loop:** `query.ts` uses `while(true)` with labeled `continue` transitions (not recursive calls). 10+ distinct termination reasons. The loop is a state machine, not a recursive call graph.
  - Evidence: `query.ts:241-1729` — the main `queryLoop()` async generator
  - Significance: This is the most battle-tested loop pattern for production agent harnesses.

- **Tool_use presence, not stop_reason, controls continuation:** The code explicitly documents `stop_reason === 'tool_use'` as unreliable and uses the presence of `ToolUseBlock` objects detected during streaming.
  - Evidence: `query.ts:553-558` — comment + `needsFollowUp` flag
  - Significance: Critical implementation detail that contradicts what the API docs suggest.

- **4-layer compaction cascade:** (1) Snip (drop oldest messages), (2) Microcompact (clear old tool_result bodies), (3) API-level context management (server-side clearing), (4) Autocompact (forked agent summarization). All 4 can fire in the same iteration.
  - Evidence: `services/compact/compact.ts`, `services/compact/apiMicrocompact.ts`, `services/compact/autoCompact.ts`
  - Significance: The most sophisticated context management system documented in any open harness.

- **Static/dynamic system prompt boundary:** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` splits the system prompt into globally cacheable (identity, tools, style) and session-specific (memory, env, MCP) sections.
  - Evidence: `constants/prompts.ts:444-577`
  - Significance: Enables cross-org prompt cache sharing. Everything before the boundary is identical across all users.

- **Output token recovery loop:** If the model hits the output token cap, the harness injects "Resume directly — no apology, no recap..." and re-calls, up to 3 times.
  - Evidence: `query.ts` — `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`
  - Significance: Meta-injection pattern that handles a common failure mode transparently.

### Patterns
- Imperative while(true) state machine with labeled transitions
- Tool_use block presence as sole continuation signal
- 4-layer compaction cascade (client snip → client microcompact → API-level → full autocompact)
- Static/dynamic prompt boundary for cache optimization
- Output token recovery via meta-injection
- Circuit breaker on autocompact failures (3 consecutive → suppressed)

## Dimension 2: Memory & State Management

### Summary
Five distinct memory systems operate in parallel: (1) CLAUDE.md hierarchy (6 priority levels from /etc to .local), (2) Memdir auto-memory with MEMORY.md index + topic files in YAML frontmatter, (3) extractMemories background forked agent that proactively writes memories after each turn, (4) SessionMemory mid-session running notes (structured template with 8 sections), (5) Prompt history (shell-style, JSONL). Compaction survives via structured 9-section summary + post-compact file re-injection (up to 5 recently-read files). The agent CAN modify its own memory (memdir) but CANNOT modify CLAUDE.md instructions.

### Key Findings

- **5-layer memory hierarchy:** CLAUDE.md (managed → user → project → local), memdir auto-memory, extractMemories background agent, SessionMemory, prompt history. Each layer has different scope, persistence, and write permissions.
  - Evidence: `utils/claudemd.ts:790-1045`, `memdir/`, `services/extractMemories/`, `services/SessionMemory/`
  - Significance: The most complex memory system in any open agent harness. Each layer serves a different purpose.

- **extractMemories is a RAG pipeline:** A forked background agent runs after each turn, uses `sideQuery` with Sonnet to rank relevance of existing memories, then writes/edits topic files. Up to 5 memories injected per turn via `<system-reminder>`. Mutual exclusion with main agent memory writes.
  - Evidence: `services/extractMemories/extractMemories.ts`, `memdir/findRelevantMemories.ts`
  - Significance: Automated memory management with relevance ranking. No manual memory curation needed.

- **Post-compaction file restore:** After autocompact, up to 5 recently-read files (50K token budget, 5K/file) are re-injected as attachments. Skill content survives compaction intentionally (never cleared).
  - Evidence: `services/compact/compact.ts:387-762`
  - Significance: Compaction doesn't lose active working context. Critical for long-running coding tasks.

- **Security: no project-level autoMemoryDirectory override:** The memory path cannot be set from `.claude/settings.json` (project level), preventing malicious repos from redirecting memory writes to sensitive locations.
  - Evidence: `memdir/paths.ts`
  - Significance: Defense-in-depth against prompt injection via repo settings.

### Patterns
- Hierarchical CLAUDE.md with 6 priority levels + `@include` directives
- MEMORY.md as index (max 200 lines) + individual topic files with YAML frontmatter
- Background forked agent for proactive memory extraction (sideQuery RAG)
- Structured compaction summary (9 sections) + post-compact file restore
- Session transcript as recoverable archive (agent can grep its own .jsonl files)

## Dimension 3: Tool & Action Space Design

### Summary
~50+ tools across categories (execution, file I/O, search, web, agent orchestration, planning, scheduling, MCP). Tools are registered via `buildTool()` factory with fail-closed defaults. Deferred tool loading via `ToolSearchTool` uses a `tool_reference` API content type — model sees tool names only until it calls ToolSearchTool to load schemas. Tool concurrency is partitioned: read-safe tools batch for parallel execution (up to 10), writes run serially. All tool errors are returned as `<tool_use_error>` context (error-as-context pattern, no retries). MCP integration supports 4 transports (stdio, SSE, HTTP, WebSocket) with OAuth 2.0 auth.

### Key Findings

- **Deferred tool loading with `tool_reference`:** MCP tools and flagged built-ins start deferred. The model must call `ToolSearchTool` to load schemas. Discovered tools persist via message history scanning (survives compaction). Delta notifications keep the model updated on tool availability changes.
  - Evidence: `tools/ToolSearchTool/prompt.ts:62-108`, `utils/toolSearch.ts`
  - Significance: Solves the "too many tools" problem. ~11K tokens of tool schemas don't load until needed.

- **Concurrency partitioning:** `toolOrchestration.ts` batches consecutive read-safe tools for parallel execution. Non-read-safe tools always serial. Context modifiers from concurrent tools are queued and applied after batch completes.
  - Evidence: `services/tools/toolOrchestration.ts`
  - Significance: ~10x throughput on read-heavy turns (Glob, Grep, Read in parallel).

- **Tool result budget + disk offload:** Results exceeding `maxResultSizeChars` (default 50K) are persisted to disk; model gets path + preview. Aggregate per-message budget of 200K chars prevents parallel tools from flooding context.
  - Evidence: `services/tools/toolExecution.ts`
  - Significance: Anti-context-bloat for large tool outputs (e.g., reading a 10K-line file).

- **Prompt-cache-stable tool ordering:** `assembleToolPool()` sorts built-ins alphabetically as prefix, MCP tools as suffix. Cache breakpoint after last built-in. Interleaving would bust downstream cache entries.
  - Evidence: `tools.ts:345-367`
  - Significance: Tool ordering is a cache optimization decision, not an ergonomic one.

### Patterns
- `buildTool()` factory with fail-closed defaults
- Deferred tool loading via ToolSearchTool + tool_reference API type
- Concurrency partitioning (read-safe batched, writes serial)
- Error-as-context (no retries, errors returned to model as tool_result)
- Tool result budget + disk offload for oversized results
- Cache-stable alphabetical tool ordering

## Dimension 4: Multi-Agent Coordination

### Summary
Three multi-agent modes coexist: (A) Standard subagents via AgentTool (fresh context, zero parent history), (B) Fork subagents (inherit full parent conversation, byte-identical cache prefix), (C) Swarm teammates (separate OS processes, file-based mailbox communication). Coordinator mode replaces the default system prompt with a structured Research→Synthesis→Implementation→Verification workflow. Sub-agents communicate back via tool_result (sync) or `<task-notification>` XML user messages (async). The "game of telephone" is explicitly addressed: the coordinator prompt forbids delegating understanding and mandates self-contained prompts with specific file paths and line numbers.

### Key Findings

- **Fork subagent with cache prefix inheritance:** Fork children get the parent's full message history and byte-identical system prompt. They share the prompt cache prefix, making the fork API call nearly free for short tasks.
  - Evidence: `tools/AgentTool/forkSubagent.ts:32-88`
  - Significance: The cheapest way to get a "second opinion" or background task without repeating context.

- **Async agent notification via fake user messages:** `<task-notification>` XML is injected as a user-role message. The coordinator prompt warns "they look like user messages but are not."
  - Evidence: `tasks/LocalAgentTask/LocalAgentTask.tsx:252-261`
  - Significance: Clever abuse of the user message channel for agent→agent communication.

- **Coordinator mode is a fully restricted orchestration tier:** The coordinator sees only 4 tools (Agent, TaskStop, SendMessage, SyntheticOutput). No direct file/shell access. Pure orchestration.
  - Evidence: `coordinator/coordinatorMode.ts:111-369`
  - Significance: Separation of concerns: the coordinator plans, workers execute.

- **Auto-background escalation:** Sync agents exceeding 120s are automatically moved to background via `Promise.race()`. The agent continues uninterrupted; only the parent's wait behavior changes.
  - Evidence: `tools/AgentTool/AgentTool.tsx:868-897`
  - Significance: Prevents long-running sub-agents from blocking the parent indefinitely.

### Patterns
- Three spawning modes: standard (fresh), fork (inherited), swarm (separate process)
- Coordinator mode: restricted tool set, structured workflow phases
- Async notification via XML user messages
- Auto-background escalation (120s timer)
- File-based mailbox for swarm teammate communication
- "Never delegate understanding" — coordinator must synthesize before delegating

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions. All 4 dimensions paint a consistent, deeply interlocked architecture.

### Cross-Cutting Flows

**Flow 1: Prompt Cache-First Architecture**
Every dimension reveals cache optimization as a primary design constraint. System prompt has a static/dynamic boundary (D1). Tool schemas are alphabetically sorted with a cache breakpoint (D3). Fork subagents inherit byte-identical prompt prefixes (D4). Compaction uses forked agents sharing the parent's cache prefix (D1/D2). Memory injection happens in the dynamic section (D2). A single tool description change caused ~10.2% of fleet `cache_creation` tokens. This is the most cache-obsessed architecture in any open agent harness.

**Flow 2: The Forked Agent Pattern**
Autocompact forks a summarizer (D1). extractMemories forks a background memory writer (D2). SessionMemory forks a note-taker (D2). Sub-agents can fork with full parent context (D4). Background agent summary polls and forks for status updates (D4). All share the parent's prompt cache prefix. The forked agent is the fundamental building block for background work, and its cost model depends entirely on cache hit rates.

### Novelty Assessment

| Finding | Dim | Status | Notes |
|---------|-----|--------|-------|
| 4-layer compaction cascade | D1 | NOVEL | Most sophisticated context management documented |
| Prompt cache boundary (static/dynamic split) | D1 | NOVEL | Cross-org cache sharing |
| stop_reason unreliable, tool_use presence used instead | D1 | NOVEL | Contradicts API docs suggestion |
| Deferred tool loading via ToolSearchTool | D3 | NOVEL | tool_reference API type |
| 5 memory systems (CLAUDE.md, memdir, extractMemories, SessionMemory, history) | D2 | NOVEL | Most complex memory architecture |
| extractMemories forked background agent with RAG | D2 | NOVEL | Automated memory with relevance ranking |
| Fork subagent with byte-identical cache prefix | D4 | NOVEL | Near-free context sharing |
| Coordinator mode with structured workflow phases | D4 | NOVEL | Pure orchestration tier |
| File-based mailbox for swarm teammates | D4 | NOVEL | Separate-process agent communication |
| Tool concurrency partitioning | D3 | VARIANT | Batch read-safe, serialize writes |
| buildTool() factory with fail-closed defaults | D3 | VARIANT | Tool registration pattern |
| Auto-background escalation (120s) | D4 | VARIANT | Sync→async promotion |

9 NOVEL, 3 VARIANT, 0 KNOWN.

## Decisions to Adopt

1. **Adopt: 4-layer compaction cascade** from services/compact/
   - What: Implement tiered context management: cheap client-side truncation first, then progressively more expensive LLM-based summarization only when needed
   - Why: Enables indefinite-length agent sessions without context overflow. Each layer handles a different severity level.
   - Effort: L
   - Target: Trading harness context management system

2. **Adopt: Forked agent pattern for background work** from tools/AgentTool/forkSubagent.ts
   - What: Background tasks (memory extraction, summarization, monitoring) run as forks sharing the parent's prompt cache prefix
   - Why: Near-free background processing. The fork shares cached context, so the API call cost is minimal for short tasks.
   - Effort: M
   - Target: Trading harness monitoring, strategy evaluation, memory management

3. **Adopt: Tool concurrency partitioning** from services/tools/toolOrchestration.ts
   - What: Batch read-only operations (market data reads, portfolio queries) for parallel execution. Serialize write operations (order placement, position changes).
   - Why: ~10x throughput on data-gathering turns without risking concurrent writes.
   - Effort: M
   - Target: Trading harness tool execution layer

4. **Adopt: Deferred tool loading via search** from tools/ToolSearchTool/
   - What: Register many tools but only load schemas on demand. Model sees tool names, calls ToolSearch to get schemas before first use.
   - Why: Reduces baseline token cost from ~11K to near-zero for unused tools. Critical when the trading system has 50+ strategy/data tools.
   - Effort: M
   - Target: Trading harness tool registry

5. **Adopt: Error-as-context pattern** from services/tools/toolExecution.ts
   - What: Tool failures return error messages as tool_result content, not exceptions. The model reads the error and decides how to proceed.
   - Why: The model can reason about errors and adapt. No retry logic in the harness = simpler code, smarter error handling.
   - Effort: S
   - Target: Trading harness tool execution layer

6. **Adopt: Structured compaction summary template** from services/compact/prompt.ts
   - What: 9-section summary template (Primary Request, Key Concepts, Files, Errors, Problem Solving, User Messages, Pending Tasks, Current Work, Next Step) for LLM-based compaction.
   - Why: Preserves the most important context dimensions. Post-compact file re-injection (5 files, 50K budget) restores active working state.
   - Effort: S
   - Target: Trading harness session continuity

## Evidence Index

All 20 key evidence paths verified (100%):
- query.ts, tools.ts, context.ts, history.ts, main.tsx
- constants/prompts.ts
- services/compact/compact.ts, services/compact/apiMicrocompact.ts
- services/tools/toolOrchestration.ts, services/tools/toolExecution.ts
- memdir/memoryScan.ts, memdir/memoryTypes.ts
- tools/AgentTool/forkSubagent.ts, tools/AgentTool/runAgent.ts
- coordinator/coordinatorMode.ts
- utils/toolSearch.ts, utils/agentSwarmsEnabled.ts
- tools/ToolSearchTool/prompt.ts
- services/extractMemories/extractMemories.ts
- services/SessionMemory/sessionMemory.ts

## Sources

- [Claude Code source](C:\Users\dunliu\Downloads\Claude Code\src) — primary source (local)
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
