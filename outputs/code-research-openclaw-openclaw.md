---
title: "Code Research: openclaw-openclaw"
source: https://github.com/openclaw/openclaw
author: "kb-code-research skill"
date: 2026-04-14
fetched: 2026-04-14
type: code-research
status: raw
tags: [code-research, extensible-harness, skill-system, multi-agent, memory-hierarchy]
relevance_score: 9
research_goal: "analyze Pi harness, SOUL.md, skill system, clawhub plugin architecture for extensible harness design"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: openclaw-openclaw

## Executive Summary

OpenClaw is a personal AI assistant with a TypeScript-based extensible harness architecture that is the most feature-complete open-source agent harness we've analyzed. It implements a two-level agent loop (outer retry/failover + inner Pi SDK ReAct cycle), a MemGPT-inspired 3-tier memory hierarchy with autonomous "dreaming" consolidation, 30+ core tools with per-provider schema normalization across 6+ LLM providers, and a mature hierarchical multi-agent system with dual-runtime spawning (native subagents + external coding agents via ACP). The 7 genuinely novel patterns — dreaming-style memory consolidation, sessions_yield cooperative abort, SOUL.md persona injection, frozen result capture, streaming JSON repair, lane-based session serialization, and dual-runtime subagent spawning — represent significant advances over our existing KB coverage. This repo is the strongest exemplar of extensible harness design we've studied, with the plugin/skill/extension architecture providing a clean model for how to make every harness subsystem replaceable without forking core code.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | openclaw-openclaw |
| Primary language | TypeScript |
| Size classification | large (11,335 .ts files, est. 200K+ LOC) |
| File count | ~14,021 total files |
| Last commit | 2026-04-14 |
| Commit frequency | active (commits today) |
| README quality | detailed |
| Relevance to goal | 9/10 — Rich extensible harness with Pi agent loop, SOUL.md persona, skill system with clawhub marketplace, 70+ extensions, multi-agent subagent system |
| Agent/harness signals | 1,154 matches (agent, harness, orchestrat, loop, tool_use, function_call) |
| Multi-agent signal count | 251 (spawn, worker, swarm, subprocess, delegate) |
| Recommended dimensions | All 4 (Architecture, Memory, Tools, Multi-Agent) |

## Dimension 1: Architecture & Loop Design

### Summary
OpenClaw implements a two-level loop architecture: an outer `while (true)` retry loop in `runEmbeddedPiAgent` that manages failover/compaction/auth-profile rotation across attempts, and an inner model-driven tool-calling loop delegated to the Pi SDK's `activeSession.prompt()`. The outer loop has a dynamic iteration cap (32–160, scaled by auth-profile count) and terminates on success, cap exhaustion, timeout, abort signal, or strict-agentic planning-only blocks. Context management layers 5 mechanisms (preemptive, overflow-triggered, timeout-triggered, safeguard extension, cache-TTL pruning). The system prompt is assembled dynamically per-attempt from ~15 named sections with provider-specific overrides.

### Key Findings
- **Two-level loop with SDK delegation:** The harness owns retry/failover via `while (true)` at `src/agents/pi-embedded-runner/run.ts:611-645`; the model drives tool sequencing inside `activeSession.prompt()` at `src/agents/pi-embedded-runner/run/attempt.ts:2013-2016`. Clean separation means harness bugs don't corrupt the tool loop.
  - Evidence: `src/agents/pi-embedded-runner/run.ts:611-645` — outer loop with runLoopIterations counter
  - Significance: Principled boundary between code-controlled retry and model-controlled reasoning

- **Dynamic retry cap scaled by auth profiles:** `resolveMaxRunRetryIterations()` computes `BASE(24) + profiles × 8`, capped at 160, at `src/agents/pi-embedded-runner/run/helpers.ts:68-79`. More API key profiles → more retry surface.
  - Evidence: `src/agents/pi-embedded-runner/run/helpers.ts:68-79`
  - Significance: Unusual design — retry limits are typically fixed. Dynamic scaling reflects multi-provider reality.

- **5-layer compaction cascade:** Preemptive pre-prompt check, overflow-triggered (max 3 retries), timeout-triggered (max 2 retries), Pi SDK safeguard extension, and cache-TTL pruning all operate independently.
  - Evidence: `src/agents/pi-embedded-runner/run.ts:452-454` — MAX_TIMEOUT/OVERFLOW constants; `src/agents/pi-hooks/compaction-safeguard.ts` — SDK extension; `src/agents/pi-hooks/context-pruning.ts` — pruning extension
  - Significance: Layered, not monolithic — progressively more expensive operations applied as needed.

