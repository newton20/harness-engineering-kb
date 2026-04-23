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
  - raw/walden_yan-2047054401341370639.md
source_count: 9
status: draft
last_compiled: 2026-04-23
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

## Walden Yan (Cognition): "Writes Stay Single-Threaded"

Walden Yan's April 2026 "Multi-Agents: What's Actually Working" is a deliberate retrospective on Cognition's 2025 post "Don't Build Multi-Agents." The original argument: parallel agents make implicit choices about style, edge cases, and code patterns that conflict with each other, leading to fragile products. That observation still holds for parallel-*writer* swarms — "most of the sexy ideas in that space still don't see meaningful adoption." But Cognition has identified a narrower class of patterns that do work. **The through-line: multi-agent systems work best today when writes stay single-threaded and additional agents contribute *intelligence* rather than actions.** [Source: raw/walden_yan-2047054401341370639.md]

### Pattern 1: The Clean-Context Code-Review Loop

Devin Review (a separate review agent invoked on every Devin PR) catches an average of **2 bugs per PR, of which ~58% are severe** (logic errors, missing edge cases, security vulnerabilities). The system iterates through multiple review cycles, finding new bugs each cycle. [Source: raw/walden_yan-2047054401341370639.md]

The counterintuitive finding: **the pattern works best when the coding and review agents do not share any context beforehand.** Yan's justification is part philosophical, part technical:

- Agents are not egos; they're systems that perform based on their context. Two instances of the same model don't self-bias the way a single human doing both tasks would.
- The clean-context review agent is forced to reason backward from implementation without the spec, so it can openly question assumptions the original agent inherited from instructions (e.g., a user specifying an insecure pattern).
- **The math of attention matters most.** Context Rot is well-documented — models make less intelligent decisions at longer context lengths. When the coding agent has been working for hours (reading the repo, running commands, fixing errors), its context is huge. The dedicated review agent skips that extraneous context, looks only at the diff, and re-discovers what it needs. Shorter context → more attention capacity → better detection of nuanced issues. [Source: raw/walden_yan-2047054401341370639.md]

The final load-bearing piece: the communication bridge back from the review agent to the coding agent must let the coder "properly use its broader context of user instructions, decisions, etc. to filter the bugs that come back." Without this, the system loops, disobeys the user, and does out-of-scope work. [Source: raw/walden_yan-2047054401341370639.md]

**Takeaway:** Clean context + generator-verifier loop = notable capability improvement. But clear synthesis back to the overall context is what makes the experience cohesive.

### Pattern 2: "Smart Friend" — Cross-Model Delegation

With the return of large models (Opus-class) and the arrival of the Mythos class, frontier intelligence will soon be too expensive and slow for most day-to-day tasks. Windsurf's SWE-1.5 (950 tok/sec sub-frontier model) was paired with Sonnet 4.5 for planning via a "smart friend" tool the primary model could invoke. [Source: raw/walden_yan-2047054401341370639.md]

The architecture inverts the usual orchestrator-worker pattern: rather than a smart primary delegating to smaller subagents, the **smaller primary decides when to consult the smarter model**. Two engineering problems emerge:

1. **How does the weaker model know it's at its limits?** This is fundamentally a calibration problem for a model that isn't the smartest. Solutions: always make at least one call per task; prompt-tune or train the primary to be more calibrated; prescribe domain-specific trigger rules (always invoke smart friend for merge conflicts).
2. **What context should the primary share?** Yan reports the 80/20 answer for today's models: share a fork of the full primary context with the smart model; encourage broad questions ("what should I do?") and let the smart model decide what's interesting.

The counter-direction also matters: the smart model needs to know not to make up theories when the primary hasn't shown it a relevant file — it should instead specifically instruct the primary to investigate that file and ask again. "Over-scoped" smart friends (suggesting important guidance the primary didn't ask for) produced more interesting interactions. [Source: raw/walden_yan-2047054401341370639.md]

### What Actually Happened with SWE-1.5

