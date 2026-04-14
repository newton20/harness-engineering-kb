<!-- /autoplan restore point: /c/Users/dunliu/.gstack/projects/newton20-harness-engineering-kb/master-autoplan-restore-20260413-223226.md -->
---
title: "feat: /kb-code-research — Parallel Deep Code Research Skill"
type: feat
status: reviewed
date: 2026-04-14
requirements: docs/brainstorms/code-repo-deep-research-requirements.md
---

# /kb-code-research — Implementation Plan

## Overview

A Claude Code skill that deploys parallel specialist agents to deeply analyze open-source code repos, extracting agent harness patterns across 4 dimensions. Two-pass system: quick triage scorecard, then 2-wave parallel deep dive with 4 specialist agents, followed by orchestrator-driven synthesis and KB integration.

**Cost:** ~$18-40/repo (Sonnet for dimension agents, Opus for synthesis + source re-reads).
**Time:** ~5 min for triage-only, ~30-75 min for full deep dive.
**Token budget:** ~900K-1M per repo.

---

## 1. File Structure

The skill lives **project-level** (not ~/.claude/) because it's specific to this harness engineering KB and tightly coupled to its raw/wiki/outputs structure.

```
harness_engineering/
├── .claude/
│   └── skills/
│       └── kb-code-research/
│           └── SKILL.md              # The skill file (orchestrator prompt)
├── outputs/
│   ├── research-learnings.md         # Accumulated learnings across runs
│   ├── code-research-{repo}.md       # Per-repo research report (copy)
│   └── code-snippets-{repo}/        # Extracted code snippets (v2)
├── raw/
│   └── code-research-{repo}.md       # Per-repo report (KB-integrated copy)
├── scripts/
│   └── compile.js                    # Existing — called for KB integration
└── wiki/                             # Updated by /kb-compile after integration
```

**Why project-level:** The existing /kb-* skills at ~/.claude/skills/ are generic (work with any KB). This skill is specific to harness engineering research — it references this project's raw/, wiki/, outputs/, and scripts/. It also needs to be version-controlled with the KB.

**Single file vs. multi-file:** The entire skill is one SKILL.md. Dimension agent prompts are embedded as template strings within the orchestrator's instructions. This avoids file-reading overhead and keeps everything in one reviewable unit. The SKILL.md will be ~400-500 lines.

---

## 2. Phased Delivery

### Phase 1: Skeleton + Triage (this session)
- Create SKILL.md with orchestrator structure
- Implement repo access (clone or local path detection)
- Implement Pass 1 triage scorecard
- Implement file index / hot path builder
- Test on `karpathy/autoresearch` (small, focused repo) — triage only

### Phase 2: Dimension Agents + Synthesis (next session)
- Implement all 4 dimension agent prompts
- Implement 2-wave dispatch (Wave 1: dims 1+2, Wave 2: dims 3+4)
- Implement synthesis logic (orchestrator merges, validates evidence)
- Implement report generation with standard frontmatter
- Test full pipeline on `karpathy/autoresearch`

### Phase 3: KB Integration + Learnings (same or next session)
- Implement raw/ file writing with proper frontmatter
- Implement `node scripts/compile.js delta` call
- Implement learnings accumulation (outputs/research-learnings.md)
- Implement log.md append
- Test full pipeline including KB integration on autoresearch
- Run on 1-2 more P0 repos to validate

### Phase 4: Polish + Queue (later session)
- Run remaining P0 repos (hermes-agent, openclaw, Claude Code local, deepagents)
- Run P1 repos (opencode, MiroFish)
- Tune dimension prompts based on learnings
- Human review of learnings file after 3-5 repos
- Consider trading KB cross-reference output (deferred per requirements)

---

## 3. Component Specifications

### 3.1 SKILL.md — Orchestrator

```yaml
---
name: kb-code-research
description: >
  Deep code research on agent/harness repos. Deploys parallel specialist agents
  to analyze architecture, memory, tools, and multi-agent patterns. Use when asked
  to "research repo", "analyze codebase", "kb code research", "deep dive on repo",
  or "extract patterns from".
argument-hint: "<repo-url-or-local-path> --goal \"<research goal>\""
---
```

The orchestrator is the SKILL.md body itself. It does NOT spawn a sub-agent for itself — it IS the top-level agent running in the user's session. It spawns sub-agents only for the 4 dimension analyses.

**Orchestrator responsibilities (performed directly, not delegated):**
1. Parse arguments (repo URL/path, --goal)
2. Access repo (git clone --depth=1 or validate local path)
3. Run Pass 1 triage
4. Build file index + identify hot paths
5. Dispatch dimension agents in 2 waves
6. Synthesize results (merge, validate evidence, resolve contradictions)
7. Write research report
8. Integrate into KB (write raw/, call compile.js)
9. Append to learnings and log

### 3.2 Repo Access

```
## Step 1: Access Repo

Parse the argument: <repo> #$ARGUMENTS

1. If argument looks like a URL (contains github.com or starts with https://):
   - Clone with shallow depth: `git clone --depth=1 <url> /tmp/kb-research-{repo-name}`
   - Set REPO_PATH to the clone directory
   - Set REPO_NAME from the URL (e.g., "karpathy-autoresearch")

2. If argument is a local path:
   - Verify path exists with `ls`
   - Set REPO_PATH to the absolute path
   - Set REPO_NAME from the directory name (e.g., "claude-code-src")

3. Extract --goal from arguments. If missing, default to:
   "Extract agent harness architecture patterns, design decisions, and reusable components"

4. Read outputs/research-learnings.md if it exists, for context from prior runs.
```

### 3.3 Pass 1 — Triage Scorecard

A single quick pass performed by the orchestrator (not a sub-agent). Takes ~5 minutes.

```
## Step 2: Triage Scorecard

Quickly assess the repo — spend no more than 5 minutes on this step.

1. Read the README (or README.md, readme.md) if it exists
2. Run: `find {REPO_PATH} -type f -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.rs" -o -name "*.go" | head -200`
   to get a file listing and estimate repo size
3. Run: `git -C {REPO_PATH} log --oneline -20` to check recent activity
4. Run: `wc -l $(find {REPO_PATH} -type f \( -name "*.py" -o -name "*.ts" -o -name "*.js" \) | head -50)` for code volume

Produce a triage scorecard:

| Dimension | Value |
|-----------|-------|
| Repo name | {name} |
| Primary language | {lang} |
| Size classification | tiny (<1K LOC) / small (1-10K) / medium (10-50K) / large (50K+) |
| File count | {N} source files |
| Last commit | {date} |
| Commit frequency | {active/maintained/stale/dead} |
| README quality | {none/stub/adequate/detailed} |
| Relevance to goal | {0-10 score with 1-sentence justification} |
| Agent/harness signals | {list of signals found: agent loop, tool definitions, memory system, etc.} |
| Recommended dimensions | {which of the 4 dimensions are applicable} |

