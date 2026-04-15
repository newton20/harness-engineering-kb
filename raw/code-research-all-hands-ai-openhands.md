---
title: "Code Research: all-hands-ai-openhands"
source: https://github.com/all-hands-ai/openhands
author: "kb-code-research skill"
date: 2026-04-14
fetched: 2026-04-14
type: code-research
status: compiled
compiled_to: [agent-memory-and-context-management, tool-design-patterns, long-running-agent-harnesses, practical-best-practices, multi-agent-reliability, agentic-design-patterns]
compiled_date: 2026-04-15
tags: [code-research, event-driven-architecture, condenser-system, runtime-sandboxing]
relevance_score: 10
research_goal: "analyze event-driven agent loop, runtime sandboxing, and micro-agent delegation patterns"
dimensions_analyzed: [architecture, memory, tools, multi-agent]
---

# Code Research: all-hands-ai-openhands

## Executive Summary

OpenHands (formerly OpenDevin) is a production AI-driven development platform achieving 77.6% on SWE-Bench, built on a fully event-driven architecture where a single append-only EventStream serves as the universal integration layer for all agent stepping, tool execution, memory management, and multi-agent coordination. Its most architecturally distinctive contribution is a pluggable condenser system with 9 composable implementations that treats condensation as a first-class event (the agent returns a CondensationAction from step(), making compression auditable and replayable). The tool layer introduces a novel security-risk-as-parameter pattern where the LLM must self-label every action's risk level. Despite this sophisticated architecture, the entire V0 codebase is deprecated as of version 1.0.0 with migration to a separate Software Agent SDK underway — making this research a snapshot of a system in transition. Key novel findings include the most extensive condenser system in any open-source framework, event store as JSON files per event, MCP stdio-over-HTTP proxy inside Docker sandboxes, and delegate history scrubbing via disjoint event ID slices.

## Triage Scorecard

| Dimension | Value |
|-----------|-------|
| Repo name | all-hands-ai-openhands |
| Primary language | Python (1,225 files) |
| Size classification | large (2,250+ source files) |
| File count | 1,225 .py + 1,021 .ts/.tsx + 7 .js |
| Last commit | 2026-04-14 (active) |
| Commit frequency | active |
| README quality | detailed (SDK, CLI, GUI, Cloud, Enterprise sections) |
| Relevance to goal | 10/10 — event-driven loop, runtime sandboxing, micro-agent delegation, 5+ agent types |
| Agent/harness signals | 478 matches |
| Multi-agent signal count | 396 (well above 10 threshold) |
| Recommended dimensions | All 4 |

## Dimension 1: Architecture & Loop Design

### Summary
OpenHands implements a fully event-driven, reactive agent loop where the controller does not poll — every event arriving on the shared EventStream triggers a `should_step()` check and, when appropriate, a synchronous `agent.step()` call. The primary agent (CodeActAgent) is ReAct-style: it receives condensed history, calls the LLM, and emits one action per step. Hierarchical delegation is first-class via parent-child AgentControllers sharing the same event stream. Five termination conditions exist: model finish/reject, iteration limit (500), budget limit (USD), stuck detection (5 scenarios), and unrecoverable errors.

### Key Findings
- **Event-driven ReAct loop:** The "loop" (`core/loop.py`) is just `while state not in end_states: await asyncio.sleep(1)` — a status poller. The actual stepping is callback-driven via `on_event` → `should_step()` → `agent.step()`.
  - Evidence: `openhands/controller/agent_controller.py:454-512` — event callback drives stepping
  - Significance: Decouples agent from runtime's async I/O; new event types can trigger steps without touching loop logic

- **Code controls the loop:** The model is a pure function (State → Action). The harness enforces iteration caps, budget caps, detects stuck states, and injects loop-recovery actions.
  - Evidence: `openhands/controller/state/control_flags.py:64-102` — IterationControlFlag and BudgetControlFlag raise RuntimeError on limit
  - Significance: Full harness control — can pause, rate-limit, inject recovery, or stop without model cooperation

- **5-scenario stuck detection:** Repeated identical actions, repeated action-error loops, repeated action-observation pairs, repeated errors, and repeated condensation events — each with configurable thresholds.
  - Evidence: `openhands/controller/stuck.py:45-138`
  - Significance: Production-grade loop-escape with interactive recovery options in CLI mode

