---
title: "Self-Evolving Agents and Skillify"
type: wiki
tags:
  - self-evolution
  - autogenesis
  - agent-protocol
  - skillify
  - resolvers
  - lifecycle
  - version-lineage
  - rollback
  - skill-refinement
sources:
  - raw/arxiv-org-pdf-2604-15034.md
  - raw/garrytan-2046876981711769720.md
  - raw/garrytan-2044479509874020852.md
  - raw/akshay_pachaar-2041146899319971922.md
  - raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md
source_count: 5
status: draft
last_compiled: 2026-04-23
---

# Self-Evolving Agents and Skillify

Self-evolving agents are systems that improve their own prompts, tools, memory, and workflows based on observed performance — without a human manually patching each failure. Two distinct but converging strands appeared in April 2026: the **Autogenesis Protocol** (a formal two-layer protocol for resource lifecycle + evolution operators, arXiv:2604.15034) and Garry Tan's **skillify** operational practice (every agent failure becomes a markdown skill with tests, until failure is structurally impossible to repeat). LangChain independently demonstrated the pattern at the harness level, using trace-analysis skills to lift a GPT-5.2-Codex agent 13.7 points on TerminalBench 2.0 without changing the model. The three approaches share a foundational claim: agents should encode their lessons as first-class evolvable objects, not as prompt edits buried in model weights. [Source: raw/arxiv-org-pdf-2604-15034.md] [Source: raw/garrytan-2046876981711769720.md] [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]

## The Core Problem: Ad-Hoc Self-Modification

Existing self-evolving agent systems are fragmented and ad hoc. Updates happen by hand-patching prompts or swapping tool implementations directly, with no shared abstraction for managing heterogeneous agent components. Without explicit lifecycle management and safe update interfaces, self-modification introduces runtime instability — erroneous updates can lead to irrecoverable errors because there is no rollback. [Source: raw/arxiv-org-pdf-2604-15034.md]

Existing connectivity protocols (Anthropic's MCP, Google's A2A) standardize **invocation** — how agents and tools call each other. They do not standardize **state mutation and management** — how resources evolve over time, how versions are tracked, or how bad updates get rolled back. "Relying solely on communication protocols is insufficient; a novel protocol capable of managing the dynamics of mutation is required." [Source: raw/arxiv-org-pdf-2604-15034.md]

The Autogenesis paper (Zhang, Zhao, Wen, Wu, Yin, An, Wang, 2026) identifies three essential problems any self-evolution protocol must solve:

- **Decoupling**: Prompts, tools, and memory must be abstracted from the agent's core logic — passive, independently managed entities rather than tightly coupled code.
- **Safety & auditability**: Strict version control and rollback to ensure every evolutionary step is traceable and reversible.
- **Formalism**: A standardized set of operators (reflect, propose, verify) that convert heuristic text modifications into a rigorous control loop. [Source: raw/arxiv-org-pdf-2604-15034.md]

## The Autogenesis Protocol (AGP)

Autogenesis is a two-layer protocol architecture that **strictly decouples the evolutionary substrate from the evolutionary logic**:

### Layer 1: Resource Substrate Protocol Layer (RSPL)

RSPL defines what can evolve, modeling five entity types as first-class, protocol-registered resources — **Prompts, Agents, Tools, Environments, and Memory**. Each resource has explicit state, lifecycle, and versioned interfaces. [Source: raw/arxiv-org-pdf-2604-15034.md]

Formally, a resource entity is a tuple `e = (n, d, φ, g, m)` where `n` is a unique name, `d` is a description, `φ: X → Y` is an input-to-output mapping, `g ∈ {0,1}` is the trainable marker (is this resource evolvable?), and `m` is a metadata dictionary. Resources in RSPL are **passive** — they encapsulate no optimization logic and cannot self-modify. All state transitions happen through controlled, interface-mediated operations invoked by higher layers. [Source: raw/arxiv-org-pdf-2604-15034.md]

Each resource type gets a dedicated **context manager** that maintains an active registry of materialized resources and a versioned history for restoration. The manager's API covers five functional groups: lifecycle and registration (init, build), retrieval and inspection (list, get_state), evolution and versioning (update, restore), execution and contract (run, load_contract), and serialization (save_to_json, load_from_json). The manager also produces a consolidated **capability and constraint specification** (a "skills.md-style" contract in the Anthropic sense) that enables controlled prompt injection while reducing prompt bloat. [Source: raw/arxiv-org-pdf-2604-15034.md]

RSPL also defines cross-cutting infrastructure services:

