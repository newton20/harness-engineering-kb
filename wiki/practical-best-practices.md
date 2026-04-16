---
title: Practical Best Practices
type: wiki
tags:
  - best-practices
  - prompt-engineering
  - sycophancy
  - progressive-disclosure
  - model-generations
  - harness-engineering
  - multi-agent
  - git
  - testing
sources:
  - raw/systematicls-2028814227004395561.md
  - raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md
  - raw/anthropic-com-engineering-harness-design-long-running-apps.md
  - raw/Hxlfed14-2028116431876116660.md
  - raw/openai-com-index-harness-engineering.md
  - raw/servasyy_ai-2042951017462169812.md
  - raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md
  - raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/code-research-claude-code-2026-04-15.md
  - raw/code-research-karpathy-autoresearch-2026-04-15.md
  - raw/code-research-all-hands-ai-openhands-2026-04-15.md
  - raw/code-research-anomalyco-opencode-2026-04-15.md
source_count: 13
status: draft
last_compiled: 2026-04-15
---

# Practical Best Practices

This article collects concrete, actionable practices for harness engineering -- the kind of hard-won lessons that emerge from building and shipping agent systems in production. The evidence comes from Anthropic's long-running agent work, OpenAI's fully agent-generated codebase, @systematicls's operator-level experience, and comparative analysis of multi-agent delegation architectures.

## Less Is More

@systematicls articulates a foundational principle: less is more. Strip dependencies, reduce complexity, give agents only what they need. [Source: raw/systematicls-2028814227004395561.md] Every unnecessary piece of context, every redundant tool, every extraneous instruction is a potential source of confusion and a drain on the model's finite attention.

This is not minimalism for aesthetic reasons. It is a performance optimization. "You don't need the latest agentic harnesses, you don't need to install a million packages and you absolutely do not need to feel the need to read a million things to stay competitive. In fact, your enthusiasm is likely doing more harm than good." [Source: raw/systematicls-2028814227004395561.md]

The pattern is consistent across top teams: Manus has rewritten their framework five times, each time removing things. Anthropic designs Claude Code's scaffold to shrink as models improve. Replit went from one agent to three but each individual agent got simpler. Over-engineering is the default failure mode. [Source: raw/Hxlfed14-2028116431876116660.md]

## Separate Research from Implementation

A common failure mode is giving an agent a vague, compound instruction: "build an auth system." The agent has to simultaneously research options and implement, leading to shallow research and confused implementation. [Source: raw/systematicls-2028814227004395561.md]

The fix is to decompose into distinct phases:

- **Research phase:** "Investigate JWT auth approaches. Report findings on bcrypt cost factors, token expiration strategies, and refresh token patterns."
- **Implementation phase:** "Implement JWT auth with bcrypt-12 cost factor, 15-minute access tokens, 7-day refresh tokens, using the jsonwebtoken library."

The implementation instruction is specific because the research phase already resolved the ambiguity. If you do not know the implementation details, create a research task first, either decide yourself or get an agent to decide, then get another agent with a fresh context to implement. [Source: raw/systematicls-2028814227004395561.md]

## Handle Sycophancy

Models have a well-documented tendency toward sycophancy -- telling the user (or the harness) what it thinks they want to hear. "If you give it an instruction to add 'happy' to every 3 words it's going to do its best to follow that instruction... Its willingness to follow is precisely what makes it such a fun product to use." [Source: raw/systematicls-2028814227004395561.md]

**Use neutral prompts.** Instead of "find bugs in this code" (which primes the model to find bugs whether they exist or not), use "search through the database, try to follow along with the logic of each component, and report back all findings." A neutral prompt sometimes surfaces bugs and sometimes just matter-of-factly states how the code runs, without biasing the agent toward any conclusion. [Source: raw/systematicls-2028814227004395561.md]

