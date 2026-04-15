---
title: "Agentic Design Patterns"
type: wiki
tags:
  - design-patterns
  - react
  - reflection
  - planning
  - tool-use
  - multi-agent
  - agentic-rag
sources:
  - raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md
  - raw/arxiv-org-html-2501-09136v4.md
  - raw/code-research-claude-code.md
  - raw/code-research-karpathy-autoresearch.md
  - raw/code-research-all-hands-ai-openhands.md
  - raw/code-research-anomalyco-opencode.md
  - raw/code-research-666ghj-mirofish.md
source_count: 7
status: draft
last_compiled: 2026-04-15
---

# Agentic Design Patterns

Agent failures are more often **architectural** than prompting failures. An agent that loops endlessly lacks a stopping condition. An agent that calls the wrong tools lacks a clear contract. Patterns provide repeatable architectural templates for the agent loop -- they are the structural decisions that determine whether an agent works reliably or fails unpredictably [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

This article catalogs the core patterns used in agentic AI systems and provides guidance on when to apply each one.

## The Selection Principle

The most important rule for pattern selection: **start with the simplest pattern that could work**. Premature complexity means more model calls, higher latency, more failure surfaces, and more coordination bugs. Every additional layer of sophistication must be justified by a specific bottleneck or failure mode in the simpler approach [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

"Start with the problem, not the pattern." If a single LLM call with good prompting solves the task, no agent loop is needed. If a simple ReAct loop works, do not introduce multi-agent coordination. Escalate complexity only when you have evidence that the current approach fails [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

## Core Patterns

### ReAct (Reasoning + Acting)

The ReAct pattern implements a **Thought -> Action -> Observation** loop. The agent reasons about what to do next (Thought), executes a tool call or action (Action), and observes the result (Observation). This cycle repeats until the agent decides it has enough information to produce a final answer [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

**When to use**: Complex, unpredictable tasks where the agent cannot plan all steps upfront. ReAct is the default pattern for most agent applications because it externalizes reasoning, making failures visible and debuggable.

**Trade-off**: Each iteration requires an additional model call. For tasks that could be solved in one or two steps, the overhead of the loop is wasteful. The pattern also requires careful stopping conditions to prevent infinite loops [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

### Reflection

The Reflection pattern implements a **Generation -> Critique -> Refinement** cycle. The agent (or a first model) generates an initial output, a critic evaluates it against quality criteria, and the generator revises based on the critique [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

**Key design decisions**:
- The critic should be **independent from the generator** -- ideally a separate prompt, or even a separate model. Self-critique using the same prompt and context tends to rubber-stamp the original output.
- **Explicit iteration bounds are required**. Without a maximum number of refinement rounds, the system can oscillate between minor changes indefinitely.
- The pattern is worth the additional cost when **quality matters more than speed** -- code review, report writing, and any task where a second look catches meaningful errors.

[Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]

### Tool Use

The Tool Use pattern gives the agent access to a **fixed catalog of tools with strict schemas**. The agent decides which tool to call, constructs the appropriate input, and processes the tool's output [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

**Design considerations**:
- **Handle failures explicitly**: Tools can time out, return errors, or produce unexpected output. The agent needs retry logic and graceful degradation paths.
- **Selection accuracy degrades with catalog size**: The more tools available, the harder it is for the model to select the right one. Keep catalogs focused and well-described.
- **Security surface**: Every tool is a potential attack vector. Sandboxed execution environments and human approval gates for destructive operations are not optional in production.

[Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]

### Planning

The Planning pattern separates strategy from execution. Two variants exist [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]:

- **Plan-and-Execute**: The agent creates a complete plan upfront, then executes each step sequentially. This works when the problem is well-understood and unlikely to require mid-course correction.
- **Adaptive Planning**: The agent creates a partial plan, executes the first steps, observes results, and re-plans based on what it learned. This handles uncertainty better but requires more model calls.

**When to use**: Tasks with high coordination complexity -- multi-step workflows where the order of operations matters and individual steps depend on prior results. Planning adds overhead, so it is not justified for tasks that can be solved reactively [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

### Multi-Agent

The Multi-Agent pattern uses **specialized agents coordinated by an orchestrator**. Each agent has a focused role (e.g., search agent, code agent, review agent), and the coordinator routes tasks and aggregates results [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

**Topologies**:
- **Sequential**: Agents process in a fixed order (pipeline). Simple but creates bottlenecks.
- **Concurrent**: Agents work in parallel on independent subtasks. Higher throughput but requires result aggregation.
- **Debate**: Agents argue different positions and a judge selects or synthesizes the best answer. Useful for tasks where correctness is critical and multiple perspectives help.

**The cardinal rule**: "Start with a single agent, move to multi-agent only when a specific bottleneck emerges." Multi-agent coordination introduces communication overhead, error propagation across agents, and debugging complexity that is not justified unless a single agent genuinely cannot handle the task [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

## Agentic RAG

Traditional Retrieval-Augmented Generation (RAG) follows a static pipeline: retrieve relevant documents, then generate an answer from them in a single pass. **Agentic RAG** embeds autonomous agents into the RAG pipeline, applying the design patterns above -- reflection, planning, tool use, and multi-agent collaboration -- to make retrieval dynamic and iterative [Source: raw/arxiv-org-html-2501-09136v4.md].

The key differences from traditional RAG:

- **Dynamic retrieval strategies**: Instead of a single retrieval step, agents decide when to retrieve, what query to use, and whether the retrieved results are sufficient or require follow-up searches.
- **Iterative refinement**: Agents can re-retrieve with modified queries based on what they learned from initial results, implementing a search-reason loop within the RAG pipeline.
- **Adaptive workflows**: The retrieval and generation process adapts to the specific query rather than following a fixed pipeline.

Agentic RAG represents the evolution from static, one-shot retrieval to adaptive, multi-step retrieval that can handle complex information needs [Source: raw/arxiv-org-html-2501-09136v4.md].

## Production Evaluation

Each pattern requires **pattern-specific evaluation criteria** [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]:

- **ReAct**: Are tool calls aligned with the reasoning in the Thought step? Does the agent converge or loop?
- **Reflection**: Are outputs actually improving across iterations? Is the critic catching real issues or generating noise?
- **Multi-Agent**: Is routing accurate? Are the right tasks going to the right agents?
- **Tool Use**: Is the agent selecting the correct tool? Are tool inputs well-formed?

General evaluation principles:

- **Build failure mode tests**: For each pattern, enumerate known failure modes (infinite loops, wrong tool selection, rubber-stamp critique) and write tests that specifically check for them.
- **Treat observability as a requirement**: If you cannot see why the agent made a decision, you cannot debug it when it fails. Log reasoning traces, tool calls, and intermediate outputs.

[Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]

## Pattern Combinations

In practice, production agents rarely use a single pattern in isolation. Common combinations include:

- **ReAct + Tool Use**: The standard agent loop -- reason about what tool to use, call it, observe results. This is so common it is often treated as a single pattern.
- **Planning + Multi-Agent**: The planning agent decomposes a task and delegates subtasks to specialized agents (the orchestrator-worker pattern used by Anthropic's research system).
- **Reflection + Tool Use**: Generate code with tools, then critique and refine it. Common in coding agents.
- **ReAct + Reflection**: The agent acts, then reflects on whether its approach is working before continuing. Prevents getting stuck in unproductive loops.

The art of agent engineering is selecting the minimal combination of patterns that addresses the specific failure modes of the task at hand [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md].

## Imperative State Machine Pattern

Claude Code's source reveals a concrete production implementation of the agent loop: an imperative `while(true)` async generator in `query.ts` with labeled `continue` transitions and 10+ distinct termination reasons. This is a state machine, not a recursive call graph -- the code controls the loop, not the model. Continuation is signaled by the presence of `tool_use` blocks in the response (not by `stop_reason`, which is documented as unreliable). Each iteration follows a fixed sequence: pre-call compaction, API call, stream tool_use detection, tool execution, post-tool attachments, tools refresh, state update, continue. This imperative pattern contrasts with reactive/event-driven patterns (like OpenClaw's event loop) and recursive patterns (where agent calls chain through function calls). [Source: raw/code-research-claude-code.md]

## Prose-as-Control-Flow Pattern

Karpathy's autoresearch demonstrates a pattern at the opposite extreme from Claude Code's imperative state machine: the system prompt IS the agent loop. The entire outer experiment loop is a numbered natural language procedure in a 115-line markdown file. There is no Python orchestrator, no state machine code, no tool API. The LLM reads the prose steps and executes them as its own decision procedure -- it is simultaneously the scheduler, state machine, and decision engine. The code (train.py) is just the worker. This prose-as-control-flow pattern works when the loop structure is fixed and simple, and the agent's creative freedom should be confined to a narrow domain (in this case, what mutation to make to the training code). [Source: raw/code-research-karpathy-autoresearch.md]

## Forked Agent Pattern

Claude Code's forked agent emerges from source analysis as a fundamental building block for multi-agent work. A fork child inherits the parent's full message history and byte-identical system prompt, sharing the parent's prompt cache prefix so the API call is nearly free for short tasks. This pattern is used pervasively: autocompact forks a summarizer, extractMemories forks a background memory writer, SessionMemory forks a note-taker, and sub-agents can fork with full parent context for "second opinion" tasks. The forked agent pattern sits between a fresh subagent (zero parent context, clean but expensive to reconstruct) and a full context handoff (complete state transfer, which is what fork provides nearly for free via cache sharing). [Source: raw/code-research-claude-code.md]

## Event-Driven vs. Polling Agent Loops

Three distinct loop architectures appear across production agent systems, each with different extension and correctness properties. OpenHands uses a callback-driven pattern: the outer status loop is passive (`while state not in end_states: await asyncio.sleep(1)`), and all actual stepping fires through `on_event → should_step() → agent.step()` callbacks registered on the EventStream. New event types can trigger agent steps without modifying the loop code at all — the architecture is open-closed by design. Claude Code and OpenCode both use imperative `while(true)` loops, but differ in their authority model: Claude Code's loop reads from in-memory state, while OpenCode re-reads from SQLite on each iteration, making the database the canonical authority and giving the loop natural crash-recovery semantics. The callback-driven pattern excels at extensibility (new event types compose cleanly); the DB-authoritative poll loop excels at durability (each iteration starts from a verified-good state). The imperative in-memory loop (Claude Code) optimizes for latency and simplicity at the cost of requiring explicit compaction and state management to remain coherent. [Source: raw/code-research-all-hands-ai-openhands.md] [Source: raw/code-research-anomalyco-opencode.md]

## Deterministic FSM with LLM Judgment at Branch Points

Karpathy's autoresearch defines its experiment loop as a numbered 9-step fixed protocol in program.md. The sequence — read branch, write mutation, run training, evaluate metric, commit or revert — is deterministic: every iteration follows the same numbered steps in the same order. LLM judgment is invoked only at specific branch points, most notably crash recovery: when a training run fails, the agent decides whether to diagnose and retry or to revert to the prior commit. This hybrid is more reliable than pure free-choice ReAct because the vast majority of the loop is not subject to LLM variance. The LLM's creative freedom is deliberately confined to a single degree of freedom (what mutation to make) and a single recovery decision (how to handle a crash). [Source: raw/code-research-karpathy-autoresearch.md]

## Round-as-Clock: Time-Driven Environment Tick

MiroFish's simulation loop is a deterministic `for round in range(total_rounds)` where each increment maps to a fixed number of simulated minutes. Agents are not selected on a fixed schedule — they are selected stochastically each round based on per-agent `activity_level` and `active_hours` parameters. The environment advances time; agents respond to it. This is not an agent-driven loop — agents do not decide when to act. It is a time-driven environment tick that samples agent activity probabilistically. The pattern enables multi-agent simulations where agent behavior is temporally heterogeneous (some agents are active at night, some at day) without requiring agent-side scheduling logic. [Source: raw/code-research-666ghj-mirofish.md]

## Condenser-as-Action Pattern

Both OpenHands and OpenCode treat context compaction as a visible, auditable architectural event rather than a hidden side-effect triggered by token count. In OpenHands, the agent returns a `CondensationAction` from `agent.step()` — this action is logged to the EventStream, persisted in the event store, and replayed on session restore, giving condensation the same first-class status as any tool call or observation. The pluggable condenser system has 9 composable implementations (LLM summarizer, no-op, amnesiac, recent-history, llm-and-no-op, summarize-then-forget, and combinations). OpenCode models compaction as a named "compaction" agent with its own model selection and no tools, making it an explicit agent dispatch rather than an inline operation. Both approaches make the context management decision inspectable in logs and auditable in post-mortems — a stark contrast to systems where compaction fires silently based on token thresholds with no trace in the event record. [Source: raw/code-research-all-hands-ai-openhands.md] [Source: raw/code-research-anomalyco-opencode.md]

## Sources

- [raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md](../raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md) — Comprehensive roadmap covering ReAct, Reflection, Tool Use, Planning, and Multi-Agent patterns with selection criteria and evaluation guidance
- [raw/arxiv-org-html-2501-09136v4.md](../raw/arxiv-org-html-2501-09136v4.md) — Survey paper on Agentic RAG covering the integration of autonomous agents into retrieval-augmented generation pipelines
- [raw/code-research-claude-code.md](../raw/code-research-claude-code.md) — Code research, Apr 2026. Imperative while(true) state machine pattern, forked agent pattern as fundamental building block, tool_use presence as continuation signal.
- [raw/code-research-karpathy-autoresearch.md](../raw/code-research-karpathy-autoresearch.md) — Code research, Apr 2026. Prose-as-control-flow pattern where the system prompt IS the agent loop; deterministic FSM with LLM judgment only at crash-recovery branch points; control inversion with LLM as scheduler.
- [raw/code-research-all-hands-ai-openhands.md](../raw/code-research-all-hands-ai-openhands.md) — Code research, Apr 2026. Callback-driven event loop as alternative to while(true); CondensationAction as first-class event; 9-implementation pluggable condenser system.
- [raw/code-research-anomalyco-opencode.md](../raw/code-research-anomalyco-opencode.md) — Code research, Apr 2026. DB-authoritative while(true) loop; compaction modeled as a named agent dispatch with its own model selection.
- [raw/code-research-666ghj-mirofish.md](../raw/code-research-666ghj-mirofish.md) — Code research, Apr 2026. Round-as-clock time-driven environment tick with stochastic per-agent activity sampling.

## Related

- [What Is Harness Engineering](what-is-harness-engineering.md) — Patterns as a framework for understanding agent harness design
- [Tool Design Patterns](tool-design-patterns.md) — Detailed treatment of the tool use pattern including schema design and catalog management
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) — The reflection pattern applied to agent self-improvement
- [Deep Research Agents](deep-research-agents.md) — Multi-agent pattern applied to research with orchestrator-worker architecture
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) — Planning pattern in the context of persistent, multi-session agents
- [Agent Memory and Context Management](agent-memory-and-context-management.md) — Memory systems that underpin the forked agent and compaction patterns
- [Practical Best Practices](practical-best-practices.md) — Error-as-context, prompt cache-first architecture, minimal-signal extraction as actionable practices
- [Multi-Agent Reliability](multi-agent-reliability.md) — Coordination patterns for multi-agent topologies

## Open Questions

- **Pattern selection automation**: Can an agent or meta-agent automatically select the right pattern for a given task, or does this always require human architectural judgment?
- **Reflection independence**: The recommendation is that critics be independent from generators. How much independence is needed -- is a different prompt sufficient, or does a genuinely different model provide meaningfully better critique?
- **Agentic RAG boundaries**: Where does Agentic RAG end and a full research agent begin? Is there a meaningful architectural distinction, or is it a spectrum?
- **Scaling limits of multi-agent**: At what number of agents does coordination overhead dominate, and does this limit depend primarily on the orchestrator's capability or on the communication protocol?
- **Pattern-specific benchmarks**: Current agent benchmarks evaluate end-to-end performance. Are there benchmarks that specifically measure whether an agent is using the right pattern for a given task type?