- **Lane-based session serialization:** Each session gets its own command queue (`session:${key}` lane) preventing concurrent runs, with separate lanes for cron jobs and nested operations.
  - Evidence: `src/agents/pi-embedded-runner/lanes.ts`
  - Significance: Novel concurrency pattern — per-session serialization without global blocking.

- **Lazy skill loading via model's read tool:** System prompt injects an XML `<available_skills>` catalog with names + descriptions. The model must call the `read` tool on the SKILL.md file before using a skill.
  - Evidence: `src/agents/skills/skill-contract.ts:44-64` — XML catalog format; `src/agents/system-prompt.ts:157-166` — instruction to read before executing
  - Significance: Keeps system prompt lean while advertising full capability. Skills are prompt injections, not tools.

- **SOUL.md as file-content persona injection:** If `soul.md` exists in workspace, the system prompt adds "embody its persona and tone." The template says "You're not a chatbot. You're becoming someone."
  - Evidence: `docs/reference/templates/SOUL.md` — full template; `src/agents/system-prompt.ts:105-113` — injection logic
  - Significance: Novel pattern — personality is user-controlled workspace content, not hardcoded in the harness.

- **Strict-agentic execution contract:** A mode that enforces tool calls on every turn; planning-only responses trigger retries with injected continuation instructions. After exhaustion, emits `STRICT_AGENTIC_BLOCKED_TEXT`.
  - Evidence: `src/agents/pi-embedded-runner/run.ts:438-447`, `run.ts:1727-1748`
  - Significance: Policy enforcement for agents expected to always act, not just plan.

### Patterns
- `while (true)` + dynamic iteration cap as safety valve
- AbortController propagation across all in-flight operations
- Dual compaction paths (harness contextEngine + Pi SDK auto-compaction)
- Stream function wrapping chain (7-10 sequential wraps for transform/logging/repair)
- Provider-portable system prompt assembly (sections compose differently per provider)

## Dimension 2: Memory & State Management

### Summary
OpenClaw implements a sophisticated 5-system, 3-tier memory architecture. Five systems operate in parallel: in-context workspace files (MEMORY.md, SOUL.md, daily files), SQLite+vector hybrid search store, external QMD backend, session transcript JSONL files, and per-agent sessions.json metadata. Memory flush is a full agent invocation (not a simple append) that writes to date-constrained daily files. Post-compaction context refresh re-reads AGENTS.md sections. Most remarkably, a "dreaming" subsystem runs three scheduled phases (light/deep/REM) for autonomous background memory consolidation.

### Key Findings
- **5 parallel memory systems:** In-context workspace files, SQLite+sqlite-vec vector store, external QMD binary, session JSONL transcripts, per-agent sessions.json metadata.
  - Evidence: `src/agents/memory-search.ts:44-54` — store config; `src/config/types.memory.ts:1-68` — MemoryBackend type; `src/config/sessions/types.ts:111-260` — SessionEntry schema
  - Significance: Multi-backend design means the memory subsystem is swappable via a single plugin registration.

- **3-tier memory hierarchy (working/short-term/long-term):** Working = in-context per turn. Short-term = daily `memory/YYYY-MM-DD.md` files. Long-term = MEMORY.md + vector store. Dreaming promotes short-term → long-term based on recall frequency.
  - Evidence: `src/memory-host-sdk/dreaming.ts:92-113` — promotion config with `minRecallCount: 3`, `recencyHalfLifeDays: 14`
  - Significance: MemGPT-inspired but with autonomous promotion via scheduled jobs, not explicit agent API calls.

- **Dreaming system (light/deep/REM):** Three cron-scheduled phases — light (every 6h, deduplication), deep (nightly 3am, recall-frequency promotion), REM (weekly, pattern synthesis).
  - Evidence: `src/memory-host-sdk/dreaming.ts:13-43` — cron schedules; `src/memory-host-sdk/events.ts:20-33` — MemoryHostPromotionAppliedEvent
  - Significance: Novel pattern — autonomous background memory consolidation inspired by sleep neuroscience.

