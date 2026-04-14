---
name: kb-code-research
description: >
  Deep code research on agent/harness repos. Deploys parallel specialist agents
  to analyze architecture, memory, tools, and multi-agent patterns. Produces
  structured research reports integrated into the KB. Use when asked to
  "research repo", "analyze codebase", "kb code research", "deep dive on repo",
  or "extract patterns from".
argument-hint: "<repo-url-or-local-path> [--goal \"<research goal>\"] [--skip-triage]"
---

# /kb-code-research

<args> #$ARGUMENTS </args>

## Step 0: Parse Arguments and Validate Environment

### 0a. Help Check

If the arguments above are empty, or contain `--help` or `-h`, print this usage guide and STOP:

```
Usage: /kb-code-research <repo-url-or-local-path> [--goal "<research goal>"] [--skip-triage]

Examples:
  /kb-code-research https://github.com/karpathy/autoresearch --goal "extract autoresearch loop patterns"
  /kb-code-research https://github.com/openclaw/openclaw --goal "analyze multi-agent orchestration and skill system"
  /kb-code-research C:\Users\dunliu\Downloads\Claude\ Code\src --goal "reverse-engineer harness architecture"
  /kb-code-research https://github.com/letta-ai/letta --skip-triage

Options:
  --goal "<text>"    Research goal (default: "Extract agent harness architecture patterns, design decisions, and reusable components")
  --skip-triage      Skip the triage scorecard and go directly to deep dive
  --help             Show this help message

Cost: ~$18-40 per repo (Sonnet for dimension agents, Opus for synthesis).
Time: ~5 min triage only, ~30-75 min full deep dive.
```

### 0b. CWD Guard

**Note:** All bash commands in this skill use Unix shell syntax. Claude Code on Windows uses Git Bash, which supports `mktemp`, `test`, `rm -rf`, and other Unix commands. This is the correct behavior.

Verify we are in the harness engineering KB project root:

```bash
ls scripts/compile.js
```

If this file does NOT exist, print this error and STOP:

```
Problem: Not in the harness engineering KB project root.
Cause: /kb-code-research must be invoked from the project directory that contains scripts/compile.js.
Fix: cd to C:\Users\dunliu\projects\knowledge_base\agents\harness_engineering and re-invoke.
```

### 0c. Parse Arguments

Parse the arguments from the `<args>` block above. The parsing rules are:

1. **Repo (required):** The first argument before any `--` flags. This is either:
   - A URL containing `github.com` or starting with `https://`
   - A local file system path (absolute or relative)

2. **--goal (optional):** Everything after `--goal` up to the next `--` flag or end of arguments. If omitted, use this default and print it:
   `"Extract agent harness architecture patterns, design decisions, and reusable components"`

3. **--skip-triage (optional):** If present, skip the triage scorecard and go directly to the deep dive.

If no repo argument is found, print:
```
Problem: No repository specified.
Cause: The first argument must be a repo URL or local path.
Fix: /kb-code-research <repo-url-or-local-path> --goal "your research goal"
```
Then STOP.

### 0d. Determine Repo Name

Sanitize the repo name for use in filenames:
- For URLs: extract `{owner}-{repo}` from the URL (e.g., `karpathy-autoresearch`)
- For local paths: use the directory name (e.g., `claude-code-src`)
- Replace any spaces, special characters, or path separators with hyphens
- Convert to lowercase

Set these variables for the rest of the skill:
- **REPO_URL_OR_PATH**: the raw argument
- **REPO_NAME**: the sanitized name
- **RESEARCH_GOAL**: the parsed or default goal
- **SKIP_TRIAGE**: true or false

### 0e. Read Prior Learnings

If `outputs/research-learnings.md` exists, read it for context from prior research runs. Note any prompt improvement suggestions that apply to the current research.

Print: **"Starting research on {REPO_NAME}. Goal: {RESEARCH_GOAL}"**

---

## Step 1: Access Repository

### 1a. Clone or Validate

