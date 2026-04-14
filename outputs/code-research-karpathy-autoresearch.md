---
title: "Code Research: karpathy-autoresearch"
source: https://github.com/karpathy/autoresearch
author: "kb-code-research skill"
date: 2026-04-14
fetched: 2026-04-14
type: code-research
status: raw
tags: [code-research, autoresearch, self-improvement, experiment-loop, git-as-state]
relevance_score: 8
research_goal: "extract autoresearch loop patterns for trading strategy self-improvement"
dimensions_analyzed: [architecture, memory, tools]
---

# Code Research: karpathy-autoresearch

## Executive Summary

Karpathy's autoresearch is a minimalist autonomous ML research loop where an LLM agent iteratively modifies training code, runs 5-minute experiments, and keeps or discards changes based on a single metric (val_bpb). The entire agent harness is a 115-line markdown file (program.md) — there is no Python orchestrator, no tool API, no formal state machine. Git branches serve as the experiment database, git commits as checkpoints, and git reset as rollback. The most transferable pattern for trading strategy self-improvement is the complete experiment lifecycle: prose-driven loop + git-as-state + fixed-budget evaluation + structural separation of mutable code from protected evaluation harness.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | karpathy-autoresearch |
| Primary language | Python |
| Size classification | small (~1K LOC) |
| File count | 2 Python source files + program.md |
| Last commit | Active (recent PR merge) |
| Commit frequency | active |
| README quality | detailed (93 lines) |
| Relevance to goal | 8/10 — directly implements autonomous research loop with code modification, evaluation, and selection |
| Agent/harness signals | 20 agent/loop, 37 memory/state, 7 multi-agent |
| Multi-agent signal count | 7 (below threshold, single-agent system) |
| Recommended dimensions | Architecture, Memory, Tools (Dim 4 skipped) |

## Dimension 1: Architecture & Loop Design

### Summary
Autoresearch implements a dual-loop architecture: an outer "experiment loop" expressed entirely as prose instructions in program.md (lines 94-112), and an inner training loop in Python (train.py while True at line 543). The LLM is the executor of the outer loop — there is no Python scheduler. Loop termination for the inner loop is wall-clock time (5 minutes fixed budget). The outer loop has no termination criterion — it runs until human interruption. Context management is architectural, not algorithmic: training output is redirected to run.log and only 2 scalar values are extracted via grep.

### Key Findings
- **Prose-as-Control-Flow:** The outer experiment loop is a numbered procedure in program.md (lines 94-106). The LLM reads these steps and executes them as its own decision procedure. No Python code drives the outer iteration.
  - Evidence: `program.md:94-106` — "LOOP FOREVER: 1. Look at the git state... 9. If val_bpb is equal or worse, you git reset back"
  - Significance: This is the simplest possible harness — the system prompt IS the harness.

- **Control Inversion:** The LLM is the scheduler, state machine, and decision engine. Python (train.py) is just the training worker.
  - Evidence: `program.md:112` — "NEVER STOP... You are autonomous."
  - Significance: Maximum model agency. The code is the tool, the model is the operator.

- **Fixed-Budget Evaluation as Convergence Proxy:** Every experiment trains for exactly 5 minutes regardless of model size, batch size, or architecture changes. This makes val_bpb directly comparable across all experiments.
  - Evidence: `prepare.py:31` — `TIME_BUDGET = 300`; `train.py:603` — `if step > 10 and total_training_time >= TIME_BUDGET: break`
  - Significance: Eliminates an entire dimension of state (training cost) from the evaluation. For trading, this maps to fixed backtest windows.

- **Minimal-Signal Extraction:** Only 2 lines of output enter the agent's context per experiment via grep.
  - Evidence: `program.md:100` — `grep "^val_bpb:\|^peak_vram_mb:" run.log`
  - Significance: Explicit anti-context-bloat pattern. Each experiment adds ~3-5 lines to context instead of 630 lines of training output.

- **No Convergence Detection:** The outer loop runs until human interruption.
  - Evidence: `program.md:112` — "The loop runs until the human interrupts you, period."
  - Significance: Deliberate design choice — the system cannot detect when it has exhausted useful mutations.

### Patterns
- Prose-as-Control-Flow: agent loop expressed as numbered natural language steps
- Control Inversion: LLM is the scheduler, code is the worker
- Fixed-Budget Evaluation: 5-min wall-clock budget makes experiments directly comparable
- Minimal-Signal Extraction: grep 2 scalars from redirected stdout
- Crash-as-Data: crashes are first-class outcomes with explicit triage heuristics