- **Memory flush as agentic sub-run:** A full `runEmbeddedPiAgent` invocation with `trigger: "memory"` writes to a date-constrained path. The LLM decides what to remember; the harness constrains where.
  - Evidence: `src/auto-reply/reply/agent-runner-memory.ts:505-848` — `runMemoryFlushIfNeeded()` full implementation
  - Significance: Agent-driven memory curation, not harness-driven extraction. But writes are path-constrained for safety.

- **Post-compaction context refresh:** After compaction, re-reads `AGENTS.md` "Session Startup" and "Red Lines" sections with current-date `YYYY-MM-DD` substitution.
  - Evidence: `src/auto-reply/reply/post-compaction-context.ts:64-156` — `readPostCompactionContext()`
  - Significance: Solves "cold start after compaction" without explicit memory tool calls — harness-driven recovery.

- **Plugin-extensible memory via registerMemoryCapability:** A single `registerMemoryCapability(pluginId, capability)` call replaces the entire memory system.
  - Evidence: `src/plugins/memory-state.ts:170-175`
  - Significance: The memory subsystem is a replaceable plugin, not a hardcoded subsystem.

- **Temporal decay + MMR in hybrid search:** Vector search supports recency-weighted scoring and Maximal Marginal Relevance diversity reranking.
  - Evidence: `src/agents/memory-search.ts:78-92`
  - Significance: Production-grade search quality, not naive cosine similarity.

### Patterns
- Memory flush fires before compaction (token threshold gate)
- Compaction checkpoint archiving with pre/post session IDs
- Session store with JSON5 serialization and async lock queues
- Embedding provider registry via `Symbol.for()` on globalThis
- Daily memory files labeled as untrusted context in prompt

## Dimension 3: Tool & Action Space Design

### Summary
OpenClaw's tool ecosystem has three layers: 30+ core tools in 11 categories (defined in TypeBox), plugin-registered dynamic tools from 70+ extensions, and MCP proxy tools materialized at runtime. All schemas are normalized per-provider (Gemini, OpenAI strict, Anthropic, xAI) via `normalizeToolParameterSchema()`. Tool failure handling is exceptionally robust: streaming JSON argument repair for malformed providers, 4-level tool name normalization, error-as-context (never crashes), head+tail result truncation with error preservation, and an unknown-tool loop guard that rewrites assistant messages after repeated hallucinated tool calls.

### Key Findings
- **30+ core tools in 11 categories:** fs, runtime, web, memory, sessions, ui, messaging, automation, nodes, agents, media. Plus plugin tools and MCP tools.
  - Evidence: `src/agents/tool-catalog.ts:53-304` — CORE_TOOL_DEFINITIONS
  - Significance: Rich but organized — tool groups enable granular access control policies.

- **TypeBox schemas with per-provider normalization:** Tools defined once in TypeBox; `normalizeToolParameterSchema()` handles all provider quirks.
  - Evidence: `src/agents/pi-tools.schema.ts:131-254` — normalization logic; `src/agents/openai-tool-schema.ts:71-130` — strict mode check
  - Significance: Write-once, run-anywhere tool schemas — a provider portability abstraction.

- **Streaming JSON argument repair:** Provider-specific (Kimi: JSON with garbage, xAI: HTML entities). Repair runs as a stream wrapper, fixing partial events before they reach the agent core.
  - Evidence: `src/agents/pi-embedded-runner/run/attempt.tool-call-argument-repair.ts:68-384`
  - Significance: Novel pattern — fixing provider bugs in the streaming pipeline, not after the fact.

- **4-level tool name normalization:** Exact → case-insensitive → structured segment matching → tool-call-ID inference. Plus unknown-tool loop guard.
  - Evidence: `src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts:699-771` — loop guard rewrite
  - Significance: Deep fallback chain reflects real-world LLM output quality issues across providers.

- **Error-as-context pattern:** All tool exceptions caught in adapter, converted to `{status: "error", tool, error}` JSON results fed back as context.
  - Evidence: `src/agents/pi-tool-definition-adapter.ts:203-228`
  - Significance: Known pattern, robustly implemented — errors are information, not crashes.

- **MCP as first-class tools:** MCP tool schemas pass through the same normalization pipeline. Tool name collisions get suffix disambiguation.
  - Evidence: `src/agents/pi-bundle-mcp-materialize.ts:84-113` — tool wrapping; per-session MCP connection caching
  - Significance: MCP tools are indistinguishable from core tools at the model level.

