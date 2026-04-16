---
title: "Code Research: claude-code"
source: "C:\\Users\\dunliu\\Downloads\\Claude Code\\src"
author: "kb-code-research skill"
date: 2026-04-15
fetched: 2026-04-15
type: code-research
status: compiled
compiled_to: [wiki/claude-code-architecture.md, wiki/agent-memory-and-context-management.md, wiki/long-running-agent-harnesses.md, wiki/tool-design-patterns.md, wiki/auto-mode-and-safety.md]
compiled_date: 2026-04-15
tags: [code-research, claude-code, compaction, permissions, memory-aging, fork-subagent]
relevance_score: 10
research_goal: "reverse-engineer Claude Code harness architecture for pattern extraction"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: claude-code

## Executive Summary

Claude Code is Anthropic's production coding agent CLI, analyzed with tuned prompts that caught extensive patterns the first run missed. The re-run uncovered: a **6-layer compaction pipeline** (snip → microcompact → context collapse → autocompact → blocking limit → reactive compact) with two interchangeable autocompact backends (session memory vs. API summarization); a **memory aging system** that injects staleness prose rather than dates (because "the model is poor at date arithmetic"); a **formal memory type taxonomy** (user/feedback/project/reference) with eval-validated extraction prompts; a **permission rule DSL** with escaped-paren grammar, shadowed-rule detection, and dangerous-pattern stripping at auto-mode entry; a **classifier-based auto mode** with denial tracking circuit breakers (3 consecutive / 20 total → loop abort); a **fork sub-agent mode** (4th multi-agent pattern) maximizing prompt cache sharing via byte-exact history inheritance; and **dual permission transport** (file-based + mailbox) for swarm coordination. The tuned prompts caught 20+ net-new patterns across all 4 dimensions.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | claude-code |
| Primary language | TypeScript (~1800 files) |
| Size classification | large |
| Relevance to goal | 10/10 |
| Multi-agent signal count | 100+ (all 4 dimensions applicable) |

## Dimension 1: Architecture & Loop Design

### Summary (RE-RUN — new findings only)
The first run identified the main while-tool-call loop and 5 termination conditions. This re-run reveals the compaction pipeline is actually a 6-layer stack, permission denial tracking creates a circuit-breaker loop terminator, and autocompact has two completely different backends.

### Key New Findings
- **6-layer compaction pipeline:** (1) Snip — removes history slices, adjusts threshold. (2) Microcompact — three sub-paths: time-based (1h idle → mutate), cached (cache_edits without mutation), legacy (removed). (3) Context collapse — runs BEFORE autocompact to preserve granularity. (4) Autocompact — session memory or API summarization. (5) Blocking limit — hard ceiling minus 3K buffer, returns synthetic error. (6) Reactive compact — handles prompt_too_long after the fact.
  - Evidence: `query.ts:396-467`, `services/compact/compact.ts`, `services/compact/apiMicrocompact.ts`

- **Two autocompact backends:** Session memory compaction (no API call — uses pre-built session notes file) vs. legacy summarization (forked Claude agent). SM-compact preserves recent tail (min 10K tokens, min 5 messages, max 40K) with remote config via GrowthBook.
  - Evidence: `services/compact/sessionMemoryCompact.ts`, `services/compact/autoCompact.ts`

- **Permission denial tracking as circuit breaker:** 3 consecutive or 20 total classifier denials → AbortError in headless mode (hard loop termination). In interactive mode, falls back to human prompting.
  - Evidence: `utils/permissions/denialTracking.ts`

- **BigQuery-based constant tuning:** MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3 cites BQ data: "1,279 sessions had 50+ consecutive failures, wasting ~250K API calls/day globally."
  - Evidence: `services/compact/autoCompact.ts:70`

- **Post-compact re-injection:** After compaction, re-injects: 5 recently-read files, deferred tool schemas delta, agent listing, MCP instructions, plan file, skill content (truncated at 5K tokens/skill).
  - Evidence: `services/compact/postCompactCleanup.ts`