- **Condenser-returns-action pattern:** Rather than the controller driving condensation externally, the agent itself returns a CondensationAction from step(). This makes condensation a first-class event in the stream — auditable, replayable, and logged.
  - Evidence: `openhands/agenthub/codeact_agent/codeact_agent.py:204-213`
  - Significance: Architecturally elegant — condensation participates in the event lifecycle like any other action

- **Multiple runtime backends:** Docker, Kubernetes, Local, Remote, CLI all implement `Runtime.run_action()`. The agent never knows which backend it's talking to.
  - Evidence: `openhands/runtime/base.py:106-248`
  - Significance: Clean abstraction enables deployment from laptop Docker to production Kubernetes without agent changes

### Patterns
- Event-stream-as-shared-bus: all components subscribe to the same EventStream
- Action-as-return, Observation-as-input: canonical pattern throughout
- Pending action gate: prevents double-stepping while action awaits observation
- Shared iteration counter across delegate chain
- Replay mode for debugging/benchmarking saved trajectories

## Dimension 2: Memory & State Management

### Summary
OpenHands has one of the most sophisticated memory architectures in any open-source agent framework. Four memory planes coexist: (1) an append-only file-backed event store as the single source of truth, (2) a pluggable condenser layer with 9 implementations that shape a "View" before the LLM sees it, (3) an in-RAM State.history list rebuilt from the event store on each restore, and (4) keyword-triggered microagent knowledge injection. State persistence uses pickle-base64 but explicitly excludes history — history is always reconstructed from the durable event store. No vector search or semantic retrieval exists; microagent matching is pure substring search.

### Key Findings
- **9 condenser implementations:** NoOp, ObservationMasking, BrowserOutput, RecentEvents, ConversationWindow, AmortizedForgetting, LLMAttention, LLMSummarizing, StructuredSummary — plus a Pipeline condenser that chains multiple condensers.
  - Evidence: `openhands/memory/condenser/impl/` directory, `openhands/core/config/condenser_config.py:188-200`
  - Significance: Most extensive pluggable condenser system in any open-source framework; pipeline composability enables combinations like BrowserOutput → LLMSummarizing

- **Event store as source of truth:** Every event is written as an individual JSON file to a FileStore backend (Local, S3, GCS, InMemory). State.history is rebuilt by replaying events from the store on each session restore.
  - Evidence: `openhands/events/event_store.py:44-183`, `openhands/controller/state/state.py:198-201` (history excluded from pickle)
  - Significance: Full audit trail preserved; condensation is a view-level concern, not deletion. But startup cost grows linearly with event count.

- **Dual reactive + proactive condensation:** Proactive: `RollingCondenser.should_condense()` triggers when `len(view) > max_size`. Reactive: Controller catches `ContextWindowExceededError` and emits `CondensationRequestAction`.
  - Evidence: `openhands/memory/condenser/condenser.py:169-193`, `openhands/controller/agent_controller.py:929-955`
  - Significance: Robust dual-strategy with stuck-loop detection as a third safety layer

- **No vector search:** Microagent knowledge injection uses pure substring matching (`trigger.lower() in message.lower()`). For a 77.6% SWE-Bench system, this is surprisingly primitive.
  - Evidence: `openhands/microagent/microagent.py:189-198`
  - Significance: Demonstrates that sophisticated retrieval is not necessary for strong benchmark performance

- **Prompt caching support:** `ConversationMemory.apply_prompt_caching()` marks system message and last user/tool message as `cache_prompt=True` for Anthropic API cache breakpoints.
  - Evidence: `openhands/memory/conversation_memory.py:696-709`
  - Significance: Cache-aware message construction, similar to Claude Code's cache-first architecture

### Patterns
- Condenser returns View or Condensation (never partial) — condensation is first-class
- CondenserPipeline composability — chain multiple strategies
- Keyword-triggered microagent injection via RecallObservation events
- Delegate history slicing via start_id/end_id ranges
- State pickle excludes history — always rebuilt from event store

## Dimension 3: Tool & Action Space Design

### Summary
OpenHands implements a dual-layer tool system: statically declared agent-specific tools (up to 10 for CodeActAgent, expressed as litellm ChatCompletionToolParam JSON schemas) plus dynamically loaded MCP tools at runtime. Every tool call resolves to a strongly typed Action dataclass dispatched via string-keyed reflection (`getattr(self, action_type)(action)`) through the runtime. A distinctive cross-cutting feature is security-risk-as-parameter: the LLM must self-declare LOW/MEDIUM/HIGH risk on every executable tool call, with a pluggable SecurityAnalyzer (Invariant, GraySwan, or LLM-based) that can independently evaluate or block.

