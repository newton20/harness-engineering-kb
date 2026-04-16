---
title: "Code Research: 666ghj-mirofish"
source: https://github.com/666ghj/MiroFish
author: "kb-code-research skill"
date: 2026-04-15
fetched: 2026-04-15
type: code-research
status: raw
tags: [code-research, swarm-simulation, graph-memory, react-agent, multi-agent]
relevance_score: 6
research_goal: "Evaluate agentic trading-specific patterns. Memory for market state, strategy evaluation loops, position management integration."
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: 666ghj-mirofish

## Executive Summary

MiroFish is a "swarm intelligence engine" that simulates thousands of LLM-backed agents interacting on parallel social platforms (Twitter + Reddit) to predict opinion dynamics. It implements a two-tier architecture: a deterministic, round-based simulation loop (OASIS engine) running as a subprocess, and a hand-rolled ReAct agent (ReportAgent) that reads from a Zep graph memory store to generate analysis reports. The most novel pattern is the **post-simulation hibernated agent interview**: after simulation rounds complete, the subprocess stays alive so the ReportAgent can interrogate individual agents via file-based IPC. The **temporal fact lifecycle** (active/historical edges in Zep) and the **minimum tool-call enforcement** (forcing retrieval diversity before Final Answer) are both genuinely new patterns not in the existing KB. While there are no trading-specific constructs in the codebase, the write-once-read-many memory architecture, platform-isolation-then-convergence pattern, and time-of-day agent scheduling provide transferable abstractions for multi-agent strategy evaluation systems.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | 666ghj-mirofish |
| Primary language | Python (35 files, ~20K LOC backend) |
| Size classification | medium (~42K LOC total across Python/JS/Vue) |
| File count | 60 source files |
| Last commit | Shallow clone — 1 commit visible (docs rename) |
| Commit frequency | active (trending on GitHub) |
| README quality | detailed (full workflow, setup, screenshots, demo videos) |
| Relevance to goal | 6/10 — Multi-agent simulation engine with transferable patterns for strategy evaluation |
| Agent/harness signals | 571 agent/loop, 399 memory/state, 28 multi-agent |
| Multi-agent signal count | 28 |
| Recommended dimensions | All 4 |

## Dimension 1: Architecture & Loop Design

### Summary
MiroFish implements two structurally distinct agent loops. The simulation layer runs a deterministic, round-based `for` loop (code-controlled) where OASIS agents execute `LLMAction()` in parallel each round — terminating by exhausting a pre-computed `total_rounds` count. The ReportAgent implements a proper ReAct loop (Thought → Tool → Observation → Final Answer) bounded at 5 iterations and 3-5 tool calls. The two layers are architecturally isolated: the simulation runs as a separate subprocess communicating with Flask only through file-based IPC (JSONL logs + JSON command/response files).

### Key Findings
- **Two-tier loop architecture:** Deterministic simulation ticker + model-driven ReAct report loop. Neither loop feeds back into the other.
  - Evidence: `backend/scripts/run_parallel_simulation.py:1228-1279` — `for round_num in range(total_rounds)` with `env.step(actions)`.
  - Evidence: `backend/app/services/report_agent.py:1285-1502` — ReAct loop with tool-call parsing.
  - Significance: Clean separation between environment simulation and analytical reasoning.

- **Code controls both loops:** The simulation loop is entirely clock-driven. The ReAct loop enforces min 3 / max 5 tool calls, rejecting premature Final Answer.
  - Evidence: `backend/app/services/report_agent.py:1287-1389` — `if tool_calls_count < min_tool_calls: messages.append(REACT_INSUFFICIENT_TOOLS_MSG)`.
  - Significance: Minimum tool-call enforcement is a quality guardrail that forces retrieval diversity.