- **Skills are prompt injections, not tools:** Skills instruct the model to use existing tools in domain-specific ways. Loaded lazily via the model's `read` tool from SKILL.md files.
  - Evidence: `src/agents/skills/skill-contract.ts:44-64` — XML catalog; `src/agents/system-prompt.ts:157-166` — load instruction
  - Significance: Fundamental architecture difference — skills are a meta-layer above tools.

- **Context guard via transformContext monkey-patch:** The harness patches a private Pi SDK method to enforce per-result and aggregate tool result size limits.
  - Evidence: `src/agents/pi-embedded-runner/tool-result-context-guard.ts:287-341`
  - Significance: Reveals that the upstream SDK lacks sufficient extension hooks for production context management.

### Patterns
- Policy pipeline: profile → provider → agent → group (multi-step tool availability filtering)
- Head+tail truncation preserving error text (30% tail budget)
- Plugin tool split: required (always present) vs optional (behind allowlist)
- Tool selection is pure LLM autonomy — no server-side routing
- `splitSdkTools()` always routes all tools as customTools, never builtInTools

## Dimension 4: Multi-Agent Coordination

### Summary
OpenClaw implements a mature hierarchical multi-agent system with two distinct spawn runtimes: native subagents (`runtime: "subagent"`) and ACP harness sessions (`runtime: "acp"`) for external coding agents. Coordination is enforced orchestrator-worker with depth-bounded nesting (default depth 1, configurable). Communication is push-based: children auto-announce results upstream via gateway steer message injection — no polling required. The sessions_yield cooperative abort lets orchestrators voluntarily terminate their LLM turn while children run. Reliability machinery includes frozen result capture (write-ahead log), orphan recovery after restart, announce retry with exponential backoff, and deduplication through latest-run indexing.

### Key Findings
- **Dual-runtime subagent spawning:** `sessions_spawn` supports both `runtime: "subagent"` (native) and `runtime: "acp"` (external: Claude Code, Codex, Gemini) under one unified tool surface.
  - Evidence: `src/agents/tools/sessions-spawn-tool.ts:22-23` — SESSIONS_SPAWN_RUNTIMES; `src/agents/acp-spawn.ts` — ACP pathway
  - Significance: Novel pattern — internal and external agents spawn through the same interface.

- **Depth-bounded hierarchy with role assignment:** Three roles: `main` (depth 0), `orchestrator` (depth 1 to maxDepth-1), `leaf` (at maxDepth). Default depth 1 = flat topology. Leaf agents cannot spawn.
  - Evidence: `src/agents/subagent-capabilities.ts:77-108` — `resolveSubagentRoleForDepth()`; `src/config/agent-limits.ts:6` — DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1
  - Significance: Ships safe (flat) by default, but supports arbitrarily deep trees via config.

- **Push-based announcements (no polling):** Children auto-announce results upstream via `runSubagentAnnounceFlow()` which injects a structured `task_completion` event into the requester's LLM context via gateway steer.
  - Evidence: `src/agents/subagent-announce.ts:194-551` — full announce flow; delivery via `callGateway({ method: "agent", deliver: false })`
  - Significance: Eliminates the "orchestrator burns context polling children" anti-pattern.

- **sessions_yield cooperative abort:** The model calls `sessions_yield` to voluntarily terminate its LLM turn. The gateway re-wakes it with aggregated child results when descendants settle.
  - Evidence: `src/agents/pi-embedded-runner/run/attempt.ts:548-553` — `runAbortController.abort("sessions_yield")`; `src/agents/subagent-announce.ts:130-192` — `wakeSubagentRunAfterDescendants()`
  - Significance: Novel pattern — model-initiated cooperative concurrency for multi-agent coordination.

- **Frozen result capture (frozenResultText):** Child's raw last assistant reply is captured immediately at run completion, before any announce delivery attempt.
  - Evidence: `src/agents/subagent-registry-lifecycle.ts:156-170` — `freezeRunResultAtCompletion()`
  - Significance: Acts as a mini write-ahead log — enables reliable retry/replay without re-querying the gateway.