Yan is explicit that SWE-1.5 was **not good enough** as the primary for this setup. The gap from Sonnet 4.5 was too wide in exactly the places that mattered — knowing when to escalate, knowing what to ask. SWE-1.6 (Opus-4.5-level on SWE-bench) closes enough of that gap that the pattern starts to pay off, but "it's still not where we want it. We're reasonably confident this is a training problem, and future SWE models will be trained with this back-and-forth in mind." [Source: raw/walden_yan-2047054401341370639.md]

Where the pattern does work well: **across frontier models.** Running Claude and GPT together in this setup produced real gains. The interesting shift: cross-frontier communication is less about a weaker model knowing when to ask a stronger one, and more about routing to whichever model is best at the specific sub-task. "The delegation logic becomes a capability router rather than a difficulty escalator." [Source: raw/walden_yan-2047054401341370639.md]

### Higher-Level Delegation: Map-Reduce-and-Manage

Devin supports a manager Devin breaking a larger task into pieces, spawning child Devins, and coordinating their progress through an internal MCP. Getting this coherent took more context engineering than Cognition expected:

- Managers trained on small-scoped delegation default to being overly prescriptive, which backfires when the manager lacks deep codebase context.
- Agents assume they share state with their children when they don't.
- Cross-agent communication (sub-agent writing messages back to its manager to pass to sibling agents) doesn't happen by default because models haven't been trained in environments where it needed to.

Cognition's take: **"Unstructured swarms — arbitrary networks of agents negotiating with each other — is mostly a distraction. The practical shape is map-reduce-and-manage: a manager splits work, children execute, the manager synthesizes and reports back."** [Source: raw/walden_yan-2047054401341370639.md]

### The Unifying Rule

"The open problems are all communication problems": how a weaker model learns when to escalate, how a child surfaces a discovery that should change its siblings' work, how to transfer context between agents without drowning the receiver. You can get decently far with prompting, but Cognition expects next-generation models (including the ones they train themselves) to close these gaps in training. [Source: raw/walden_yan-2047054401341370639.md]

The Walden Yan rule for 2026: **multi-agent works when writes are single-threaded and the additional agents contribute intelligence, not actions.** A clean-context reviewer catches bugs the coder can't see. A frontier-level smart friend catches subtleties a weaker primary misses. A manager coordinates scope across child agents without fragmenting decisions.

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
- [raw/walden_yan-2047054401341370639.md](../raw/walden_yan-2047054401341370639.md) — Walden Yan (Cognition), Apr 22, 2026. "Multi-Agents: What's Actually Working" — 10-month retrospective on Cognition's 2025 "Don't Build Multi-Agents" post. Devin Review catches 2 bugs/PR, ~58% severe. Clean-context generator-verifier loop. Smart Friend cross-model delegation. Map-reduce-and-manage as practical higher-level pattern. "Writes stay single-threaded" as unifying rule.

## Related

- [Deep Research Agents](deep-research-agents.md) — Multi-agent research architecture where these reliability patterns are applied
- [Auto Mode and Safety](auto-mode-and-safety.md) — Safety classifiers and permission systems that complement adversary resistance
- [Practical Best Practices](practical-best-practices.md) — Evaluation approaches and production deployment guidance
- [Thin Harness, Fat Skills](thin-harness-fat-skills.md) — Cognition's Devin Review as an example of the generator-verifier pattern
- [Self-Evolving Agents and Skillify](self-evolving-agents.md) — LangChain's trace-analyzer skill as a harness-level analog of Devin Review
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) — Map-reduce-and-manage as the practical shape for multi-session delegation

## Open Questions

- **Credibility cold start**: How should new agents be treated before they have a track record? Optimistic initialization (assume good) risks early corruption; pessimistic initialization (assume bad) delays productive contribution.
- **Adversary adaptation**: The credibility scoring framework assumes adversaries do not adapt their strategy to game the scoring system. How robust is the framework to adversaries that strategically build credibility before deploying attacks?
- **Cross-system transfer**: Can credibility scores earned in one task domain transfer to another, or must they be re-earned for each new context?
- **Evaluation cost scaling**: Human evaluation is the gold standard but does not scale. At what point does automated evaluation (LLM-as-judge) become reliable enough to reduce human evaluation to spot-checking?
- **Emergent failure detection**: Is there a principled way to detect emergent failures in multi-agent systems before they reach production, or is this inherently a problem that can only be caught through extensive runtime monitoring?
