---
title: "Code Research: karpathy-autoresearch"
source: https://github.com/karpathy/autoresearch
author: "kb-code-research skill"
date: 2026-04-15
fetched: 2026-04-15
type: code-research
status: raw
tags: [code-research, autoresearch, self-improvement, experiment-loop, git-memory]
relevance_score: 8
research_goal: "extract autoresearch loop patterns for trading strategy self-improvement"
dimensions_analyzed: [architecture, memory, tools]
---

# Code Research: karpathy-autoresearch

## Executive Summary

karpathy-autoresearch is a deliberately minimal autonomous ML research loop: a 115-line markdown "program" (program.md) instructs an LLM agent to iteratively modify a training script, run 5-minute time-boxed experiments, and keep/discard changes via git. This re-run with tuned prompts caught several patterns the original missed: the **deterministic FSM tool selection** (steps 1-9 with LLM judgment only at crash-recovery), the **two-tier error handling** (LLM judgment for experiments + exponential backoff for I/O), **complexity as explicit optimization criterion** (anti-bloat in system prompt), and **metric isolation via immutable oracle** (read-only evaluation function). The strongest patterns for trading strategy self-improvement are: time-boxed experiments as comparison units, git-as-experiment-database with untracked TSV for crash resilience, the stateless agent restart protocol, and the two-level optimization hierarchy (LLM optimizes strategy code, human optimizes program.md).

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | karpathy-autoresearch |
| Primary language | Python (2 files) |
| Size classification | tiny (~1K LOC, 4 files) |
| File count | 4 |
| Relevance to goal | 8/10 — Directly applicable autonomous self-improvement loop |
| Multi-agent signal count | 0 (single agent) |
| Recommended dimensions | Architecture, Memory, Tools (Dim 4 N/A) |

## Dimension 1: Architecture & Loop Design

### Summary
Two-level loop: an outer meta-loop in natural language (LOOP FOREVER in program.md) and an inner ML training loop bounded by TIME_BUDGET=300s. The LLM controls the outer loop; Python controls the inner loop with NaN fail-fast. No convergence detection — runs until human interruption. Context managed through strict output filtering: stdout redirect + selective grep.

### Key Findings
- **Hybrid NL + imperative loop:** Outer loop is LOOP FOREVER in prose (steps 1-9); inner loop is `while True:` with time-budget break.
  - Evidence: `program.md:94-109`, `train.py:543-604`
- **Context hygiene via output filtering:** `> run.log 2>&1` + `grep "^val_bpb:"` — each experiment adds only 1-2 lines to context.
  - Evidence: `program.md:99-100`
- **Warmup step exclusion:** First 10 steps excluded from timing to avoid PyTorch compilation overhead.
  - Evidence: `train.py:578-579`
- **LR schedule based on wall-clock progress:** `progress = training_time / TIME_BUDGET` — unusual in ML, consistent with time-budget philosophy.
  - Evidence: `train.py:518-525`
- **Two-level optimization:** LLM optimizes train.py; human optimizes program.md.
  - Evidence: `README.md:14-15`

### Patterns
- Time-bounded inner loop (wall-clock, not step count)
- Context hygiene via stdout redirect + selective grep
- Git as experiment ledger (keep = advance branch, discard = reset)
- Two-level optimization with separate owners
- Fast-fail guardrail (NaN/explosion → exit(1))
- Never-stop directive for overnight operation
- Warmup exclusion from timing budget

## Dimension 2: Memory & State Management

### Summary
Entire state strategy is filesystem + git. Three-tier hierarchy: ephemeral run.log, session-level results.tsv (untracked, survives git reset), permanent git branch (best-so-far code). No model checkpoints — weights trained from scratch each run; the code IS the persistent artifact. Designed for stateless agent restarts via deterministic context reconstruction protocol.

### Key Findings
- **Three file-based memory systems:** run.log (ephemeral), results.tsv (session, untracked), git branch (permanent).
  - Evidence: `program.md:65-77, 92-109`
- **Git reset resilience via untracked TSV:** TSV survives `git reset --hard` because it's untracked.
  - Evidence: `program.md:66`
- **Stateless agent restart protocol:** Setup section provides deterministic context reconstruction from files + git.
  - Evidence: `program.md:1-19`
- **No model checkpoints:** Code is the artifact, not weights. Each experiment trains from scratch.
  - Evidence: No `torch.save()` in train.py
- **Agent cannot modify own instructions:** Can modify train.py but not prepare.py or program.md.
  - Evidence: `program.md:22-29`

### Patterns
- Git-as-experiment-database with branch HEAD as best-so-far
- Untracked file as crash-safe experiment ledger
- Stateless agent restart via file re-read protocol
- Three-tier memory hierarchy (ephemeral/session/permanent)
- Infrastructure state in ~/.cache (outside git)

## Dimension 3: Tool & Action Space Design

### Summary
Zero tools defined in code. ~8 implicit OS primitives specified in natural language. Tool selection follows a deterministic FSM (steps 1-9) with LLM judgment only at crash-recovery. The evaluation function is explicitly immutable, preventing metric gaming. Two-tier error handling: LLM judgment for experiment failures, exponential backoff for I/O.