- **Subprocess firewall with file-based IPC:** Simulation runs as `subprocess.Popen` with `start_new_session=True`. Monitor thread polls JSONL every 2s. Post-simulation, IPC command bus via filesystem polling.
  - Evidence: `backend/app/services/simulation_runner.py:436-468` — subprocess spawn + monitor thread.
  - Evidence: `backend/app/services/simulation_ipc.py:117-187` — UUID-keyed command/response JSON files.
  - Significance: Process isolation prevents simulation crashes from taking down the Flask harness.

- **Post-simulation hibernated agent interview:** After rounds complete, subprocess enters IPC wait loop. Flask can send INTERVIEW commands to live OASIS agents whose in-memory state reflects end-of-simulation world.
  - Evidence: `backend/scripts/run_parallel_simulation.py:1595-1631` — post-simulation polling loop.
  - Significance: Genuinely novel — agents maintain full state for post-hoc interrogation.

- **Character-level context management:** No compaction or summarization. Prior sections truncated to 4000 chars. Time config to 10000 chars. Chat report to 15000 chars.
  - Evidence: `backend/app/services/report_agent.py:1265-1272` — `truncated = sec[:4000] + "..."`.
  - Significance: Simple but brittle — no token-counting guard exists.

### Patterns
- Round-as-Clock: each round maps to `minutes_per_round` of simulated time with stochastic agent activation
- Subprocess Firewall: `subprocess.Popen` + `start_new_session=True` isolates heavy OASIS workload
- File-based IPC Command Bus: JSON files in `ipc_commands/` and `ipc_responses/` directories
- Hard-enforced ReAct budget: min 3, max 5 tool calls per section, with unused-tool diversity hints
- Backward-compatible tool aliasing: old tool names silently redirected in `_execute_tool()`

## Dimension 2: Memory & State Management

### Summary
MiroFish implements a four-layer memory architecture: Zep Cloud graph (long-term, external), local JSON files (project/simulation state), SQLite per-simulation databases (agent actions/posts), and in-memory Python dicts (transient). Agent actions are converted to natural-language episode text via deterministic templates and batch-pushed to Zep. The ReportAgent reads from Zep using 3 purpose-built retrieval tools. A sharp boundary exists between ephemeral in-context state and permanent Zep graph — the graph is the only memory that "survives everything."

### Key Findings
- **Four memory systems:** Zep Cloud graph (external, permanent), local JSON files (per-entity), SQLite per-simulation DBs (relational, OASIS-native), in-memory dicts (transient).
  - Evidence: `backend/app/services/zep_graph_memory_updater.py:1-20` — Zep client; `backend/app/models/project.py:167-174` — `project.json`; `backend/app/api/simulation.py:2023-2044` — SQLite query.
  - Significance: Memory split across 3 query surfaces — complex but durable.

- **Episode-to-graph pipeline:** Agent actions → `to_episode_text()` (deterministic NL) → `client.graph.add(type="text")` → Zep auto-extracts entities/relations.
  - Evidence: `backend/app/services/zep_graph_memory_updater.py:35-62, 396-424`.
  - Significance: No manual schema mapping at write time. Zep handles entity extraction automatically.

- **Temporal fact lifecycle:** Zep edges carry `valid_at`, `invalid_at`, `expired_at`. PanoramaResult splits `active_facts` from `historical_facts`.
  - Evidence: `backend/app/services/zep_tools.py:91-135, 214-281`.
  - Significance: Built-in temporal dimension to the knowledge graph — facts "expire" as simulation progresses.

- **Platform-partitioned batch queue:** Two in-memory buffers (`twitter`/`reddit`) accumulate actions up to BATCH_SIZE=5, flushed by daemon thread at 0.5s intervals.
  - Evidence: `backend/app/services/zep_graph_memory_updater.py:249-256, 364-394`.
  - Significance: Prevents overwhelming Zep API with per-action writes during high-concurrency simulation.

- **Dual-path state persistence:** Every state object written both to in-memory dict AND JSON file. Reads check dict first, fall back to file.
  - Evidence: `backend/app/services/simulation_manager.py:157-192`.
  - Significance: Fast reads with durable fallback. No migration layer though.