## Dimension 2: Memory & State Management

### Summary
Autoresearch uses three memory systems, all external to the model: (1) the agent's context window (working memory, reconstructed each loop iteration), (2) results.tsv as a flat-file experiment ledger (deliberately gitignored to separate outcomes from code state), and (3) git commit history on a dedicated branch as the code-state database. There is no vector store, no database, no MemGPT-style archival memory. State persistence relies entirely on git + one untracked file. Model weights are never saved — each experiment starts from random initialization. The "progress" being accumulated is improvement in code quality (the recipe), not in trained weights.

### Key Findings
- **Git-as-Experiment-Database:** Every git commit before a run is a hypothesis insertion into a content-addressed database. Git reset is the rollback/discard operation. Branch HEAD always points to the current best-performing hypothesis.
  - Evidence: `program.md:92-108` — "The experiment runs on a dedicated branch... if val_bpb improved, advance the branch... if worse, git reset back"
  - Significance: Complete, versioned, diffable experiment history with zero additional infrastructure.

- **Outcome-Memory Separated from Code-State:** results.tsv is deliberately untracked (gitignored) to separate what-was-tried-and-its-outcome from the-code-that-worked.
  - Evidence: `program.md:102` — "do not commit the results.tsv file, leave it untracked by git"
  - Significance: Clean git history (only kept experiments as commits), comprehensive experiment log (all attempts including failures).

- **Meta-Learning, Not Model Training:** Each run starts from torch.manual_seed(42) random init. No checkpoint saving. Progress = better code, not better weights.
  - Evidence: `train.py:458` — random init each run
  - Significance: This is a meta-learning setup. The agent learns a better training algorithm, not a better model.

- **Context-Safe Logging:** stdout redirect + targeted grep prevents context flooding.
  - Evidence: `program.md:99` — "redirect everything — do NOT use tee or let output flood your context"
  - Significance: The agent's context window is treated as a scarce resource with explicit architectural protection.

### Patterns
- Git branches as experiment namespaces
- Structured stdout as evaluation oracle
- results.tsv as long-term memory index that git cannot provide
- Context-safe logging via redirect

## Dimension 3: Tool & Action Space Design

### Summary
The tool space is maximally simple: 6-8 shell commands (git, edit, uv run, grep, tail) defined entirely in natural language in program.md. There is no formal tool API, no JSON schema, no MCP integration. Tool selection is not an LLM decision — it is a hardcoded sequence in the loop definition. LLM creativity is bottlenecked to a single degree of freedom: what mutation to make to train.py. The mutable/immutable boundary (train.py vs prepare.py) is enforced by natural language contract and file separation, not access control.

### Key Findings
- **Prose-as-Schema:** The entire tool API is natural language in program.md. No JSON schema, no typed interfaces.
  - Evidence: `program.md:96-104` — numbered steps with inline shell commands
  - Significance: Any LLM with shell access can participate. Zero tool infrastructure.

- **Structural Immutability via File Separation:** The boundary between mutable (train.py) and immutable (prepare.py) is enforced by physical file separation + natural language rule.
  - Evidence: `program.md:29` — "Modify prepare.py. It is read-only." Under CANNOT DO list.
  - Significance: The agent cannot game the metric without violating the read-only contract. For trading: backtest harness must be in a protected file.

- **Metric Oracle in Protected File:** evaluate_bpb lives in prepare.py (off-limits). The agent cannot modify the ground truth.
  - Evidence: `prepare.py` contains `evaluate_bpb()` function; `program.md:32` — "The evaluate_bpb function in prepare.py is the ground truth metric."
  - Significance: Tamper-proof evaluation. For trading: Sharpe/PnL calculation must be in the immutable harness, not the mutable strategy.

- **Deterministic Tool Sequence:** The loop defines a strict numbered procedure. The LLM has no choice about which tool to invoke — only what hypothesis to test.
  - Evidence: `program.md:94-104` — numbered 1-9 steps executed in order
  - Significance: Tool selection is NOT an LLM decision — it is a hardcoded state machine.

### Patterns
- Shell commands as tool API (no abstraction layer)
- Log-as-observation (redirect + grep, never direct output)
- Structural immutability via file separation
- Metric oracle in protected file
- Deterministic tool sequence (agent creativity confined to mutation only)

## Dimension 4: Multi-Agent Coordination