**Gate:** If relevance score < 3, print the scorecard and stop:
"Triage score {N}/10 — below threshold. Skipping deep dive. Scorecard saved."
Save scorecard to outputs/triage-{repo-name}.md and append to log.md. STOP HERE.

**If score >= 3:** Continue to Step 3.
```

### 3.4 File Index & Hot Path Builder

```
## Step 3: Build File Index

Identify the most important files for focused agent work. This prevents dimension
agents from wasting tokens on irrelevant code.

1. Use Glob to find all source files by type
2. Identify "hot paths" — files most likely to contain harness patterns:
   - Entry points: main.*, index.*, app.*, cli.*, __main__.py
   - Agent/loop files: *agent*, *loop*, *run*, *harness*, *orchestrat*
   - Tool files: *tool*, *action*, *command*, *function*
   - Memory files: *memory*, *state*, *context*, *session*, *persist*
   - Config files: *config*, *setting*, *prompt*, *.env*, CLAUDE.md, AGENTS.md, SOUL.md
   - Multi-agent: *worker*, *swarm*, *delegate*, *dispatch*, *coordinator*
3. Read the top-level directory listing and any src/ or lib/ directory
4. Produce a HOT_PATHS list (max 40 files) sorted by likely relevance

This HOT_PATHS list is passed to each dimension agent in their prompt to focus
their exploration. Agents CAN read files outside this list but should START here.
```

### 3.5 Dimension Agents — Dispatch Pattern

```
## Step 4: Dispatch Dimension Agents

Use the Agent tool to spawn dimension agents. Run in 2 waves of 2 agents each
to stay within concurrency limits.

### Wave 1 (parallel):
- Agent 1: Architecture & Loop Design
- Agent 2: Memory & State Management

Wait for both Wave 1 agents to complete before starting Wave 2.

### Wave 2 (parallel):
- Agent 3: Tool & Action Space Design
- Agent 4: Multi-Agent Coordination

Skip any dimension marked as "N/A" in the triage scorecard (e.g., if the repo
is single-agent, skip Dimension 4). Replace skipped dimensions with a note:
"Dimension N/A: {reason from triage}"

Each agent receives this context in its prompt:
- REPO_PATH: {path}
- REPO_NAME: {name}
- RESEARCH_GOAL: {goal}
- HOT_PATHS: {list from Step 3}
- TRIAGE_SUMMARY: {scorecard from Step 2}

Each agent MUST use model: "sonnet" to control costs.

Each agent returns a structured report as its final message. Parse these
reports from the agent results for synthesis.
```

### 3.6 Dimension Agent Prompts

Each dimension agent prompt follows the same structure:

```
You are a specialist code researcher analyzing {REPO_NAME} for {DIMENSION_NAME}.

## Context
- Repo path: {REPO_PATH}
- Research goal: {RESEARCH_GOAL}
- Repo size: {SIZE_CLASSIFICATION}
- Triage summary: {TRIAGE_SUMMARY}

