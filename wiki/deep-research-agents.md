---
title: "Deep Research Agents"
type: wiki
tags:
  - deep-research
  - multi-agent
  - search-reason-loops
  - convergence
  - orchestrator-worker
sources:
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/firecrawl-dev-blog-deep-research-for-ai-agents.md
  - raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md
  - raw/arxiv-org-html-2512-20491v1.md
source_count: 4
status: draft
last_compiled: 2026-04-14
---

# Deep Research Agents

Deep research agents represent a qualitative leap beyond single-turn LLM interactions. Rather than answering a question in one pass, these systems run **search-reason loops** -- search, read, update internal knowledge, then re-search with better-informed questions -- repeating until the agent reaches convergence or exhausts its budget. The performance gap is dramatic: standard LLMs without iteration score below 10% on multi-step research benchmarks, while deep research agents achieve 51.5% [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md] [Source: raw/firecrawl-dev-blog-deep-research-for-ai-agents.md].

This article synthesizes architectural patterns, production lessons, and open challenges from Anthropic's multi-agent research system, open-source deep research frameworks, and academic work on training dedicated research models.

## Anthropic's Orchestrator-Worker Architecture

Anthropic's Research feature uses an **orchestrator-worker pattern**: a lead agent coordinates the overall research plan while specialized subagents execute searches in parallel [Source: raw/anthropic-com-engineering-multi-agent-research-system.md].

The performance results are striking. Multi-agent with Claude Opus 4 as the lead and Claude Sonnet 4 subagents **outperformed single-agent Opus 4 by 90.2%** on Anthropic's internal evaluation [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]. This is not just a marginal gain -- the multi-agent setup nearly doubles effective performance.

A key empirical finding is that **token usage alone explains 80% of performance variance** on BrowseComp. Agents use approximately 4x more tokens than standard chat interactions, and multi-agent configurations use approximately 15x more [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]. This suggests that, within the current paradigm, scaling compute (via more thorough search and reasoning) is the primary lever for research quality.

The workflow proceeds as follows:

1. The **lead agent** receives the user query, creates a research plan, and saves it to Memory (important because context may be truncated at 200K tokens).
2. The lead agent **spawns subagents** with specific, well-scoped tasks.
3. Subagents execute their searches and return results.
4. A **CitationAgent** processes the final output to ensure proper attribution.

[Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

## Three-Layer Architecture

A useful conceptual model decomposes deep research agents into three layers [Source: raw/firecrawl-dev-blog-deep-research-for-ai-agents.md] [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]:

- **Retrieval layer**: Search APIs, web scrapers, and content extractors that gather raw information from the web or document stores.
- **Orchestration layer**: Agent frameworks that decide *when* to search, *what* to search for, and how to route tasks across multiple agents or tool calls.
- **Reasoning layer**: LLMs that interpret retrieved results, synthesize findings, and decide whether the research is complete or more searching is needed.

This separation of concerns allows each layer to be improved independently. A better scraper improves retrieval without touching orchestration; a better planning prompt improves orchestration without changing the retrieval stack.

## The Convergence Problem

Knowing when to stop is the hardest engineering challenge in deep research systems. An agent that stops too early misses critical information; one that runs too long wastes tokens and money without meaningful gains. Several strategies exist [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md]:

- **Information gain thresholds**: Stop when the marginal new information from the latest search drops below a defined threshold. This requires a way to measure "new information," which is non-trivial.
- **Query saturation**: Stop when the agent begins generating semantically similar queries to ones it has already issued. If the agent is asking the same questions in different words, it has likely exhausted the available information.
- **Coverage checklists**: Decompose the original question into sub-questions and track which have been adequately answered. This provides an explicit progress measure.
- **Budget-based cutoffs**: Set hard limits on maximum iterations, tokens consumed, or dollar cost.
- **Best practice**: Combine coverage checklists with budget cutoffs. Checklists provide a quality signal; budgets provide a safety net.

## Economics and Effort Scaling

Deep research is expensive relative to single-turn chat. Typical costs range from **$2-5 per session** for moderate complexity queries. Google Gemini's deep research reportedly runs 80-160 searches per task [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md].

A critical optimization is **effort scaling** -- classifying query complexity before committing resources:

- **Simple queries**: 1 agent, 3-10 tool calls
- **Moderate queries**: 2-4 agents, 10-15 tool calls
- **Complex queries**: 10+ agents, many parallel search streams

Without effort scaling, simple factual questions consume the same budget as genuinely complex research tasks, wasting resources and adding unnecessary latency [Source: raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md].

## Prompting Multi-Agent Research Systems

Anthropic documented several hard-won lessons for prompting and configuring multi-agent research systems [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]:

1. **Think like your agents**: Build simulations and watch agent behavior step-by-step. You cannot debug what you cannot observe.
2. **Teach the orchestrator how to delegate**: Provide detailed task descriptions to subagents, not vague directives. The quality of delegation determines the quality of results.
3. **Scale effort to query complexity**: Include explicit guidelines in the orchestrator prompt for how many agents and searches to use based on query difficulty.
4. **Tool descriptions are critical**: Bad tool descriptions send agents down wrong paths. One team found that having a tool-testing agent rewrite tool descriptions led to a **40% decrease in task completion time**.
5. **Let agents improve themselves**: The system can be designed so agents iteratively refine their own tool usage and delegation strategies.
6. **Start wide, then narrow**: Issue broad queries first to establish the landscape, then follow up with targeted searches.
7. **Guide the thinking process**: Extended thinking serves as a controllable scratchpad where agent reasoning can be shaped via prompting.
8. **Parallel tool calling transforms speed**: Enabling parallel tool calls cut research time by up to 90%.

## Step-DeepResearch: Training Dedicated Research Models

Academic work has explored training models specifically for deep research rather than relying solely on prompting general-purpose LLMs. **Step-DeepResearch** (arxiv 2512.20491) is a 32B parameter model that achieves 61.4% on Scale AI Research Rubrics, rivaling OpenAI and Gemini Deep Research products [Source: raw/arxiv-org-html-2512-20491v1.md].

The key innovation is a **"Data Synthesis Strategy Based on Atomic Capabilities"** that decomposes research into four atomic skills: planning, information gathering, reflection, and report writing. Training proceeds progressively:

1. **Agentic mid-training**: The base model is trained on agentic interaction traces.
2. **Supervised fine-tuning (SFT)**: The model is fine-tuned on high-quality research demonstrations.
3. **Reinforcement learning with Checklist-style Judger rewards**: A reward model evaluates outputs against structured checklists rather than holistic quality scores.

This progressive training pipeline builds research capabilities incrementally, allowing a relatively small model to compete with much larger systems augmented by extensive engineering [Source: raw/arxiv-org-html-2512-20491v1.md].

## Production Reliability

Running deep research agents in production introduces challenges distinct from serving standard LLM chat [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]:

- **Agents are stateful, and errors compound**: A single bad search result or misinterpretation can cascade through subsequent reasoning steps. Unlike stateless chat, you cannot simply retry the last turn.
- **Resume from checkpoints, don't restart**: When failures occur, resuming from the last good state is vastly more efficient than restarting from scratch, especially for long-running research tasks.
- **Rainbow deployments**: Standard blue-green deployments cut off running agents mid-task. Rainbow deployments allow existing agent sessions to complete on the old version while new sessions start on the updated version.
- **Monitor agent decision patterns, not conversation contents**: For privacy, track metrics like tool call frequency, search query patterns, and convergence speed rather than logging the actual content of research.
- **Subagent output to filesystem**: Having subagents write their results to a shared filesystem rather than passing everything through the orchestrator minimizes the "game of telephone" effect where information degrades as it passes through multiple agents.

## Sources

- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) — Anthropic's engineering blog post on building their multi-agent research system, covering architecture, prompting lessons, and production reliability
- [raw/firecrawl-dev-blog-deep-research-for-ai-agents.md](../raw/firecrawl-dev-blog-deep-research-for-ai-agents.md) — Firecrawl's guide to deep research for AI agents, covering the three-layer architecture and retrieval infrastructure
- [raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md](../raw/tianpan-co-zh-blog-2026-04-12-deep-research-agents-orchestrating-multi-.md) — Comprehensive overview of deep research agent orchestration, convergence strategies, economics, and effort scaling
- [raw/arxiv-org-html-2512-20491v1.md](../raw/arxiv-org-html-2512-20491v1.md) — Step-DeepResearch paper describing a 32B model trained with atomic capability decomposition and progressive RL

## Related

- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) — Multi-session patterns and persistent agent execution
- [Agent Memory and Context Management](agent-memory-and-context-management.md) — Context overflow handling relevant to 200K token truncation
- [Tool Design Patterns](tool-design-patterns.md) — Tool description lessons and catalog design
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) — Self-improvement loops and agent-driven optimization
- [Practical Best Practices](practical-best-practices.md) — Effort scaling and production evaluation

## Open Questions

- **Optimal convergence detection**: Coverage checklists combined with budget cutoffs is the current best practice, but no principled method exists for setting information gain thresholds or determining when query saturation is truly reached. Is there a generalizable stopping criterion?
- **Multi-agent vs. single-agent scaling laws**: Anthropic found multi-agent outperforms single-agent by 90.2%, but is this a fixed advantage or does it depend on task type? At what complexity threshold does multi-agent coordination overhead exceed its benefits?
- **Small model viability**: Step-DeepResearch shows a 32B model can rival proprietary systems. How far can this be pushed -- can even smaller models be effective research agents with the right training pipeline?
- **Citation accuracy**: The CitationAgent addresses attribution, but how reliable are citations in practice? What percentage of claims are correctly attributed vs. hallucinated or mis-attributed?
- **Privacy-preserving monitoring**: Anthropic recommends monitoring decision patterns rather than content. What specific metrics are most predictive of research quality without exposing user data?