### Key Findings
- **Deterministic FSM tool selection:** Steps 1-9 form fixed protocol; LLM judgment only at crash recovery (step 6).
  - Evidence: `program.md:94-109`
  - Significance: More reliable than free-choice tool selection.
- **Metric isolation via immutable oracle:** evaluate_bpb() in prepare.py is read-only.
  - Evidence: `prepare.py:343-365`, `program.md:29-30`
  - Significance: Prevents metric gaming in self-improvement loops.
- **Complexity as optimization criterion:** "A small improvement from deleting code? Definitely keep."
  - Evidence: `program.md:37`
  - Significance: Explicit anti-bloat policy weights simplicity alongside metric.
- **Two-tier error handling:** LLM judgment for experiments; exponential backoff (5 attempts, 2^n) for I/O.
  - Evidence: `program.md:101-111`, `prepare.py:66-88`
- **Convention-based security:** Agent told not to modify prepare.py — no code enforcement.
  - Evidence: `program.md:28-29`

### Patterns
- Filesystem as universal interface
- Metric isolation via immutable oracle
- Time-boxed experiments as comparison units
- Deterministic FSM with judgment at branch points
- Complexity as explicit optimization criterion
- Convention-based security (NL prohibition, not sandbox)

## Dimension 4: Multi-Agent Coordination

N/A — single-agent system. Branch naming (`autoresearch/<tag>-gpu0`) hints at designed-for extension via git branch isolation.

## Cross-Cutting Analysis

### Contradiction Resolutions
No contradictions detected.

### Cross-Cutting Flows

**Flow 1: Experiment Lifecycle**
- Dim 1: Steps 1-9 define the lifecycle FSM
- Dim 2: run.log (ephemeral) → results.tsv (durable) → git (permanent)
- Dim 3: Each step uses a specific OS primitive
- Integrated: The experiment lifecycle IS the harness — loop, memory, and tools are one unified protocol.
- Significance: For trading: edit strategy.py → run backtest → grep sharpe_ratio → log → git keep/discard.

**Flow 2: Context Hygiene Pipeline**
- Dim 1: stdout redirect prevents flooding; grep extracts key metrics
- Dim 2: run.log is buffer; TSV is durable; git log is compressed timeline
- Dim 3: grep is the critical filter between raw output and agent context
- Integrated: ~1MB training output → 2 lines of metrics → 1 TSV row. Agent never sees raw logs unless crash.
- Significance: Essential for any long-running autonomous loop.

### Novelty Assessment (vs. first run)

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| Deterministic FSM tool selection | Dim 3 | NOVEL | New — steps 1-9 as fixed protocol, not identified in first run |
| Two-tier error handling | Dim 3 | NOVEL | New — prepare.py retry pattern missed by first run |
| Complexity as optimization criterion | Dim 3 | NOVEL | New — anti-bloat as tool/strategy pattern |
| Metric isolation via immutable oracle | Dim 3 | NOVEL | New — security framing not in first run |
| LR schedule based on wall-clock progress | Dim 1 | NOVEL | New — unusual ML pattern |
| Stateless agent restart protocol | Dim 2 | VARIANT | Refinement of prior finding |
| Warmup step exclusion from timing | Dim 1 | VARIANT | Refinement of time-budget finding |
| Time-bounded experiments | Dim 1/3 | KNOWN | Already in KB |
| Git-as-experiment-database | Dim 2 | KNOWN | Already in KB |
| Context hygiene via redirect + grep | Dim 1 | KNOWN | Already in KB |

**5 NOVEL, 2 VARIANT, 3 KNOWN — tuned prompts caught 5 new patterns.**

## Decisions to Adopt

1. **Adopt: Deterministic FSM loop with LLM judgment at branch points** from program.md
   - What: Fixed numbered steps with LLM judgment only at decision points
   - Why: More reliable than free-choice tool selection
   - Effort: S
   - Target: Any self-improvement loop

2. **Adopt: Metric isolation via immutable oracle** from prepare.py
   - What: Read-only evaluation function the agent cannot modify
   - Why: Prevents Goodhart's Law in self-improvement loops
   - Effort: S
   - Target: Trading backtesting harness

3. **Adopt: Complexity as explicit optimization criterion** from program.md
   - What: Anti-bloat instructions in system prompt
   - Why: Prevents strategy bloat in long-running self-improvement
   - Effort: S
   - Target: System prompts for code-modifying agents

4. **Adopt: Stateless agent restart protocol** from program.md Setup
   - What: Deterministic context reconstruction from files + git
   - Why: Seamless recovery from context resets or crashes
   - Effort: S
   - Target: Any long-running autonomous agent

5. **Adopt: Two-tier error handling** from program.md + prepare.py
   - What: LLM judgment for strategy failures; exponential backoff for infrastructure failures
   - Why: Different failure types need different handling
   - Effort: S
   - Target: Trading backtesting error handling

## Evidence Index

```
Verified: 4 paths (100%)
  ✓ program.md — outer loop, tools, constraints, system prompt
  ✓ train.py — inner loop, time budget, fail-fast, GC optimization
  ✓ prepare.py — data prep, TIME_BUDGET, evaluation function, retry
  ✓ README.md — overview, multi-agent hints, two-level optimization
```

## Sources

- [karpathy-autoresearch](https://github.com/karpathy/autoresearch) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