**Use adversarial agent setups.** @systematicls describes a three-agent pattern: a bug-finder agent that aggressively identifies all possible bugs (the superset), an adversarial agent that tries to disprove each bug (the subset), and a referee agent that scores both. "Whatever the referee says is the truth, I inspect to make sure it's the truth. For the most part this is frighteningly high fidelity." [Source: raw/systematicls-2028814227004395561.md]

Anthropic's work confirms the pattern from a different angle: self-evaluation fails because models confidently praise their own work. A separate evaluator agent is tractable to tune into a skeptical critic. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Progressive Disclosure

Agents work best when they discover context incrementally rather than receiving everything at once. Anthropic formalized this with Agent Skills: Claude reads skill files that reference other files that the model can read recursively. "Over the course of a year Claude went from not really being able to build its own context, to being able to do nested search across several layers of files." [Source: raw/Hxlfed14-2028116431876116660.md]

The quantitative case is strong: Claude-Mem documentation shows static loading injects 25,000 tokens at 0.8% efficiency (one relevant observation in noise), while progressive disclosure uses 955 tokens at 100% efficiency -- a 26x improvement. [Source: raw/Hxlfed14-2028116431876116660.md]

Dex Horthy (creator of the "12 Factor Agents" methodology) puts the threshold at 40% of the model's input capacity: push past that and you enter the "dumb zone" where signal-to-noise degrades and agents start making mistakes that look like reasoning failures but are actually information overload. [Source: raw/Hxlfed14-2028116431876116660.md]

## Every New Model Generation Forces Rethinking

Harness components encode assumptions about model limitations. @systematicls warns: "the most important principle to hold is the realization that every new generation of agents will force you to rethink what is optimal, which is why less is more." [Source: raw/systematicls-2028814227004395561.md]

Anthropic's experience confirms this concretely. When Opus 4.6 dropped, they were able to remove the sprint construct from their long-running harness entirely because the model could natively handle coherent work without that decomposition. The evaluator's role changed too -- tasks that used to need the evaluator's check were now within what the generator handled well on its own. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