- **Orphan recovery after restart:** Post-restart, `recoverOrphanedSubagentSessions()` scans the registry for aborted runs and re-dispatches resume messages with exponential backoff.
  - Evidence: `src/agents/subagent-orphan-recovery.ts:146-281`
  - Significance: Production-grade reliability — multi-agent state survives process restarts.

- **Announce queue with batching:** Concurrent child completions are batched into a single summary delivery, preventing context floods during fan-out.
  - Evidence: `src/agents/subagent-announce-queue.ts:60-238` — per-requester queue with `dropPolicy: "summarize"`, cap 20
  - Significance: Prevents O(n) context injection during parallel fan-out bursts.

### Patterns
- Steer-restart suppression: old run marked with `suppressAnnounceReason: "steer-restart"` before replacing
- controllerSessionKey vs requesterSessionKey distinction (ownership transfer after steer)
- Child-count deduplication via latestByChildSessionKey index
- Fire-and-forget spawning: `{status: "accepted"}` returned immediately, no blocking
- Telephone mitigation: frozen capture + direct injection, not multi-hop LLM paraphrasing

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions detected. All 4 dimension reports are consistent.

### Cross-Cutting Flows

**Flow 1: Memory Flush → Compaction → Post-Compaction Refresh**
- Dim 1 (Architecture): Compaction triggered by token overflow/timeout in outer loop
- Dim 2 (Memory): Memory flush runs first (full agent invocation writing to daily files), then compaction summarizes, then post-compaction refresh re-reads AGENTS.md with current date
- Dim 3 (Tools): Tool catalog is preserved through compaction because it's rebuilt per-attempt, not stored in history
- INTEGRATED VIEW: The system preserves actionable knowledge through a 3-step pipeline: flush memories → compact conversation → refresh critical instructions. Tool availability is maintained because it's regenerated per-attempt. This is a fully automated knowledge preservation pipeline that requires zero agent cooperation.
- SIGNIFICANCE: The harness owns the entire memory-through-compaction flow. The agent doesn't need to "remember" to save important context — the harness does it autonomously.

**Flow 2: Subagent Spawn → Tool Execution → Announce → Context Injection**
- Dim 1 (Architecture): The outer loop handles sessions_yield abort and re-wake
- Dim 2 (Memory): Subagent sessions get their own session files and session-store entries with compaction checkpoints
- Dim 3 (Tools): sessions_spawn is a tool the model calls — tool selection is LLM-driven
- Dim 4 (Multi-Agent): Fire-and-forget spawn → frozen result capture → push announcement → steer injection into orchestrator context
- INTEGRATED VIEW: The multi-agent flow is fully mediated through the tool layer. The model decides to spawn, the harness manages lifecycle, the registry captures results, and the gateway injects them back. No dimension owns this flow alone — it spans all 4.
- SIGNIFICANCE: This is the most complete tool-mediated multi-agent flow we've documented. Every step is observable, every result is captured, every failure has a recovery path.

**Flow 3: Skill Discovery → Lazy Loading → Tool Use → Memory Persist**
- Dim 1 (Architecture): System prompt includes XML catalog of available skills
- Dim 2 (Memory): Skills can trigger memory flush (via the tools they instruct the agent to use)
- Dim 3 (Tools): Skills are NOT tools — they're prompt injections that instruct the model to use existing tools
- Dim 4 (Multi-Agent): Skills have agent-filter controlling which agents see which skills
- INTEGRATED VIEW: The skill system is an orchestration layer above both tools and memory. A skill is a "script" the model follows, using tools to accomplish tasks and memory to persist results. Loading itself is mediated through the read tool — making both meta-level (skill loading) and object-level (skill execution) tool-mediated.
- SIGNIFICANCE: Skills-as-prompt-injections is a fundamentally different architecture from tool registration, enabling domain-specific behavior without modifying the tool set.