- **Model manager** — unified API layer across providers (OpenAI, Anthropic, Google, OpenRouter) with routing, fallback, and cost-aware selection.
- **Version manager** — auto-incremented semantic versions on every register/update, each referencing an immutable snapshot. Enables rollback, branching, diffing.
- **Dynamic manager** — serialization and hot-swapping of resource configurations at runtime without restarting the agent system.
- **Tracer module** — fine-grained execution traces for interpretability, debugging, and as training signals for retrospective improvement. [Source: raw/arxiv-org-pdf-2604-15034.md]

### Layer 2: Self-Evolution Protocol Layer (SEPL)

SEPL specifies **how** evolution occurs — a control-theoretic formalism with atomic operators. The paper models continuous improvement as a generalized optimization problem over a heterogeneous state space. All state mutations flow through standardized RSPL interfaces, guaranteeing that evolution is **traceable, reversible, and safe-by-construction**. [Source: raw/arxiv-org-pdf-2604-15034.md]

The five atomic operators, each defined over a typed space:

1. **Reflect** (`ψ: Z × Vevo → P(H)`) — map execution traces to causal failure hypotheses. "The semantic gradient of the system."
2. **Select** (`σ: Vevo × P(H) → P(D)`) — translate hypotheses into concrete update proposals.
3. **Improve** (`μ: Vevo × P(D) → Vevo`) — mutate the evolvable variable set.
4. **Evaluate** — score the modified configuration against the objective specification and safety constraints.
5. **Commit** — persist the improvement if it passes evaluation; otherwise roll back via the version manager. [Source: raw/arxiv-org-pdf-2604-15034.md]

A central abstraction is **variable lifting**: projecting the heterogeneous RSPL resources (tool code, system prompts, memory contents) onto a unified representation of **evolvable variables** `Vevo`. Each variable carries a binary learnability constraint `g_v ∈ {0,1}`, which explicitly delineates the trainable subspace. This homogenizes the interaction surface so the same optimization algorithm — TextGrad, GRPO, Reinforce++ — can be applied uniformly across prompts, tools, and memory. [Source: raw/arxiv-org-pdf-2604-15034.md]

### Autogenesis Agent System (AGS)

Building on AGP, the paper presents AUTOGENESIS-AGENT, a reasoning-and-acting tool-calling agent that **dynamically instantiates, retrieves, and refines resources via protocol interfaces during execution** rather than relying on hardcoded components. The system was evaluated on GPQA, AIME, GAIA, and LeetCode and reports consistent improvements over strong baselines. Code at https://github.com/DVampire/Autogenesis. [Source: raw/arxiv-org-pdf-2604-15034.md]

The paper's framing: "A potential shift from manual prompt engineering to automated protocol engineering."

## Skillify: The Operational Practice

Garry Tan's **skillify** (April 22, 2026) is a lighter-weight, markdown-native instantiation of the same principle: every failure becomes a first-class, versioned, testable resource. Where Autogenesis proposes a formal protocol, skillify proposes a 10-step operational checklist — no protocol, just discipline plus GBrain tooling. [Source: raw/garrytan-2046876981711769720.md]

The full practice is detailed in [Thin Harness, Fat Skills](thin-harness-fat-skills.md). The key self-evolution claim:

> In a healthy software engineering team, every bug gets a test. That test lives forever. The bug becomes structurally impossible to recur. AI agents should work the same way. Every failure becomes a skill. Every skill has evals. Every eval runs daily. The agent's judgment improves permanently, not just for the current session, not just while the context window holds. [Source: raw/garrytan-2046876981711769720.md]

The 10-step skillify checklist maps cleanly onto Autogenesis operators:

| Skillify Step | Autogenesis Analog |
|---|---|
| 1. SKILL.md (the contract) | RSPL: register Prompt/Tool resource with versioned interface |
| 2. Deterministic code | RSPL: register Tool resource (native script) |
| 3. Unit tests | SEPL: Evaluate operator (structural gate) |
| 4. Integration tests | SEPL: Evaluate operator (live data) |
| 5. LLM evals | SEPL: Evaluate operator (semantic judgment) |
| 6. Resolver trigger | RSPL: update routing resource in registry |
| 7. Resolver eval | SEPL: Evaluate operator (routing correctness) |
| 8. Check-resolvable + DRY audit | SEPL: graph-level safety constraint |
| 9. E2E smoke test | SEPL: objective-specification verification |
| 10. Brain filing rules | RSPL: Memory resource with explicit lifecycle |