**If REPO_URL_OR_PATH is a URL** (contains `github.com` or starts with `https://`):

```bash
CLONE_DIR=$(mktemp -d)
echo "Cloning to $CLONE_DIR"
```

Then clone:
```bash
git clone --depth=1 --no-recurse-submodules {REPO_URL_OR_PATH} "$CLONE_DIR/repo"
```

If the clone fails, print this error and STOP:
```
Problem: Failed to clone {REPO_URL_OR_PATH}.
Cause: {the actual error from git — auth failure, network issue, repo not found, etc.}
Fix: Clone manually with `git clone {url} /path/to/local` and re-run with the local path.
```

Set **REPO_PATH** to `$CLONE_DIR/repo`.

**If REPO_URL_OR_PATH is a local path:**

Verify the path exists and is a directory:
```bash
test -d "{REPO_URL_OR_PATH}" && echo "OK" || echo "NOT_A_DIRECTORY"
```

If NOT_A_DIRECTORY, print:
```
Problem: Path "{REPO_URL_OR_PATH}" is not a valid directory.
Cause: The path doesn't exist or points to a file, not a directory.
Fix: Provide the path to the repo root directory.
```
Then STOP.

Set **REPO_PATH** to the absolute path. Set CLONE_DIR to empty (no cleanup needed).

### 1b. Check for Existing Research

Check if `raw/code-research-{REPO_NAME}.md` already exists:

```bash
ls raw/code-research-{REPO_NAME}.md 2>/dev/null
```

If it exists, print: **"Note: Previous research exists for {REPO_NAME}. Will overwrite."**

Print: **"Repository accessed. Path: {REPO_PATH}"**

---

## Step 2: Triage Scorecard