The principle: "every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve." [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Find the Simplest Solution Possible

Anthropic's guidance is direct: "Find the simplest solution possible, only increase complexity when needed." [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md] When Anthropic attempted to simplify their harness radically all at once, they could not replicate performance and could not tell which pieces were load-bearing. They moved to a more methodical approach: removing one component at a time and reviewing the impact. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

If your agent harness is getting more complex while models get better, something is wrong. [Source: raw/Hxlfed14-2028116431876116660.md]

## Git as Recovery Mechanism

For coding agents, git serves as a critical safety net. Anthropic's long-running agent harness uses git commits at meaningful checkpoints. If the agent goes down a bad path, the harness (or the user) can revert to a known-good state. The commit history provides an audit trail of what the agent did and why. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

OpenAI's fully agent-generated codebase takes this further: agents use standard development tools directly (gh, local scripts, repository-embedded skills) to gather context, open pull requests, respond to feedback, and often squash and merge their own PRs. The repository operates with minimal blocking merge gates and short-lived PRs. "In a system where agent throughput far exceeds human attention, corrections are cheap, and waiting is expensive." [Source: raw/openai-com-index-harness-engineering.md]

## Feature Lists as JSON, Not Markdown

Anthropic found that representing feature lists as JSON rather than markdown prevents the model from inappropriately modifying them. They use "strongly-worded instructions like 'It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality.'" The model is less likely to inappropriately change or overwrite JSON files compared to markdown files. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

## Browser Testing Dramatically Improves Performance

Adding browser testing capabilities (Playwright, Puppeteer MCP) to coding agents catches bugs that are invisible from code alone. Anthropic found that Claude "did well at verifying features end-to-end once explicitly prompted to use browser automation tools and do all testing as a human user would." Screenshots taken through the Puppeteer MCP allowed the agent to identify and fix bugs not obvious from code alone. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

OpenAI similarly wired the Chrome DevTools Protocol into the agent runtime and created skills for working with DOM snapshots, screenshots, and navigation, enabling Codex to reproduce bugs, validate fixes, and reason about UI behavior directly. [Source: raw/openai-com-index-harness-engineering.md]

## Mitigating Unattended Execution Risk

Simon Willison identified that agents are most productive when running unattended ("YOLO mode"), but this introduces three distinct risk categories: bad shell commands that damage the local environment, exfiltration attacks where injected instructions cause the agent to leak data, and proxy attacks where the agent is tricked into taking actions on behalf of an attacker. His recommended mitigations are layered: sandbox the agent (Docker containers, Apple containers), run on someone else's computer (GitHub Codespaces or similar ephemeral environments), or accept the risk with eyes open. [Source: raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md] For credential scoping specifically, Willison recommends creating dedicated organizations with hard budget limits -- for example, a Fly.io org with a $5 spending cap -- so that even if an agent goes off the rails, the blast radius is financially bounded. [Source: raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md] This complements the git-as-recovery-mechanism approach: git handles rollback after mistakes, while sandboxing and credential scoping prevent the mistakes from having irreversible consequences in the first place.

## AGENTS.md as TOC, Not Encyclopedia

OpenAI learned that the "one big AGENTS.md" approach fails: context is a scarce resource that crowds out the task, too much guidance becomes non-guidance, it rots instantly, and it is hard to verify. [Source: raw/openai-com-index-harness-engineering.md]

Instead, they treat AGENTS.md as the table of contents -- roughly 100 lines injected into context as a map with pointers to deeper sources of truth in a structured docs/ directory. Design docs, execution plans, product specs, and references all live in version-controlled files the agent can discover via progressive disclosure. [Source: raw/openai-com-index-harness-engineering.md]

This mirrors @systematicls's advice: "treat your CLAUDE.md as a logical, nested directory of where to find context given a scenario and an outcome. It should be as barebones as possible, and only contain the IF-ELSE of where to go to seek the context." [Source: raw/systematicls-2028814227004395561.md]

OpenAI enforces this mechanically with dedicated linters and CI jobs that validate the knowledge base is up to date, cross-linked, and structured correctly. A recurring "doc-gardening" agent scans for stale documentation and opens fix-up PRs. [Source: raw/openai-com-index-harness-engineering.md]

## Multi-Agent Delegation Patterns: Hermes vs. OpenClaw

@servasyy_ai provides a deep technical comparison of two multi-agent delegation architectures, representing opposite ends of the design space: [Source: raw/servasyy_ai-2042951017462169812.md]

**Hermes (delegate_task):** A synchronous "contractor-subcontractor" model. The parent agent dispatches tasks to child agents, then blocks until all results return. Child agents' intermediate reasoning never enters the parent's context window -- only a compressed summary returns. This achieves zero context inflation but means the parent cannot respond to new demands while children run. [Source: raw/servasyy_ai-2042951017462169812.md]
- Max 3 concurrent children, max depth of 2 (no grandchildren)
- Child agents are stripped of delegate_task, clarify, memory, execute_code, and send_message tools to prevent escaping the sandbox
- No timeout mechanism -- children are bounded only by iteration count (default 50)

**OpenClaw (subagent system):** An asynchronous event-driven "orchestra" model. The parent defines a global topology, then child agents run asynchronously with results pushed back via events. The parent can send "steer" messages to redirect running children (rate-limited to once per 2 seconds, max 4000 characters). [Source: raw/servasyy_ai-2042951017462169812.md]
- Max 8 concurrent agents, 5 active children per agent, configurable nesting depth
- Supports external agents (Claude Code, Codex) as children via ACP
- Timeout control (default 300 seconds), persistent run records, session recovery
- Tradeoff: announce pushes inject child results into parent context (~12% more tokens for same 3-task scenario)

**Choosing between them:** Hermes is optimal for parallel processing of 3 independent tasks with clean isolation and zero context pollution. OpenClaw is necessary when tasks require mid-flight direction changes, timeout control, external agent integration, or more than 3 parallel workers. The recommended hybrid: use OpenClaw's async orchestration for complex workflows with Hermes-style delegate calls embedded for isolated parallel subtasks. [Source: raw/servasyy_ai-2042951017462169812.md]

## Entropy and Garbage Collection

OpenAI found that full agent autonomy introduces entropy: agents replicate patterns that already exist in the repository, including suboptimal ones. Initially, "our team used to spend every Friday (20% of the week) cleaning up 'AI slop.'" [Source: raw/openai-com-index-harness-engineering.md]

The fix: encode "golden principles" directly into the repository and build recurring cleanup processes. Background Codex tasks scan for deviations, update quality grades, and open targeted refactoring PRs on a regular cadence. "This functions like garbage collection. Technical debt is like a high-interest loan: it's almost always better to pay it down continuously in small increments than to let it compound." [Source: raw/openai-com-index-harness-engineering.md]

## Effort Scaling: Classify Before You Start

Not every query deserves the same amount of compute. A practical best practice emerging from deep research systems is to classify query complexity before dispatching agents. Simple factual questions can be answered by a single agent in one pass. Moderate questions benefit from a few search-reason cycles. Complex research questions justify multiple parallel subagents running extended search loops. [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

Anthropic's Research feature validates this: token usage explains 80% of performance variance. Under-investing in a complex query produces thin results; over-investing in a simple query wastes money (deep research sessions cost $2-5 and use 15x the tokens of a standard chat turn). The harness should make this classification explicitly rather than treating all queries identically. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md, raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

## Convergence Detection: Know When to Stop

A critical and often overlooked best practice is detecting when further search iterations produce diminishing returns. Two complementary strategies have emerged as best practice: [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

- **Coverage checklists:** Before starting research, decompose the question into sub-questions or aspects that must be covered. Track which aspects have been addressed. When coverage reaches a threshold (e.g., 90%), stop searching. This is deterministic and auditable.
- **Budget cutoffs:** Set hard limits on token spend, time, or iteration count. When the budget is exhausted, synthesize from whatever has been gathered. This prevents runaway sessions.

The best systems combine both: the coverage checklist provides a quality signal, and the budget cutoff provides a safety net. Neither alone is sufficient -- checklists without budgets can loop indefinitely on hard-to-find information, and budgets without checklists can stop before critical aspects are covered. [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

## Minimal-Signal Extraction

Karpathy's autoresearch demonstrates a concrete anti-context-bloat pattern: redirect all program output to a log file, then grep only the key metrics into the agent's context. In autoresearch, each 5-minute training run produces roughly 630 lines of output, but only 2 scalar values (val_bpb and peak_vram_mb) enter the agent's context via `grep "^val_bpb:\|^peak_vram_mb:" run.log`. This adds 3-5 lines per experiment instead of hundreds. The system prompt explicitly warns "do NOT use tee or let output flood your context." The pattern generalizes to any agent that runs evaluation processes: redirect stdout, extract the metrics that matter, discard the rest. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Complexity as Explicit Optimization Criterion

Karpathy's autoresearch instructs the LLM to treat code complexity as a first-class cost that must be weighed against metric gains: "A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not worth it. A 0.001 val_bpb improvement from deleting code? Definitely keep." This anti-bloat policy is enforced as an explicit directive in the system prompt, not as a passive preference. It prevents strategy drift in long-running self-improvement loops: without a complexity penalty, agents tend to accumulate patches and workarounds that individually seem justified but collectively degrade maintainability. By making complexity an explicit optimization criterion alongside the primary metric, the system keeps the mutation search space honest. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Metric Isolation via Immutable Oracle

The evaluation function in autoresearch's prepare.py is declared read-only by natural language contract — the agent is explicitly told it cannot modify this file. This prevents Goodhart's Law: an agent optimizing for a metric it can also redefine will eventually redefine the metric rather than improving the underlying capability. Structural file separation enforces the boundary between mutable strategy code (train.py) and the protected evaluation oracle (prepare.py). For production deployments, Karpathy's natural language contract should be reinforced by file system permissions or sandboxing — the agent cannot be trusted to honor the read-only constraint under adversarial conditions or in long-running sessions where the original instruction may be compacted away. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Stateless Agent Restart Protocol

The Setup section of autoresearch's program.md defines a deterministic context reconstruction procedure: read the current branch name, read 3 specific files (program.md, train.py, run.log tail), check the experiment cache. Any new agent instance — including one starting from a blank context after a reset — can resume seamlessly by following these steps. The procedure is designed for LLM context resets: it assumes nothing is in working memory and everything must be re-derived from durable state. This is a concrete implementation of the "stateless agent" principle: the agent's effective state is always reconstructible from artifacts in the repository, not from conversation history. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Structural Immutability via File Separation

A related autoresearch pattern: enforce the boundary between mutable strategy code and protected evaluation harness through physical file separation. In autoresearch, train.py is mutable (the agent can edit it freely), while prepare.py containing the evaluation function is read-only by natural language contract. The agent cannot game the metric without violating the read-only rule. This is not access control -- it is a structural design decision where the evaluation oracle lives in a file the agent is instructed never to modify. For any self-improving agent system, the backtest harness or evaluation function must be structurally separated from the code the agent is allowed to mutate. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Prompt Cache-First Architecture

Claude Code's source reveals that prompt cache optimization is a primary design constraint permeating the entire architecture. The system prompt is split at a `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker into a globally cacheable prefix (identity, tools, style rules) and a session-specific suffix (memory, environment, MCP tools). Everything before the boundary is identical across all users, enabling cross-org prompt cache sharing. Tool schemas are sorted alphabetically with a cache breakpoint after the last built-in tool -- tool ordering is a cache optimization decision, not an ergonomic one. Forked sub-agents inherit byte-identical prompt prefixes to share the parent's cache. A single tool description change was observed to cause roughly 10.2% of fleet `cache_creation` tokens, illustrating how consequential cache stability is at scale. [Source: raw/code-research-claude-code-2026-04-15.md]

## Error-as-Context Over Harness Retries

Claude Code returns all tool errors as `<tool_use_error>` content in the tool_result message rather than throwing exceptions or triggering automatic retries. The model reads the error and decides how to proceed -- it may retry with different parameters, try an alternative approach, or inform the user. This error-as-context pattern produces simpler harness code (no retry logic, no backoff configuration) and smarter error handling, since the model can reason about what went wrong contextually rather than blindly retrying the same operation. The pattern is particularly effective for coding agents where errors are often informative (e.g., a compilation error tells the agent exactly what to fix). [Source: raw/code-research-claude-code-2026-04-15.md]

## Deterministic Scaffolds Around Non-Deterministic Reasoning

A principle from the deep research literature that applies broadly to harness engineering: "deterministic scaffolds around non-deterministic reasoning." [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md] The search loop control -- when to search, when to stop, how many subagents to spawn, what budget to allocate -- should be regular code, not LLM decisions. The LLM's role is to reason about content: what to search for, how to interpret results, what to synthesize. Mixing control flow with content reasoning leads to unpredictable behavior: agents that search forever, agents that stop too early, agents that spawn too many or too few workers.

This reinforces the "thin harness, fat skills" principle (see [Tool Design Patterns](tool-design-patterns.md)): the harness handles deterministic orchestration, the model handles non-deterministic reasoning.

## Source Reliability and Contradiction Detection

When agents gather information from multiple sources, they must handle conflicting claims. Two practices have emerged: [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

- **Multi-source corroboration:** Treat a claim as reliable only when multiple independent sources support it. A single source, no matter how authoritative, may be wrong or outdated.
- **Contradiction detection:** Actively identify cases where sources disagree and flag these for explicit resolution rather than silently choosing one. The agent should note the disagreement in its output rather than presenting one version as fact.

These practices apply beyond deep research -- any agent that gathers context from external sources (documentation, codebases, web search) faces the same reliability challenge.

## Start Wide, Then Narrow

For research and exploration tasks, the recommended search strategy is to start with broad queries that establish the landscape, then progressively narrow based on what was found. Starting narrow risks missing important context; starting broad and filtering is more robust. This mirrors the progressive disclosure principle applied to the agent's own information gathering rather than to the information presented to the agent. [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]

## Doom-Loop Detection Variants

Both OpenHands and OpenCode implement explicit detection of stuck-loop conditions, but with distinct intervention strategies. OpenHands maintains a 5-scenario stuck detector covering repeated identical actions, action-error loops, action-observation pairs that cycle, chains of pure errors, and repeated condensation events; when any scenario triggers, the system halts and prompts the user interactively via CLI rather than auto-aborting. OpenCode's detector is simpler but routed differently: three consecutive identical tool calls trigger a permission-system `ask("doom_loop")` event, giving the user three choices — allow, deny, or auto-configure. The key architectural difference is where the decision lands: OpenHands escalates to a human as a hard stop, while OpenCode routes through the same allow/deny/ask flow used for all other permission decisions, keeping doom-loop handling consistent with the rest of the permission model. Neither system attempts to auto-recover without user input. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md] [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Two-Part System Prompt for Cache Efficiency

OpenCode normalizes the system prompt array to exactly two entries on every request: a stable base prompt containing identity and invariant rules, and a variable context entry holding session-specific content such as environment facts, loaded AGENTS.md files, and tool descriptions. This two-entry invariant is enforced regardless of how many sources contribute to the prompt. By keeping the first entry byte-identical across requests, every call gets a cache hit on the base prompt; only the second entry needs fresh caching. The pattern is architecturally similar to Claude Code's `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` split into cacheable prefix and session-specific suffix. For high-volume agent deployments where the base prompt dominates token count, the cache savings from this discipline are significant. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Claims-Based Instruction Deduplication

OpenCode tracks which AGENTS.md files have already been injected into the current session using a per-turn injection registry keyed by file path. Before adding an AGENTS.md file's content to the system prompt, the harness checks whether that file has already been included earlier in the session; if so, it is skipped. This prevents exponential instruction growth in long sessions where many tool calls might each re-trigger AGENTS.md discovery. The pattern generalizes to any harness that dynamically discovers and injects context files: without deduplication, the same file can be injected dozens of times across a long session, silently consuming token budget and potentially confusing the model with repeated identical instructions. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Summary of Principles

1. Strip everything that is not load-bearing [Source: raw/systematicls-2028814227004395561.md]
2. Separate research from implementation [Source: raw/systematicls-2028814227004395561.md]
3. Design around sycophancy with neutral prompts and adversarial setups [Source: raw/systematicls-2028814227004395561.md]
4. Let agents discover context progressively, not all at once [Source: raw/Hxlfed14-2028116431876116660.md]
5. Revisit the entire harness with each new model generation [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
6. Default to the simplest solution that works [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
7. Use JSON for data the model should not edit, markdown for content it should [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
8. Leverage git for recovery, not just version control [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
9. Add browser testing to close the visual feedback loop [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
10. Treat AGENTS.md/CLAUDE.md as a table of contents, not an encyclopedia [Source: raw/openai-com-index-harness-engineering.md]
11. Build garbage collection for agent-generated codebases [Source: raw/openai-com-index-harness-engineering.md]
12. Classify query complexity before dispatching agents -- effort scaling prevents both under- and over-investment [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]
13. Use coverage checklists + budget cutoffs for convergence detection [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]
14. Keep search loop control in deterministic code; let the LLM reason about content, not control flow [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]
15. Corroborate across sources and flag contradictions explicitly [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]
16. Extract minimal signals from evaluation output -- redirect stdout, grep key metrics, protect context from flooding [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]
17. Separate mutable code from immutable evaluation harness via physical file boundaries [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]
18. Design for prompt cache stability -- static/dynamic boundaries, sorted tool schemas, byte-identical fork prefixes [Source: raw/code-research-claude-code-2026-04-15.md]
19. Return errors as context for the model to reason about rather than retrying automatically in the harness [Source: raw/code-research-claude-code-2026-04-15.md]
20. Use SOUL.md for personality/persona definition -- separate from code instructions (CLAUDE.md) and agent guidelines (AGENTS.md). Personality as user-controlled workspace content survives harness upgrades [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]
21. Use per-provider schema normalization so tool authors write schemas once (TypeBox) and provider quirks (Gemini, OpenAI strict, xAI) are handled at the normalization layer [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]
22. For multi-provider deployments, implement streaming JSON argument repair to handle provider-specific tool call bugs in the pipeline, not after the fact [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]
23. Implement post-compaction context refresh -- re-inject critical config file sections (AGENTS.md startup/safety rules) after compaction with current-date substitution [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]
24. Implement doom-loop detection and route it through the permission system (allow/deny/ask) rather than auto-aborting -- gives users control without silently stopping work [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md] [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]
25. Normalize system prompt to exactly 2 entries (stable base + variable context) to maximize prompt cache hit rate on the base prefix [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]
26. Track injected instruction files per session turn and deduplicate — prevent exponential token growth from repeated AGENTS.md injections in long sessions [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]
27. Treat code complexity as an explicit optimization criterion alongside the primary metric — a small improvement that adds hacky code may not be worth it; a small improvement from deleting code usually is [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]
28. Isolate the evaluation oracle from mutable strategy code via physical file separation; reinforce with file permissions or sandboxing rather than relying on natural language read-only contracts [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]
29. Define a stateless restart protocol (read branch, read key files, check cache) so any new agent instance can reconstruct working context without relying on conversation history [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

## Related

- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- Context curation, CLAUDE.md as directory, compaction handling
- [Tool Design Patterns](tool-design-patterns.md) -- Progressive disclosure, fewer tools, lazy loading
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) -- Adversarial evaluation, GAN-inspired patterns, self-improvement loops
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- Initializer agents, incremental progress, context resets
- [Claude Code Architecture](claude-code-architecture.md) -- System prompt layering, tool result injection, progressive disclosure
- [OpenAI Codex Harness](openai-codex-harness.md) -- AGENTS.md as TOC, repository as system of record, garbage collection
- [Deep Research Agents](deep-research-agents.md) -- convergence detection, effort scaling, search strategies, and economics of deep research sessions
- [Agentic Design Patterns](agentic-design-patterns.md) -- ReAct, Reflection, Planning as formal patterns underlying these best practices
- [Multi-Agent Reliability](multi-agent-reliability.md) -- source reliability via credibility scoring and adversary-resistant multi-agent coordination

## Open Questions

- At what point does harness simplification go too far? Anthropic found that radical simplification all at once failed -- they could not determine which pieces were load-bearing. Methodical one-at-a-time removal was necessary. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
- How should the Hermes vs. OpenClaw delegation tradeoff evolve as models get better at managing their own context and sub-tasks? [Source: raw/servasyy_ai-2042951017462169812.md]
- OpenAI found conventional merge gates became counterproductive at high agent throughput. What new quality assurance patterns replace traditional code review in fully agent-generated systems? [Source: raw/openai-com-index-harness-engineering.md]
- How does the "doc-gardening" agent approach scale? Is there a risk of agents maintaining documentation that drifts from unstated human preferences?

## Sources

- [raw/systematicls-2028814227004395561.md](../raw/systematicls-2028814227004395561.md) -- @systematicls on less is more, context is everything, separating research from implementation, handling sycophancy with neutral prompts and adversarial agents, CLAUDE.md as directory, iterative rule/skill building.
- [raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md](../raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md) -- Anthropic on initializer agents, incremental progress, feature lists as JSON, git as recovery, browser testing with Puppeteer MCP, progress files for session bridging.
- [raw/anthropic-com-engineering-harness-design-long-running-apps.md](../raw/anthropic-com-engineering-harness-design-long-running-apps.md) -- Anthropic's Prithvi Rajasekaran on GAN-inspired evaluators, simplest solution principle, harness simplification methodology, model generation rethinking (Opus 4.5 to 4.6 transition).
- [raw/Hxlfed14-2028116431876116660.md](../raw/Hxlfed14-2028116431876116660.md) -- Himanshu's survey of harness architectures. Progressive disclosure quantified (26x efficiency gain), CORE-Bench scaffold comparison, Vercel tool deletion, 12 Factor Agents 40% threshold.
- [raw/openai-com-index-harness-engineering.md](../raw/openai-com-index-harness-engineering.md) -- OpenAI's Ryan Lopopolo on building a million-line codebase with zero manually-written code. AGENTS.md as TOC, repository as system of record, entropy and garbage collection, agent legibility, throughput changing merge philosophy.
- [raw/servasyy_ai-2042951017462169812.md](../raw/servasyy_ai-2042951017462169812.md) -- @servasyy_ai deep technical comparison of Hermes delegate_task (synchronous, isolated, token-efficient) vs. OpenClaw subagent system (asynchronous, event-driven, steerable). Architecture tradeoffs, hybrid patterns.
- [raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md](../raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md) -- Simon Willison, Sep 2025. YOLO mode risk taxonomy (bad commands, exfiltration, proxy attacks), sandbox mitigations, tightly scoped credentials with budget limits.
- [raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md](../raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md) -- Tian Pan, Apr 2026. Convergence detection (coverage checklists, budget cutoffs), effort scaling, source reliability (multi-source corroboration, contradiction detection), deterministic scaffolds around non-deterministic reasoning, deep research economics ($2-5/session, 15x chat tokens).
- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) -- Anthropic, Apr 2026. Token usage explains 80% of performance variance. Parallel tool calling cut research time by 90%. Effort scaling validation.
- [raw/code-research-claude-code-2026-04-15.md](../raw/code-research-claude-code-2026-04-15.md) -- Code research, Apr 2026. Prompt cache-first architecture with static/dynamic boundary, cache-stable tool ordering, error-as-context pattern.
- [raw/code-research-karpathy-autoresearch-2026-04-15.md](../raw/code-research-karpathy-autoresearch-2026-04-15.md) -- Code research, Apr 2026. Minimal-signal extraction via redirect+grep, structural immutability via file separation for tamper-proof evaluation.
- [raw/code-research-openclaw-openclaw-2026-04-15.md](../raw/code-research-openclaw-openclaw-2026-04-15.md) -- Code research, Apr 2026. SOUL.md persona pattern, per-provider schema normalization, streaming JSON argument repair, post-compaction context refresh.
- [raw/code-research-all-hands-ai-openhands-2026-04-15.md](../raw/code-research-all-hands-ai-openhands-2026-04-15.md) -- Code research, Apr 2026. 5-scenario doom-loop detector with interactive CLI recovery; stuck detection covering repeated actions, action-error cycles, condensation events.
- [raw/code-research-anomalyco-opencode-2026-04-15.md](../raw/code-research-anomalyco-opencode-2026-04-15.md) -- Code research, Apr 2026. 3-identical-calls doom-loop detector via permission system; two-entry system prompt normalization for cache efficiency; per-turn AGENTS.md injection deduplication.