The mapping isn't perfect — skillify lacks formal version lineage and rollback operators — but the structural bet is the same: treat every skill as a versioned, testable, registered resource rather than a prompt edit.

## Resolvers as Self-Healing Routing

Tan's **resolver** pattern (see [Thin Harness, Fat Skills](thin-harness-fat-skills.md)) is the self-evolution story for routing. A resolver is a ~200-line markdown routing table that maps task types to context/skills. It decays over time as new skills are added but not registered, and as user phrasing drifts from trigger descriptions. [Source: raw/garrytan-2044479509874020852.md]

Tan's proposed endgame: **RLM-based resolver self-healing**. The system observes every task dispatch — which skill fired, which didn't, which tasks had no match, which tasks matched the wrong skill. Periodically (nightly or weekly) it rewrites the resolver based on observed evidence. "Eight hundred task dispatches over a month. The system sees that 'is my flight on time' never triggers flight-tracker but 'check my flight' does. It rewrites the trigger description." [Source: raw/garrytan-2044479509874020852.md]

Tan notes Claude Code's **AutoDream** memory consolidation as a primitive version of this pattern. AutoDream merges, dedupes, removes contradictions, and aggressively prunes memory during idle time. Apply the same principle specifically to the resolver and you get a routing table that improves with use. "A resolver that learns from its own traffic. That's the endgame for agent governance." [Source: raw/garrytan-2044479509874020852.md]

## LangChain's Trace-Analyzer Skill: Self-Improvement at the Harness Level

LangChain's February 2026 Deep Agents post demonstrated the pattern at the harness-engineering level rather than the task level. Their deepagents-cli went from **52.8% to 66.5% on TerminalBench 2.0 (13.7-point improvement) without changing the model**. They changed only the harness — system prompt, tools, and middleware. [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]

The mechanism was a **Trace Analyzer Skill** — an agent skill that:

1. Fetches experiment traces from LangSmith.
2. Spawns parallel error-analysis agents; a main agent synthesizes findings and suggestions.
3. Aggregates feedback and makes targeted changes to the harness.

"This works similarly to boosting — focusing on mistakes from previous runs." [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]

The pattern echoes Autogenesis SEPL almost directly:

- **Reflect** ≈ parallel error-analysis agents attributing failures to causes.
- **Select/Improve** ≈ main agent proposing and applying harness changes.
- **Evaluate** ≈ re-running TerminalBench to verify improvement.
- **Commit** ≈ locking in changes that improve the score.

LangChain's specific harness fixes discovered via trace analysis:

- **Build-verify loop**: Agent was writing a solution, re-reading its own code, confirming it looked OK, and stopping. Fix: system-prompt guidance on Planning → Build → Verify → Fix, plus a **PreCompletionChecklistMiddleware** that intercepts the agent before exit and forces a verification pass.
- **LocalContextMiddleware**: Maps cwd and child/parent directories at agent start, runs bash to find tools like Python. "Context discovery and search are error-prone; injecting context reduces this error surface."
- **LoopDetectionMiddleware**: Tracks per-file edit counts; after N edits to the same file, injects "consider reconsidering your approach" into context. Catches doom loops.
- **Reasoning sandwich (xhigh-high-xhigh)**: xhigh reasoning on planning, high during implementation, xhigh during verification. Running xhigh throughout scored 53.9% (timeouts); the sandwich scored 66.5%. [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]

LangChain's framing: "The purpose of the harness engineer is to prepare and deliver context so agents can autonomously complete work."

## The 12 Components of a Self-Evolving Harness

Akshay Pachaar's "Anatomy of an Agent Harness" (April 2026) synthesizes the self-evolution machinery into component #10 of a production harness: **verification loops**. Without verification, errors compound — a 10-step process at 99% per-step success still has only ~90.4% end-to-end success. [Source: raw/akshay_pachaar-2041146899319971922.md]

Anthropic's three verification approaches:

1. **Rules-based feedback** — tests, linters, type checkers as deterministic ground truth.
2. **Visual feedback** — screenshots via Playwright for UI tasks.
3. **LLM-as-judge** — a separate subagent evaluates output.

Boris Cherny (creator of Claude Code): "giving the model a way to verify its work improves quality by 2 to 3x." [Source: raw/akshay_pachaar-2041146899319971922.md]

## Common Architectural Patterns Across Approaches

Despite different terminology, the four approaches converge on a shared pattern:

1. **Externalized, addressable state** — prompts and skills as markdown files (Claude Code, GStack, Autogenesis Prompt resources); memory as git repos (GBrain); tools as versioned registrations (AGP/RSPL).
2. **Passive resources + active operators** — resources don't self-modify; evolution happens only through named operators invoked by higher layers (Autogenesis SEPL, skillify's checklist, LangChain's trace-analyzer loop).
3. **Atomic test/commit/rollback** — no change survives without passing evaluation (skillify step 9 smoke test, Autogenesis evaluate-then-commit, LangChain's benchmark-gated harness changes).
4. **Traces as primary input** — execution traces drive all three learning loops. "Models today are largely black-boxes... we can see their inputs and outputs in text space which we then use in our improvement loops." [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]
5. **Healing by re-reading rather than retraining** — none of these systems finetune model weights. Every fix is a new file the model reads on its next call.

## Where the Approaches Diverge

- **Autogenesis** is a formal protocol with type theory; best suited for research agents that need reproducibility and rigorous version lineage. Heavier machinery.
- **Skillify** is a daily operational habit; best suited for personal agents or teams willing to adopt a 10-step checklist. Lighter but requires discipline.
- **LangChain middleware harness engineering** is a benchmark-driven improvement loop; best suited when you have a scored benchmark and can A/B harness changes. Focused on the harness, not the skills.

None of the three is strictly incompatible with the others. A production system might use skillify for daily skill creation, trace-analyzer skills for harness-level tuning, and Autogenesis-style version lineage for rollback when a skill regression ships.

## Related

- [Thin Harness, Fat Skills](thin-harness-fat-skills.md) — Tan's architecture thesis; skillify is its operational surface
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) — the broader automated-optimization category this fits into
- [Agent Memory and Context Management](agent-memory-and-context-management.md) — Memory as a first-class resource type (RSPL) and as markdown file substrate (GBrain)
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) — skillify and resolvers as continuity patterns across sessions
- [Claude Code Architecture](claude-code-architecture.md) — AutoDream as a primitive of self-healing memory; description-based resolver built into every skill
- [Practical Best Practices](practical-best-practices.md) — concrete testing and verification practices
- [What Is Harness Engineering?](what-is-harness-engineering.md) — the three-layer learning model (model / harness / context) that maps onto Autogenesis's RSPL/SEPL split

## Open Questions

- Does Autogenesis's formal protocol provide real safety gains over skillify's markdown-and-git-discipline approach, or is the formalism overhead most projects can skip? The paper reports benchmark wins but doesn't compare against lighter-weight alternatives.
- When RLM-based resolver self-healing actually ships, will it converge? Tan explicitly notes it's not yet built. [Source: raw/garrytan-2044479509874020852.md]
- Can LangChain's trace-analyzer loop generalize to non-coding benchmarks? The 13.7-point TerminalBench improvement used Codex-specific prompting; Claude Opus 4.6 scored 59.6% with an earlier harness version. "Many principles generalize... but running a few rounds of harness iterations for your task helps maximize agent performance across tasks." [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]
- How does the "skill rot" problem scale? With 40+ skills, Tan found 15% unreachable. At 400+ skills, do the audit tools hold up?
- The shared assumption that traces are interpretable enough to drive improvement may break as models get smaller and more opaque. None of these approaches currently handles cases where the trace doesn't reveal why a decision was made.

## Sources

- [raw/arxiv-org-pdf-2604-15034.md](../raw/arxiv-org-pdf-2604-15034.md) — Zhang, Zhao, Wen, Wu, Yin, An, Wang (NTU, Stanford, City U HK, Princeton), April 2026. "Autogenesis: A Self-Evolving Agent Protocol" — the two-layer RSPL/SEPL protocol and the Autogenesis agent system.
- [raw/garrytan-2046876981711769720.md](../raw/garrytan-2046876981711769720.md) — Garry Tan, April 22, 2026. "How to really stop your agents from making the same mistakes" — the skillify 10-step practice and two concrete failure-to-skill conversions.
- [raw/garrytan-2044479509874020852.md](../raw/garrytan-2044479509874020852.md) — Garry Tan, April 15, 2026. "Resolvers: The Routing Table for Intelligence" — check-resolvable, trigger evals, RLM-based resolver self-healing.
- [raw/akshay_pachaar-2041146899319971922.md](../raw/akshay_pachaar-2041146899319971922.md) — Akshay Pachaar, April 6, 2026. "The Anatomy of an Agent Harness" — verification loops as the 10th of 12 harness components.
- [raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md](../raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md) — Vivek Trivedy (LangChain), February 2026. "Improving Deep Agents with harness engineering" — trace-analyzer skill pattern, 13.7-point TerminalBench improvement.