### Key Findings
- **Security-risk-as-parameter:** Every executable tool (bash, file edit, browser, IPython) requires `security_risk: LOW|MEDIUM|HIGH` as a mandatory parameter. The LLM self-labels each action's risk.
  - Evidence: `openhands/agenthub/codeact_agent/tools/security_utils.py`, `openhands/runtime/base.py:1099-1119`
  - Significance: Novel inversion — the agent classifies its own actions. Serves as input to pluggable SecurityAnalyzer and confirmation UI.

- **MCP stdio-over-HTTP proxy in sandbox:** Stdio MCP servers run inside the Docker action execution server container, proxied by FastMCP to an HTTP endpoint reachable by the outer coordinator.
  - Evidence: `openhands/runtime/mcp/proxy/manager.py:29-87`, `openhands/core/config/mcp_config.py:380`
  - Significance: Solves a key distribution problem — stdio servers can run sandboxed while remaining callable from outside.

- **Condensation request as a tool:** The `request_condensation` tool lets the agent proactively request memory compression — context management exposed as a deliberate LLM action.
  - Evidence: `openhands/agenthub/codeact_agent/tools/condensation_request.py`
  - Significance: The agent can decide when to compress, not just react to overflow — a unique level of agent agency over memory.

- **Action-as-typed-dataclass with runtime reflection:** Actions are Python dataclasses with an `action: str` field matching ActionType enum. Runtime dispatch is `getattr(self, action_type)(action)` — the string type is also the method name.
  - Evidence: `openhands/runtime/base.py:1105-1119`, `openhands/core/schema/action.py:11-109`
  - Significance: Clean dispatch pattern but requires synchronization between enum values and runtime methods.

- **Dual file editor modes:** `FileEditAction` supports two backends: `LLM_BASED_EDIT` (LLM generates diff-like drafts) and `OH_ACI` (str_replace editor from openhands-aci package).
  - Evidence: `openhands/events/action/files.py:62-138`
  - Significance: Allows A/B testing of edit strategies; ACI is the default, LLM-based is fallback for models with weaker structured output.

- **LoC tools via Jupyter RPC:** LoC agent's graph-traversal tools execute as `print(func_name(**arguments))` strings sent to IPythonRunCellAction — Jupyter kernel as a generic RPC mechanism.
  - Evidence: `openhands/agenthub/loc_agent/function_calling.py:78-83`
  - Significance: Reuses IPython channel for graph queries without a separate execution path.

### Patterns
- Security-risk-as-parameter on every executable tool
- Pending actions queue (multi-action responses drip-fed one per step)
- Platform-adaptive tool descriptions (bash → PowerShell on Windows)
- MCP tools injected after runtime initialization (dynamic loading)
- Tool description length adapts per-model (long vs. short for older GPT-4)

## Dimension 4: Multi-Agent Coordination

### Summary
OpenHands implements a true multi-agent system built on strict orchestrator-worker hierarchy. The primary coordination primitive is AgentDelegateAction/AgentDelegateObservation: a parent AgentController spawns a child controller in-process, routes all events to it via the shared EventStream, and blocks until the delegate finishes. Only one delegate can be active at a time (single-slot). Microagents are NOT autonomous agents — they are keyword-triggered knowledge injections (prompt augmentation). The only hardwired delegation target from CodeActAgent is BrowsingAgent. Work isolation uses disjoint event ID ranges on the shared stream, and a global iteration counter crosses the delegation boundary.

### Key Findings
- **Single-slot sequential delegation:** Parent AgentController holds exactly one `delegate` reference. Only one child runs at a time; no parallelism within a session.
  - Evidence: `openhands/controller/agent_controller.py:120-123`, `openhands/controller/agent_controller.py:413-420`
  - Significance: Simple, deadlock-free, but throughput-limited. Fan-out requires session-level parallelism.

- **Shared EventStream, disjoint ID slices:** Parent and delegate share the same EventStream object. Isolation is via `start_id` offset at spawn time — delegate only sees events from its birth ID onward.
  - Evidence: `openhands/controller/agent_controller.py:762-794`
  - Significance: Avoids channel overhead; all events interleaved in one log; NestedEventStore provides HTTP-backed read-only view for Docker deployment.