- **API microcompact strategy types:** Date-stamped type names (`clear_tool_uses_20250919`, `clear_thinking_20251015`). Tool clearing: trigger at 180K tokens, target 40K. Asymmetric: write-tool history never cleared (audit trail); read-tool results cleared first (reconstructable).
  - Evidence: `services/compact/apiMicrocompact.ts`

- **Forked agent recursion guards:** autocompact checks `querySource === 'session_memory' || 'compact'` and returns false, preventing compaction deadlock.

### Patterns
- 6-layer compaction stack with progressive degradation
- Session memory as zero-cost compaction alternative
- Denial tracking → circuit breaker → loop termination
- Production telemetry (BigQuery) directly informing source code constants
- Date-stamped API type names for forward compatibility
- Asymmetric tool clearing (write preserved, read cleared)
- Post-compact re-injection of working context

## Dimension 2: Memory & State Management

### Summary (RE-RUN — new findings only)
The first run documented 5 memory systems. This re-run reveals a formal memory type taxonomy, a memory aging/staleness system, agent memory snapshots for team distribution, session memory compaction, and two separate MemoryType enums.

### Key New Findings
- **Memory aging system:** `memdir/memoryAge.ts` computes days-since-write and injects staleness prose (not dates) because "the model is poor at date arithmetic." Warns: "claims about code behavior or file:line citations may be outdated."
  - Evidence: `memdir/memoryAge.ts`, `utils/attachments.ts`

- **Formal memory type taxonomy:** 4 content types (user/feedback/project/reference) with XML-structured prompt guidance including `<when_to_save>`, `<how_to_use>`, `<body_structure>`, `<examples>`. Key design: feedback must record confirmations AND corrections (correction-only recording causes model to "grow overly cautious").
  - Evidence: `memdir/memoryTypes.ts`

- **Agent memory snapshots:** Team distribution mechanism at `.claude/agent-memory-snapshots/{agentType}/`. Three actions: `initialize` (first use), `prompt-update` (newer snapshot), `none` (up to date). Bridges project-scope VCS knowledge to local-scope agent memory.
  - Evidence: `tools/AgentTool/agentMemorySnapshot.ts`

- **Session memory compaction:** Pre-built session notes file replaces API summarization. Extraction triggers on: 10K+ tokens crossed AND 5K+ growth AND 3+ tool calls. Serialized via `sequential()` wrapper. 15s wait, 60s stale threshold.
  - Evidence: `services/compact/sessionMemoryCompact.ts`, `services/SessionMemory/sessionMemory.ts`

- **WHAT_NOT_TO_SAVE section:** Eval-validated (case 3: 0/2 → 3/3) gate that refuses to save PR lists even when user explicitly asks — redirects to "what was surprising or non-obvious."
  - Evidence: `memdir/memoryTypes.ts`

- **Two separate MemoryType enums:** `utils/memory/types.ts` (storage layer: User/Project/Local/Managed/AutoMem/TeamMem) vs. `memdir/memoryTypes.ts` (content taxonomy: user/feedback/project/reference). Naming collision mitigated by import paths.

- **Staleness is read-path only:** Compaction summarizer receives raw content without staleness annotations. Staleness caveats are injected only when the model reads memories, not when they're summarized.

### Patterns
- Staleness-as-prose (operationalizing model weakness in harness)
- Eval-validated extraction prompt sections
- Snapshot distribution for team agent memory bootstrap
- Session memory as zero-cost compaction alternative
- Dual MemoryType taxonomy (storage vs. content)
- Confirmation+correction recording to prevent overcaution drift

## Dimension 3: Tool & Action Space Design

### Summary (RE-RUN — new findings only)
The first run documented 50+ tools and deferred loading. This re-run reveals a permission rule DSL with escaped-paren grammar, three-tier shell matching, shadowed-rule detection, classifier-based auto mode, dangerous pattern stripping, and policy lockdown.