- **No self-modification:** Agents cannot modify their system prompts, ontology, or Zep graph structure. Report Agent has only read tools. Zep writes are driven by the action log pipe, not agent self-reflection.
  - Evidence: `backend/app/services/zep_tools.py:401-419` — read-only tools; `backend/app/config.py:52-59` — static action lists.
  - Significance: Intentional for simulation integrity, but limits adaptive learning.

### Patterns
- Episode-to-graph pipeline via deterministic templates (no LLM summarization)
- Platform-partitioned batch queue for Zep writes
- Cursor-based pagination with per-page retry for graph reads
- Ontology-gated entity filtering at read time
- Temporal fact lifecycle (active vs. historical via timestamps)
- Dual-path state persistence (dict + file)
- File-based IPC for inter-process communication

## Dimension 3: Tool & Action Space Design

### Summary
MiroFish operates two distinct tool layers that never interact. The ReportAgent has 4 high-level Zep retrieval tools (insight_forge, panorama_search, quick_search, interview_agents) defined as natural-language Python dicts — no JSON Schema, no OpenAI function-calling API. The simulation layer has 17 OASIS platform actions (6 Twitter, 13 Reddit). Tools are parsed from free-text LLM output using a custom XML-tag protocol with JSON fallback. Tool failures are handled via 3-tier strategy: retry with backoff, local keyword search fallback, and error-as-observation.

### Key Findings
- **21 total action types across two namespaces:** 4 ReportAgent tools + 17 OASIS simulation actions. Completely decoupled — different dispatch, different schemas, different processes.
  - Evidence: `backend/app/services/report_agent.py:919-954` — 4 tools defined; `backend/app/config.py:52-59` — OASIS action lists.
  - Significance: No shared action registry — adding a new action requires modifying both config and dispatch.

- **High-level composite tools:** InsightForge is a 5-step pipeline: LLM sub-query generation → semantic search per sub-query → entity UUID extraction → per-node detail fetch → result aggregation. PanoramaSearch fetches ALL nodes and edges, then does local relevance sorting.
  - Evidence: `backend/app/services/zep_tools.py:944-1090` — insight_forge pipeline; `backend/app/services/zep_tools.py:1145-1235` — panorama full dump.
  - Significance: Tool granularity hierarchy (quick_search < panorama_search < insight_forge) maps to time-sensitive vs. deep analysis.

- **Custom XML-tag tool protocol:** `<tool_call>{"name": ..., "parameters": {...}}</tool_call>` parsed via regex. Fallback: bare JSON detection. No OpenAI `tools=` parameter used.
  - Evidence: `backend/app/services/report_agent.py:1067-1125` — dual-tier parser.
  - Significance: Model-agnostic but brittle. Avoids vendor lock-in to specific structured-output format.

- **3-tier failure handling:** (1) Exponential backoff retry for Zep API (max 3 retries, 2s initial, 2x). (2) Local keyword search fallback on API failure. (3) Error string returned as Observation to LLM.
  - Evidence: `backend/app/services/zep_tools.py:442-462, 541-544`; `backend/app/services/report_agent.py:1060-1062`.
  - Significance: Error-as-context preserves ReAct loop continuity. Local fallback maintains functional continuity.

- **Minimum tool-call enforcement:** ReAct loop requires ≥3 tool calls before accepting Final Answer. Unused-tool hints nudge diversity.
  - Evidence: `backend/app/services/report_agent.py:1379-1388, 1451-1454`.
  - Significance: Novel quality guardrail — forces broad evidence gathering before synthesis.

- **No MCP integrations, no dynamic tool loading.** All tools registered at construction time. Static tool set.
  - Evidence: `backend/app/services/report_agent.py:910` — eager registration.
  - Significance: Simple but not extensible. Domain-specific analysis requires forking the tool dict.