### Novelty Assessment

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| Dreaming system (light/deep/REM) | Dim 2 | NOVEL | No KB article covers autonomous background memory consolidation with sleep-phase scheduling |
| sessions_yield cooperative abort | Dim 4 | NOVEL | No KB article covers model-initiated voluntary turn abort for multi-agent coordination |
| SOUL.md as file-content persona injection | Dim 1 | NOVEL | KB covers CLAUDE.md/AGENTS.md but not a personality/soul workspace file |
| Frozen result capture (frozenResultText WAL) | Dim 4 | NOVEL | No KB article covers write-ahead log patterns for multi-agent result delivery |
| Streaming JSON argument repair | Dim 3 | NOVEL | KB mentions tool failures but not in-stream provider-specific argument fixing |
| Lane-based session serialization | Dim 1 | NOVEL | No KB article covers per-session command queues as a concurrency control pattern |
| Dual-runtime subagent spawning (native + ACP) | Dim 4 | NOVEL | KB covers orchestrator-worker but not unified tool surface for internal + external agents |
| Skills as lazy-loaded prompt injections (XML catalog) | Dim 1,3 | VARIANT | KB covers lazy tool loading but not skill-as-prompt with XML catalog + lazy read |
| Post-compaction context refresh | Dim 2 | VARIANT | KB covers compaction cascade but not AGENTS.md re-injection with date substitution |
| Two-level loop (retry outer + ReAct inner) | Dim 1 | VARIANT | KB documents while-tool-call loops but not explicit retry/failover outer loop |
| Depth-bounded role assignment (main/orchestrator/leaf) | Dim 4 | VARIANT | KB covers orchestrator-worker but not explicit depth limits with role enums |
| Error-as-context tool failure handling | Dim 3 | KNOWN | Already documented in wiki/tool-design-patterns.md |
| Plugin tool extensibility via registration | Dim 3 | KNOWN | Already covered in wiki/tool-design-patterns.md |

## Decisions to Adopt

1. **Adopt: Dreaming-style background memory consolidation** from `src/memory-host-sdk/dreaming.ts`
   - What: Schedule three phases of autonomous memory consolidation — light (6h dedup), deep (nightly promotion by recall frequency), REM (weekly synthesis)
   - Why: Eliminates the need for agents to explicitly manage memory tiers; the harness handles promotion autonomously
   - Effort: L
   - Target: Agent memory systems, long-running agent harnesses

2. **Adopt: sessions_yield cooperative abort** from `src/agents/pi-embedded-runner/run/attempt.ts:548-553`
   - What: A tool the model calls to voluntarily terminate its LLM turn while waiting for children, with gateway re-wake when descendants settle
   - Why: Prevents orchestrator from burning context/tokens polling for child results — a direct solution to the most wasteful multi-agent anti-pattern
   - Effort: M
   - Target: Multi-agent coordination patterns

3. **Adopt: Post-compaction context refresh** from `src/auto-reply/reply/post-compaction-context.ts`
   - What: After compaction, re-read critical AGENTS.md sections with current-date substitution so agent accesses today's daily memory files
   - Why: Solves "cold start after compaction" — agent re-anchors to critical instructions without explicit memory tool calls
   - Effort: S
   - Target: Context management, long-running agent harnesses

4. **Adopt: Streaming JSON argument repair** from `src/agents/pi-embedded-runner/run/attempt.tool-call-argument-repair.ts`
   - What: Provider-specific in-stream repair of malformed JSON tool call arguments (handles Kimi garbage, xAI HTML entities, etc.)
   - Why: Significant reliability improvement for multi-provider deployments — fix provider bugs in the pipeline, not after
   - Effort: M
   - Target: Tool design patterns, multi-provider harness reliability

5. **Adopt: Skills as lazy-loaded prompt injections** from `src/agents/skills/skill-contract.ts` and `src/agents/system-prompt.ts`
   - What: XML catalog in system prompt with name+description; model reads full SKILL.md via read tool only when needed
   - Why: Keeps system prompt lean while advertising full capability; skills compose over existing tools without modifying them
   - Effort: M
   - Target: Skill/plugin architecture, practical best practices

6. **Adopt: Frozen result capture** from `src/agents/subagent-registry-lifecycle.ts:156-170`
   - What: Capture child agent's raw output immediately at completion into a `frozenResultText` field for reliable replay/retry
   - Why: Acts as a mini write-ahead log — enables reliable announcement retry without re-querying the gateway
   - Effort: S
   - Target: Multi-agent reliability

7. **Adopt: Lane-based session serialization** from `src/agents/pi-embedded-runner/lanes.ts`
   - What: Per-session command queues preventing concurrent agent runs on the same session, with separate lanes for cron/nested operations
   - Why: Prevents race conditions in multi-channel deployments without global blocking
   - Effort: S
   - Target: Long-running agent harnesses, concurrency patterns

