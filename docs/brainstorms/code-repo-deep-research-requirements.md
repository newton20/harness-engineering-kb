# Code Repo Deep Research Skill — Requirements

**Date:** 2026-04-14
**Status:** Draft
**Scope:** Deep — cross-cutting skill design + research methodology + dual-KB integration
**Approach:** Layered Research Protocol (Approach C) — two-pass: quick triage scorecard for all repos, then deep multi-phase analysis for high-scoring ones. Use a low triage threshold to avoid filtering out potentially important findings.

---

## 1. Problem Statement

We are building an auto-agentic Polymarket trading system that requires a world-class agent harness supporting long-running agents, massive sub-agent swarms, persistent memory, and autonomous self-improvement. To design this harness, we need to systematically extract architectural patterns, reusable components, and design decisions from the world's best open-source agent projects.

Current state: manual, ad-hoc repo browsing produces shallow observations. No structured process exists for deep code research that produces durable, cross-referenced knowledge artifacts.

Desired state: a reusable Claude Code skill (`/kb-code-research`) that deploys a fleet of parallel specialist agents to deeply analyze any code repo against targeted research goals, producing structured artifacts that flow into both the harness engineering KB and the trading KB.

## 2. Success Criteria

1. The skill produces a comprehensive research report for any given repo in a single invocation
2. Research findings are automatically ingested into the harness engineering KB (raw/ + wiki/)
3. Cross-references to the trading KB are identified and flagged for manual integration
4. The skill is reusable across agent/harness-related repositories and accepts customizable research goals within the agent engineering domain
5. Research depth is sufficient to identify: architecture patterns, reusable components, memory systems, tool interfaces, multi-agent coordination, safety mechanisms, and self-improvement loops
6. Convergence is measurable — the skill knows when it has "enough" and stops
7. Total execution time per repo: 15-45 minutes depending on repo size
8. The skill improves over time — learnings from each research session feed back into the skill's prompts

## 3. Skill Design

### 3.1 Invocation

```
/kb-code-research <repo-url-or-local-path> --goal "<research goal>"
```

Examples:
```
/kb-code-research https://github.com/karpathy/autoresearch --goal "extract autoresearch loop patterns for trading strategy self-improvement"
/kb-code-research https://github.com/openclaw/openclaw --goal "analyze multi-agent orchestration, memory persistence, and skill system for long-running harness design"
/kb-code-research C:\Users\dunliu\Downloads\Claude\ Code\src --goal "reverse-engineer Claude Code harness architecture for pattern extraction"
```

### 3.2 Architecture: Per-Dimension Parallel Agents

The skill uses a **two-pass Layered Research Protocol**:

**Pass 1 — Triage (~5 min per repo):** Single agent quickly evaluates repo maturity, relevance, code volume, and maintenance activity. Produces a triage scorecard with a relevance estimate. Uses a **low threshold** to avoid filtering out potentially important findings — only obviously irrelevant or dead repos are skipped.

**Pass 2 — Deep Dive (~30-75 min per repo):** Spawns 3-4 parallel specialist agents (v1 core dimensions), each attacking a different research dimension. Results merge into a unified report. Uses **2-wave parallelism** (2 agents per wave) to stay within Claude Code concurrency limits.

```
/kb-code-research invoked
        |
        v
  [Orchestrator Agent]
        |
        ├── clone/access repo (shallow clone)
        ├── PASS 1: Triage scorecard
        │   ├── repo size classification (tiny/small/medium/large)
        │   ├── relevance estimate (0-10) to research goal
        │   ├── maintenance activity, language detection
        │   └── if score < 3: return scorecard only, skip deep dive
        |
        ├── determine applicable dimensions (skip N/A dims)
        ├── build file index + hot paths for focused agent work
        |
        ├── PASS 2: Deep Dive (2-wave parallel)
        │   ├── Wave 1:
        │   │   ├──► [Agent 1: Architecture & Loop Design]
        │   │   └──► [Agent 2: Memory & State Management]
        │   ├── Wave 2:
        │   │   ├──► [Agent 3: Tool & Action Space Design]
        │   │   └──► [Agent 4: Multi-Agent Coordination]
        │   └── (v2: additional dimensions as optional wave 3)
        |
        ├── Synthesis (orchestrator performs directly, no nested agent)
        │   ├── merge dimension reports
        │   ├── resolve contradictions
        │   ├── validate evidence (file-existence checks)
        │   ├── produce unified research report
        │   └── assess novelty vs. existing KB
        |
        ├── KB Integration (orchestrator performs directly)
        │   ├── write raw/code-research-{repo-name}.md with standard frontmatter
        │   ├── run: node scripts/compile.js delta (to verify)
        │   ├── flag trading KB cross-references in report appendix
        │   └── append to research log
        v
      DONE + learnings appended to outputs/research-learnings.md
```

