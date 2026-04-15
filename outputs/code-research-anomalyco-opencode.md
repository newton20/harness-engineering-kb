---
title: "Code Research: anomalyco-opencode"
source: https://github.com/anomalyco/opencode
author: "kb-code-research skill"
date: 2026-04-14
fetched: 2026-04-14
type: code-research
status: raw
tags: [code-research, tool-design, context-management, fuzzy-editing]
relevance_score: 9
research_goal: "Analyze alternative harness design — compare with Claude Code. Look for novel patterns in tool design or context management."
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: anomalyco-opencode

## Executive Summary

OpenCode is an open-source AI coding agent built on TypeScript/Bun with an Effect-TS functional runtime, achieving feature parity with Claude Code while introducing several novel patterns. Its most architecturally distinctive contributions are: (1) a 9-strategy fuzzy edit replacer cascade that makes file editing extremely tolerant of LLM formatting drift, (2) tree-sitter AST-based permission detection for bash commands, (3) model-gated tool selection that automatically swaps edit/write for apply_patch on GPT models, (4) snapshot-per-step git-backed time travel enabling full session revert, and (5) doom-loop detection routed through the permission system rather than auto-stopping. The tool layer is exceptionally well-designed with description-as-template from .txt sidecar files, uniform truncation middleware, and an `invalid` tool that makes schema mismatches first-class results. Context management uses dual-trigger compaction (predictive + reactive overflow) with compaction modeled as a named agent. The multi-agent system supports resumable subagent sessions via task_id and mode-typed agent registry (primary/subagent/all).

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | anomalyco-opencode |
| Primary language | TypeScript (Bun runtime) |
| Size classification | large (502 core TS files, 1,056 total excl tests) |
| File count | ~4,596 total files |
| Last commit | 2026-04-14 (active) |
| Commit frequency | active |
| README quality | detailed (multi-language, installation, usage) |
| Relevance to goal | 9/10 — direct Claude Code alternative with novel tool and context patterns |
| Agent/harness signals | high (agent/, session/, tool/ directories) |
| Multi-agent signal count | 128 (above 10 threshold) |
| Recommended dimensions | All 4 |

## Dimension 1: Architecture & Loop Design

### Summary
OpenCode implements a code-controlled `while(true)` ReAct loop where the harness evaluates continuation after each LLM stream. Five termination paths exist: natural finish, max steps (via synthetic assistant prefill), processor stop signal, compaction failure, and doom-loop interrupt. Context overflow is handled through two distinct mechanisms: reactive compaction (a named "compaction" agent rewrites history into a structured summary) and proactive pruning (tool outputs erased in-place). An Effect-based PubSub event bus propagates all state changes. The v2 rewrite defines a new event-sourced schema but remains unimplemented stubs.

### Key Findings
- **Code-controlled while(true) loop:** Each iteration issues one LLM stream call; harness checks finish reason and pending tool calls. Model cannot self-terminate except via finish field.
  - Evidence: `packages/opencode/src/session/prompt.ts:1323-1540`

- **Max-steps via synthetic assistant prefill:** When step limit reached, a synthetic assistant-turn message is injected (not a system prompt), exploiting instruction-following for prefill.
  - Evidence: `packages/opencode/src/session/prompt/max-steps.txt`

- **Doom-loop detector:** 3 identical consecutive tool calls trigger a permission `ask("doom_loop")` — user can allow, deny, or auto-configure. Permission-gated, not auto-stopping.
  - Evidence: `packages/opencode/src/session/processor.ts:25,306-329`

- **Compaction as named agent:** Context compaction is a dedicated agent entry in the registry with its own model, `summary: true` flag, no tools. Runs as a separate LLM call.
  - Evidence: `packages/opencode/src/agent/agent.ts:188-202`

- **Two-tier context reduction:** Pruning erases old tool output text in-place (protecting 40K token window); compaction rewrites history via LLM. Prune avoids unnecessary LLM round-trips.
  - Evidence: `packages/opencode/src/session/compaction.ts:33-35,91-136`

- **7 provider-specific base system prompts:** anthropic.txt, gpt.txt, beast.txt, gemini.txt, kimi.txt, codex.txt, trinity.txt — selected at runtime by model API ID prefix.
  - Evidence: `packages/opencode/src/session/system.ts:20-34`

- **Two-part system prompt for cache efficiency:** System array normalized to exactly 2 entries (stable base, variable rest) to maximize prompt cache hits.
  - Evidence: `packages/opencode/src/session/llm.ts:114-119`

### Patterns
- Effect-based PubSub bus for all state changes
- Subtask queue in message store (pull-based, not push-based)
- DB re-read per iteration (database is single source of truth for loop state)
- Plugin hooks at compaction boundary for extensibility

## Dimension 2: Memory & State Management