## Priority files to start with
{HOT_PATHS — filtered to this dimension's likely files}

## Your Research Checklist
{dimension-specific questions — see below}

## Instructions
1. Start with the hot path files. Use Read to examine them.
2. Use Grep to search for patterns across the codebase.
3. Use Glob to find additional relevant files.
4. For each checklist question, find SPECIFIC code evidence (file path + line range).
5. If a question is not applicable, explicitly state "N/A: {reason}".
6. Do NOT speculate. If you can't find evidence, say "Not found in codebase."

## Output Format
Return your findings as a structured report in this exact format:

# {DIMENSION_NAME} — {REPO_NAME}

## Summary
{3-5 sentence summary of key findings for this dimension}

## Findings

### {Checklist Question 1}
**Answer:** {concise answer}
**Evidence:** `{file_path}:{line_start}-{line_end}` — {what the code shows}
**Significance:** {why this matters for harness design}

### {Checklist Question 2}
...

## Key Patterns Discovered
- {Pattern 1}: {description} — Evidence: `{path}`
- {Pattern 2}: ...

## Architecture Diagram (if applicable)
```mermaid
{diagram}
```

## Novelty Assessment
{What in these findings is NOT already covered by common agent harness patterns?
What is genuinely novel or surprising?}
```

#### Dimension 1: Architecture & Loop Design — Checklist

```
1. What is the core agent loop structure?
   Search for: while loops with tool/LLM calls, ReAct patterns, plan-execute cycles.
   Grep patterns: "while", "loop", "iterate", "step", "turn", "cycle"

2. Who controls the loop — the model or the code?
   "Model controls": model decides when to stop (tool_use vs. end_turn)
   "Code controls": code checks conditions, calls model in a loop

3. How does the loop terminate?
   Search for: break conditions, max iterations, token limits, convergence checks,
   task_complete signals, error thresholds.

4. What is the orchestration topology?
   Single agent? Sequential pipeline? Parallel dispatch? Hierarchical?
   Search for: agent spawning, subprocess calls, worker pools, delegation.

5. How are context windows managed?
   Search for: compaction, summarization, context reset, sliding window,
   message truncation, handoff artifacts, checkpoint files.

6. What is the system prompt structure?
   Look for: prompt templates, system message construction, conditional
   prompt sections, layered prompts, CLAUDE.md loading.

7. How does progressive disclosure work?
   Are all tools/context given upfront? Loaded lazily? On-demand?
   Search for: deferred loading, tool filtering, dynamic prompt assembly.
```

#### Dimension 2: Memory & State Management — Checklist

```
1. What memory systems exist?
   Search for: in-context memory, file-based state, databases, vector stores,
   git-backed state, session files, checkpoint mechanisms.

2. Is there short-term vs. long-term memory separation?
   Short-term: current conversation, working memory, scratchpad.
   Long-term: persistent files, databases, cross-session state.

3. How does state persist across sessions?
   Search for: state files, checkpoints, git commits as state, databases,
   serialization, progress tracking files.

4. What survives context window compaction?
   Search for: compaction hooks, summary generation, priority markers,
   "must keep" annotations, pinned messages.

5. Is there a memory hierarchy? (e.g., core/working/archival like MemGPT)
   Search for: memory tiers, eviction policies, promotion/demotion logic.

6. Can agents modify their own instructions or memory?
   Search for: self-modifying prompts, learned preferences, memory write tools,
   instruction updates, CLAUDE.md auto-editing.

7. How is conversation history managed for long-running tasks?
   Search for: message trimming, history summarization, context budgets,
   conversation splitting, session handoffs.
```

#### Dimension 3: Tool & Action Space Design — Checklist

```
1. How many tools are defined? What categories?
   Search for: tool definitions, function schemas, action handlers.
   Categorize: file ops, code execution, search, web, git, communication, MCP.

2. Are tools primitives or high-level integrations?
   Primitives: read_file, write_file, run_bash.
   High-level: create_pr, deploy_app, run_test_suite.

3. How are tool schemas defined?
   Search for: JSON schema, function signatures, TypeScript types,
   tool description strings, parameter definitions.

4. Is there lazy/dynamic tool loading?
   Search for: deferred tools, tool registries, conditional tool injection,
   tool search/discovery mechanisms.

5. How are tool failures handled?
   Search for: try/catch around tool calls, retry logic, fallback tools,
   error-as-context patterns, graceful degradation.

6. Are there MCP integrations?
   Search for: MCP, model context protocol, server definitions,
   tool providers, external tool sources.

7. What is the tool selection mechanism?
   LLM native choice? Routing logic? Classifier? Hybrid?
   Search for: tool routing, tool selection, tool filtering.
```

#### Dimension 4: Multi-Agent Coordination — Checklist

```
1. Is this a single-agent or multi-agent system?
   Search for: agent spawning, worker processes, subprocess calls,
   parallel execution, delegation patterns.

2. What is the coordination topology?
   Orchestrator-worker? Peer-to-peer? Hierarchy? Swarm?
   Search for: orchestrator, coordinator, dispatcher, worker, swarm, fleet.

3. How are tasks delegated?
   Explicit decomposition by a planner? Dynamic spawning on demand?
   Search for: task queue, work items, delegation, decompose, assign.

4. How do agents communicate?
   Shared files? Message passing? Shared context? Tool calls?
   Search for: message, channel, pipe, shared state, artifact, handoff.

5. Is there a "game of telephone" problem?
   How much information is lost in agent-to-agent handoffs?
   Search for: context passing, summary generation, handoff protocols.

6. How is work deduplicated across agents?
   Search for: dedup, already done, conflict detection, merge logic.

7. Is execution synchronous or asynchronous?
   Search for: await, Promise, async, parallel, concurrent, background.
```

### 3.7 Synthesis (Orchestrator-Performed)

```
## Step 5: Synthesize Results

After all dimension agents complete, the orchestrator synthesizes directly.
Do NOT spawn a sub-agent for synthesis.

### 5a: Evidence Validation
For every file path cited in any dimension report:
- Check if the file exists: `ls {REPO_PATH}/{cited_path}`
- If the file does NOT exist, mark that evidence as [UNVERIFIED]
- If >30% of evidence is unverified, flag a warning in the report

### 5b: Cross-Dimension Contradiction Resolution
Read all 4 dimension reports. Look for contradictions:
- Dimension 1 says "model controls loop" but Dimension 4 says "orchestrator dispatches"
- Dimension 2 says "no persistence" but Dimension 3 says "checkpoint tool exists"

For each contradiction:
- Check the cited evidence for both sides
- Resolve based on what the code actually shows
- Note the resolution in the report

### 5c: Novelty Assessment
Read wiki/_index.md and skim 3-5 relevant wiki articles.
For each key finding, assess:
- Is this already documented in the KB? → Mark as "KNOWN"
- Is this a new variation of a known pattern? → Mark as "VARIANT"
- Is this genuinely novel? → Mark as "NOVEL"

The report MUST contain at least 1 NOVEL or VARIANT finding.
If not, note: "No novel findings beyond existing KB coverage."
```

### 3.8 Research Report Format

```
## Step 6: Write Research Report

Write the report to BOTH locations:
1. `raw/code-research-{REPO_NAME}.md` (for KB integration)
2. `outputs/code-research-{REPO_NAME}.md` (archival copy)
```

**Report template:**

```markdown
---
title: "Code Research: {Repo Name}"
source: {repo_url_or_path}
author: "kb-code-research skill"
date: {YYYY-MM-DD}
fetched: {YYYY-MM-DD}
type: code-research
status: raw
tags: [code-research, {primary-patterns-found}]
relevance_score: {0-10}
research_goal: "{goal}"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
token_estimate: {approximate tokens used}
---

# Code Research: {Repo Name}

## Executive Summary

{3-5 sentences: what this repo is, what harness patterns it demonstrates,
what's novel, and what's most relevant to our goals.}

## Triage Scorecard

{Scorecard table from Step 2}

## Dimension 1: Architecture & Loop Design

### Summary
{3-5 sentence dimension summary}

### Key Findings
{Merged and validated findings from the dimension agent.
Each finding has: description, evidence (file:line), significance.}

### Patterns
{Bulleted list of patterns with evidence}

### Architecture Diagram
```mermaid
{diagram from dimension agent, validated}
```

## Dimension 2: Memory & State Management
{Same structure as Dimension 1}

## Dimension 3: Tool & Action Space Design
{Same structure as Dimension 1}

## Dimension 4: Multi-Agent Coordination
{Same structure as Dimension 1, or "N/A — single-agent system" with explanation}

## Cross-Cutting Analysis

### Contradiction Resolutions
{Any contradictions found between dimensions and how they were resolved}

### Novelty Assessment
| Finding | Status | Notes |
|---------|--------|-------|
| {finding} | NOVEL / VARIANT / KNOWN | {context} |

### Adoption Recommendations
{Prioritized list of what to adopt from this repo, ordered by impact.
Each item: what it is, why it matters, rough effort to extract/adapt.}

## Evidence Index

{All file paths cited in the report, with existence verification status.
Format: ✓ path/to/file.py:42-67 — description
         ✗ path/to/missing.py — [UNVERIFIED]}

## Sources

- [{Repo Name}]({repo_url}) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
```

### 3.9 KB Integration (Orchestrator-Performed)

```
## Step 7: KB Integration

1. The raw/ copy was already written in Step 6.
   Verify it exists: `ls raw/code-research-{REPO_NAME}.md`

2. Run compile delta to verify it's detected:
   ```bash
   node scripts/compile.js delta
   ```

3. Do NOT auto-compile into wiki. Print:
   "Research report saved to raw/code-research-{REPO_NAME}.md
   Run /kb-compile to integrate findings into wiki articles."

   Rationale: Wiki compilation involves cross-pollination across 10-15 articles.
   This is a significant operation that should be a deliberate step, not a side
   effect of research. The user can batch-compile multiple research reports.
```

### 3.10 Learnings Accumulation

```
## Step 8: Accumulate Learnings

Append a structured entry to outputs/research-learnings.md:

### {REPO_NAME} — {YYYY-MM-DD}

**Research goal:** {goal}
**Relevance score:** {score}/10
**Dimensions analyzed:** {list}
**Wall time:** ~{estimate}
**Evidence validation:** {X}% verified, {Y}% unverified

**What worked well:**
- {e.g., "Hot path builder correctly identified the main loop in src/agent.py"}
- {e.g., "Dimension 2 agent found memory system that wasn't obvious from README"}

**What didn't work:**
- {e.g., "Dimension 4 was N/A — wasted time before determining single-agent"}
- {e.g., "Grep for 'tool' returned too many false positives in docs/"}

**Missed findings (discovered during synthesis, not by any dimension agent):**
- {e.g., "The error recovery pattern was in util/retry.py but no dimension looked there"}

**Prompt improvement suggestions:**
- {e.g., "Dimension 1 should also grep for 'retry' and 'backoff' — these are loop-relevant"}
- {e.g., "Hot path builder should include test files — they reveal intended usage patterns"}

---
```

### 3.11 Log Entry

```
## Step 9: Append to Log

Append to log.md:
- **[{timestamp}]** — /kb-code-research on {REPO_NAME}: relevance {score}/10,
  {N} novel findings, {M} variant findings. Report: raw/code-research-{REPO_NAME}.md
```

---

## 4. Error Handling & Edge Cases

### Repo Access Failures

| Scenario | Handling |
|----------|----------|
| Git clone fails (auth, network) | Print error, suggest manual clone or local path. STOP. |
| Local path doesn't exist | Print error with path. STOP. |
| Repo is empty (no source files) | Triage produces score 0, skip deep dive. |
| Repo is a monorepo | Triage notes subproject structure. User should pass path to specific subdir. |

### Dimension Agent Failures

| Scenario | Handling |
|----------|----------|
| Agent returns empty/minimal results | Flag in synthesis as "Dimension {N}: insufficient findings". Include what WAS found. |
| Agent times out | Use whatever partial results were returned. Note timeout in report. |
| Agent hallucinates file paths | Evidence validation in Step 5a catches this. Mark as [UNVERIFIED]. |
| Dimension is N/A for this repo | Skip agent, note "N/A: {reason}" in report. Saves ~$2-5 per skipped dimension. |

### Repo Size Edge Cases

| Size | Handling |
|------|----------|
| Tiny (<1K LOC) | All 4 dimensions may finish in <5 min each. Proceed normally. Report may be short. |
| Large (50K+ LOC) | Hot path builder is critical. Dimension agents should use Grep heavily, Read selectively. May hit token limits — agents should prioritize checklist coverage over exhaustive reading. |
| Non-code repo (mostly docs) | Triage catches this (low agent/harness signals). Score likely <3, skip deep dive. |

### Existing Research

| Scenario | Handling |
|----------|----------|
| raw/code-research-{repo}.md already exists | Print warning: "Previous research exists. Overwrite? (default: yes, append -v2 suffix)". In auto mode, overwrite. |
| Repo was previously triaged as irrelevant | Note in log that it was re-evaluated. Proceed normally. |

---

## 5. Testing Strategy

### Phase 1 Test: Triage Only
```
/kb-code-research https://github.com/karpathy/autoresearch --goal "extract autoresearch loop patterns"
```
**Verify:** Triage scorecard is produced. Score is reasonable (expect 7-9 for this repo). Hot paths identify the main loop files. No dimension agents dispatched yet.

### Phase 2 Test: Full Pipeline (Single Repo)
```
/kb-code-research https://github.com/karpathy/autoresearch --goal "extract autoresearch loop patterns for trading strategy self-improvement"
```
**Verify:**
- 4 dimension agents dispatched in 2 waves (or fewer if N/A)
- Each dimension report has evidence with real file paths
- Synthesis validates evidence (>70% verified)
- Research report is well-structured with all sections
- At least 1 NOVEL or VARIANT finding

### Phase 3 Test: KB Integration
**Verify after Phase 2 test:**
- `raw/code-research-karpathy-autoresearch.md` exists with valid frontmatter
- `node scripts/compile.js delta` shows it as uncompiled
- `outputs/code-research-karpathy-autoresearch.md` exists (archival copy)
- `outputs/research-learnings.md` has an entry
- `log.md` has a new line

### Phase 3 Test: Large Repo
```
/kb-code-research C:\Users\dunliu\Downloads\Claude\ Code\src --goal "reverse-engineer Claude Code harness architecture"
```
**Verify:** Hot path builder handles the larger file set. Dimension agents stay within token budget. Evidence validation works on local paths.

---

## 6. Cost & Performance Estimates

| Component | Model | Est. Tokens | Est. Cost |
|-----------|-------|-------------|-----------|
| Triage (orchestrator) | Opus (inherited) | 10-20K | $0.50-1.00 |
| File index builder | Opus (inherited) | 5-10K | $0.25-0.50 |
| Dimension agent (×4) | Sonnet (explicit) | 150-200K each | $1.50-2.00 each |
| Synthesis (orchestrator) | Opus (inherited) | 50-100K | $2.50-5.00 |
| KB integration | Opus (inherited) | 5-10K | $0.25-0.50 |
| **Total per repo** | | **~700K-1.1M** | **$18-40** |

> **Note (from autoplan review):** Original estimate was $8-15. Revised to $18-40 to account for: (1) orchestrator context accumulation during synthesis (~300-500K tokens of Opus), (2) source file re-reads during contradiction resolution (user-approved override, adds 50-100K tokens). The $18-40 range is realistic for medium repos. Tiny repos may cost $10-15, large repos $30-45.

Wave 1 and Wave 2 run sequentially (not all 4 at once), so peak concurrency is 2 sub-agents + the orchestrator. This stays well within Claude Code's concurrency limits.

---

## 7. Deferred to v2

These items are explicitly NOT in v1:

1. **Dimensions 5-8** (Safety, Self-Improvement, Reusable Components, Domain Mapping) — add as optional Wave 3 later
2. **Automated self-improvement** — v1 accumulates learnings, human reviews every 3-5 repos
3. **Trading KB cross-reference** — v1 notes trading relevance in the main report; a separate `outputs/trading-xref-{repo}.md` is deferred
4. **Code snippets extraction** — v1 cites code by path+line; physical extraction to `outputs/code-snippets-{repo}/` is deferred
5. **Knowledge graph** — deferred indefinitely
6. **`--domain` flag** — pluggable domain relevance (trading, security, etc.) for Dimension 8
7. **Incremental re-research** — re-analyzing a previously researched repo for updates
8. **Web UI / dashboard** — markdown artifacts are sufficient for v1

---

## 8. Implementation Sequence (Detailed Steps)

### Step-by-step for Phase 1 (Skeleton + Triage):

1. Create `.claude/skills/kb-code-research/SKILL.md` with:
   - Frontmatter (name, description, argument-hint)
   - Step 1: Argument parsing + repo access
   - Step 2: Triage scorecard (inline, not sub-agent)
   - Step 3: File index + hot path builder
   - Stub for Step 4-9: "Coming in Phase 2"
2. Create `outputs/research-learnings.md` with initial header
3. Test: invoke `/kb-code-research https://github.com/karpathy/autoresearch --goal "test triage"`
4. Verify triage scorecard output quality

### Step-by-step for Phase 2 (Dimension Agents + Synthesis):

5. Add Step 4 to SKILL.md: Wave dispatch with all 4 dimension prompts
6. Add Step 5: Synthesis (evidence validation, contradiction resolution, novelty assessment)
7. Add Step 6: Report generation with full template
8. Test: full pipeline on `karpathy/autoresearch`
9. Review dimension agent output quality, tune prompts if needed

### Step-by-step for Phase 3 (KB Integration + Learnings):

10. Add Step 7: KB integration (write raw/, run compile delta)
11. Add Step 8: Learnings accumulation
12. Add Step 9: Log entry
13. Add error handling for all edge cases
14. Test: full pipeline with KB integration
15. Run on 2nd P0 repo to validate generalization

---

## 9. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Skill location | Project-level (.claude/skills/) | Tightly coupled to this KB's structure; needs version control |
| Dimension agent model | Sonnet (explicit) | Cost control: 4 × Sonnet ≪ 4 × Opus |
| Synthesis model | Opus (inherited from session) | Synthesis requires judgment that benefits from Opus |
| Sub-agent count | 4 max, 2 per wave | Claude Code concurrency limits; diminishing returns beyond 4 |
| KB integration | Write raw/ only, don't auto-compile | Wiki compilation is a heavy cross-pollination step; keep deliberate |
| Triage threshold | Score < 3 skips | Low bar intentionally — avoid filtering potentially valuable repos |
| Evidence validation | File-existence checks on all paths | Hallucinated paths are the #1 quality risk for code research |
| Learnings | Append-only file, human reviews | v1 simplicity; automated prompt revision needs quality baselines |
| Single SKILL.md | All prompts embedded | One reviewable unit; no file-reading overhead for dimension prompts |
| No nested sub-agents | Synthesis + KB done by orchestrator | Avoids "game of telephone"; synthesis needs full context from all dims |

---

## 10. Open Decisions (Resolved from Requirements)

| Question | Resolution |
|----------|------------|
| Dimension agents see existing KB? | **No.** Research "blind" to avoid confirmation bias. Novelty check happens in synthesis only. |
| Token budget per dimension? | **150-200K** (Sonnet). Enough for thorough exploration of medium repos. |
| Trading KB cross-reference automated? | **Manual.** v1 produces recommendations in the main report. Separate xref doc is v2. |
| Incremental re-research? | **Deferred to v2.** v1 overwrites previous research. |
| Web UI for browsing results? | **No.** Markdown artifacts are sufficient. |

---

<!-- AUTONOMOUS DECISION LOG -->
## Decision Audit Trail

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|---------------|-----------|-----------|----------|
| 1 | CEO-0F | Mode: SELECTIVE EXPANSION | Mechanical | P6 | New feature on existing platform → default SELECTIVE | EXPANSION, HOLD, REDUCTION |
| 2 | CEO-0C | Approach C (Layered Research Protocol) | Mechanical | P6 | Already validated in requirements, Agent tool access is the differentiator | Approaches A, B |
| 3 | CEO-1 | Fix /tmp/ path to use mktemp -d | Mechanical | P5 | Windows compatibility — /tmp/ unreliable on Git Bash | Keep /tmp/ |
| 4 | CEO-1 | Add clone cleanup step at end of skill | Mechanical | P1 | Prevents temp dir accumulation across runs | Leave clones |
| 5 | CEO-1 | Revise cost estimate to $15-35/repo | Mechanical | P3 | Orchestrator context accumulation adds $5-10 over dimension costs | Keep $8-15 |
| 6 | CEO-2 | Add error handling for compile.js failures | Mechanical | P1 | Node.js missing or script error is unhandled | Skip |
| 7 | CEO-2 | Add error handling for total agent failure | Mechanical | P1 | If Agent tool crashes, continue with remaining dimensions | Skip |
| 8 | CEO-4 | Sanitize repo name for filenames | Mechanical | P5 | Special chars in repo names break file paths | Skip |
| 9 | CEO-4 | Warn if zero hot paths found | Mechanical | P5 | Agents need guidance — "explore from root" is better than silent | Skip |
| 10 | CEO-6 | Add test scenarios for edge cases | Mechanical | P3 | 4 gaps in test coverage for manual testing | Skip |
| 11 | CEO-7 | Add --no-recurse-submodules to clone | Mechanical | P5 | Prevent unexpected large clone for repos with submodules | Skip |
| 12 | CEO-8 | Add progress messages between waves | Mechanical | P1 | 30-75 min with no feedback is poor DX | Skip |
| 13 | CEO-10 | Add "Decision Extraction" section to report | Taste | P1 | Both dual voices flagged adoption as #1 risk — reports need actionable output | Skip section |
| 14 | CEO-DV | Add cross-cutting integration pass in synthesis | Taste | P1 | Subagent: dimensions miss cross-cutting patterns. Worth adding. | Skip |
| 15 | CEO-DV | Add Dimension 4 skip signal (grep count) | Taste | P3 | Subagent: most repos are single-agent, dim 4 wastes $2-5 | Always run dim 4 |
| 16 | CEO-DV | Consider --skip-triage for curated queue | Taste | P3 | Both voices: triage is ritual for known-good repos | Always triage |
| 17 | CEO-DV | Validate single-pass baseline first | Taste | P6 | Subagent: prove multi-agent is better than one Sonnet pass | Build multi-agent directly |

---

## CEO Review Outputs

### NOT in scope (deferred from CEO review)

1. **Automated self-improvement** — v1 accumulates learnings, human reviews. Rationale: need quality baseline from 10+ repos first.
2. **Trading KB cross-reference doc** — v1 notes relevance in main report. Separate xref is v2.
3. **Code snippets physical extraction** — v1 cites code by path+line. Extraction to files is v2.
4. **Knowledge graph** — deferred indefinitely.
5. **Incremental re-research / delta tracking** — Codex flagged this as 6-month regret. Acknowledged, still v2.
6. **Evidence content validation** (beyond file-existence) — Codex flagged file-existence as too weak. Acknowledged, but content-level validation requires reading cited code during synthesis, which would push token costs even higher. Deferred to v2 with explicit warning in report when >30% evidence is unverified.

### What already exists

| Sub-problem | Existing code | Status |
|-------------|--------------|--------|
| KB raw file schema | CLAUDE.md frontmatter spec | Reuse directly |
| Wiki compilation | /kb-compile skill + compile.js | Reuse (user invokes after research) |
| File search/read in repos | Agent tool (Glob, Grep, Read) | Reuse via dimension agents |
| Status tracking | log.md convention | Reuse |
| Health checks | compile.js health command | Reuse |

### Dream state delta

This plan takes us from "zero code research capability" to "structured 4-dimension research pipeline producing KB-integrated reports." The 12-month ideal includes self-improving prompts, 50+ repos analyzed, and a knowledge graph. This plan gets us to ~30% of the 12-month ideal — the pipeline exists but lacks self-improvement, delta tracking, and depth beyond 4 dimensions.

### Error & Rescue Registry

| Codepath | Failure Mode | Handled? | Action | User Sees |
|----------|-------------|----------|--------|-----------|
| git clone | Auth failure | Y | Print error, STOP | "Clone failed — try manual clone or local path" |
| git clone | Network timeout | Y | Print error, STOP | "Clone failed" |
| git clone | Disk full | N → FIX | Check exit code | "Clone failed: disk full" |
| local path check | Path is file not dir | N → FIX | Check with test -d | "Path is a file, not a directory" |
| Agent dispatch | Agent crashes | N → FIX | Catch, continue with remaining dims | "Dimension N failed, continuing with remaining" |
| Agent dispatch | Agent returns empty | Y | Flag in synthesis | Report notes "insufficient findings" |
| compile.js delta | Node.js missing | N → FIX | Check which node first | "Node.js not found — install to enable KB integration" |
| compile.js delta | Script error | N → FIX | Check exit code | "compile.js failed — check scripts/" |
| Evidence validation | Path has spaces | N → FIX | Quote paths | (transparent) |

### Failure Modes Registry

| Codepath | Failure Mode | Rescued? | Test? | User Sees? | Logged? |
|----------|-------------|----------|-------|------------|---------|
| Clone + research | Private repo | Y | Phase 1 | Error + stop | Y (log.md) |
| Dim agent | Hallucinated paths | Y | Phase 2 | [UNVERIFIED] tags | Y (report) |
| Dim agent | All 4 fail | N ← **CRITICAL** | No | Unclear | N |
| Synthesis | >30% unverified | Y | Phase 2 | Warning in report | Y (report) |
| KB integration | compile.js fails | N ← **CRITICAL** | No | Silent failure | N |

**2 CRITICAL GAPS** identified and addressed in Decision Audit Trail items #6 and #7.

### CEO Completion Summary

```
  +====================================================================+
  |            MEGA PLAN REVIEW — COMPLETION SUMMARY (CEO)              |
  +====================================================================+
  | Mode selected        | SELECTIVE EXPANSION                          |
  | Step 0               | Approach C confirmed, 5 premises valid       |
  | Section 1  (Arch)    | 3 issues (temp path, cleanup, cost model)    |
  | Section 2  (Errors)  | 4 gaps found, all fixable                    |
  | Section 3  (Security)| 1 low-severity (prompt injection awareness)  |
  | Section 4  (Data/UX) | 2 edge cases (hot path empty, name sanitize) |
  | Section 5  (Quality) | 0 issues                                     |
  | Section 6  (Tests)   | 4 test scenario gaps                         |
  | Section 7  (Perf)    | 1 low (submodules flag)                       |
  | Section 8  (Observ)  | 1 medium (progress messaging)                 |
  | Section 9  (Deploy)  | 0 issues (skill file, trivial rollback)       |
  | Section 10 (Future)  | 1 high (adoption forcing function)            |
  | Section 11 (Design)  | SKIPPED (no UI scope)                         |
  +--------------------------------------------------------------------+
  | NOT in scope         | 6 items deferred                              |
  | What already exists  | 5 reuse points mapped                         |
  | Dream state delta    | ~30% of 12-month ideal                        |
  | Error/rescue registry| 9 codepaths, 4 gaps → fixed                   |
  | Failure modes        | 5 total, 2 CRITICAL GAPS → fixed              |
  | TODOS.md updates     | 0 (all scope decided)                         |
  | Scope proposals      | 5 surfaced from dual voices, 5 taste decisions |
  | CEO plan             | Not written (SELECTIVE — inline only)           |
  | Dual voices          | Codex + Claude subagent, 3/6 confirmed         |
  | Diagrams produced    | 1 (architecture pipeline)                      |
  | Unresolved decisions | 0 (all auto-decided, 5 taste → final gate)     |
  +====================================================================+
```

---

## Eng Review Outputs

### Eng Step 0: Scope Challenge

1. **Existing code leverage:** 5 reuse points mapped (same as CEO). No unnecessary rebuilding.
2. **Minimum changes:** 1 new file (SKILL.md) + 1 new file (research-learnings.md) + generated outputs. Under the 8-file smell threshold.
3. **Complexity check:** PASS — no new classes/services. One prompt file.
4. **Search check:** The Agent tool's `model` parameter is confirmed (checked against tool schema). Sub-agents WILL run on Sonnet as specified.
5. **Distribution check:** No new artifact type. SKILL.md is consumed by Claude Code directly.
6. **Completeness check:** The plan is the complete version with all 4 dimensions, 2-wave parallelism, synthesis, and KB integration. No shortcuts.

Scope accepted as-is.

### Eng Section 1: Architecture Review (confidence 8/10)

**[P1] (confidence 9/10)** SKILL.md Step 2, line 143-147 — **Shell commands assume Unix.** `find`, `wc -l $(find ...)`, and pipe patterns are central to triage and file indexing. On Windows Git Bash, `wc -l $(find ...)` with 50+ files can hit argument list limits. The path `C:\Users\dunliu\Downloads\Claude\ Code\src` has a space — unquoted `find` will break.
**Fix:** Use Claude Code's Glob and Grep tools instead of shell find/wc. These are cross-platform and handle spaces correctly. This also makes the skill more consistent with how dimension agents explore code.

**[P2] (confidence 9/10)** SKILL.md Step 1, line 119 — **CWD guard missing.** The skill writes to relative paths (`raw/`, `outputs/`, `log.md`). If invoked from a different CWD than the project root, all writes land in wrong locations.
**Fix:** Add a CWD check at Step 1: verify `scripts/compile.js` exists, or cd to the project root explicitly.

**[P2] (confidence 7/10)** Synthesis step — **Token budget for re-reading evidence.** Contradiction resolution (Step 5b) requires re-reading source files at synthesis time, after the orchestrator already holds 4 dimension reports. This adds 50-100K tokens on top of the existing ~800K context.
**Fix:** Acknowledge this in the cost model. Consider: synthesis reads dimension reports but does NOT re-read source files for contradiction resolution. Instead, trust the evidence cited by dimension agents and resolve based on report content alone.

No issues found in: dependency graph, coupling (appropriate for project-level skill), rollback (delete file), scaling (2 concurrent agents is the limit).

### Eng Section 2: Code Quality Review (confidence 8/10)

**[P1] (confidence 8/10)** `#$ARGUMENTS` parsing, line 115 — **Argument parsing is fragile.** The orchestrator must correctly parse: `https://github.com/foo/bar --goal "extract patterns"` vs `C:\path\with spaces --goal "goal"` vs `./local/path`. This is natural-language instruction to the LLM with no regex or validation. Edge cases: paths containing `--`, goals with quotes, URLs with query strings.
**Fix:** Add explicit parsing instructions in the skill: "The first argument (before --goal) is the repo. Everything after --goal is the goal text. If --goal is absent, use the default goal."

**No issues** in: DRY (appropriate duplication in dimension prompts for reviewability), naming, over/under-engineering.

### Eng Section 3: Test Review

```
CODE PATH COVERAGE
===========================
[+] Step 1: Repo Access
    ├── [★★  TESTED] URL clone — Phase 1 test (autoresearch)
    ├── [★★  TESTED] Local path — Phase 3 test (Claude Code src)
    ├── [GAP]         Path is file not dir — NO TEST
    ├── [GAP]         Path with spaces — NO TEST
    └── [GAP]         URL with query params — NO TEST

[+] Step 2: Triage
    ├── [★★  TESTED] Score >= 3 path — Phase 1+2 tests
    ├── [GAP]         Score < 3 (skip deep dive) — NO TEST
    └── [GAP]         Repo with no README — NO TEST

[+] Step 3: File Index
    ├── [★★  TESTED] Normal repo — Phase 1+2 tests
    └── [GAP]         Zero hot paths found — NO TEST

[+] Step 4: Dimension Agents
    ├── [★★  TESTED] All 4 succeed — Phase 2 test
    ├── [GAP]         1-3 agents fail — NO TEST
    ├── [GAP]         All 4 fail — NO TEST ← CRITICAL
    └── [GAP]         Dimension N/A skip — NO TEST

[+] Step 5: Synthesis
    ├── [★★  TESTED] Normal synthesis — Phase 2 test
    ├── [GAP]         >30% evidence unverified — NO TEST
    ├── [GAP]         Zero novel findings — NO TEST
    └── [GAP]         Contradictions between dims — NO TEST

[+] Steps 7-9: KB Integration + Learnings
    ├── [★★  TESTED] Normal flow — Phase 3 test
    ├── [GAP]         compile.js fails — NO TEST ← CRITICAL
    └── [GAP]         raw/ file already exists — NO TEST

─────────────────────────────────
COVERAGE: 6/18 paths tested (33%)
QUALITY:  ★★★: 0  ★★: 6  ★: 0
GAPS: 12 paths need tests (2 CRITICAL)
─────────────────────────────────
```

**Auto-decided (P1 — completeness):** Add these 12 test scenarios to the plan's testing section as manual invocation tests. The 2 CRITICAL gaps (all-agents-fail, compile.js-fails) must be handled in the SKILL.md error handling before testing.

### Eng Section 4: Performance Review (confidence 8/10)

**[P1] (confidence 8/10)** — **Cost model needs revision.** The plan's $8-15 estimate doesn't account for orchestrator context accumulation during synthesis. Realistic: $15-35/repo. At 50 repos: $750-$1,750 instead of $400-$750. Decision Audit Trail #5 already flagged this but the plan body (§6) still shows old numbers.
**Fix:** Update §6 cost table with revised estimates.

No issues in: token limits per dimension (200K is reasonable for Sonnet), wall time (30-75 min is dominated by LLM inference, no optimization possible).

### Eng Failure Modes Registry

| Codepath | Failure Mode | Rescued? | Test? | User Sees? | Logged? |
|----------|-------------|----------|-------|------------|---------|
| #$ARGUMENTS | Path with spaces | N | No | Wrong clone path | N |
| #$ARGUMENTS | Missing --goal | Y (default) | No | Default goal | N |
| Triage | No README | Y | No | "README: none" | N |
| File index | Zero hot paths | N ← **GAP** | No | Silent | N |
| Agent wave | 1-3 agents fail | N ← **GAP** | No | Missing dimensions | N |
| Agent wave | All 4 fail | N ← **CRITICAL** | No | Silent completion | N |
| Synthesis | Context limit hit | N ← **GAP** | No | Truncated report | N |
| compile.js | Node missing | N ← **CRITICAL** | No | Silent | N |

**2 CRITICAL GAPS, 3 regular gaps.**

### Eng Worktree Parallelization

Sequential implementation, no parallelization opportunity. The skill is a single SKILL.md file built in 3 phases. Each phase adds to the same file.

### Eng Completion Summary

```
  +====================================================================+
  |         ENG REVIEW — COMPLETION SUMMARY                            |
  +====================================================================+
  | Step 0: Scope        | Accepted as-is (2 files, under thresholds)   |
  | Section 1 (Arch)     | 3 issues (shell commands, CWD guard, tokens) |
  | Section 2 (Quality)  | 1 issue (argument parsing)                   |
  | Section 3 (Tests)    | 12 gaps, 2 CRITICAL                          |
  | Section 4 (Perf)     | 1 issue (cost model revision)                 |
  | NOT in scope         | Same as CEO (6 items)                         |
  | What already exists  | Same as CEO (5 reuse points)                  |
  | Failure modes        | 8 total, 2 CRITICAL, 3 gaps                   |
  | Dual voices          | Codex + Claude subagent, 4/6 confirmed        |
  | Parallelization      | Sequential (single file)                       |
  | TODOS.md updates     | 0 (eng issues folded into plan fixes)         |
  +====================================================================+
```

---

## DX Review Outputs (Phase 3.5)

### DX Scope Assessment

Product type: **Claude Code skill (CLI tool for developers)**
Primary persona: **The skill author/user** — a developer using Claude Code who wants to research agent repos
TTHW target: Invoke `/kb-code-research <url>` and get a useful triage scorecard in <5 minutes

### Developer Journey

| Stage | Current Plan | Score |
|-------|-------------|-------|
| 1. Discovery | User reads CLAUDE.md or remembers skill exists | 3/10 |
| 2. First invocation | `/kb-code-research <url> --goal "..."` | 7/10 |
| 3. Argument parsing | LLM parses #$ARGUMENTS | 5/10 |
| 4. Progress feedback | Agent spinners during waves, otherwise silent | 4/10 |
| 5. Output review | Read raw/ or outputs/ file manually | 5/10 |
| 6. KB integration | User runs /kb-compile separately | 6/10 |
| 7. Error recovery | Partial — some errors handled, some silent | 4/10 |
| 8. Iteration | Overwrite previous, check learnings | 5/10 |

**DX Overall: 5/10**

### Key DX Findings

1. **Progress feedback is the biggest DX gap.** A 30-75 minute skill with no structured progress updates is painful. The user should see: "Triage complete (score: 7/10)... Wave 1 dispatched... Wave 1 complete (Architecture: 12 findings, Memory: 8 findings)... Wave 2 dispatched..." This is cheap to add and high-impact.

2. **Error messages need problem + cause + fix.** Current plan: "Print error, suggest manual clone or local path. STOP." Better: "Clone failed for https://github.com/foo/bar: authentication required (HTTP 401). Fix: clone manually with `git clone <url> /path/to/local` and re-run with the local path."

3. **The --goal argument is invisible UX.** The skill description says `argument-hint: "<repo-url-or-local-path> --goal \"<research goal>\""`. Users will forget --goal on first invocation. The default goal ("Extract agent harness architecture patterns...") should be shown explicitly: "No --goal specified — using default: '...'. Override with --goal."

4. **Output location is implicit.** After a 45-minute run, the skill says "Research report saved to raw/code-research-{repo}.md. Run /kb-compile to integrate." The user has to navigate to the file manually. Better: print the first 10 lines of the executive summary inline, then the file path.

5. **No --help or usage guide.** If the user types `/kb-code-research` with no arguments, the plan says to ask the user. Better: show usage with examples from the requirements doc.

### DX Consensus Table

```
DX DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Review   CEO/Eng  Consensus
  ──────────────────────────────────── ──────── ──────── ─────────
  1. Getting started < 5 min?          Yes      Yes      CONFIRMED
  2. Argument naming guessable?        Mostly   N/A      —
  3. Error messages actionable?        No       No       CONFIRMED
  4. Progress feedback adequate?       No       N/A      —
  5. Output useful without manual work? No      No       CONFIRMED
  6. Upgrade/iteration path safe?      Yes      Yes      CONFIRMED
═══════════════════════════════════════════════════════════════
CONFIRMED: 4/6.
```

### DX Implementation Checklist

- [ ] Add structured progress messages between each major step
- [ ] Show default --goal when none specified
- [ ] Print executive summary inline after completion
- [ ] Add --help output with usage examples
- [ ] Format error messages as "Problem → Cause → Fix"
- [ ] Add CWD validation at startup

### DX Completion Summary

DX Overall: 5/10 → target 7/10 with checklist items above.
TTHW: ~2 minutes (fast — just invoke the skill) for triage; ~45 minutes for full pipeline (inherent, not improvable).

---

## Additional Eng Decision Audit Trail Entries

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|---------------|-----------|-----------|----------|
| 18 | ENG-1 | Use Glob/Grep instead of shell find/wc | Mechanical | P5 | Cross-platform, handles spaces, consistent with dimension agents | Keep shell commands |
| 19 | ENG-1 | Add CWD guard (verify scripts/compile.js exists) | Mechanical | P5 | Prevents silent miswrite of all output files | Skip |
| 20 | ENG-1 | Synthesis: DO re-read source files for contradiction resolution | Taste (USER OVERRIDE) | P1 | User chose accuracy over cost savings. Re-read cited files during synthesis. Adds 50-100K tokens per repo. | Trust reports only |
| 21 | ENG-2 | Add explicit argument parsing instructions | Mechanical | P5 | Reduce fragility of #$ARGUMENTS split | Skip |
| 22 | ENG-3 | Add 12 test scenarios to plan | Mechanical | P1 | 33% coverage → ~80% with manual test plan | Skip |
| 23 | ENG-4 | Update §6 cost table to $15-35/repo | Mechanical | P3 | Orchestrator context makes $8-15 unrealistic | Keep old estimate |
| 24 | DX | Add progress messages | Mechanical | P1 | 30-75 min with no feedback is unacceptable DX | Skip |
| 25 | DX | Show default --goal when none specified | Mechanical | P5 | Invisible default is confusing | Skip |
| 26 | DX | Print executive summary inline | Taste | P1 | Immediate value after long wait. But adds tokens to orchestrator context. | Only print file path |
| 27 | DX | Format errors as Problem → Cause → Fix | Mechanical | P5 | Standard DX best practice | Skip |

---

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 1 | CLEAR (via /autoplan) | 5 premises valid, 5 taste decisions, 2 critical gaps fixed |
| Eng Review | `/plan-eng-review` | Architecture & tests | 1 | ISSUES_OPEN (PLAN via /autoplan) | 7 issues, 2 critical gaps, 12 test gaps |
| DX Review | `/plan-devex-review` | Developer experience | 1 | ISSUES_OPEN (via /autoplan) | DX 5/10, 6 implementation items |
| CEO Voices | `/autoplan` | Cross-model consensus | 1 | 3/6 confirmed | Adoption forcing function, cost model, evidence validation |
| Eng Voices | `/autoplan` | Cross-model consensus | 1 | 4/6 confirmed | Shell commands, CWD guard, synthesis tokens, arg parsing |
| DX Voices | `/autoplan` | Cross-model consensus | 1 | 4/6 confirmed | Progress feedback, error formatting |

**VERDICT:** APPROVED with 1 user override (#6: re-read source files in synthesis). 27 mechanical decisions applied. 6 taste decisions accepted as recommended. Cost model revised to $18-40/repo. Plan ready for Phase 1 implementation.

### Plan Amendments Summary (changes from review)

These items must be incorporated when building the SKILL.md:

1. **Use Glob/Grep tools** instead of shell find/wc in triage and file indexing (cross-platform)
2. **Add CWD guard** at Step 1 (verify scripts/compile.js exists)
3. **Use mktemp -d** for clone directory (not /tmp/)
4. **Add --no-recurse-submodules** to git clone
5. **Add progress messages** between each major step
6. **Add 'Decisions to Adopt' section** in report template (adoption forcing function)
7. **Add Step 5d: cross-cutting integration pass** in synthesis
8. **Synthesis re-reads source files** for contradiction resolution (user override)
9. **Gate Dimension 4** on multi-agent grep count (<10 matches → skip)
10. **Add --skip-triage flag** (optional, triage remains default)
11. **Print executive summary inline** after completion
12. **Add explicit argument parsing instructions** ("first arg = repo, after --goal = goal text")
13. **Show default --goal** when none specified
14. **Format errors** as Problem → Cause → Fix
15. **Add error handling** for compile.js failures and total agent failure
16. **Sanitize repo name** for filenames (replace special chars with hyphens)
17. **Add cleanup step** for cloned repos at end of skill
18. **Add --help output** with usage examples
19. **Cost estimate:** $18-40/repo (revised from $8-15)