**v1 Core Dimensions (4):** Architecture & Loop Design, Memory & State Management, Tool & Action Space Design, Multi-Agent Coordination.

**v2 Additional Dimensions:** Safety & Error Recovery, Self-Improvement & Eval Loops, Reusable Components Extraction, Domain Relevance Mapping (pluggable via `--domain` flag).

### 3.3 Research Dimensions (Parallel Agent Specs)

Each dimension agent receives: the repo path, the research goal, and a dimension-specific prompt with a checklist of questions to answer.

#### Dimension 1: Architecture & Loop Design
- What is the core agent loop? (while-tool-call, ReAct, plan-execute, custom?)
- Is it "model controls the loop" or "code controls the model"?
- How is the loop terminated? (token limit, task complete, error threshold, convergence?)
- What is the orchestration topology? (single agent, sequential, parallel, hierarchical?)
- How are context windows managed? (compaction, context resets, handoff artifacts?)
- What is the system prompt structure? (static, conditional, layered?)
- How does progressive disclosure work? (all upfront, lazy loading, on-demand?)
- **Produce:** Architecture diagram (mermaid), loop flow description, key design decisions list

#### Dimension 2: Memory & State Management
- What memory systems exist? (in-context, file-based, database, vector store, git-backed?)
- Is there short-term vs long-term memory separation?
- How does state persist across sessions? (progress files, checkpoints, git commits?)
- What survives context compaction and what is lost?
- Is there a memory hierarchy? (core/working/archival like MemGPT?)
- Can agents modify their own instructions or memory?
- How is conversation history managed for long-running tasks?
- **Produce:** Memory architecture diagram, persistence mechanisms list, key code paths

#### Dimension 3: Tool & Action Space Design
- How many tools are defined? What categories?
- Are tools primitives or high-level integrations?
- How are tool schemas defined? (JSON schema, function signatures, natural language?)
- Is there lazy/dynamic tool loading?
- How are tool failures handled? (retry, fallback, error-as-context?)
- Are there MCP integrations?
- How are tool descriptions maintained? (static, auto-generated, agent-improved?)
- What is the tool selection mechanism? (LLM choice, routing logic, hybrid?)
- **Produce:** Tool catalog with categories, schema examples, failure handling patterns

#### Dimension 4: Multi-Agent Coordination
- Is it single-agent or multi-agent?
- What is the coordination topology? (orchestrator-worker, peer, hierarchy, swarm?)
- How are tasks delegated? (explicit decomposition, dynamic spawning?)
- How do agents communicate? (shared files, message passing, shared context, tool calls?)
- Is there a "game of telephone" problem? How is it mitigated?
- How is work deduplicated across agents?
- Is execution synchronous or asynchronous?
- **Produce:** Coordination diagram, delegation patterns, communication protocols

#### Dimension 5: Safety & Error Recovery
- What safety mechanisms exist? (permission systems, sandboxing, classifiers?)
- How are destructive actions prevented?
- How does the system recover from errors? (retry, checkpoint resume, human escalation?)
- Is there prompt injection defense?
- Are there budget/cost controls?
- How are agent actions audited?
- Is there a "deny and continue" pattern?
- **Produce:** Safety mechanism inventory, error recovery flow, risk assessment

#### Dimension 6: Self-Improvement & Eval Loops
- Does the system improve itself? (prompt evolution, strategy optimization, skill learning?)
- What is the eval/feedback loop? (autoresearch, reflection, human review?)
- How are changes validated? (A/B testing, replay, shadow execution?)
- Is there a champion/challenger pattern?
- How is overfitting/gaming prevented?
- What metrics drive improvement?
- Is there a "garbage collection" process for degraded components?
- **Produce:** Self-improvement architecture, eval criteria, promotion/revert logic

#### Dimension 7: Reusable Components Extraction
- Which modules are cleanly separated and could be extracted?
- Are there utility libraries worth forking? (file handling, git operations, API clients?)
- What abstractions would transfer to a trading context?
- Are there configuration patterns worth adopting?
- What is the test infrastructure like?
- License compatibility for reuse?
- **Produce:** Extractable components catalog with effort estimates, code snippet collection