### Summary
OpenCode implements a triple-storage architecture: SQLite/Drizzle for structured session data, JSON files for large blobs, and a git-backed snapshot store for filesystem state at each step. Compaction is dual-triggered (predictive via token counting + reactive via provider rejection). The snapshot-per-step system enables full time-travel revert to any point in a session. A TodoWrite tool provides agent-managed task tracking backed by SQL. Skill tool output is permanently protected from pruning.

### Key Findings
- **Triple storage:** SQLite (sessions/messages/parts/todos), JSON filesystem (diffs/blobs), git bare repo (snapshots per step).
  - Evidence: `packages/opencode/src/storage/db.ts:31-44`, `packages/opencode/src/storage/storage.ts:60-333`, `packages/opencode/src/snapshot/index.ts:83-91`

- **Snapshot-per-step time machine:** Every LLM step records git tree hashes. Any point in session timeline can be restored via `SessionRevert`.
  - Evidence: `packages/opencode/src/session/revert.ts:43-92`

- **Claims-based instruction deduplication:** AGENTS.md files tracked per-message to prevent repeat injection across loop iterations.
  - Evidence: `packages/opencode/src/session/instruction.ts:76-100`

- **Hierarchical AGENTS.md resolution:** Walks directory tree upward, attaches nearest matching instruction file when the agent reads a file in that directory.
  - Evidence: `packages/opencode/src/session/instruction.ts:186-227`

- **TodoWrite as SQL-backed agent memory:** Session-scoped todo list with atomic delete-and-reinsert. Agent-writable persistent state.
  - Evidence: `packages/opencode/src/session/todo.ts:41-59`

- **Skill output permanently protected from pruning:** `PRUNE_PROTECTED_TOOLS = ["skill"]` — skill instructions survive indefinitely in context.
  - Evidence: `packages/opencode/src/session/compaction.ts:33-35`

### Patterns
- Dual-trigger compaction (predictive + reactive overflow)
- Media stripping as first-class compaction feature
- Plugin hooks at compaction boundary
- Compaction failure safety valve (graceful stop, not infinite loop)

## Dimension 3: Tool & Action Space Design

### Summary
OpenCode has 17 built-in tools with an exceptionally well-engineered tool layer. Descriptions are loaded from .txt sidecar files with runtime template substitution. The edit tool uses a 9-strategy fuzzy replacer cascade. Bash permission detection uses tree-sitter AST parsing. Tool selection is model-adaptive (apply_patch for GPT, edit/write for others). An `invalid` tool makes schema mismatches first-class. MCP supports full OAuth 2.0/PKCE. The skill system supports remote CDN discovery.

### Key Findings
- **9-strategy fuzzy edit replacer cascade:** Exact → line-trimmed → block-anchor (Levenshtein) → whitespace-normalized → indentation-flexible → escape-normalized → trimmed-boundary → context-aware → multi-occurrence.
  - Evidence: `packages/opencode/src/tool/edit.ts:651-688`
  - Significance: Makes the edit tool extremely tolerant of LLM formatting drift — the most sophisticated fuzzy matching in any open-source harness.

- **Tree-sitter bash AST for permission detection:** Parses commands with tree-sitter-bash to extract file paths from file-modifying commands, builds glob-pattern permission requests.
  - Evidence: `packages/opencode/src/tool/bash.ts`
  - Significance: Significantly more sophisticated than string matching for security.

- **Model-gated tool selection:** GPT-4 models get apply_patch instead of edit/write automatically.
  - Evidence: `packages/opencode/src/tool/registry.ts:279-288`

- **`invalid` tool as first-class error handler:** Registered as a real tool; LLM routed to it when calling non-existent tools.
  - Evidence: `packages/opencode/src/tool/invalid.ts`

- **Description-as-template from .txt sidecar files:** Tool descriptions in separate .txt files with runtime substitutions (`${shell}`, `${directory}`, etc.).
  - Evidence: `packages/opencode/src/tool/bash.ts:461-510`

- **MCP with full OAuth 2.0/PKCE:** Local callback HTTP server, credential vault per-server, transport auto-negotiation (StreamableHTTP → SSE fallback), hot reload via ToolListChanged.
  - Evidence: `packages/opencode/src/mcp/index.ts:306-385`

- **Remote skill CDN discovery:** `Discovery.pull(url)` fetches index.json listing skills, downloads and caches SKILL.md files locally.
  - Evidence: `packages/opencode/src/skill/discovery.ts`

- **LSP integration (experimental):** 9 operations including goToDefinition, findReferences, hover, callHierarchy.
  - Evidence: `packages/opencode/src/tool/lsp.ts:12-22`

### Patterns
- Uniform truncation middleware on every tool output
- Permission-as-continuation via Effect Deferred
- Plugin hooks mutate tool definitions before LLM sees them
- ACP (Agent Communication Protocol) for IDE embedding — separate from MCP