8. **Adopt: SOUL.md as persona injection** from `docs/reference/templates/SOUL.md`
   - What: A workspace file that defines the agent's personality, injected into system prompt with "embody its persona and tone"
   - Why: Separates personality from code — user-controlled, file-based identity that survives harness upgrades
   - Effort: S
   - Target: System prompt design, harness configuration patterns

## Evidence Index

**Verified: 50 paths (100%)**

Verified:
  ✓ src/agents/pi-embedded-runner/run.ts — outer agent loop
  ✓ src/agents/pi-embedded-runner/run/attempt.ts — inner attempt logic
  ✓ src/agents/pi-embedded-runner/run/helpers.ts — retry limit calculation
  ✓ src/agents/pi-embedded-runner/extensions.ts — extension factory
  ✓ src/agents/pi-embedded-runner/lanes.ts — session serialization
  ✓ src/agents/system-prompt.ts — dynamic prompt assembly
  ✓ src/agents/compaction.ts — compaction summarization
  ✓ src/agents/harness/builtin-pi.ts — Pi harness registration
  ✓ src/agents/harness/selection.ts — harness selection
  ✓ src/agents/memory-search.ts — vector search store
  ✓ src/auto-reply/reply/agent-runner-memory.ts — memory flush agent
  ✓ src/auto-reply/reply/post-compaction-context.ts — post-compaction refresh
  ✓ src/plugin-sdk/memory-core.ts — memory core plugin
  ✓ src/plugin-sdk/memory-host-core.ts — memory host core
  ✓ src/plugins/memory-state.ts — memory state registration
  ✓ src/agents/subagent-spawn.ts — subagent spawn logic
  ✓ src/agents/subagent-registry.ts — subagent registry
  ✓ src/agents/subagent-capabilities.ts — role/depth assignment
  ✓ src/agents/subagent-announce.ts — announcement flow
  ✓ src/agents/subagent-depth.ts — depth limiting
  ✓ src/agents/openclaw-tools.ts — core tool assembly
  ✓ src/agents/openclaw-tools.registration.ts — tool registration
  ✓ src/agents/bash-tools.ts — bash tool definitions
  ✓ src/agents/bash-tools.exec.ts — bash execution
  ✓ src/agents/pi-bundle-mcp-tools.ts — MCP tool bridge
  ✓ src/agents/model-tool-support.ts — model tool compat
  ✓ src/agents/pi-embedded-runner/tool-schema-runtime.ts — runtime schema
  ✓ src/agents/pi-embedded-runner/tool-result-truncation.ts — result truncation
  ✓ src/agents/pi-embedded-runner/tool-result-context-guard.ts — context guard
  ✓ src/agents/pi-embedded-runner/run/attempt.tool-call-argument-repair.ts — JSON repair
  ✓ src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts — name normalization
  ✓ src/agents/skills.ts — skill loading
  ✓ src/agents/skills-clawhub.ts — clawhub marketplace
  ✓ src/agents/skills/skill-contract.ts — skill contract/catalog
  ✓ src/agents/subagent-announce-queue.ts — announce batching
  ✓ src/agents/subagent-announce-output.ts — announce output
  ✓ src/agents/subagent-orphan-recovery.ts — orphan recovery
  ✓ src/agents/subagent-system-prompt.ts — subagent prompt
  ✓ src/agents/pi-hooks/compaction-safeguard.ts — safeguard extension
  ✓ src/agents/pi-hooks/context-pruning.ts — pruning extension
  ✓ src/config/types.memory.ts — memory config types
  ✓ src/agents/tool-catalog.ts — tool catalog definitions
  ✓ src/agents/pi-tools.schema.ts — schema normalization
  ✓ src/agents/pi-tool-definition-adapter.ts — tool adapter
  ✓ src/agents/tool-policy-pipeline.ts — access control pipeline
  ✓ src/plugins/tools.ts — plugin tool resolution
  ✓ src/agents/pi-bundle-mcp-materialize.ts — MCP materialization
  ✓ src/agents/openai-tool-schema.ts — OpenAI schema compat
  ✓ src/memory-host-sdk/dreaming.ts — dreaming consolidation
  ✓ src/global-state.ts — global state (trivially small)

Unverified: 0 paths (0%)

## Sources

- [openclaw-openclaw](https://github.com/openclaw/openclaw) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
