---
title: "Code Research: openclaw-openclaw"
source: https://github.com/openclaw/openclaw
author: "kb-code-research skill"
date: 2026-04-15
fetched: 2026-04-15
type: code-research
status: raw
tags: [code-research, openclaw, compaction-pipeline, dreaming, security-architecture, snapshot-system]
relevance_score: 9
research_goal: "analyze Pi harness, SOUL.md, skill system, clawhub plugin architecture for extensible harness design"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: openclaw-openclaw

## Executive Summary

OpenClaw is a production multi-channel AI agent platform with one of the most sophisticated harness architectures analyzed in this KB. This re-run with tuned prompts uncovered massive new depth: the **compaction system is a full multi-stage pipeline** with preemptive/overflow/queued/manual paths, checkpoint-based undo, plugin provider override, and identifier preservation instructions; the **dreaming system** has asymmetric phase budgets (light=cheap, deep=quality, REM=expensive), a deep recovery self-healing loop (auto-promotes at 97% confidence), corpus self-ingestion detection/repair, and subagent-delegated narrative generation with atomic file writes; the **security architecture** includes sandbox bind-mount validation with symlink escape hardening, host-env variable sanitization with three distinct boundaries, plugin security scanning with mtime-keyed caching, and 13-pattern prompt injection detection; and a **pervasive snapshot system** for config, runtime, auth profiles, skills, and sessions with a two-layer redaction/restore round-trip for credential safety. The tuned prompts caught 18+ net-new patterns.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | openclaw-openclaw |
| Primary language | TypeScript (~14K files) |
| Size classification | large |
| Relevance to goal | 9/10 |
| Multi-agent signal count | 100+ |

## Dimension 1: Architecture & Loop Design (RE-RUN additions)

### Key New Findings
- **5-path compaction pipeline:** Preemptive (pre-flight before every prompt), queued (session+global lock), manual (boundary hardening via firstKeptEntryId rewrite), safety timeout (15min wrapper + AbortSignal), staged summarization with progressive fallback (chunk→parallel summarize→merge→drop oversized→hard fallback).
  - Evidence: `src/agents/pi-embedded-runner/run/preemptive-compaction.ts`, `compact.queued.ts`, `manual-compaction-boundary.ts`, `compaction-safety-timeout.ts`, `src/agents/compaction.ts`

- **Compaction checkpoint/undo system:** Before every compaction, captures physical `.checkpoint.*.jsonl` file copy. Up to 25 checkpoints per session with reason tracking. Enables time-travel recovery.
  - Evidence: `src/gateway/session-compaction-checkpoints.ts`

- **Plugin compaction provider interface:** Plugins can fully replace the built-in summarization via `CompactionProvider`. Registry uses `Symbol.for()` pattern to survive duplicate bundle loading.
  - Evidence: `src/plugins/compaction-provider.ts`

- **Post-compaction context injection:** Re-injects AGENTS.md sections ("Session Startup", "Red Lines") after compaction. Date placeholders resolved at runtime. Legacy fallback to older section names.
  - Evidence: `src/auto-reply/reply/post-compaction-context.ts`

- **Identifier preservation instructions:** Every summarization prompt includes instructions to preserve UUIDs, hashes, tokens, API keys, URLs, file names verbatim. Configurable to "off" or "custom".
  - Evidence: `src/agents/compaction.ts`

- **Compaction failure reason classifier:** String-matching maps free-text errors to structured codes (no_compactable_entries, below_threshold, timeout, provider_error_4xx/5xx, etc.).
  - Evidence: `src/agents/pi-embedded-runner/compact-reasons.ts`

### Patterns
- Compaction as distributed transaction (snapshot → lock → execute → checkpoint → side-effects → release)
- Provider override without auth bleed (drops authProfileId on cross-provider compaction)
- SECURITY comment annotation convention for LLM-facing code
- Manual compaction = metadata rewrite, not content change
- Post-compaction memory sync configurable to off/async/await