### Key New Findings
- **Permission rule DSL:** `ToolName(content)` grammar with escaped parentheses. `Bash(npm install)` or bare `Bash`. Legacy tool name aliases normalized at parse time (Task→Agent, KillShell→TaskStop).
  - Evidence: `utils/permissions/permissionRuleParser.ts`

- **Three-tier shell matching:** exact literal, prefix (`npm:*`), wildcard (`git *` → regex with dotAll for heredocs). Trailing ` *` made optional so `git *` matches bare `git`.
  - Evidence: `utils/permissions/shellRuleMatching.ts`

- **Shadowed rule detection:** Static analysis warns when a specific allow rule is unreachable because a broader deny/ask rule takes precedence.
  - Evidence: `utils/permissions/shadowedRuleDetection.ts`

- **Classifier auto mode:** LLM classifier replaces manual prompts. Fast paths skip classifier for safe tools (20+ allowlisted) and acceptEdits-compatible actions. Classifier transcript from `auto_mode_system_prompt.txt`. Iron gate (fail-closed vs. fail-open) via GrowthBook.
  - Evidence: `utils/permissions/permissions.ts`, `yoloClassifier.ts`

- **Dangerous pattern stripping at auto-mode entry:** Before entering auto mode, all rules are scanned for patterns that would bypass the classifier: tool-wide allows, interpreter prefixes (python, node, ruby, ssh — 20+ patterns), eval/exec/sudo. Dangerous rules stripped from in-memory context.
  - Evidence: `utils/permissions/permissionSetup.ts`, `dangerousPatterns.ts`

- **Policy lockdown:** `allowManagedPermissionRulesOnly` enterprise flag ignores all user/project rules, hides "always allow" UI.
  - Evidence: `utils/permissions/permissionsLoader.ts`

- **No fuzzy matching for tool names:** Confirmed — exact match or alias lookup only. `findSimilarFile` exists for file paths but nothing analogous for tools.

- **Security review as slash command:** Markdown template with three-phase methodology. Launches parallel sub-tasks for false-positive filtering with 8/10 confidence threshold.
  - Evidence: `commands/security-review.ts`

### Patterns
- Permission rule DSL with escape sequences
- Legacy name normalization at parse time (backward compat without migration)
- Three-tier shell matching (exact/prefix/wildcard)
- Shadowed rule static analysis
- Classifier-as-permission-gatekeeper with fast paths
- Dangerous pattern stripping at mode entry
- Enterprise policy lockdown for managed-only rules

## Dimension 4: Multi-Agent Coordination

### Summary (RE-RUN — new findings only)
The first run documented 3 modes (standard, fork, swarm). This re-run reveals fork sub-agent mode as a distinct 4th pattern, dual permission transport, memory snapshot distribution, agent summarization, tier-matched model inheritance, and in-process teammate spawning.

### Key New Findings
- **Fork sub-agent mode (4th mode):** Inherits parent's full conversation history byte-exact for prompt cache sharing. Identical placeholder tool_result blocks maximize cache hits. Anti-recursion guard via `<fork-boilerplate>` tag detection. 10 behavioral rules prompt-injected.
  - Evidence: `tools/AgentTool/forkSubagent.ts`

- **Dual permission transport for swarms:** File-based (legacy: JSON files with file locking, 1h GC) + mailbox-based (current: structured messages via teammate mailbox). Sandbox network approval is mailbox-only.
  - Evidence: `utils/swarm/permissionSync.ts`

- **Agent summarization:** 30s background ticker uses forked agent against live transcript. Shares prompt cache by NOT stripping tools (canUseTool always denies). NOT setting maxOutputTokens (would bust cache key). Elicits 3-5 word present-tense description.
  - Evidence: `services/AgentSummary/agentSummary.ts`

