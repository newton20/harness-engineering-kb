---
title: "Multi-Agent Reliability and Adversary Resistance"
type: wiki
tags:
  - multi-agent
  - reliability
  - adversary-resistance
  - credibility-scoring
  - evaluation
sources:
  - raw/arxiv-org-html-2505-24239v1.md
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md
  - raw/code-research-openclaw-openclaw-2026-04-15.md
  - raw/code-research-all-hands-ai-openhands-2026-04-15.md
  - raw/code-research-anomalyco-opencode-2026-04-15.md
  - raw/code-research-openclaw-openclaw-2026-04-14.md
  - raw/code-research-claude-code-2026-04-14.md
source_count: 8
status: draft
last_compiled: 2026-04-15
---

# Multi-Agent Reliability and Adversary Resistance

Multi-agent systems amplify the capabilities of individual LLM agents, but they also amplify failure modes. A single compromised, malfunctioning, or poorly performing agent in a multi-agent system can corrupt the collective output in ways that are difficult to detect and debug. This article covers the vulnerability landscape, defense mechanisms, and production reliability practices for multi-agent AI systems.

## The Core Vulnerability

Multi-agent systems are **highly sensitive to adversarial and low-performing agents**. A subset of compromised members can corrupt collective output, and this vulnerability is amplified by a fundamental property of LLMs: they are susceptible to persuasive inputs. An adversarial agent that generates confident, well-structured but incorrect output can sway other agents' reasoning more effectively than a human adversary might expect [Source: raw/arxiv-org-html-2505-24239v1.md].

This is not a hypothetical concern. In any system where multiple agents contribute to a shared conclusion -- whether through voting, aggregation, or sequential processing -- the integrity of the final output depends on the integrity of each contributor. Without explicit defenses, the system's reliability ceiling is set by its weakest or most compromised member.

## Credibility Scoring Framework

Research on adversary-resistant multi-agent collaboration (arxiv 2505.24239) proposes a **credibility scoring framework** that addresses the vulnerability systematically [Source: raw/arxiv-org-html-2505-24239v1.md]:

- **Game-theoretic formulation**: Collaborative query-answering is modeled as an iterative game where agents contribute answers over multiple rounds.
- **Credibility scores**: Each agent receives a credibility score that is used to weight its contributions during output aggregation. Higher-credibility agents have more influence on the final answer.
- **Gradual learning**: Scores are not assigned upfront but learned gradually from past contributions. An agent that consistently provides accurate, corroborated answers earns higher credibility over time.
- **Adversary-majority robustness**: The framework remains effective even when adversarial agents outnumber honest ones. Because credibility is earned through track record rather than majority vote, a majority of low-quality agents cannot simply outvote a minority of high-quality ones.

This approach treats agent reliability as an empirical property to be measured and tracked, rather than an assumption to be made at design time [Source: raw/arxiv-org-html-2505-24239v1.md].

## Source Reliability Defenses

Beyond agent-level credibility, deep research systems must also handle unreliable *information sources*. Several defensive strategies address this layer [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]:

- **Multi-source corroboration**: Require 2 or more independent sources before treating a factual claim as established. This is the simplest and most effective defense against individual source errors.
- **Source type weighting**: Primary sources (original research, official documentation, firsthand accounts) are weighted more heavily than secondary sources (summaries, news articles, blog posts). This prevents SEO-optimized but low-quality content from dominating results.
- **Contradiction detection**: When sources disagree, flag the conflict explicitly rather than silently choosing one version. This gives downstream consumers (whether human or agent) the information needed to make a judgment.
- **Dynamic credibility scoring**: Update source credibility scores based on how well a source's claims align with the emerging consensus across all sources. Sources that are frequently contradicted by multiple other sources have their influence reduced.

These defenses operate at the information level rather than the agent level, and are complementary to the agent-level credibility scoring described above [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md].

## Production Reliability Patterns

Anthropic's experience building and operating a multi-agent research system surfaced several production reliability patterns [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]:

### Stateful Error Compounding