#### Dimension 8: Trading System Relevance Mapping
- Map each finding to the trading system's 8 subsystems: data ingestion, strategy engine, execution engine, evaluation engine, autoresearch loop, monitoring, portfolio management, risk gateway
- Which patterns directly apply vs. need adaptation?
- What gaps does this repo NOT address that the trading system needs?
- What novel ideas does this repo introduce that the trading system design hadn't considered?
- Priority ranking: which findings should be adopted first?
- **Produce:** Relevance matrix, adoption priority list, gap analysis

### 3.4 Convergence Criteria

Research on a dimension is "sufficient" when:

1. **Coverage checklist**: All questions in the dimension's checklist have been answered or explicitly marked as "not applicable to this repo"
2. **Evidence threshold**: Each answer is supported by at least one specific file path or code reference (validated via file-existence check)

Research on a repo is "complete" when:
1. All applicable dimensions have converged
2. Cross-dimension contradiction resolution has completed (synthesis agent responsibility)
3. The synthesis agent has produced a unified report with at least one novel finding vs. existing KB
4. KB integration has completed (raw/ ingested, wiki updated)
5. A relevance score (0-10) has been assigned

**Budget cutoffs** (hard limits regardless of convergence):
- Pass 1 triage: max 5 minutes, ~10K tokens
- Pass 2 deep dive: max 75 minutes wall time per repo
- Max 200K tokens per dimension agent (Sonnet-class model)
- Max 4 parallel dimension agents per repo (v1), 2 per wave
- Synthesis + KB integration: max 100K tokens (Opus-class model)
- Total token budget per repo: ~900K-1M (4 dimensions × 200K + synthesis overhead)
- Estimated cost per repo: $8-20 (Sonnet for dimensions, Opus for synthesis)

### 3.5 Output Artifacts

For each repo, the skill produces:

1. **Research Report** (`raw/code-research-{repo-name}.md`)
   - Frontmatter with repo URL, research goal, date, relevance score
   - Executive summary (3-5 sentences)
   - Per-dimension findings with code references
   - Architecture diagram (mermaid)
   - Relevance matrix to trading system
   - Adoption recommendations (prioritized)

2. **Code Snippets Catalog** (`outputs/code-snippets-{repo-name}/`)
   - Extracted code files organized by pattern/component
   - Each with a README explaining what it does and how to adapt it

3. **Wiki Updates** (via /kb-compile)
   - New wiki articles for novel patterns discovered
   - Updates to existing articles with cross-references
   - Source citations back to the research report

4. **Trading KB Cross-Reference** (`outputs/trading-xref-{repo-name}.md`)
   - Mapping of findings to trading system subsystems
   - Specific adoption recommendations with effort estimates
   - Flagged for manual review and integration into trading KB

5. **Research Log Entry** (appended to `log.md`)
   - Timestamp, repo, goal, relevance score, key findings count, artifacts produced

### 3.6 Repo-Specific Research Goals

#### Initial Research Queue

| # | Repo | Research Goal | Priority |
|---|------|---------------|----------|
| 1 | `karpathy/autoresearch` | Extract the autoresearch loop mechanics — mutation, eval, selection, git-based versioning of experiments. Map to trading strategy self-improvement. | P0 |
| 2 | `NousResearch/hermes-agent` | Analyze multi-model orchestration, tool routing, and agent persona system. Extract patterns for heterogeneous agent swarms. | P0 |
| 3 | `openclaw/openclaw` | Deep dive into Pi harness, SOUL.md, skill system, clawhub, plugin architecture. Extract patterns for extensible harness with community skills. | P0 |
| 4 | Local Claude Code source (`Downloads/Claude Code/src`) | Reverse-engineer the full Claude Code harness: loop, tools, TodoWrite, compaction, auto mode classifier, subagent handoffs. Gold standard reference. | P0 |
| 5 | `anomalyco/opencode` | Analyze alternative harness design — compare with Claude Code. Look for novel patterns in tool design or context management. | P1 |
| 6 | `666ghj/MiroFish` | Evaluate agentic trading-specific patterns. Memory for market state, strategy evaluation loops, position management integration. | P1 |
| 7 | `langchain-ai/deepagents` | LangChain's deep agents framework — multi-step research, harness optimization patterns, TerminalBench performance. | P0 |

#### Additional Repos to Consider (from KB + research)

| Repo | Rationale |
|------|-----------|
| `anthropics/claude-code-sdk-python` | Official Claude Agent SDK — the building block for custom harnesses |
| `langchain-ai/langgraph` | State machine orchestration for multi-agent workflows |
| `microsoft/autogen` | Multi-agent conversation framework with code execution |
| `crewAI-Inc/crewAI` | Role-based multi-agent orchestration |
| `letta-ai/letta` | Memory-first agent harness (MemGPT successor) |
| `All-Hands-AI/OpenHands` | Full coding agent with sandbox execution |
| `princeton-nlp/SWE-agent` | Interface design for coding agents (64% improvement from harness alone) |
| `Significant-Gravitas/AutoGPT` | Long-running autonomous agent with memory and planning |
| `joonspk-research/generative_agents` | Memory-driven generative agents (Stanford) |