## Dimension 2: Memory & State Management (RE-RUN additions)

### Key New Findings
- **Asymmetric dreaming phase budgets:** Light=fast/cheap (quick ingestion), Deep=balanced/high-thinking/medium (quality scorer), REM=slow/expensive (reflective, no token limit). Each phase can use a different model.
  - Evidence: `src/memory-host-sdk/dreaming.ts`

- **Deep recovery self-healing loop:** When memory health drops below 35%, re-examines last 30 days of candidates. Auto-writes at 97% confidence; interactive review below that.
  - Evidence: `src/memory-host-sdk/dreaming.ts` — recovery config block

- **Corpus self-ingestion detection and repair:** Detects when narrative generation prompts leaked back into the corpus. Archives (not deletes) corrupted files with UUID suffix to `.openclaw-repair/dreaming/`.
  - Evidence: `extensions/memory-core/src/dreaming-repair.ts`

- **Daily snippet chunking in light phase:** Parses markdown daily-memory files into typed chunks (list/paragraph). Strips previous dreaming output markers to prevent self-poisoning. Content-addressed cosine scoring at 0.62 threshold.
  - Evidence: `extensions/memory-core/src/dreaming-phases.ts`

- **Subagent-delegated narrative generation:** Diary entries dispatched as real agent turns via SubagentSurface with idempotency keys, 60s timeout, 120s settle wait. Per-file async locking via global Symbol registry. Atomic write pattern (tmp → chmod → rename).
  - Evidence: `extensions/memory-core/src/dreaming-narrative.ts`

- **Backfill diary lane:** Historical replay with HTML comment markers. Fully reversible — `removeBackfillDiaryEntries()` only strips marked entries. Content deduplication via SHA fingerprint.
  - Evidence: `extensions/memory-core/src/dreaming-narrative.ts`

- **Pervasive snapshot system:** Runtime config snapshots (module-level singleton with refresh handlers), auth profile snapshots (structuredClone copies, atomic replace), skills snapshots (lazy-loaded, version-keyed, stale detection), session snapshots (freshness evaluation with daily-reset and idle-timeout policies).
  - Evidence: `src/config/runtime-snapshot.ts`, `src/agents/auth-profiles/runtime-snapshots.ts`, `src/cron/isolated-agent/skills-snapshot.ts`, `extensions/whatsapp/src/auto-reply/session-snapshot.ts`

- **Two-layer config redaction with restore:** Schema-derived hint lookup + pattern-guessing fallback. Sentinel `__OPENCLAW_REDACTED__` enables round-trip restore on write. SecretRef objects get partial redaction (id redacted, source/provider preserved).
  - Evidence: `src/config/redact-snapshot.ts`

### Patterns
- Deep recovery self-healing loop for memory health
- Corpus self-ingestion detection and archival repair
- Subagent-as-worker for narrative generation
- Per-file async locking via global Symbol registry
- Atomic file writes (tmp → chmod → rename)
- Snapshot-plus-restore round trip for config credentials
- Content-addressed deduplication with SHA fingerprints

## Dimension 3: Tool & Action Space Design (RE-RUN additions)

### Key New Findings
- **Sandbox security validation:** Docker bind-mount validator with host path denylist, home subdirectory denylist (.aws/.ssh/.docker/.gnupg), symlink escape hardening via ancestor resolution, reserved container target path blocking.
  - Evidence: `src/agents/sandbox/validate-sandbox-security.ts`

- **Host environment sanitization (3 boundaries):** isDangerousHostEnvVarName (execution-time), isDangerousHostInheritedEnvVarName (child process), isDangerousHostEnvOverrideVarName (agent/gateway overrides). PATH always blocked for overrides. Shell wrappers restricted to terminal/locale allowlist.
  - Evidence: `src/infra/host-env-security.ts`