- **Tier-matched model inheritance:** `aliasMatchesParentTier()` prevents `model: 'opus'` on Vertex-hosted parent from resolving to a different Opus version. Bedrock cross-region prefix inherited.
  - Evidence: `utils/model/agent.ts`

- **Three memory scopes for agents:** user (cross-project, ~/.claude/), project (VCS-tracked, .claude/), local (machine-specific, .claude-local/). CLAUDE_CODE_REMOTE_MEMORY_DIR redirects local to mounted storage.
  - Evidence: `tools/AgentTool/agentMemory.ts`

- **Context isolation via AsyncLocalStorage:** SubagentContext and TeammateAgentContext carry identity in concurrent Node.js. Sparse telemetry edges via `consumeInvokingRequestId()` (fires once per spawn/resume).
  - Evidence: `utils/agentContext.ts`

- **In-process teammates:** Same Node.js process, AsyncLocalStorage for isolation. Cannot spawn background agents. Flat roster (no nested teams). Plan mode required flag forces teammate into plan mode first.
  - Evidence: `utils/swarm/spawnInProcess.ts`

### Patterns
- Prompt-cache-first forking (byte-exact history inheritance)
- Dual permission transport (file + mailbox)
- Background summarization sharing parent's cache
- Tier-matched model inheritance (prevents cross-provider downgrade)
- Three-scope agent memory (user/project/local)
- AsyncLocalStorage for concurrent agent isolation
- Sparse telemetry edges (one-shot invocation attribution)

## Cross-Cutting Analysis

### Contradiction Resolutions
No contradictions detected.

### Cross-Cutting Flows

**Flow 1: Cache-First Everything**
- Dim 1: Compaction pipeline designed around cache preservation (snip adjusts thresholds, MC uses cache_edits)
- Dim 2: Session memory compaction is zero-API-call (pre-built file)
- Dim 3: API microcompact sends server-side clearing strategies to avoid client-side cache busting
- Dim 4: Fork children inherit byte-exact history; summarization avoids tool stripping to share cache
- Integrated: The entire architecture is organized around prompt cache hit rates as the primary performance metric.

**Flow 2: Permission System as Harness Control Plane**
- Dim 1: Denial tracking → circuit breaker → loop termination (AbortError)
- Dim 3: Permission DSL + classifier + dangerous pattern stripping gate all tool execution
- Dim 4: Permission mode propagated to sub-agents via CLI flags; swarm permission delegation via dual transport
- Integrated: Permissions are not just access control — they are the primary loop control mechanism, agent coordination protocol, and enterprise governance layer.

**Flow 3: Memory Aging ↔ Compaction ↔ Extraction Pipeline**
- Dim 1: Compaction strips old context; post-compact re-injects memory files
- Dim 2: Memory aging injects staleness prose at read time; extraction validates what to save; session memory replaces API summarization
- Dim 3: Asymmetric tool clearing preserves write history (audit) while clearing read results (reconstructable)
- Integrated: The memory lifecycle is: extract (session memory) → age (staleness annotations) → compact (preserve or summarize) → re-inject (post-compact attachment).

### Novelty Assessment (vs. first run)

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| 6-layer compaction pipeline | Dim 1 | NOVEL | First run said "compaction exists" — now fully mapped |
| Session memory as zero-cost compaction | Dim 1/2 | NOVEL | Entirely new finding |
| Memory aging/staleness-as-prose | Dim 2 | NOVEL | Not in first run or any wiki article |
| Formal memory type taxonomy (4 types) | Dim 2 | NOVEL | First run didn't analyze memdir/ |
| Agent memory snapshots for team distribution | Dim 2/4 | NOVEL | First run missed this |
| Permission rule DSL with escape grammar | Dim 3 | NOVEL | First run didn't analyze rule parsing |
| Shadowed rule detection | Dim 3 | NOVEL | Static analysis for unreachable rules |
| Classifier auto mode with denial tracking | Dim 3 | NOVEL | First run noted auto mode existed but not the internals |
| Dangerous pattern stripping | Dim 3 | NOVEL | Not documented anywhere |
| Fork sub-agent mode (4th mode) | Dim 4 | NOVEL | First run documented 3 modes, missed fork |
| Dual permission transport for swarms | Dim 4 | NOVEL | File + mailbox dual system |
| Agent summarization (cache-sharing ticker) | Dim 4 | NOVEL | 30s background summarization |
| Tier-matched model inheritance | Dim 4 | NOVEL | Prevents cross-provider model downgrade |
| API microcompact with asymmetric clearing | Dim 1/3 | NOVEL | Write-tool audit trail preservation |
| Eval-validated extraction prompt sections | Dim 2 | NOVEL | A/B test results embedded in prompts |
| BigQuery-based constant tuning | Dim 1 | VARIANT | Production telemetry informing code |
| Post-compact re-injection | Dim 1 | VARIANT | Refinement of compaction finding |