## Dimension 4: Multi-Agent Coordination

### Summary
OpenCode is a full multi-agent system with mode-typed agent registry (primary/subagent/all). A primary agent dispatches to typed subagents via the `task` tool. Each subagent runs in its own child session with inherited-but-restricted permissions. Subagent sessions are resumable via task_id. The plan agent implements a structured 5-phase workflow (explore → design → review → write → exit) as prompt-injected instructions, not hard-coded TypeScript. LLM-directed parallelism enables multiple subagent dispatches per response.

### Key Findings
- **Mode-typed agent registry:** Agents classified as primary, subagent, or all. Subagents cannot be user-selected defaults.
  - Evidence: `packages/opencode/src/agent/agent.ts:30`

- **Resumable subagent sessions:** task_id returned from TaskTool enables multi-turn interactions with same child session.
  - Evidence: `packages/opencode/src/tool/task.ts:25-29,64-66`

- **Subtask queue as main-loop primitive:** Slash commands targeting subagent-mode agents create SubtaskPart records, handled without LLM token cost.
  - Evidence: `packages/opencode/src/session/prompt.ts:1332-1379`

- **Permission inheritance with scoped denial:** Child sessions inherit permissions but todowrite and recursive task access denied by default.
  - Evidence: `packages/opencode/src/tool/task.ts:74-98`

- **Plan agent as prompt-injected orchestrator:** 5-phase workflow encoded as `<system-reminder>`, not compiled behavior. Hackable but fragile.
  - Evidence: `packages/opencode/src/session/prompt.ts:274-354`

- **ACP for IDE embedding:** JSON-RPC-over-stdio protocol for external clients (Zed). Supports session fork, permission flows, usage tracking.
  - Evidence: `packages/opencode/src/acp/agent.ts`

### Patterns
- LLM-directed parallelism (multiple task calls in one response)
- Narrow text-only result channel (one message back from subagent)
- Session forking for user branching (not agent spawning)
- Recursive delegation prevented by permission system, not architecture

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions detected.

### Cross-Cutting Flows

**Flow 1: Tool Output Lifecycle (Dims 1, 2, 3)**
- Dim 3: Tool.wrap() applies truncation middleware to every tool result
- Dim 2: Pruning erases old tool outputs in-place (PRUNE_PROTECT = 40K tokens); skill tool outputs permanently protected
- Dim 1: If context still overflows after pruning, compaction agent rewrites entire history
- **Integrated view:** Three-stage tool output management: truncate on generation → prune on aging → compact on overflow. Each stage is lighter than the next.

**Flow 2: Permission System as Cross-Cutting Concern (Dims 1, 3, 4)**
- Dim 1: Doom-loop detection routes through permission system
- Dim 3: Every tool call goes through permission.ask() with Effect Deferred; bash uses tree-sitter AST for permission patterns
- Dim 4: Child sessions inherit permissions with programmatic overrides (deny todowrite, deny recursive task)
- **Integrated view:** The permission system is the harness's central control plane — it gates tool execution, loop termination, and delegation scope. More unified than separate systems for each concern.

**Flow 3: Agent Mode Switching (Dims 1, 3, 4)**
- Dim 1: Main loop pulls SubtaskPart from queue and handles before LLM call
- Dim 3: plan_exit tool switches agent mode by creating synthetic user message with `agent: "build"`
- Dim 4: Plan agent delegates explore/design phases to subagents via task tool
- **Integrated view:** Agent switching is multi-modal: tool-call (plan_exit), queue-based (subtask), or config-based (default agent per project). All paths converge at the session message layer.

### Novelty Assessment

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| 9-strategy fuzzy edit replacer cascade | Dim 3 | NOVEL | Most sophisticated fuzzy matching in any open harness |
| Tree-sitter bash AST for permission detection | Dim 3 | NOVEL | Structural command analysis, not string matching |
| Model-gated tool selection (apply_patch vs edit) | Dim 3 | NOVEL | Automatic model-capability routing |
| `invalid` tool as first-class error handler | Dim 3 | NOVEL | Schema mismatches become structured results |
| Max-steps via synthetic assistant prefill | Dim 1 | NOVEL | Instruction-following exploit for loop control |
| Doom-loop detector as permission-gated | Dim 1 | NOVEL | User-configurable, not auto-stop |
| Snapshot-per-step git time-travel revert | Dim 2 | NOVEL | Git object store as session checkpoint |
| Remote skill CDN discovery | Dim 3 | NOVEL | Marketplace-style skill distribution |
| Resumable subagent sessions via task_id | Dim 4 | NOVEL | Stateful multi-turn subagent interactions |
| Description-as-template from .txt sidecar files | Dim 3 | NOVEL | Prompt engineering separated from code |
| Dual-trigger compaction (predictive + reactive) | Dim 2 | VARIANT | Similar to OpenHands but with prune as third stage |
| Two-part system prompt for cache efficiency | Dim 1 | VARIANT | Same idea as Claude Code cache-first |
| Permission-as-continuation (Effect Deferred) | Dim 3 | VARIANT | Functional variant of Claude Code permissions |
| Claims-based instruction dedup | Dim 2 | VARIANT | Progressive disclosure variant |
| Compaction as named agent | Dim 1 | VARIANT | Similar to OpenHands condenser-as-action |
| TodoWrite as agent task tracking | Dim 2 | VARIANT | SQL-backed version of existing pattern |