- **Plugin security scanning:** 4 scan functions (bundle, package, dependency tree, file) with lazy-loaded runtime. Skill installs get separate path. mtime-keyed file cache (5000 entries) for hot-path performance.
  - Evidence: `src/plugins/install-security-scan.ts`

- **Prompt injection detection:** 13 regex patterns in `external-content.ts` covering "ignore all previous instructions", "you are now a", `<system>` tags, role-change markers, etc.
  - Evidence: `src/security/external-content.ts` (referenced in audit system)

- **Dangerous config flag aggregation:** `dangerous-config-flags.ts` aggregates all `dangerously*` and `allowInsecure*` flags across core config and plugin-declared flags via `configContracts.dangerousFlags`.

### Patterns
- Symlink escape hardening via ancestor resolution
- Three-boundary env var sanitization (execution/inheritance/override)
- Lazy-loaded security scanning with mtime cache
- Prompt injection pattern matching (13 patterns)
- Dangerous flag aggregation across core + plugin configs

## Dimension 4: Multi-Agent Coordination (RE-RUN additions)

### Key New Findings
- **Subagent-delegated dreaming narrative:** Narrative generation dispatched as real agent turns, not inline LLM calls. Idempotency keys derived from workspace hash + timestamp. Silent (deliver: false) with timeout and settle wait.
  - Evidence: `extensions/memory-core/src/dreaming-narrative.ts`

- **Legacy export migration in progress:** `resolveMemoryCorePluginConfig` aliased to `resolveMemoryDreamingPluginConfig` with migration comment. Indicates rename from memory-core to dreaming branding.
  - Evidence: `src/memory-host-sdk/dreaming.ts:346`

### Patterns
- Subagent surface for background work (non-interactive agent turns)
- Per-path async locking via global Symbol for concurrent safety

## Cross-Cutting Analysis

### Cross-Cutting Flows

**Flow 1: Compaction-Dreaming Memory Lifecycle**
- Architecture: Compaction pipeline triggers when context exceeds budget; captures checkpoint
- Memory: Post-compaction re-injects AGENTS.md sections. Dreaming system processes daily memories on cron schedule (light→REM→deep). Deep recovery re-examines candidates when health drops.
- Tools: Plugin compaction provider can replace built-in summarization. Identifier preservation protects opaque IDs.
- Integrated: Compaction reduces context; dreaming promotes important memories to permanent store; recovery self-heals when promotion starvation occurs. The three systems form a memory lifecycle: accumulate → compress → promote → recover.

**Flow 2: Security Defense-in-Depth**
- Architecture: Sandbox validation gates container execution with symlink hardening
- Tools: Host env sanitization (3 boundaries), plugin security scanning (4 scan types), prompt injection detection (13 patterns)
- Integrated: Five independent security layers that together cover: execution environment (sandbox), environment variables (host-env), supply chain (plugin scanning), content injection (prompt patterns), and config safety (dangerous flags). Each layer operates independently — no single bypass compromises all layers.