### 3.7 Learnings Accumulation (v1) / Self-Improvement (v2)

**v1: Learnings accumulation (no automated prompt revision).**
After each research session:
1. Append structured observations to `outputs/research-learnings.md` (what worked, what failed, what was missed, timing data)
2. The skill reads this file at invocation start for context
3. Human reviews accumulated learnings every 3-5 repos and manually updates dimension prompts
4. Prompt versions are tracked in the learnings file for rollback

**v2 (deferred): Automated self-improvement.**
Requires: (1) human-rated quality baseline across 10+ repos, (2) eval pipeline with before/after comparison, (3) prompt version control with automated rollback on quality regression. Not attempted until v1 is stable and a quality signal exists.

## 4. KB Integration Strategy

### 4.1 Harness Engineering KB (primary)
- Each research report is ingested as `raw/code-research-{repo-name}.md`
- `/kb-compile` creates or updates wiki articles
- Cross-pollination: new findings update existing wiki articles (e.g., a novel memory pattern in repo X updates `agent-memory-and-context-management.md`)

### 4.2 Trading KB (cross-reference)
- The Dimension 8 agent produces a trading-specific cross-reference doc
- This doc is saved in `outputs/` of the harness KB AND flagged for manual integration into `C:\Users\dunliu\projects\knowledge_base\agents\trading\`
- Specific design recommendations are tagged with the trading system subsystem they affect

### 4.3 Knowledge Graph (deferred to v2+)
- **Not in scope for v1.** As the number of researched repos grows, a future version could build a knowledge graph mapping repos to patterns to subsystems to extraction status. This is tracked as a future idea, not a v1 requirement.

## 5. Implementation Plan

### Phase 1: Skill Skeleton (1 session)
- Create the skill file at `~/.claude/skills/kb-code-research/`
- Implement orchestrator agent with repo cloning/access
- Implement initial triage (README, structure analysis)
- Implement 2-3 dimension agents as proof of concept
- Test on `karpathy/autoresearch` (smallest, most focused repo)

### Phase 2: Full Dimension Coverage (1-2 sessions)
- Implement all 8 dimension agents
- Implement synthesis agent and report generation
- Implement KB integration agent
- Test on Claude Code local source (largest, most complex)

### Phase 3: Refinement (1 session)
- Run on remaining P0 repos
- Calibrate convergence criteria
- Tune parallel agent prompts based on results
- Add self-improvement loop

### Phase 4: Skill Packaging (1 session)
- Package as a proper Claude Code skill with install instructions
- Write skill documentation
- Consider publishing to the agent harness engineering community (not as a general code analysis tool)

## 6. Non-Goals

- We are NOT building a general-purpose code analysis tool (focused on agent harness patterns)
- We are NOT building an automated code migration/porting tool (extraction is manual)
- We are NOT replacing human judgment on what to adopt (the skill recommends, human decides)
- We are NOT researching closed-source agents (only repos we can read)

## 7. Open Questions

1. Should the skill support incremental re-research (re-run on a previously analyzed repo to check for updates)?
2. Should dimension agents have access to the existing KB wiki articles for context, or should they research "blind" to avoid confirmation bias?
3. What is the right token budget per dimension agent? (Estimated: 50-80K tokens per dimension)
4. Should the trading KB cross-reference be automated (agents write to trading KB directly) or manual (produce a recommendation doc)?
5. Should we build a web UI / dashboard for browsing research results, or are the markdown artifacts sufficient?

## 8. Dependencies

- Claude Code with Agent tool support (subagent spawning)
- Git access for cloning repos
- Existing KB infrastructure (ingest.js, compile.js, /kb-* skills)
- Sufficient API budget (~$5-15 per deep repo research session)

## 9. Risks

| Risk | Mitigation |
|------|------------|
| Token cost per repo too high | Budget cutoffs, dimension relevance triage |
| Parallel agents produce contradictory findings | Synthesis agent explicitly resolves contradictions |
| Research quality varies by repo size/quality | Convergence criteria enforce minimum evidence threshold |
| Skill prompts need tuning per repo type | Self-improvement loop after every 5 repos |
| Large repos (100K+ files) overwhelm agents | Initial triage identifies hot paths; agents focus on relevant directories |