### Patterns
- XML-tag tool call protocol with JSON fallback
- Error-as-Observation pattern for ReAct loop continuity
- Zep Search → local keyword fallback chain
- File-based IPC as cross-process tool channel (interview_agents)
- Temporal edge classification for time-aware memory retrieval
- Legacy tool aliasing (silent redirect from old to new tool names)
- Tool diversity enforcement via minimum-call + unused-tool hints

## Dimension 4: Multi-Agent Coordination

### Summary
MiroFish runs a genuine multi-agent swarm built on the CAMEL-AI OASIS engine. Within each platform, agents act simultaneously in each time-step round, coordinated only through a shared SQLite environment (bulletin-board model) — no direct agent-to-agent messages. Twitter and Reddit simulations run in parallel via asyncio.gather, each capped at 30 concurrent LLM calls. The two swarms are completely isolated during simulation but converge at the Zep graph level when the ReportAgent analyzes both.

### Key Findings
- **Swarm peer topology within platform, orchestrator-worker between Flask and subprocess:** Agents share a single SQLite DB and interact through it. No direct message passing.
  - Evidence: `backend/scripts/run_parallel_simulation.py:1155-1160` — `oasis.make(semaphore=30)`; `backend/app/services/simulation_runner.py:438-468` — `subprocess.Popen`.
  - Significance: Clean two-level topology but with asyncio/threading impedance mismatch at the boundary.

- **Stochastic time-step task delegation:** `get_active_agents_for_round()` selects agents based on per-agent `activity_level`, `active_hours`, peak/off-peak multipliers.
  - Evidence: `backend/scripts/run_parallel_simulation.py:1040-1090`.
  - Significance: Novel — models real-world behavioral patterns for realistic simulation.

- **Bulletin-board communication via shared SQLite:** Agents post, like, repost, follow. Later agents see previous posts when OASIS builds their observation. Zero direct agent-to-agent messages.
  - Evidence: `backend/scripts/run_parallel_simulation.py:517-558, 657-746`.
  - Significance: Equivalent to environment-mediated communication — no game-of-telephone corruption.

- **Platform isolation during simulation, convergence at analysis:** Twitter and Reddit swarms never interact during simulation. Both push to the same Zep graph. ReportAgent queries the unified graph.
  - Evidence: `backend/scripts/run_parallel_simulation.py:1585-1588` — `asyncio.gather` with separate coroutines.
  - Significance: Isolation-then-convergence prevents cross-contamination during data collection while enabling holistic analysis.

- **Dual-LLM split for platform parallelism:** Twitter uses default LLM; Reddit uses `LLM_BOOST_*` env-var config, allowing different API providers per platform.
  - Evidence: `backend/scripts/run_parallel_simulation.py:984-1037`.
  - Significance: Practical for managing API rate limits across concurrent simulation tracks.

- **SQLite rowid cursor for action deduplication:** `WHERE rowid > ?` ensures each action row is processed exactly once. No cross-restart persistence of the cursor.
  - Evidence: `backend/scripts/run_parallel_simulation.py:685-696`.
  - Significance: Sound replay-prevention mechanism but vulnerable to subprocess crash mid-round.

### Patterns
- Swarm peer topology with bulletin-board (SQLite) communication
- Time-of-day activity modeling for agent scheduling
- Platform isolation during simulation, convergence at graph level
- Dual-LLM configuration for parallel platform workloads
- Post-simulation IPC command-wait loop for agent interviews
- Deterministic action→text serialization to Zep (no LLM summarization in write path)
- Semaphore-bounded async concurrency (30 per platform)

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions detected. All four dimensions converge on the same two-tier architecture.

### Cross-Cutting Flows