### Novelty Assessment (vs. first run)

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| 5-path compaction pipeline | Dim 1 | NOVEL | First run noted compaction but not the 5 paths |
| Compaction checkpoint/undo | Dim 1 | NOVEL | Time-travel recovery for compaction |
| Plugin compaction provider | Dim 1 | NOVEL | Extensible summarization via interface |
| Identifier preservation instructions | Dim 1 | NOVEL | UUID/hash preservation in summarization prompts |
| Deep recovery self-healing loop | Dim 2 | NOVEL | Auto-promotion at 97% confidence |
| Corpus self-ingestion detection | Dim 2 | NOVEL | Repair system for dreaming self-poisoning |
| Subagent-delegated narrative | Dim 2/4 | NOVEL | Real agent turns for diary writing |
| Daily snippet chunking | Dim 2 | NOVEL | Content-addressed cosine scoring |
| Pervasive snapshot system | Dim 2 | NOVEL | Config/auth/skills/session snapshots |
| Two-layer config redaction | Dim 2 | NOVEL | Sentinel-based round-trip restore |
| Sandbox symlink escape hardening | Dim 3 | NOVEL | Ancestor resolution prevents symlink bypass |
| 3-boundary env var sanitization | Dim 3 | NOVEL | Execution/inheritance/override boundaries |
| Plugin security scanning with mtime cache | Dim 3 | NOVEL | Lazy-loaded, hot-path optimized |
| Prompt injection detection (13 patterns) | Dim 3 | NOVEL | Content-level security scanning |
| Compaction failure reason classifier | Dim 1 | VARIANT | String-matching to structured codes |
| Post-compaction context injection | Dim 1 | VARIANT | Similar to Claude Code's re-injection |
| Asymmetric phase budgets | Dim 2 | VARIANT | Refinement of dreaming phase finding |
| Backfill diary lane | Dim 2 | VARIANT | Historical replay with reversibility |

**14 NOVEL, 4 VARIANT findings — massive new coverage from tuned prompts.**

## Decisions to Adopt

1. **Adopt: Compaction checkpoint/undo system** — Physical file snapshot before every compaction. Enables time-travel recovery. Effort: M. Target: Long-running agent harness.

2. **Adopt: Deep recovery self-healing loop** — Auto-promote memories when health drops below threshold. Effort: M. Target: Any agent with persistent memory.

3. **Adopt: Corpus self-ingestion detection** — Check whether LLM output leaked back into training/ingestion corpus. Archive (not delete) corrupted files. Effort: S. Target: Any system with cyclic LLM read/write.

4. **Adopt: Identifier preservation in summarization** — Instruct summarizer to preserve UUIDs, hashes, API keys verbatim. Effort: S. Target: Any compaction/summarization system.

5. **Adopt: Three-boundary env var sanitization** — Separate dangerous-var checks for execution, child-process inheritance, and API-supplied overrides. Effort: M. Target: Sandbox/execution harness.

6. **Adopt: Plugin compaction provider interface** — Allow plugins to replace built-in summarization. Use Symbol.for() for cross-bundle registry. Effort: M. Target: Extensible harness design.

## Evidence Index

```
Verified: 30+ files across all dimensions (100%)
Key new files examined:
  ✓ src/agents/compaction.ts — staged summarization
  ✓ src/agents/pi-embedded-runner/compact.queued.ts — queued compaction
  ✓ src/agents/pi-embedded-runner/run/preemptive-compaction.ts — preemptive path
  ✓ src/agents/pi-embedded-runner/compact-reasons.ts — failure classification
  ✓ src/agents/pi-embedded-runner/manual-compaction-boundary.ts — manual path
  ✓ src/agents/pi-embedded-runner/compaction-safety-timeout.ts — timeout wrapper
  ✓ src/gateway/session-compaction-checkpoints.ts — checkpoint system
  ✓ src/plugins/compaction-provider.ts — provider interface
  ✓ src/auto-reply/reply/post-compaction-context.ts — post-compaction injection
  ✓ extensions/memory-core/src/dreaming-phases.ts — light phase chunking
  ✓ extensions/memory-core/src/dreaming-narrative.ts — narrative generation
  ✓ extensions/memory-core/src/dreaming-repair.ts — self-ingestion repair
  ✓ src/memory-host-sdk/dreaming.ts — phase config, recovery
  ✓ src/config/redact-snapshot.ts — credential redaction
  ✓ src/config/runtime-snapshot.ts — runtime snapshots
  ✓ src/agents/sandbox/validate-sandbox-security.ts — sandbox validation
  ✓ src/infra/host-env-security.ts — env var sanitization
  ✓ src/plugins/install-security-scan.ts — plugin scanning
```

## Sources

- [openclaw-openclaw](https://github.com/openclaw/openclaw) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