Agents are stateful systems, and **errors compound across long runs**. A misinterpreted search result in step 3 can lead to an incorrect conclusion in step 10 that looks internally consistent but is factually wrong. Unlike stateless API calls where each request is independent, agent errors have downstream consequences that grow over time [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

### Checkpoint and Resume

When failures occur (tool errors, context overflow, model errors), the system should **resume from the last good checkpoint rather than restarting from scratch**. For a research task that has consumed thousands of tokens gathering and synthesizing information, losing that state and starting over is both wasteful and potentially produces different (not necessarily better) results [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

### Rainbow Deployments

Standard blue-green deployments are inappropriate for agent systems because they cut off running agents mid-task. **Rainbow deployments** allow existing agent sessions to complete on the current version while routing new sessions to the updated version. This avoids disrupting long-running research tasks that may take minutes to complete [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

### Synchronous vs. Asynchronous Execution

Synchronous execution creates bottlenecks -- the orchestrator waits for each subagent to complete before proceeding. Asynchronous execution improves throughput but adds coordination complexity: the orchestrator must handle out-of-order results, partial failures, and the question of when to proceed with incomplete information [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

## Evaluation Challenges

Evaluating multi-agent systems is fundamentally harder than evaluating single-agent or non-agentic systems [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]:

### Emergent Behavior

Multi-agent systems exhibit **emergent behaviors** that are not predictable from the behavior of individual agents. Small changes to the lead agent's prompt can unpredictably change subagent behavior through cascading effects on task delegation, query formulation, and result interpretation. This makes traditional A/B testing unreliable -- a change that improves average performance may introduce catastrophic failures on specific query types [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

### Judging Outcomes, Not Steps

The correct evaluation approach is to **judge outcomes, not individual steps**. An agent that takes an unconventional path to a correct answer is not failing; an agent that follows the expected steps but arrives at a wrong answer is. This means evaluation must focus on final output quality rather than process adherence [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

### Evaluation Methods

Anthropic employs multiple evaluation methods [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]:

- **LLM-as-judge**: Automated evaluation across multiple dimensions -- factual accuracy, citation accuracy, completeness, source quality, and tool efficiency. This scales but has known blind spots.
- **Human evaluation**: Catches what automation misses. Humans are better at distinguishing authoritative sources from SEO farms, identifying subtle factual errors, and assessing whether an answer actually addresses the user's intent.
- **Start small, scale later**: Begin with small evaluation samples (approximately 20 queries) to establish baseline quality and identify obvious failure modes. Scale to larger eval sets only after the system is stable enough to benefit from statistical significance.

## OpenClaw: Production Multi-Agent Reliability Machinery

OpenClaw's multi-agent system provides the most concrete production implementation of multi-agent reliability patterns we've documented. Several mechanisms address the failure modes described above: [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Frozen result capture (write-ahead log pattern).** When a subagent run completes, the registry immediately captures the child's raw last assistant reply into a `frozenResultText` field before any announcement delivery attempt. If the announce fails and needs to be retried (possibly after a process restart), the actual child output is already durably persisted in `subagentRuns.json` without needing to re-query the gateway. This is a mini write-ahead log -- a concrete implementation of the checkpoint pattern described above. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Cooperative yield (sessions_yield).** A tool the model can call to voluntarily terminate its own LLM turn while waiting for child agents. The gateway re-wakes the orchestrator with aggregated child results when descendants settle via `wakeSubagentRunAfterDescendants()`. The runner strips yield artifacts from the session transcript so it appears clean. This directly solves the "orchestrator burns context polling for children" anti-pattern, which is a specific form of the synchronous vs. asynchronous execution problem. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Push-based announcements with batching.** Rather than requiring orchestrators to poll for child results, children auto-announce their results upstream via gateway steer message injection. An announce queue with configurable drop policy (`"summarize"`) batches concurrent completions into a single delivery, preventing context floods during fan-out bursts. The queue has a cap of 20 items per requester. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Orphan recovery after restart.** Post-SIGUSR1 reload, the system scans the subagent registry for runs with `abortedLastRun: true` and re-dispatches synthetic resume messages with exponential backoff (2s, 4s, 8s up to 5-minute gateway delay). This is a concrete implementation of the checkpoint-and-resume pattern for multi-agent coordination. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Depth-bounded hierarchy.** Three roles are assigned at spawn time: `main` (depth 0), `orchestrator` (depth 1 to maxDepth-1), and `leaf` (at maxDepth). Default `maxSpawnDepth = 1` means the system ships as a flat orchestrator-worker topology. Leaf agents have `canSpawn: false`, preventing spawning cascades. True multi-level orchestration requires explicit config opt-in. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

**Telephone mitigation.** The system uses frozen result capture + direct re-injection into the requester's context (not multi-hop LLM paraphrasing). For intermediate hops (sub-subagent announces), the instruction asks the orchestrator to summarize in its own words -- but when the requester is the main agent, the instruction asks for user-facing conversion. The `dedupeLatestChildCompletionRows()` function prevents duplicate completion events. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

## OpenHands: Delegate History Scrubbing

OpenHands solves the parent-context-inflation problem in hierarchical delegation by filtering the parent's history view of any events that occurred between an `AgentDelegateAction` and its corresponding `AgentDelegateObservation`. The parent sees only the bookend events — the dispatch and the result — and never sees the child's intermediate actions, observations, or errors. This prevents child work from ballooning the parent's context window, but the signal loss is real: the child's reasoning, intermediate tool calls, and recovery attempts are invisible to the parent. A TODO comment in the codebase explicitly notes that the current scrubbing should eventually be replaced with AI-generated summaries that preserve meaningful signal without full verbatim replay. Additionally, the iteration counter is global across the delegation boundary: a child's steps count against the parent's 500-step cap, meaning there is no per-agent budget isolation. Deep delegation trees can exhaust the iteration budget faster than flat architectures. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

## OpenCode: Mode-Typed Agent Registry and Permission Inheritance

OpenCode classifies all agents into three modes at registration time: `primary` (top-level session), `subagent` (dispatched worker), or `all` (both contexts). This mode typing controls which tools are available to an agent and how its permissions are scoped. Child sessions inherit the parent's permission set but apply scoped denials by default: `todowrite` and recursive task dispatch are denied for subagent sessions, preventing uncontrolled nesting. Subagent sessions are resumable via a stable `task_id`, enabling multi-turn interactions where the orchestrator can send follow-up messages to a running child rather than dispatching a fresh session each time. The permission inheritance model is explicit and auditable — each child session's effective permissions can be inspected as a diff from the parent's. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## OpenHands Microagents: Prompt Augmentation, Not Autonomous Agents

Despite their name, OpenHands microagents are not autonomous sub-agents — they are prompt augmentation mechanisms. `KnowledgeMicroagent`, `RepoMicroagent`, and `TaskMicroagent` each operate by injecting additional content into the primary agent's system prompt when triggered: `KnowledgeMicroagent` fires on keyword match in user messages, `RepoMicroagent` reads `.openhands/microagents/repo.md` from the repository root and injects it at session start, and `TaskMicroagent` injects task-specific instructions. None of them issue actions, observe results, or maintain their own state — they are stateless text injectors that extend the primary agent's context. The naming reflects how the term "agent" is overloaded in practice: any configurable system component gets the label, whether or not it acts autonomously. Harness engineers should read "microagent" as "context plugin" in the OpenHands documentation. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

## OpenClaw: Additional Multi-Agent Patterns (2026-04-14 Research)

### Announce Queue Batching with Summarize Drop Policy

When multiple child agents complete concurrently during a fan-out burst, OpenClaw's announce queue batches their completion notifications into a single delivery to the requester rather than flooding the orchestrator's context with sequential announcements. The queue has a configurable `dropPolicy: "summarize"` and a hard cap of 20 items per requester. When the cap is reached, excess completions are summarized rather than dropped or queued indefinitely. This directly prevents the context-flood anti-pattern that arises in large fan-out topologies where dozens of leaf agents complete in a short window. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

### Depth-Bounded Role Assignment

OpenClaw assigns one of three explicit roles to each agent at spawn time based on its depth in the hierarchy: `main` (depth 0), `orchestrator` (depth 1 to `maxDepth-1`), and `leaf` (at `maxDepth`). Leaf agents have `canSpawn: false` and cannot create children. The default `maxSpawnDepth = 1` means the system ships with a flat topology -- orchestrator plus leaves -- and true multi-level orchestration must be explicitly opted into by raising the depth limit. This default-flat design prevents accidental spawning cascades and makes the common case (shallow orchestration) safe without configuration. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

### Fire-and-Forget Spawning

OpenClaw's `spawn` operation returns `{status: "accepted"}` immediately with no blocking on child initialization or execution. The caller does not wait for the child to start, complete, or return a result. Result delivery is entirely push-based: when a child completes, it auto-announces its result upstream via the gateway's steer message injection path. The spawning agent can immediately continue its own work after receiving the `"accepted"` status. This fire-and-forget model maximizes parallelism in fan-out scenarios and prevents the orchestrator's context from accumulating polling overhead while waiting for children. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## Claude Code: "Never Delegate Understanding" Principle (2026-04-14 Research)

Claude Code's coordinator (orchestrator) prompt contains an explicit prohibition on delegating synthesis tasks to subagents. The principle, stated directly in the prompt, is "Never Delegate Understanding": the coordinator must itself synthesize, analyze, and reason about results -- these cognitive steps cannot be handed off to a subagent that does not share the coordinator's full context. The prompt further mandates that instructions to subagents must be self-contained, including specific file paths and line numbers rather than vague references that require the subagent to infer context. This addresses the game-of-telephone anti-pattern where each relay through an intermediate agent risks information loss, summarization artifacts, and context-dependent references that become ambiguous when the receiving agent lacks the original context. [Source: raw/code-research-claude-code-2026-04-14.md]

## Combining Defenses

No single defense mechanism is sufficient. A robust multi-agent system combines multiple layers:

1. **Agent-level credibility scoring** prevents compromised or low-quality agents from dominating outputs [Source: raw/arxiv-org-html-2505-24239v1.md].
2. **Source-level reliability defenses** prevent bad information from corrupting good agents [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md].
3. **Production infrastructure** (checkpoints, rainbow deployments, monitoring) prevents operational failures from destroying research state [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].
4. **Multi-dimensional evaluation** catches failures that any single evaluation method would miss [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].
5. **History scrubbing with AI summarization** (when available) prevents child context from inflating parent windows while preserving signal [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md].
6. **Mode-typed agent registries with scoped permission inheritance** make each agent's effective authority explicit and auditable [Source: raw/code-research-anomalyco-opencode-2026-04-15.md].

The defense-in-depth approach reflects a fundamental reality: multi-agent systems have a larger attack and failure surface than single-agent systems, and reliability must be engineered at every layer.

## Sources

- [raw/arxiv-org-html-2505-24239v1.md](../raw/arxiv-org-html-2505-24239v1.md) — Research paper on credibility scoring for adversary-resistant multi-agent collaboration, modeling collaboration as an iterative game
- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) — Anthropic's engineering blog on production reliability patterns, evaluation methods, and operational lessons from their multi-agent research system
- [raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md](../raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md) — Source reliability defenses including multi-source corroboration, source weighting, and contradiction detection
- [raw/code-research-openclaw-openclaw-2026-04-15.md](../raw/code-research-openclaw-openclaw-2026-04-15.md) — Code research, Apr 2026. OpenClaw's production multi-agent reliability: frozen result capture, sessions_yield cooperative abort, push-based announce batching, orphan recovery, depth-bounded hierarchy.
- [raw/code-research-all-hands-ai-openhands-2026-04-15.md](../raw/code-research-all-hands-ai-openhands-2026-04-15.md) — Code research, Apr 2026. Delegate history scrubbing (bookend-only parent view), global iteration counter across delegation boundary, microagents as prompt augmentation mechanisms (not autonomous agents).
- [raw/code-research-anomalyco-opencode-2026-04-15.md](../raw/code-research-anomalyco-opencode-2026-04-15.md) — Code research, Apr 2026. Mode-typed agent registry (primary/subagent/all), permission inheritance with scoped denial, resumable subagent sessions via task_id.
- [raw/code-research-openclaw-openclaw-2026-04-14.md](../raw/code-research-openclaw-openclaw-2026-04-14.md) — Code research, Apr 2026. Announce queue batching with summarize drop policy (cap 20); depth-bounded role assignment (main/orchestrator/leaf); fire-and-forget spawning with push-based result delivery.
- [raw/code-research-claude-code-2026-04-14.md](../raw/code-research-claude-code-2026-04-14.md) — Code research, Apr 2026. "Never Delegate Understanding" principle in coordinator prompt; mandate for self-contained subagent instructions with specific file paths and line numbers.

## Related

- [Deep Research Agents](deep-research-agents.md) — Multi-agent research architecture where these reliability patterns are applied
- [Auto Mode and Safety](auto-mode-and-safety.md) — Safety classifiers and permission systems that complement adversary resistance
- [Practical Best Practices](practical-best-practices.md) — Evaluation approaches and production deployment guidance

## Open Questions

- **Credibility cold start**: How should new agents be treated before they have a track record? Optimistic initialization (assume good) risks early corruption; pessimistic initialization (assume bad) delays productive contribution.
- **Adversary adaptation**: The credibility scoring framework assumes adversaries do not adapt their strategy to game the scoring system. How robust is the framework to adversaries that strategically build credibility before deploying attacks?
- **Cross-system transfer**: Can credibility scores earned in one task domain transfer to another, or must they be re-earned for each new context?
- **Evaluation cost scaling**: Human evaluation is the gold standard but does not scale. At what point does automated evaluation (LLM-as-judge) become reliable enough to reduce human evaluation to spot-checking?
- **Emergent failure detection**: Is there a principled way to detect emergent failures in multi-agent systems before they reach production, or is this inherently a problem that can only be caught through extensive runtime monitoring?