N/A — single-agent system. 7 multi-agent signals found in triage (mostly `subprocess` in prepare.py for data loading), below the threshold of 10. The README mentions running multiple instances on separate branches (`autoresearch/mar5-gpu0`) but coordination is by convention, not by code.

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions detected. All three dimensions agree on the core architecture.

### Cross-Cutting Flows

**Flow 1: The Experiment Lifecycle**
The full experiment lifecycle spans all three dimensions: program.md defines the prose loop (Dim 1), git + results.tsv provide state persistence (Dim 2), and shell commands provide the tool interface (Dim 3). The lifecycle is completely stateless between iterations — the agent can recover from a context reset by reading git state + results.tsv. Each iteration adds exactly 2 scalar values + 1 TSV row to the agent's working memory.

**Flow 2: The Tamper-Proof Evaluation**
The evaluation oracle is structurally protected from the agent through file separation (Dim 3), natural language contract (Dim 1), and outcome storage in an untracked file (Dim 2). The agent cannot game the metric without violating multiple constraints simultaneously.

### Novelty Assessment

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| Prose-as-Control-Flow | Dim 1 | NOVEL | No code-level analysis in existing wiki |
| Fixed-Budget Eval as Convergence Proxy | Dim 1 | NOVEL | Not discussed in any wiki article |
| Context-safe logging via redirect | Dim 1+2 | NOVEL | Concrete anti-flooding pattern not in wiki |
| Structural immutability via file separation | Dim 3 | NOVEL | Enforcement mechanism not in wiki |
| Metric oracle in protected file | Dim 3 | NOVEL | Ground truth protection pattern not in wiki |
| Outcome-memory separated from code-state | Dim 2 | NOVEL | Deliberate separation not in wiki |
| Multi-objective optimization (metric + simplicity) | Dim 1 | VARIANT | Wiki discusses eval criteria but not simplicity criterion |
| Git-as-State-Machine | Dim 2 | VARIANT | Wiki mentions git versioning but not branch-as-namespace |
| No convergence detection (runs until interrupted) | Dim 1 | KNOWN | Wiki discusses "LOOP FOREVER" pattern |

7 NOVEL, 2 VARIANT, 1 KNOWN findings.

## Decisions to Adopt

1. **Adopt: Git-branch-as-experiment-namespace** from program.md
   - What: Each strategy optimization run gets its own branch (`autoresearch/<date>`). Commits = kept hypotheses, reset = discarded.
   - Why: Complete, versioned, diffable experiment history with zero infrastructure. Branch HEAD always carries the current best strategy.
   - Effort: S
   - Target: Trading harness experiment management

2. **Adopt: Mutable/immutable file separation for tamper-proof evaluation** from program.md + prepare.py
   - What: Strategy code (mutable) in one file, backtest harness (immutable) in another. Agent cannot modify the evaluation metric.
   - Why: Prevents the agent from gaming the metric. The evaluation oracle must be structurally protected.
   - Effort: S
   - Target: Trading harness evaluation subsystem

3. **Adopt: Fixed-budget evaluation as convergence proxy** from prepare.py TIME_BUDGET
   - What: Every strategy backtest runs on a fixed time window (e.g., rolling 252-day period). Results are directly comparable.
   - Why: Eliminates training cost as a variable. Makes all experiments comparable without controlling for compute.
   - Effort: M
   - Target: Trading strategy evaluation engine

4. **Adopt: Minimal-signal extraction via targeted parsing** from program.md grep pattern
   - What: Redirect all backtest output to a log file, extract only 2-3 key metrics (Sharpe, max drawdown, PnL) via grep into the agent's context.
   - Why: Prevents context flooding. Each experiment adds ~5 lines to context instead of thousands.
   - Effort: S
   - Target: Trading harness context management

5. **Adopt: Untracked experiment ledger (results.tsv pattern)** from program.md
   - What: Keep an untracked TSV/CSV with all experiment outcomes (kept, discarded, crashed) separate from git history.
   - Why: Clean git history (only winning strategies as commits) + comprehensive experiment log (all attempts).
   - Effort: S
   - Target: Trading harness experiment tracking

## Evidence Index

Verified:
- program.md — agent instructions (the harness itself)
- train.py — mutable training code
- prepare.py — fixed utilities and evaluation
- README.md — project documentation
- .gitignore — tracks what's excluded
- pyproject.toml — dependencies

Unverified (runtime artifacts, not in static clone):
- results.tsv — created during experiment execution
- run.log — created during experiment execution

## Sources

- [karpathy/autoresearch](https://github.com/karpathy/autoresearch) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