**Flow 1: Simulation-to-Report Memory Pipeline**
- Dim 1: Simulation loop ticks through rounds, writing actions to JSONL
- Dim 2: ZepGraphMemoryUpdater batches JSONL actions into episode text, pushes to Zep
- Dim 3: ReportAgent's tools read FROM Zep (InsightForge, PanoramaSearch, QuickSearch)
- Dim 4: Multiple simulation agents produce data; single ReportAgent consumes
- Integrated: Write-once-read-many memory architecture. Zep graph is a one-way knowledge bridge — simulation writes, report reads, no feedback loop.
- Significance: For trading harness design, maps to agents writing market observations to shared graph, analysis agents reading post-hoc. Temporal fact lifecycle maps to price validity windows.

**Flow 2: Post-Simulation Interview Bridge**
- Dim 1: After loop ends, subprocess enters IPC command-wait polling loop
- Dim 2: Agent in-memory state (OASIS environment) preserved — no serialization
- Dim 3: `interview_agents` tool routes through file-system IPC to live OASIS env
- Dim 4: Interview targets specific agents by ID across process boundary
- Integrated: "Hibernated agent" pattern — agents maintain full internal state between simulation and analysis phases. Fragile (crash = state loss) but enables rich post-hoc interrogation.
- Significance: For trading harness design, maps to keeping strategy agents alive after backtesting for decision rationale interrogation.

**Flow 3: Dual-Platform Swarm Convergence**
- Dim 1: Two coroutines via asyncio.gather
- Dim 2: Each platform has own SQLite + Zep buffer, merging into single Zep graph
- Dim 3: PanoramaSearch queries unified graph — no platform filter
- Dim 4: Two swarms completely isolated during simulation, converge at graph level
- Integrated: Platform isolation during collection, convergence at analysis. ReportAgent performs implicit cross-platform analysis.
- Significance: For trading harness design, maps to separate swarms for different data feeds (news, social, market) converging into unified knowledge graph.

### Novelty Assessment

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| Round-as-Clock environment ticker | Dim 1 | NOVEL | Deterministic time-step loop not in any wiki article |
| Post-simulation hibernated agent interview | Dim 1/3 | NOVEL | Agents kept alive for interrogation — entirely new pattern |
| Minimum tool-call enforcement (3-5 with diversity nudge) | Dim 1/3 | NOVEL | Quality guardrail forcing retrieval breadth |
| Temporal fact lifecycle (active/historical edges) | Dim 2 | NOVEL | valid_at/invalid_at on graph edges — not in KB |
| Time-of-day activity modeling for agent scheduling | Dim 4 | NOVEL | Stochastic per-agent diurnal scheduling |
| Episode-to-graph pipeline via Zep | Dim 2 | VARIANT | Similar to MemGPT archival but graph DB not vector store |
| Subprocess firewall for simulation | Dim 1 | VARIANT | Similar to OpenHands sandbox but for simulation isolation |
| File-based IPC command bus | Dim 1/4 | VARIANT | Similar to event-stream but file-based |
| XML-tag tool protocol with JSON fallback | Dim 3 | VARIANT | Alternative to function-calling; model-agnostic |
| Zep Search → local keyword fallback | Dim 3 | VARIANT | General fallback chain already documented |
| Error-as-Observation | Dim 3 | KNOWN | In wiki/tool-design-patterns.md |
| ReAct pattern | Dim 1 | KNOWN | In wiki/agentic-design-patterns.md |
| Swarm peer topology via shared DB | Dim 4 | KNOWN | In wiki/agentic-design-patterns.md |

## Decisions to Adopt

1. **Adopt: Minimum tool-call enforcement with diversity nudging** from `report_agent.py`
   - What: Require agents to call at least N different tools before accepting Final Answer. Track used vs. unused tools and inject hints encouraging diversity.
   - Why: Prevents lazy single-tool-call analysis patterns. Forces evidence breadth before synthesis.
   - Effort: S
   - Target: Any ReAct-based agent harness with multiple retrieval tools