**15 NOVEL, 2 VARIANT findings — the tuned prompts massively expanded coverage.**

## Decisions to Adopt

1. **Adopt: Memory aging with staleness-as-prose** from memdir/memoryAge.ts
   - What: Inject pre-computed staleness warnings as prose, not dates
   - Why: Models are poor at date arithmetic; prose operationalizes the weakness
   - Effort: S
   - Target: Any system with persistent memory files

2. **Adopt: 6-layer compaction pipeline design** from services/compact/
   - What: Progressive context reduction: snip → microcompact → collapse → autocompact → blocking limit → reactive
   - Why: Each layer handles different context pressure levels; earlier layers preserve more granularity
   - Effort: L
   - Target: Long-running agent harness context management

3. **Adopt: Permission denial tracking as circuit breaker** from denialTracking.ts
   - What: Track consecutive and total denials; abort after thresholds
   - Why: Prevents infinite permission loops in headless mode
   - Effort: S
   - Target: Any autonomous agent with permission systems

4. **Adopt: Prompt-cache-first forking** from forkSubagent.ts
   - What: Fork children inherit byte-exact history with identical placeholder results
   - Why: Maximizes cache hit rates for parallel sub-agents
   - Effort: M
   - Target: Multi-agent systems using API prompt caching

5. **Adopt: Memory type taxonomy with eval-validated prompts** from memdir/memoryTypes.ts
   - What: Formal content types (user/feedback/project/reference) with tested extraction prompts
   - Why: Structured taxonomy prevents memory drift; eval validation ensures quality
   - Effort: M
   - Target: Any agent with persistent memory

6. **Adopt: Asymmetric tool clearing** from apiMicrocompact.ts
   - What: Clear read-tool results (reconstructable) before write-tool history (audit trail)
   - Why: Preserves ability to reason about what was changed while freeing context
   - Effort: M
   - Target: Context management in tool-heavy agents

## Evidence Index

```
Verified: 35+ files across all 4 dimensions (100% verified)
Key new files examined:
  ✓ services/compact/apiMicrocompact.ts
  ✓ services/compact/sessionMemoryCompact.ts
  ✓ services/compact/autoCompact.ts
  ✓ services/compact/postCompactCleanup.ts
  ✓ memdir/memoryAge.ts
  ✓ memdir/memoryScan.ts
  ✓ memdir/memoryTypes.ts
  ✓ tools/AgentTool/agentMemorySnapshot.ts
  ✓ tools/AgentTool/forkSubagent.ts
  ✓ utils/permissions/permissionRuleParser.ts
  ✓ utils/permissions/shellRuleMatching.ts
  ✓ utils/permissions/shadowedRuleDetection.ts
  ✓ utils/permissions/denialTracking.ts
  ✓ utils/swarm/permissionSync.ts
  ✓ services/AgentSummary/agentSummary.ts
  ✓ utils/model/agent.ts
  ✓ utils/agentContext.ts
```

## Sources

- [claude-code](C:\Users\dunliu\Downloads\Claude Code\src) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