**If --skip-triage was specified**, print "Skipping triage (--skip-triage)." Set the multi-agent signal count to 999 (so Dimension 4 is always dispatched when triage is skipped, since we can't assess applicability). Jump to Step 3.

Otherwise, quickly assess the repo. Spend no more than 5 minutes on this step.

### 2a. Gather Repo Metrics

1. **README:** Use Glob to find README files at the repo root only, then Read it:
   ```
   Glob: {REPO_PATH}/README*
   ```
   If not found at root, try one level deep: `Glob: {REPO_PATH}/*/README*`

2. **Source files:** Use Glob to count source files:
   ```
   Glob: {REPO_PATH}/**/*.py
   Glob: {REPO_PATH}/**/*.ts
   Glob: {REPO_PATH}/**/*.js
   Glob: {REPO_PATH}/**/*.rs
   Glob: {REPO_PATH}/**/*.go
   ```
   Count total files by type. The type with the most files is the primary language.

3. **Repo size:** Estimate lines of code by reading a sample of 10-20 representative source files and extrapolating. Classify:
   - tiny: <1K LOC
   - small: 1-10K LOC
   - medium: 10-50K LOC
   - large: 50K+ LOC

4. **Git activity:**
   ```bash
   git -C "{REPO_PATH}" log --oneline -20 2>/dev/null
   ```
   Assess: active (commits in last month), maintained (last 3 months), stale (last year), dead (>1 year).

5. **Agent/harness signals:** Use Grep across the repo for key patterns:
   ```
   Grep: "agent|harness|orchestrat|loop|tool_use|function_call" in {REPO_PATH}
   Grep: "memory|state|context|session|persist" in {REPO_PATH}
   Grep: "spawn|worker|swarm|subprocess|delegate" in {REPO_PATH}
   ```
   Count matches to identify which dimensions are applicable.

### 2b. Produce Scorecard

Produce a triage scorecard table:

| Dimension | Value |
|-----------|-------|
| Repo name | {REPO_NAME} |
| Primary language | {detected language} |
| Size classification | {tiny/small/medium/large} |
| File count | {N} source files |
| Last commit | {date from git log} |
| Commit frequency | {active/maintained/stale/dead} |
| README quality | {none/stub/adequate/detailed} |
| Relevance to goal | {0-10 score with 1-sentence justification} |
| Agent/harness signals | {list of signals found} |
| Multi-agent signal count | {count of spawn/worker/swarm/subprocess matches} |
| Recommended dimensions | {which of the 4 dimensions are applicable} |

### 2c. Triage Gate

**If relevance score < 3:**

Print: **"Triage score {N}/10 — below threshold. Skipping deep dive."**

Save the scorecard to `outputs/triage-{REPO_NAME}.md` with frontmatter:
```yaml
---
title: "Triage: {REPO_NAME}"
type: triage
date: {YYYY-MM-DD}
relevance_score: {N}
research_goal: "{GOAL}"
---
```

Append to log.md:
```
- **[{timestamp}]** — /kb-code-research triage on {REPO_NAME}: relevance {N}/10. Below threshold, skipped deep dive.
```

If CLONE_DIR is set, clean up: `rm -rf "$CLONE_DIR"`

**STOP HERE.**

**If relevance score >= 3:** Print: **"Triage complete. Score: {N}/10. Proceeding to file indexing."**

---

## Step 3: Build File Index

Identify the most important files for focused agent work. This prevents dimension agents from wasting tokens on irrelevant code.

### 3a. Find Hot Paths

Use Glob to find files matching these categories. For each category, collect matching paths:

**Entry points:**
```
Glob: {REPO_PATH}/**/main.*
Glob: {REPO_PATH}/**/index.*
Glob: {REPO_PATH}/**/app.*
Glob: {REPO_PATH}/**/cli.*
Glob: {REPO_PATH}/**/__main__.py
```

**Agent/loop files:**
```
Glob: {REPO_PATH}/**/*agent*
Glob: {REPO_PATH}/**/*loop*
Glob: {REPO_PATH}/**/*run*
Glob: {REPO_PATH}/**/*harness*
Glob: {REPO_PATH}/**/*orchestrat*
Glob: {REPO_PATH}/**/*train*
Glob: {REPO_PATH}/**/*eval*
Glob: {REPO_PATH}/**/*experiment*
Glob: {REPO_PATH}/**/*research*
```

**Tool files:**
```
Glob: {REPO_PATH}/**/*tool*
Glob: {REPO_PATH}/**/*action*
Glob: {REPO_PATH}/**/*command*
Glob: {REPO_PATH}/**/*skill*
Glob: {REPO_PATH}/**/*plugin*
```

**Memory/state files:**
```
Glob: {REPO_PATH}/**/*memory*
Glob: {REPO_PATH}/**/*state*
Glob: {REPO_PATH}/**/*context*
Glob: {REPO_PATH}/**/*session*
Glob: {REPO_PATH}/**/*persist*
Glob: {REPO_PATH}/**/*dream*
Glob: {REPO_PATH}/**/*consolidat*
Glob: {REPO_PATH}/**/*compact*
```

**Config files:**
```
Glob: {REPO_PATH}/**/CLAUDE.md
Glob: {REPO_PATH}/**/AGENTS.md
Glob: {REPO_PATH}/**/SOUL.md
Glob: {REPO_PATH}/**/*config*
Glob: {REPO_PATH}/**/*setting*
Glob: {REPO_PATH}/**/*prompt*
```

**Multi-agent files:**
```
Glob: {REPO_PATH}/**/*worker*
Glob: {REPO_PATH}/**/*swarm*
Glob: {REPO_PATH}/**/*delegate*
Glob: {REPO_PATH}/**/*dispatch*
Glob: {REPO_PATH}/**/*coordinator*
```

### 3b. Deduplicate and Rank

1. Combine all matches, remove duplicates
2. Exclude files in `node_modules/`, `.git/`, `vendor/`, `__pycache__/`, `dist/`, `build/`
3. Prioritize: entry points > agent/loop > tool > memory > config > multi-agent
4. Cap at **40 files maximum**
5. Include the top-level directory listing for orientation

**Tiny repo optimization:** If the repo has fewer than 10 source files total (across all types), skip the hot path builder entirely and pass ALL source files to every dimension agent.

If zero hot paths are found, print:
**"Warning: No hot paths identified via standard patterns. Dimension agents will explore from the repo root using Grep."**

### 3c. Build Index Summary

Produce a HOT_PATHS summary listing each file with its category:
```
HOT_PATHS ({N} files):
  [entry]    src/main.py
  [agent]    src/agent/loop.py
  [tool]     src/tools/bash.py
  [memory]   src/memory/store.py
  [config]   CLAUDE.md
  ...
```

Print: **"File index built. {N} hot paths identified across {M} categories. Ready for deep dive."**

---

## Step 4: Dispatch Dimension Agents

Spawn specialist agents using the Agent tool. Run in 2 waves of 2 agents each.

Build each agent's prompt by combining the **Shared Context Block** (from the Dimension Agent Prompts section at the end of this file) with the dimension-specific checklist. Replace all `{PLACEHOLDER}` values with actual data from Steps 0-3.

For the `## Priority files to start with` section in each prompt, filter HOT_PATHS to the categories relevant to that dimension:
- Dimension 1 (Architecture): entry points + agent/loop files
- Dimension 2 (Memory): memory/state files + config files
- Dimension 3 (Tools): tool files + config files
- Dimension 4 (Multi-Agent): multi-agent files + agent/loop files

If HOT_PATHS is sparse (<5 files for a dimension), add this note to the prompt:
"Hot paths are sparse for this repo. Start with a broad Grep sweep using the search patterns below, then Read the most promising files."

### 4a. Wave 1 (parallel)

Print: **"Wave 1 dispatched: Architecture & Loop Design, Memory & State Management..."**

Dispatch these two agents **in a single message** (parallel) using the Agent tool:

**Agent 1: Architecture & Loop Design**
- description: "Dimension 1: Architecture & Loop Design"
- model: "sonnet"
- prompt: Shared Context Block (with DIMENSION_NAME = "Architecture & Loop Design") + Dimension 1 checklist + search patterns

**Agent 2: Memory & State Management**
- description: "Dimension 2: Memory & State Management"
- model: "sonnet"
- prompt: Shared Context Block (with DIMENSION_NAME = "Memory & State Management") + Dimension 2 checklist + search patterns

Wait for both Wave 1 agents to complete. Save their results as DIMENSION_1_REPORT and DIMENSION_2_REPORT.

If an agent fails entirely (error, not just empty results):
1. Print: "Dimension {N} agent failed: {error}. Continuing with remaining dimensions."
2. Set the failed dimension's report to a placeholder:
   "# {DIMENSION_NAME} — {REPO_NAME}\n\n## Summary\nDimension agent failed. Error: {error}\n\n## Findings\nNo findings (agent failure).\n\n## Key Patterns Discovered\nNone (agent failure).\n\n## Novelty Assessment\nN/A (agent failure)."

Print: **"Wave 1 complete. Architecture: {summary}. Memory: {summary}."**

### 4b. Wave 2 (parallel)

Print: **"Wave 2 dispatched: Tool & Action Space Design{, Multi-Agent Coordination if applicable}..."**

**Agent 3: Tool & Action Space Design** — always dispatched.
- description: "Dimension 3: Tool & Action Space Design"
- model: "sonnet"
- prompt: Shared Context Block (with DIMENSION_NAME = "Tool & Action Space Design") + Dimension 3 checklist + search patterns

**Agent 4: Multi-Agent Coordination** — dispatch ONLY if multi-agent signal count from triage >= 10.
- description: "Dimension 4: Multi-Agent Coordination"
- model: "sonnet"
- prompt: Shared Context Block (with DIMENSION_NAME = "Multi-Agent Coordination") + Dimension 4 checklist + search patterns

If multi-agent signal count < 10, set DIMENSION_4_REPORT to:
"# Multi-Agent Coordination — {REPO_NAME}\n\n## Summary\nDimension 4: N/A — single-agent system ({N} multi-agent signals found in triage). This repo does not implement multi-agent coordination patterns."

If dispatching both agents, send them **in a single message** (parallel). If only Agent 3, send it alone.

Wait for Wave 2 to complete. Save results as DIMENSION_3_REPORT and DIMENSION_4_REPORT.

### 4c. Validate Results

Count how many dimension reports contain actual findings (not just "N/A" or failures).

If ALL dimension agents failed or returned empty:
```
Problem: All dimension agents failed to produce findings.
Cause: {list the errors or empty results from each agent}
Fix: Check that the repo contains readable source code and retry. For very large repos, try passing a specific subdirectory.
```
Then STOP.

Print: **"All dimension reports received. {N}/4 dimensions produced findings. Starting synthesis."**

---

## Step 5: Synthesize Results

After all dimension agents complete, the orchestrator synthesizes directly. Do NOT spawn a sub-agent for synthesis.

### 5a. Evidence Validation

Extract every file path cited in any dimension report (look for patterns like `file_path:line_start-line_end` in **Evidence:** fields).

For each cited path, check if the file exists in the repo:
```bash
test -f "{REPO_PATH}/{cited_path}" && echo "VERIFIED" || echo "UNVERIFIED"
```

Build an evidence index:
```
EVIDENCE INDEX:
  Verified: {count} paths
  Unverified: {count} paths ({percentage}%)

  Verified:
    ✓ path/to/file.py:42-67 — description from dim report
    ✓ path/to/other.py:10-25 — description

  Unverified:
    ✗ path/to/missing.py:15 — [UNVERIFIED] claimed by Dimension {N}
```

If >30% of cited paths are unverified, print:
**"Warning: {X}% of evidence is unverified. Dimension agent findings may contain hallucinated file paths. Treat [UNVERIFIED] citations with skepticism."**

### 5b. Cross-Dimension Contradiction Resolution

Read all 4 dimension reports. Look for contradictions such as:
- Dimension 1 says "model controls loop" but Dimension 4 says "orchestrator dispatches tasks"
- Dimension 2 says "no persistence" but Dimension 3 says "checkpoint tool exists"
- Two dimensions cite the same file with conflicting interpretations

For each potential contradiction:
1. Read the actual source file(s) at the cited paths using the Read tool
2. Determine which dimension's interpretation is correct based on the code
3. Record the resolution:
   ```
   CONTRADICTION: Dim {A} says X, Dim {B} says Y
   SOURCE CHECK: Read {file}:{lines} — the code shows {what it actually does}
   RESOLUTION: Dim {A/B} is correct because {reason}
   ```

If no contradictions are found, note: "No cross-dimension contradictions detected."

### 5c. Novelty Assessment

Read `wiki/_index.md` to get the list of existing wiki articles.
Read 3-5 wiki articles most relevant to the findings (based on tag overlap).

For each key finding (the "Key Patterns Discovered" items from each dimension report), assess:
- **KNOWN**: This pattern is already well-documented in the KB with similar evidence
- **VARIANT**: This is a new variation of a known pattern (different implementation, same concept)
- **NOVEL**: This is genuinely new — not covered in any existing wiki article

Build a novelty table:
```
| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| {pattern} | Dim {N} | NOVEL | Not in any wiki article |
| {pattern} | Dim {N} | VARIANT | Similar to wiki/article.md but different approach |
| {pattern} | Dim {N} | KNOWN | Already in wiki/article.md |
```

If zero NOVEL or VARIANT findings, note: "No novel findings beyond existing KB coverage. This repo confirms known patterns but doesn't add new ones."

### 5d. Cross-Cutting Integration Pass

Review all dimension reports together. Identify 2-3 flows that span multiple dimensions. These cross-cutting patterns are often the most valuable findings because individual dimension agents can't see them.

For each cross-cutting flow:
```
CROSS-CUTTING FLOW: {name}
  Dim 1 (Architecture): {how it appears in the loop}
  Dim 2 (Memory): {how state is involved}
  Dim 3 (Tools): {which tools participate}
  Dim 4 (Multi-Agent): {coordination aspect, if any}
  INTEGRATED VIEW: {the full picture that no single dimension captured}
  SIGNIFICANCE: {why this matters for harness design}
```

Print: **"Synthesis complete. {N} evidence paths verified, {M} contradictions resolved, {K} novel findings, {J} cross-cutting flows identified."**

---

## Step 6: Write Research Report

Write the research report to BOTH locations using the Write tool:
1. `raw/code-research-{REPO_NAME}.md` (for KB integration)
2. `outputs/code-research-{REPO_NAME}.md` (archival copy)

Use this template, filling in all `{PLACEHOLDER}` values from the data gathered in Steps 0-5:

```markdown
---
title: "Code Research: {REPO_NAME}"
source: {REPO_URL_OR_PATH}
author: "kb-code-research skill"
date: {TODAY YYYY-MM-DD}
fetched: {TODAY YYYY-MM-DD}
type: code-research
status: raw
tags: [code-research, {2-3 primary pattern tags from findings}]
relevance_score: {score from triage, or 7 if triage was skipped}
research_goal: "{RESEARCH_GOAL}"
dimensions_analyzed: [{list of dimensions that produced findings}]
---

# Code Research: {REPO_NAME}

## Executive Summary

{3-5 sentences: what this repo is, what harness patterns it demonstrates,
what's novel vs. existing KB, and what's most relevant to our research goal.
This section is printed inline at the end, so make it count.}

## Triage Scorecard

{Paste the scorecard table from Step 2, or "Triage skipped (--skip-triage)" if applicable}

## Dimension 1: Architecture & Loop Design

### Summary
{3-5 sentence summary from DIMENSION_1_REPORT}

### Key Findings
{Merged findings from the dimension agent. Each finding:}
- **{Finding title}:** {description}
  - Evidence: `{file}:{lines}` — {what the code shows}
  - Significance: {why it matters}

### Patterns
{Bulleted list of patterns from the "Key Patterns Discovered" section}

{Include architecture diagram from the agent if one was produced}

## Dimension 2: Memory & State Management

{Same structure as Dimension 1, from DIMENSION_2_REPORT}

## Dimension 3: Tool & Action Space Design

{Same structure as Dimension 1, from DIMENSION_3_REPORT}

## Dimension 4: Multi-Agent Coordination

{Same structure as Dimension 1 from DIMENSION_4_REPORT, OR:}
{If N/A: "N/A — single-agent system. {N} multi-agent signals found in triage, below the threshold of 10."}

## Cross-Cutting Analysis

### Contradiction Resolutions
{From Step 5b. If none: "No cross-dimension contradictions detected."}

### Cross-Cutting Flows
{From Step 5d. The 2-3 flows that span multiple dimensions.}

### Novelty Assessment
{The novelty table from Step 5c}

## Decisions to Adopt

Concrete, actionable items extracted from the research. Each should be specific enough
that someone could start implementing without re-reading the full report.

1. **Adopt: {specific pattern/technique}** from {file(s)}
   - What: {1-2 sentence description}
   - Why: {concrete benefit for our harness design}
   - Effort: {S/M/L}
   - Target: {which part of our system this applies to}

2. **Adopt: {pattern}** ...

3. **Adopt: {pattern}** ...

{If no actionable patterns found: "No directly adoptable patterns identified.
This repo confirms existing knowledge but does not introduce new techniques
worth extracting."}

## Evidence Index

{From Step 5a — the full verified/unverified path listing}

## Sources

- [{REPO_NAME}]({REPO_URL_OR_PATH}) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
```

Write the report to both file paths. They should be identical.

Print: **"Research report written to raw/code-research-{REPO_NAME}.md and outputs/code-research-{REPO_NAME}.md"**

---

## Step 7: KB Integration

### 7a. Verify Raw File

```bash
ls raw/code-research-{REPO_NAME}.md
```

If the file doesn't exist, something went wrong in Step 6. Print error and STOP.

### 7b. Run Compile Delta

Check if Node.js is available:
```bash
which node 2>/dev/null && echo "NODE_OK" || echo "NODE_MISSING"
```

If NODE_MISSING:
```
Problem: Node.js is not installed.
Cause: KB integration requires node to run scripts/compile.js.
Fix: Install Node.js and re-run, or manually run /kb-compile later.
```
Skip the compile delta check but continue to Step 8.

If NODE_OK, run the delta check:
```bash
node scripts/compile.js delta
```

If the command fails (non-zero exit), print:
```
Problem: compile.js delta failed.
Cause: {the error output}
Fix: Check scripts/compile.js for issues. The research report is still saved in raw/.
```
Continue to Step 8 regardless.

### 7c. Print Integration Message

Print: **"Research report saved to raw/code-research-{REPO_NAME}.md. Run /kb-compile to integrate findings into wiki articles."**

Do NOT auto-compile into wiki. Wiki compilation involves cross-pollination across 10-15 articles and should be a deliberate step.

---

## Step 8: Accumulate Learnings

Read `outputs/research-learnings.md`. If the file does not exist, create it first with the header:
```
# Research Learnings

Accumulated observations from /kb-code-research runs.

---
```

Then append a new entry using the Edit tool.

Append this structured block:

```markdown
### {REPO_NAME} — {TODAY YYYY-MM-DD}

**Research goal:** {RESEARCH_GOAL}
**Relevance score:** {score}/10
**Dimensions analyzed:** {list of dimensions that produced findings}
**Evidence validation:** {X}% verified, {Y}% unverified

**What worked well:**
- {Reflect on what the dimension agents did well — e.g., "Dimension 1 correctly identified the main loop pattern in train.py"}
- {e.g., "Hot path builder found the key config file (program.md)"}

**What didn't work:**
- {Reflect on failures — e.g., "Dimension 4 was N/A — correctly gated by multi-agent signal count"}
- {e.g., "Hot paths were sparse for this tiny repo — agents relied on Grep instead"}

**Missed findings (discovered during synthesis, not by any dimension agent):**
- {Any cross-cutting patterns from Step 5d that no individual agent found}
- {Any findings you noticed during synthesis that the agents missed}

**Prompt improvement suggestions:**
- {Specific suggestions — e.g., "Dimension 1 should search for 'experiment' and 'trial' patterns, not just 'loop'"}
- {e.g., "Hot path builder should also glob for *train*, *eval*, *test* files"}

---
```

Print: **"Learnings appended to outputs/research-learnings.md"**

---

## Step 9: Finalize

### 9a. Log Entry

Read `log.md`. If it does not exist, create it with a `# Log` header first.

Append to `log.md` using the Edit tool:

```
- **[{YYYY-MM-DD HH:MM}]** — /kb-code-research on {REPO_NAME}: relevance {score}/10, {N} novel findings, {M} variant findings, {K} decisions to adopt. Report: raw/code-research-{REPO_NAME}.md
```

### 9b. Print Executive Summary

Print the report's Executive Summary section inline so the user gets immediate value:

```
--- EXECUTIVE SUMMARY ---
{The 3-5 sentence executive summary from the report}
--- END SUMMARY ---
```

### 9c. Cleanup

If CLONE_DIR is set (repo was cloned from a URL, not a local path):
```bash
rm -rf "$CLONE_DIR"
```

### 9d. Final Message

Print:
```
Research complete on {REPO_NAME}.
  Relevance: {score}/10
  Novel findings: {N}
  Decisions to adopt: {K}
  Report: raw/code-research-{REPO_NAME}.md
  Next: Run /kb-compile to integrate findings into wiki articles.
```

---

## Dimension Agent Prompts (Phase 2 Reference)

These prompts are used by Step 4 when dispatching dimension agents. Each agent receives the shared context block below plus its dimension-specific checklist.

### Shared Context Block (injected into each agent prompt)

```
You are a specialist code researcher analyzing {REPO_NAME} for {DIMENSION_NAME}.

## Context
- Repo path: {REPO_PATH}
- Research goal: {RESEARCH_GOAL}
- Repo size: {SIZE_CLASSIFICATION}
- Primary language: {PRIMARY_LANGUAGE}
- Triage summary: {TRIAGE_SCORECARD}

## Priority files to start with
{HOT_PATHS — filtered to this dimension's categories}

## Instructions
1. Start with the hot path files. Use Read to examine them.
2. Use Grep to search for patterns across the codebase.
3. Use Glob to find additional relevant files beyond the hot paths.
4. For each checklist question, find SPECIFIC code evidence (file path + line range).
5. If a question is not applicable, explicitly state "N/A: {reason}".
6. Do NOT speculate. If you can't find evidence, say "Not found in codebase."
7. Spend your token budget on thorough investigation, not lengthy prose.
8. Use REPO-RELATIVE paths in all Evidence fields (e.g., `src/agents/loop.ts:42-67`), never full absolute paths.
9. For large repos (50K+ LOC): prioritize breadth of checklist coverage over exhaustive reading. Do not read test files unless the source file is ambiguous.
10. Cross-cutting question: What patterns in your dimension also appear in other dimensions?

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
(mermaid diagram)

## Surprising Observations
{What in this codebase is unexpected or unusual? What deviates from the standard approach
you'd expect given the repo's stated purpose? Stick to what the CODE shows, not what
you think is "novel" compared to other projects.}
```

### Dimension 1: Architecture & Loop Design

Checklist:
1. What is the core agent loop structure? (while-tool-call, ReAct, plan-execute, custom?)
2. Who controls the loop — the model or the code?
3. How does the loop terminate? (token limit, task complete, error threshold, convergence?)
4. What is the orchestration topology? (single agent, sequential, parallel, hierarchical?)
5. How are context windows managed? (compaction, resets, handoff artifacts?)
6. What is the system prompt structure? (static, conditional, layered?)
7. How does progressive disclosure work? (all upfront, lazy loading, on-demand?)

Search patterns: "while", "loop", "iterate", "step", "turn", "cycle", "break", "max_iter", "converge"

### Dimension 2: Memory & State Management

Checklist:
1. What memory systems exist? (in-context, file-based, database, vector store, git-backed?)
2. Is there short-term vs. long-term memory separation?
3. How does state persist across sessions? (files, checkpoints, git commits, databases?)
4. What survives context window compaction?
5. Is there a memory hierarchy? (core/working/archival like MemGPT?)
6. Can agents modify their own instructions or memory?
7. How is conversation history managed for long-running tasks?

Search patterns: "memory", "state", "persist", "checkpoint", "save", "load", "history", "session"

### Dimension 3: Tool & Action Space Design

Checklist:
1. How many tools are defined? What categories?
2. Are tools primitives or high-level integrations?
3. How are tool schemas defined? (JSON schema, function signatures, natural language?)
4. Is there lazy/dynamic tool loading?
5. How are tool failures handled? (retry, fallback, error-as-context?)
6. Are there MCP integrations?
7. What is the tool selection mechanism? (LLM choice, routing logic, hybrid?)

Search patterns: "tool", "function", "schema", "action", "execute", "invoke", "MCP", "retry", "skill", "plugin", "extension", "normalize"

### Dimension 4: Multi-Agent Coordination

**Gate:** Only dispatch this agent if multi-agent signal count >= 10 in triage. Otherwise, note "Dimension 4: N/A — single-agent system" in the report.

Checklist:
1. Is this a single-agent or multi-agent system?
2. What is the coordination topology? (orchestrator-worker, peer, hierarchy, swarm?)
3. How are tasks delegated? (explicit decomposition, dynamic spawning?)
4. How do agents communicate? (shared files, message passing, shared context, tool calls?)
5. Is there a "game of telephone" problem? How is it mitigated?
6. How is work deduplicated across agents?
7. Is execution synchronous or asynchronous?

Search patterns: "spawn", "worker", "swarm", "delegate", "dispatch", "coordinate", "agent", "subprocess"