- **Delegate history scrubbing:** Parent's history loader filters out all events between AgentDelegateAction and AgentDelegateObservation — parent sees only bookend events.
  - Evidence: `openhands/controller/state/state_tracker.py:152-179`
  - Significance: Prevents telephone problem at the cost of signal loss. TODO comment notes this should use AI-generated summaries (#2395).

- **Microagents are prompt injection, not agents:** KnowledgeMicroagent (keyword-triggered), RepoMicroagent (always active), TaskMicroagent (slash-command template) — all inject content into context via RecallObservation. None are autonomous processes.
  - Evidence: `openhands/microagent/microagent.py:174-275`, `openhands/events/observation/agent.py:48-99`
  - Significance: Naming is misleading. Architecturally these are context augmentation mechanisms, not coordination patterns.

- **Global iteration budget shared across delegation:** `iteration_flag` is passed by reference; parent and delegate consume from the same counter.
  - Evidence: `openhands/controller/agent_controller.py:766`, `openhands/controller/agent_controller.py:805-808`
  - Significance: Deep delegation chains can exhaust budget rapidly; no per-agent budget isolation.

- **Only BrowsingAgent hardwired as delegation target:** Despite generic AgentDelegateAction(agent: str), CodeActAgent exposes only `delegate_to_browsing_agent` to the LLM.
  - Evidence: `openhands/agenthub/codeact_agent/function_calling.py:145-148`
  - Significance: The multi-agent capability is infrastructure-ready but feature-limited in practice.

### Patterns
- Single-slot delegation (strict linear hierarchy)
- Shared stream with ID-range isolation
- Microagents = keyword-triggered prompt augmentation
- Global iteration counter crossing delegation boundary
- Critic system is post-hoc evaluation, not coordination

## Cross-Cutting Analysis

### Contradiction Resolutions
No cross-dimension contradictions detected. All 4 dimensions are consistent on: single-active delegation, microagents-as-knowledge, event stream as spine, code controls loop.

### Cross-Cutting Flows

**Flow 1: Condensation Lifecycle (all 4 dims)**
- Dim 1: Agent returns CondensationAction from step(); controller publishes to event stream and re-steps immediately
- Dim 2: 9 condenser implementations shape the View; pipeline composability chains them
- Dim 3: Condensation exposed as callable tool — LLM can proactively trigger compression
- Dim 4: Delegates have their own condenser pipeline but share event store
- **Integrated view:** Condensation is a first-class event stream participant — auditable, replayable, agent-initiated. Dual path (agent-requested + error-triggered) with stuck-loop detection forms a robust three-layer defense. This is the most sophisticated condensation architecture documented in the KB.

**Flow 2: Event Stream as Universal Integration Layer (all 4 dims)**
- Dim 1: EventStream is the shared bus; on_event callback drives stepping
- Dim 2: Events are the source of truth; history rebuilt from file-backed event store
- Dim 3: Tool execution flows through stream (Action → Runtime → Observation)
- Dim 4: Delegation uses disjoint ID slices on the same stream
- **Integrated view:** The EventStream is the spine of the entire system. All memory, coordination, tool execution, and audit trail flow through a single append-only file-backed queue. No other harness we've studied uses a single stream this comprehensively.

**Flow 3: Security Assessment Across Execution Layers (Dims 1, 3)**
- Dim 1: Controller blocks actions based on security analysis before publishing
- Dim 3: LLM self-labels risk as required parameter; pluggable SecurityAnalyzer can override
- **Integrated view:** Security is split between model (self-labeling) and harness (pluggable analyzer), with harness having final authority. Unique among harnesses studied.

### Novelty Assessment

| Finding | Dimension | Status | Notes |
|---------|-----------|--------|-------|
| Condenser-returns-action pattern | Dim 1 | NOVEL | Agent returns CondensationAction as first-class event |
| 9 composable condenser implementations with pipeline | Dim 2 | NOVEL | Most extensive pluggable condenser system in open source |
| Security-risk-as-parameter on every tool call | Dim 3 | NOVEL | LLM self-labels risk as required param |
| MCP stdio-over-HTTP proxy inside Docker sandbox | Dim 3 | NOVEL | FastMCP proxying stdio servers inside containers |
| Condensation request as agent-callable tool | Dim 3 | NOVEL | Agent proactively requests own memory compression |
| Delegate history scrubbing with disjoint ID slices | Dim 4 | NOVEL | Isolation by event ID ranges on shared stream |
| Event store as JSON files per event | Dim 2 | NOVEL | File-per-event persistence with page caching |
| V0→SDK migration (entire agentic core deprecated) | All | NOVEL | Production code scheduled for replacement |
| Event-driven (callback) loop | Dim 1 | VARIANT | ReAct via callbacks, not while-loop |
| 5-scenario stuck detection with interactive recovery | Dim 1 | VARIANT | Richer than existing KB coverage |
| Polymorphic runtime (Docker/K8s/local/remote/CLI) | Dim 1 | VARIANT | More extensive runtime abstraction |
| Single-slot sequential delegation | Dim 4 | VARIANT | Minimal strict variant of orchestrator-worker |
| Microagents as keyword-triggered prompt injection | Dim 4 | VARIANT | Similar to CLAUDE.md but with keyword matching |
| Dual reactive + proactive condensation | Dim 2 | VARIANT | Dual strategy is new |

## Decisions to Adopt

1. **Adopt: Condenser-as-action pattern** from `openhands/agenthub/codeact_agent/codeact_agent.py:204-213`
   - What: Make condensation/compaction a first-class event in the agent event stream rather than a background side-effect. The agent returns a condensation action that gets logged, replayed, and audited.
   - Why: Enables debugging compaction issues via event replay; makes compaction timing visible to monitoring; allows composable condenser pipelines.
   - Effort: M
   - Target: Any harness with compaction/condensation — extend the action type system to include condensation events.

2. **Adopt: Pluggable condenser pipeline with composability** from `openhands/memory/condenser/impl/pipeline.py`
   - What: Define an abstract Condenser interface with `condensed_history(state) → View | Condensation`, implement 3-5 strategies (NoOp, SlidingWindow, LLMSummarizing), and compose them via Pipeline.
   - Why: Different repos/tasks benefit from different condensation strategies; pipeline allows combining "mask browser output" + "LLM summarize" without code changes.
   - Effort: L
   - Target: Agent memory systems that currently have hardcoded compaction logic.

3. **Adopt: Security-risk-as-parameter** from `openhands/agenthub/codeact_agent/tools/security_utils.py`
   - What: Add a required `security_risk: LOW|MEDIUM|HIGH` enum field to tool schemas for dangerous actions (shell, file write). The LLM self-labels; the harness can gate.
   - Why: Forces the LLM to reason about risk before executing. Even if the label is unreliable, it provides a signal for the harness's security analyzer and creates an audit trail.
   - Effort: S
   - Target: Any agent harness with shell/file tools that needs a lightweight security layer.

4. **Adopt: MCP stdio-over-HTTP proxy pattern** from `openhands/runtime/mcp/proxy/manager.py`
   - What: Run MCP stdio servers inside sandboxed containers (Docker/K8s) and proxy them via FastMCP to an HTTP endpoint reachable by the outer coordinator.
   - Why: Solves the distribution problem — stdio servers get sandboxing "for free" without requiring HTTP-native MCP implementations.
   - Effort: M
   - Target: Any Docker-based agent runtime that needs MCP tool integration.

5. **Adopt: Condensation request as agent-callable tool** from `openhands/agenthub/codeact_agent/tools/condensation_request.py`
   - What: Expose memory compression as a tool the LLM can proactively call, not just a harness-triggered side-effect.
   - Why: Gives the agent agency over its own memory management. The agent knows when it's about to do complex work and can preemptively free context space.
   - Effort: S
   - Target: Long-running agent harnesses where the agent has better signal about context pressure than the harness.

6. **Adopt: Delegate history scrubbing with ID-range isolation** from `openhands/controller/state/state_tracker.py:152-179`
   - What: When a parent agent delegates, filter the delegate's internal events from the parent's visible history. Use event ID ranges (start_id offset) for isolation on a shared event stream.
   - Why: Prevents the telephone problem (parent context bloated with delegate internal states). Uses a single event stream without separate channels.
   - Effort: M
   - Target: Multi-agent harness systems with parent-child delegation patterns.

7. **Adopt: Event-driven agent stepping (callback, not polling)** from `openhands/controller/agent_controller.py:454-512`
   - What: Drive agent stepping via event callbacks (`on_event` → `should_step()` → `step()`) rather than a polling while-loop.
   - Why: Decouples agent from runtime I/O timing. New event types trigger stepping without loop modifications. More extensible than polling.
   - Effort: M
   - Target: Agent harnesses currently using while-loop patterns that want to support multiple event sources.

## Evidence Index

**Verified: 49 paths (100%)**

Verified:
- ✓ openhands/controller/agent_controller.py — core agent loop and delegation
- ✓ openhands/core/loop.py — outer status poller
- ✓ openhands/controller/state/control_flags.py — iteration/budget limits
- ✓ openhands/controller/stuck.py — 5-scenario stuck detection
- ✓ openhands/core/main.py — entry point and MCP initialization
- ✓ openhands/agenthub/codeact_agent/codeact_agent.py — main agent with condenser integration
- ✓ openhands/runtime/base.py — polymorphic runtime and action dispatch
- ✓ openhands/events/stream.py — EventStream shared bus
- ✓ openhands/events/event_store.py — file-backed event persistence
- ✓ openhands/memory/memory.py — microagent knowledge injection
- ✓ openhands/memory/condenser/condenser.py — abstract condenser interface
- ✓ openhands/memory/condenser/impl/llm_summarizing_condenser.py — LLM-based summarization
- ✓ openhands/memory/condenser/impl/pipeline.py — composable condenser pipeline
- ✓ openhands/memory/view.py — View construction from events + condensation actions
- ✓ openhands/memory/conversation_memory.py — message construction and prompt caching
- ✓ openhands/controller/state/state.py — state persistence (excludes history)
- ✓ openhands/controller/state/state_tracker.py — history reconstruction and delegate filtering
- ✓ openhands/microagent/microagent.py — microagent types and keyword matching
- ✓ openhands/microagent/types.py — KNOWLEDGE/REPO_KNOWLEDGE/TASK enum
- ✓ openhands/events/tool.py — tool event types
- ✓ openhands/mcp/tool.py — MCP tool schema conversion
- ✓ openhands/agenthub/codeact_agent/tools/bash.py — bash tool schema
- ✓ openhands/agenthub/codeact_agent/tools/str_replace_editor.py — file edit tool
- ✓ openhands/agenthub/codeact_agent/tools/browser.py — BrowserGym integration
- ✓ openhands/agenthub/codeact_agent/tools/security_utils.py — security risk parameter
- ✓ openhands/agenthub/loc_agent/function_calling.py — LoC IPython dispatch
- ✓ openhands/agenthub/codeact_agent/function_calling.py — action dispatch
- ✓ openhands/agenthub/readonly_agent/function_calling.py — readonly dispatch
- ✓ openhands/mcp/utils.py — MCP tool loading and proxy setup
- ✓ openhands/mcp/client.py — MCP client with SSE/SHTTP/stdio
- ✓ openhands/events/action/mcp.py — MCPAction dataclass
- ✓ openhands/core/config/mcp_config.py — MCP configuration
- ✓ openhands/runtime/action_execution_server.py — sandbox execution server
- ✓ openhands/runtime/mcp/proxy/manager.py — FastMCP stdio proxy
- ✓ openhands/events/action/agent.py — AgentDelegateAction
- ✓ openhands/events/observation/agent.py — agent observations
- ✓ openhands/events/observation/delegate.py — delegate observation
- ✓ openhands/events/nested_event_store.py — HTTP-backed event view
- ✓ openhands/events/observation/task_tracking.py — task tracking
- ✓ openhands/critic/finish_critic.py — post-hoc evaluator
- ✓ openhands/security/options.py — security analyzer options
- ✓ openhands/core/config/condenser_config.py — condenser config types
- ✓ openhands/events/action/files.py — dual file edit modes
- ✓ openhands/storage/memory.py — in-memory file store
- ✓ openhands/agenthub/codeact_agent/tools/prompt.py — platform-adaptive prompts
- ✓ openhands/agenthub/codeact_agent/prompts/system_prompt.j2 — system prompt template
- ✓ openhands/llm/tool_names.py — tool name constants
- ✓ openhands/core/schema/action.py — ActionType enum (20 types)
- ✓ openhands/core/config/sandbox_config.py — sandbox configuration

Unverified: 0 paths (0%)

## Sources

- [all-hands-ai-openhands](https://github.com/all-hands-ai/openhands) — primary source
- [Harness Engineering KB](../wiki/_index.md) — cross-reference baseline