2. **Adopt: Temporal fact lifecycle for graph memory** from `zep_tools.py`
   - What: Attach valid_at/invalid_at/expired_at timestamps to knowledge graph edges. Split retrieval results into active vs. historical facts.
   - Why: Enables time-aware reasoning over evolving world state — essential for any simulation or market-state tracking.
   - Effort: M
   - Target: Memory systems in long-running agents or simulation harnesses

3. **Adopt: Post-simulation hibernated agent pattern** from `run_parallel_simulation.py`
   - What: After agent loop completes, keep the subprocess alive in a command-polling state so the orchestrator can interrogate individual agents about their decisions.
   - Why: Enables rich post-hoc analysis without re-instantiating agents or losing their accumulated state.
   - Effort: M
   - Target: Any multi-agent system where post-run analysis is valuable

4. **Adopt: Episode-to-graph pipeline with deterministic serialization** from `zep_graph_memory_updater.py`
   - What: Convert structured agent actions to natural-language text via deterministic templates (not LLM summarization), then batch-push to a graph knowledge store.
   - Why: Deterministic serialization prevents hallucination in the write path. Batching prevents API overload during high-concurrency simulation.
   - Effort: M
   - Target: Any multi-agent system needing persistent shared memory

5. **Adopt: Platform isolation during simulation, convergence at analysis** from the dual-platform architecture
   - What: Run separate agent swarms for different data feeds in isolated environments. Merge outputs into a unified knowledge graph for cross-feed analysis.
   - Why: Prevents cross-contamination during data collection while enabling holistic analysis. Each feed can have its own LLM provider and rate limits.
   - Effort: L
   - Target: Multi-source trading or prediction harnesses

## Evidence Index

```
EVIDENCE INDEX:
  Verified: 26 paths (100%)
  Unverified: 0 paths (0%)

  Verified:
    ✓ backend/scripts/run_parallel_simulation.py — simulation loop, parallel coordination, IPC
    ✓ backend/app/services/report_agent.py — ReAct loop, tool definitions, enforcement
    ✓ backend/app/services/simulation_runner.py — subprocess spawn, monitor thread, run state
    ✓ backend/app/services/simulation_ipc.py — file-based IPC client
    ✓ backend/app/services/simulation_config_generator.py — time/event config generation
    ✓ backend/app/services/simulation_manager.py — state persistence, simulation lifecycle
    ✓ backend/app/services/zep_graph_memory_updater.py — episode pipeline, batch queue
    ✓ backend/app/services/zep_tools.py — retrieval tools, temporal facts, local fallback
    ✓ backend/app/services/zep_entity_reader.py — ontology-gated entity filtering
    ✓ backend/app/services/oasis_profile_generator.py — persona generation, fix_truncated_json
    ✓ backend/app/services/ontology_generator.py — entity type deduplication
    ✓ backend/app/services/graph_builder.py — Zep graph creation, episode waiting
    ✓ backend/app/api/simulation.py — API endpoints, SQLite queries
    ✓ backend/app/api/graph.py — graph API endpoints
    ✓ backend/app/api/report.py — report API endpoints
    ✓ backend/app/config.py — OASIS action lists, environment config
    ✓ backend/app/models/project.py — project.json persistence
    ✓ backend/app/models/task.py — TaskManager in-memory dict
    ✓ backend/app/utils/llm_client.py — OpenAI SDK client
    ✓ backend/app/utils/retry.py — retry decorator (unused by Zep)
    ✓ backend/app/utils/zep_paging.py — cursor-based pagination
    ✓ backend/app/utils/file_parser.py — file parsing utilities
    ✓ backend/scripts/action_logger.py — legacy action log interface
    ✓ backend/scripts/run_reddit_simulation.py — Reddit simulation runner
    ✓ backend/scripts/run_twitter_simulation.py — Twitter simulation runner
    ✓ backend/run.py — Flask entry point
```

## Sources

- [666ghj-mirofish](https://github.com/666ghj/MiroFish) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