## Decisions to Adopt

1. **Adopt: 9-strategy fuzzy edit replacer cascade** from `packages/opencode/src/tool/edit.ts:651-688`
   - What: Implement a multi-strategy fallback chain for str_replace operations (exact → trimmed → block-anchor → whitespace-normalized → indentation-flexible → escape-normalized → boundary → context → multi-occurrence)
   - Why: Dramatically reduces edit failures from LLM formatting drift — the #1 source of tool call retries
   - Effort: M
   - Target: Any harness with a str_replace/edit tool

2. **Adopt: Tree-sitter AST for bash permission detection** from `packages/opencode/src/tool/bash.ts`
   - What: Parse bash commands with tree-sitter to extract file paths and operations, build structured permission requests
   - Why: String matching misses escaped paths, subshells, redirections. AST parsing is structurally correct.
   - Effort: M
   - Target: Any harness with a bash tool and permission system

3. **Adopt: Model-gated tool selection** from `packages/opencode/src/tool/registry.ts:279-288`
   - What: Automatically swap tool variants based on model capabilities (e.g., apply_patch for GPT, str_replace for Claude)
   - Why: Different models perform better with different tool schemas. Automatic routing eliminates user configuration.
   - Effort: S
   - Target: Any multi-model harness

4. **Adopt: `invalid` tool as first-class error handler** from `packages/opencode/src/tool/invalid.ts`
   - What: Register an `invalid` tool that handles non-existent tool calls as structured results rather than exceptions
   - Why: Keeps the conversation loop intact; gives the LLM a structured error message to retry
   - Effort: S
   - Target: Any harness that encounters schema mismatches from LLM tool calls

5. **Adopt: Description-as-template from .txt sidecar files** from `packages/opencode/src/tool/bash.ts:461-510`
   - What: Store tool descriptions in separate .txt files with ${variable} template substitution at runtime
   - Why: Separates prompt engineering from code; allows tuning descriptions without touching TypeScript
   - Effort: S
   - Target: Any harness where tool descriptions are frequently iterated

6. **Adopt: Snapshot-per-step git time-travel** from `packages/opencode/src/snapshot/index.ts`
   - What: Record git tree hashes at each LLM step; enable full filesystem revert to any session checkpoint
   - Why: First-class undo across arbitrary multi-step operations without manual git management
   - Effort: L
   - Target: Agent harnesses where filesystem modifications need reliable rollback

7. **Adopt: Resumable subagent sessions via task_id** from `packages/opencode/src/tool/task.ts:25-29`
   - What: Return a task_id from subagent dispatch that enables resuming the same child session for follow-up queries
   - Why: Avoids re-establishing context for multi-turn subagent interactions; preserves subagent working memory
   - Effort: M
   - Target: Multi-agent harnesses with iterative delegation patterns

8. **Adopt: Doom-loop detection as configurable permission** from `packages/opencode/src/session/processor.ts:25,306-329`
   - What: Detect N consecutive identical tool calls and route through the permission system (allow/deny/ask) rather than auto-stopping
   - Why: Gives users control over loop behavior; some legitimate patterns involve repeated identical calls
   - Effort: S
   - Target: Any agent harness with loop termination logic

## Evidence Index

**Verified: 36 paths (100%)**

All paths verified: packages/opencode/src/{agent/agent.ts, session/prompt.ts, session/processor.ts, session/compaction.ts, session/overflow.ts, session/llm.ts, session/system.ts, session/instruction.ts, session/message-v2.ts, session/run-state.ts, session/todo.ts, session/revert.ts, tool/tool.ts, tool/registry.ts, tool/bash.ts, tool/edit.ts, tool/task.ts, tool/plan.ts, tool/invalid.ts, tool/skill.ts, tool/truncate.ts, tool/apply_patch.ts, tool/lsp.ts, mcp/index.ts, mcp/auth.ts, skill/index.ts, skill/discovery.ts, snapshot/index.ts, storage/db.ts, storage/storage.ts, bus/index.ts, permission/index.ts, acp/agent.ts, acp/session.ts, v2/session.ts, config/config.ts}

Unverified: 0 paths (0%)

## Sources

- [anomalyco-opencode](https://github.com/anomalyco/opencode) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
