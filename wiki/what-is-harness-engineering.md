---
title: "What Is Harness Engineering?"
type: wiki
tags:
  - harness-engineering
  - context-engineering
  - agent-architecture
  - definitions
  - benchmarks
sources:
  - raw/openai-com-index-harness-engineering.md
  - raw/Hxlfed14-2028116431876116660.md
  - raw/garrytan-2042925773300908103.md
  - raw/hwchase17-2040467997022884194.md
  - raw/hwchase17-2042978500567609738.md
  - raw/rohit4verse-2041548810804211936.md
  - raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md
  - raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md
  - raw/akshay_pachaar-2041146899319971922.md
  - raw/akshay_pachaar-2045404494641733962.md
  - raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md
  - raw/arxiv-org-html-2604-14228.md
source_count: 12
status: draft
last_compiled: 2026-04-23
---

Harness engineering is the discipline of configuring, extending, and structuring the integration points of an existing AI agent to maximize its effectiveness for a particular task or team. The term was coined by @dexhorthy in a November 4, 2025 thread on X, where he observed that the people getting dramatically better outcomes from coding agents were not using better models -- they were engineering the wrapper around the model more effectively. [Source: raw/Hxlfed14-2028116431876116660.md] The concept has since been validated by benchmark evidence, independent practitioners, and both Anthropic and OpenAI, establishing it as one of the central organizing ideas in AI agent development.

## Definition and Origin

In dexhorthy's original formulation, harness engineering is distinct from context engineering. Context engineering concerns how context -- long or short, agentic or not -- is passed to an LLM to get the best results, and is primarily about agent design and construction. Harness engineering concerns how a user or team configures, extends, and instruments an existing agent to maximize its effectiveness. As dexhorthy clarified: "No, context engineering was all about agent design and construction. This is about how you *use* the agent." [Source: raw/Hxlfed14-2028116431876116660.md]

Harrison Chase (@hwchase17) of LangChain provided the sharpest distinction between frameworks and harnesses: "A framework is abstractions... pretty unopinionated. Harnesses are batteries included." [Source: raw/Hxlfed14-2028116431876116660.md]

Akshay Pachaar's April 2026 synthesis "The Anatomy of an Agent Harness" offers the cleanest working definition by building up through three concentric rings: **prompt engineering** crafts the instructions the model receives; **context engineering** manages what the model sees and when; **harness engineering** encompasses both, plus orchestration, tool execution, state persistence, error recovery, verification loops, safety enforcement, and lifecycle management. "The harness is not a wrapper around a prompt. It is the complete system that makes autonomous agent behavior possible." [Source: raw/akshay_pachaar-2041146899319971922.md] Pachaar also cites LangChain's Vivek Trivedy with the canonical formula: **"If you're not the model, you're the harness."** [Source: raw/akshay_pachaar-2041146899319971922.md]

### The Von Neumann Analogy

Beren Millidge's 2023 essay "Scaffolded LLMs as Natural Language Computers" is the architectural antecedent that best explains why harness engineering became a distinct discipline. A raw LLM is a CPU with no RAM, no disk, and no I/O. The context window serves as RAM (fast, limited). External databases function as disk storage (large, slow). Tool integrations act as device drivers. **The harness is the operating system.** As Millidge wrote: "We have reinvented the Von Neumann architecture" because it's a natural abstraction for any computing system. [Source: raw/akshay_pachaar-2041146899319971922.md]

### Precursor: "Designing Agentic Loops" (Sep 2025)

One month before dexhorthy coined "harness engineering" in November 2025, Simon Willison published "Designing agentic loops" (September 30, 2025), identifying the same core idea under a different name. His framing: "An LLM agent is something that runs tools in a loop to achieve a goal. The art of using them well is to carefully design the tools and loop for them to use." [Source: raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md] This was one of the earliest pieces to name the design of the agent loop itself -- the tool selection, the safety mitigations, the credential scoping, the choice of when agentic loops are appropriate -- as a distinct skill. Willison identified that agentic loops work best on problems with clear success criteria amenable to trial-and-error: debugging, performance optimization, dependency upgrades, and Docker optimization, especially when amplified by good test suites. [Source: raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md] When dexhorthy's "harness engineering" thread gained traction, Willison replied noting the overlap between the two framings. The difference is primarily one of emphasis: Willison's "designing agentic loops" focuses on the loop and tool design from the agent builder's perspective, while dexhorthy's "harness engineering" foregrounds the user's configuration and extension of an existing agent system.

## Three-Layer Model

Harrison Chase mapped any agentic system to three distinct layers where learning can occur:

1. **Model**: The model weights themselves (e.g., claude-sonnet, GPT-5). Learning at this layer involves updating weights via SFT, RL (e.g., GRPO), etc. A central challenge is catastrophic forgetting. [Source: raw/hwchase17-2040467997022884194.md]
2. **Harness**: The code that drives the agent, plus any instructions or tools that are always part of the harness. Learning at this layer involves optimizing orchestration code and base instructions. Papers like "Meta-Harness: End-to-End Optimization of Model Harnesses" describe running agents over tasks, evaluating results, then using a coding agent to suggest changes to harness code. [Source: raw/hwchase17-2040467997022884194.md]
3. **Context**: Additional instructions, skills, and memory that live outside the harness and configure it. This is where most harness engineering happens in practice. Learning here can be done at the agent level (e.g., OpenClaw's SOUL.md updated over time), at the tenant/user/org level (e.g., Hex's Context Studio, Decagon's Duet), or both. Updates can happen offline via traces or in the hot path as the agent runs. [Source: raw/hwchase17-2040467997022884194.md]

Applied to Claude Code, this maps as: Model = claude-sonnet etc., Harness = Claude Code itself, User context = CLAUDE.md, /skills, mcp.json. Applied to OpenClaw: Model = many, Harness = Pi + scaffolding, Agent context = SOUL.md, skills from clawhub. [Source: raw/hwchase17-2040467997022884194.md]

Chase emphasizes that traces are the core driver of all three learning loops, and that LangSmith CLI and LangSmith Skills give coding agents access to these traces -- this is how LangChain improved Deep Agents on TerminalBench. [Source: raw/hwchase17-2040467997022884194.md]

## The Four-Layer Extension

Rohit (@rohit4verse) argued that the standard three-layer model is incomplete after analyzing Claude Code's 55-directory, 331-module source. Claude Code implements a four-level CLAUDE.md hierarchy (enterprise, project, user, local), disk-backed task coordination with file-based locking, git worktree isolation for parallel sub-agents, and a permission pipeline cascading deny rules from enterprise to session level. None of that is model, context, or harness in the traditional sense. [Source: raw/rohit4verse-2041548810804211936.md]

Rohit's proposed four-layer model:

1. **Model Weights**: Frozen intelligence.
2. **Context**: Runtime input.
3. **Harness**: The agent's designed environment.
4. **Infrastructure**: Multi-tenancy, RBAC, resource isolation, state persistence, distributed coordination. [Source: raw/rohit4verse-2041548810804211936.md]

"Most teams talk about the first three because they are interesting to think about. The fourth is where products die." [Source: raw/rohit4verse-2041548810804211936.md]

## Benchmark Evidence: The Harness Matters More Than the Model

Himanshu (@Hxlfed14) crystallized the argument with cross-company data in a widely-shared March 2026 thread:

- **CORE-Bench**: Claude Opus 4.5 scores 42% with one scaffold and 78% with another. Same model, same benchmark. Sonnet 4: 33% vs 47%. Sonnet 4.5: 44% vs 62%. [Source: raw/Hxlfed14-2028116431876116660.md]
- **LangChain's deepagents-cli**: Went from 52.8% to 66.5% on TerminalBench 2.0 (a 13.7-point improvement) by changing only the harness. [Source: raw/Hxlfed14-2028116431876116660.md]
- **Cursor's lazy tool loading**: 46.9% token reduction in A/B testing (statistically significant). [Source: raw/Hxlfed14-2028116431876116660.md]
- **Vercel**: Deleted 80% of their agent's tools and watched it go from failing tasks to completing them. Tokens dropped from 145,463 to 67,483, steps from 100 to 19, latency from 724 to 141 seconds. [Source: raw/Hxlfed14-2028116431876116660.md]
- **SWE-Agent (Princeton NLP)**: 64% relative improvement on SWE-bench by changing nothing but the interface design. Same GPT-4. Same tasks. [Source: raw/rohit4verse-2041548810804211936.md]
- **LangChain's deepagents-cli (direct measurement)**: LangChain's own post documents the TerminalBench 2.0 run in detail. Using GPT-5.2-Codex fixed, they moved 52.8% → 66.5% by tuning only system prompt, tools, and middleware. The biggest wins came from a build-verify loop guidance, a PreCompletionChecklistMiddleware that intercepts exit for verification, a LocalContextMiddleware that maps cwd at agent start, a LoopDetectionMiddleware that breaks doom-loops after N edits to the same file, and an "xhigh-high-xhigh" reasoning sandwich (high reasoning during planning and verification, moderate during implementation). [Source: raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md]
- **Claude Code dissection (UCL/MBZUAI, arXiv:2604.14228)**: After analyzing the leaked Claude Code v2.1.88 source, Liu et al. found the core agent loop is a `while-true` cycle with state management. "Most of the code, however, lives in the systems around this loop: a permission system with seven modes and an ML-based classifier, a five-layer compaction pipeline for context management, four extensibility mechanisms (MCP, plugins, skills, and hooks), a subagent delegation and orchestration mechanism, and append-oriented session storage." Akshay Pachaar's summary: **only 1.6% of the Claude Code codebase is AI decision logic; the other 98.4% is operational infrastructure.** The model reasons; the harness does everything else. [Source: raw/arxiv-org-html-2604-14228.md] [Source: raw/akshay_pachaar-2045404494641733962.md]

As Himanshu put it: "The model is the engine. The harness is the car. Nobody buys an engine." [Source: raw/Hxlfed14-2028116431876116660.md]

## The 12 Components of a Production Harness

Pachaar's synthesis across Anthropic, OpenAI, LangChain, CrewAI, AutoGen, and LangGraph identifies 12 distinct components in a production agent harness:

1. **Orchestration loop** — the Thought-Action-Observation (ReAct) heartbeat. Anthropic describes theirs as a "dumb loop" where all intelligence lives in the model.
2. **Tools** — schemas injected into the LLM's context plus registration, validation, sandboxed execution, result capture, and formatting.
3. **Memory** — operates at multiple timescales. Short-term (conversation history). Long-term (CLAUDE.md, MEMORY.md, LangGraph Stores, OpenAI Sessions).
4. **Context management** — the single most common failure site. Compaction, observation masking, just-in-time retrieval, sub-agent delegation.
5. **Prompt construction** — hierarchical assembly: system prompt, tool definitions, memory files, conversation history, current message. OpenAI Codex uses a strict priority stack.
6. **Output parsing** — native tool calling preferred over regex parsing; legacy RetryWithErrorOutputParser remains for edge cases.
7. **State management** — LangGraph models state as typed dicts flowing through graph nodes with reducers; OpenAI offers four mutually exclusive strategies; Claude Code uses git commits as checkpoints.
8. **Error handling** — transient (retry with backoff), LLM-recoverable (return as ToolMessage so the model can adjust), user-fixable (interrupt), unexpected (bubble up).
9. **Guardrails and safety** — OpenAI's tiered input/output/tool guardrails with tripwire halting; Anthropic's ~40 discrete tool capabilities gated independently.
10. **Verification loops** — rules-based (tests, linters), visual (Playwright screenshots), LLM-as-judge. Boris Cherny: "giving the model a way to verify its work improves quality by 2 to 3x."
11. **Subagent orchestration** — Fork (byte-identical parent context), Teammate (file-based mailbox), Worktree (isolated git worktree), agents-as-tools, handoffs, nested state graphs.
12. **(Not explicitly numbered but implied across the article) Persistence and lifecycle** — how state survives across sessions, how old artifacts get compacted or discarded. [Source: raw/akshay_pachaar-2041146899319971922.md]

Pachaar also identifies **seven decisions that define every harness**: single-agent vs. multi-agent, ReAct vs. plan-and-execute, context-window management strategy, verification loop design, permission architecture, tool scoping strategy, and harness thickness. [Source: raw/akshay_pachaar-2041146899319971922.md] See [Thin Harness, Fat Skills](thin-harness-fat-skills.md) for the architectural debate around decision #7.

## Thin Harness, Fat Skills

Garry Tan (@garrytan), President and CEO of Y Combinator, contributed a concrete architectural philosophy after reading Anthropic's accidentally-published Claude Code source (512,000 lines shipped to npm on March 31, 2026). [Source: raw/garrytan-2042925773300908103.md]

The principle: **"thin harness, fat skills"**:

- The **harness** should be thin -- it does four things: runs the model in a loop, reads and writes files, manages context, and enforces safety. About 200 lines of code. JSON in, text out. Read-only by default. [Source: raw/garrytan-2042925773300908103.md]
- The **skills** should be fat -- reusable markdown documents that encode judgment, process, and domain knowledge. This is where 90% of the value lives. A skill file works like a method call: it takes parameters and produces radically different capabilities depending on what you pass in. [Source: raw/garrytan-2042925773300908103.md]

The anti-pattern is a fat harness with thin skills: 40+ tool definitions eating half the context window, God-tools with 2-to-5-second MCP round-trips, REST API wrappers turning every endpoint into a separate tool. Three times the tokens, three times the latency, three times the failure rate. [Source: raw/garrytan-2042925773300908103.md]

Tan's three-layer architecture:

1. **Fat skills on top**: Markdown procedures encoding judgment, process, and domain knowledge.
2. **Thin CLI harness in the middle**: ~200 lines of code.
3. **Application on the bottom**: QueryDB, ReadDoc, Search, Timeline -- the deterministic foundation. [Source: raw/garrytan-2042925773300908103.md]

The principle is directional: push intelligence up into skills, push execution down into deterministic tooling, keep the harness thin. Every improvement to the model automatically improves every skill, while the deterministic layer stays perfectly reliable. [Source: raw/garrytan-2042925773300908103.md]

Tan also introduces the concept of **resolvers** -- routing tables for context that load the right document when a task type appears. Claude Code's built-in resolver uses skill description fields to match user intent automatically. Tan's own CLAUDE.md was 20,000 lines; Claude Code told him to cut it back. The fix was ~200 lines of pointers with the resolver loading the right document on demand. [Source: raw/garrytan-2042925773300908103.md]

## OpenAI's Validation: Zero Hand-Written Code

OpenAI's Codex team provided the most extreme validation. In a February 2026 article, Ryan Lopopolo described building a full internal product with zero hand-written code: ~1 million lines of code, 1,500 merged PRs, 3 engineers (later 7), all written by Codex agents. The humans' role was entirely harness engineering: "design environments, specify intent, and build feedback loops that allow Codex agents to do reliable work." [Source: raw/openai-com-index-harness-engineering.md]

Their key lesson aligned with the broader consensus: AGENTS.md should be a ~100-line table of contents, not a monolithic manual. Knowledge lives in a structured docs/ directory with mechanical enforcement via linters and CI. [Source: raw/openai-com-index-harness-engineering.md]

## Design Patterns as Architectural Vocabulary

The five core agentic design patterns -- **ReAct** (reason-then-act loops), **Reflection** (self-critique and iterative refinement), **Tool Use** (external tool integration), **Planning** (task decomposition and sequencing), and **Multi-Agent** (coordinated specialized agents) -- form the architectural vocabulary of harness engineering. Each pattern addresses a specific failure mode: ReAct prevents blind action without reasoning, Reflection prevents unchecked errors, Planning prevents incoherent multi-step work, and Multi-Agent prevents single points of failure on complex tasks. [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md]

The critical principle: **"Start with the problem, not the pattern."** Pattern selection should be treated as a production architecture decision, not a theoretical exercise. A simple ReAct loop may outperform a complex multi-agent system if the task does not justify the coordination overhead. The harness engineer's job is to match the minimal pattern to the task's actual requirements. [Source: raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md] See [Agentic Design Patterns](agentic-design-patterns.md) for a full treatment of each pattern and its tradeoffs.

## Industry Convergence and Disagreement

Near-consensus across teams: the single flat while(tool_call) loop wins over complex orchestration; file system as extended memory; errors preserved not cleaned; fake planning tools (TodoWrite, todo.md) for coherence; primitives (bash, grep, filesystem) over custom integrations. [Source: raw/Hxlfed14-2028116431876116660.md]

Active disagreement exists on how to handle tool overload. Manus uses logit masking (all ~29 tools permanently loaded, availability controlled via output token probability constraints). Cursor uses lazy loading (tool descriptions fetched on demand). Opposite strategies; both work. [Source: raw/Hxlfed14-2028116431876116660.md]

The pattern worth noting: the teams shipping the best agents keep simplifying. Manus: five rewrites, each one removed things. Anthropic designs Claude Code's scaffold to shrink as models improve. Replit went from one agent to three, but each individual agent got simpler. Over-engineering is the default failure mode. [Source: raw/Hxlfed14-2028116431876116660.md]

Dex Horthy (creator of "12 Factor Agents") puts the threshold at 40% of the model's input capacity: push past that and you enter the "dumb zone" -- signal-to-noise degrades, attention fragments, and agents start making mistakes that look like reasoning failures but are actually information overload. [Source: raw/Hxlfed14-2028116431876116660.md]

## Related

- [Claude Code Architecture](claude-code-architecture.md) -- the most analyzed production harness
- [Thin Harness, Fat Skills](thin-harness-fat-skills.md) -- the architecture thesis (Tan) and the Chase-Tan memory-ownership debate, with the Cursor 3.0 validation
- [Self-Evolving Agents and Skillify](self-evolving-agents.md) -- Autogenesis protocol, skillify 10-step practice, LangChain trace-analyzer skill
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- multi-session patterns that extend harness engineering across context windows
- [OpenAI Codex Harness](openai-codex-harness.md) -- the zero-hand-written-code case study
- [Auto Mode and Safety](auto-mode-and-safety.md) -- harness-level safety via classifiers
- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- memory as core harness function, context curation, and compaction strategies
- [Practical Best Practices](practical-best-practices.md) -- the concrete "how" of harness engineering: progressive disclosure, simplification, sycophancy handling
- [Tool Design Patterns](tool-design-patterns.md) -- "thin harness, fat skills" architecture and action space design principles
- [Agentic Design Patterns](agentic-design-patterns.md) -- ReAct, Reflection, Planning, Tool Use, Multi-Agent as the five core patterns forming the architectural vocabulary of harness engineering
- [Deep Research Agents](deep-research-agents.md) -- orchestrator-worker architectures, convergence detection, and economics of extended agent sessions
- [Multi-Agent Reliability](multi-agent-reliability.md) -- adversary-resistant multi-agent coordination via credibility scoring

## Open Questions

- No standard benchmarks exist for comparing harness designs head-to-head. Cursor's 46.9% token reduction is one of very few published numbers. [Source: raw/Hxlfed14-2028116431876116660.md]
- When to share sub-agent state vs. isolate it is still purely empirical. [Source: raw/Hxlfed14-2028116431876116660.md]
- How architectural coherence evolves over years in a fully agent-generated system remains unknown. [Source: raw/openai-com-index-harness-engineering.md]
- It is unclear whether Rohit's fourth layer (infrastructure) is a genuinely separate concern or an extension of harness engineering. [Source: raw/rohit4verse-2041548810804211936.md]

## Sources

- [raw/openai-com-index-harness-engineering.md](../raw/openai-com-index-harness-engineering.md) -- Ryan Lopopolo, OpenAI, Feb 2026. Building a product with zero hand-written code using Codex agents.
- [raw/Hxlfed14-2028116431876116660.md](../raw/Hxlfed14-2028116431876116660.md) -- Himanshu (@Hxlfed14), Mar 2026. Cross-company analysis of agent harness patterns with benchmark data.
- [raw/garrytan-2042925773300908103.md](../raw/garrytan-2042925773300908103.md) -- Garry Tan (@garrytan), Apr 2026. "Thin harness, fat skills" architecture and five definitions.
- [raw/hwchase17-2040467997022884194.md](../raw/hwchase17-2040467997022884194.md) -- Harrison Chase (@hwchase17), Apr 2026. Three-layer model of continual learning in agentic systems.
- [raw/rohit4verse-2041548810804211936.md](../raw/rohit4verse-2041548810804211936.md) -- Rohit (@rohit4verse), Apr 2026. Four-layer model derived from Claude Code source analysis.
- [raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md](../raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md) -- Simon Willison, Sep 2025. "Designing agentic loops" as a precursor framing to harness engineering, defining the skill of designing tools and loops for agents. YOLO mode risks, shell commands over MCP, tightly scoped credentials.
- [raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md](../raw/machinelearningmastery-com-the-roadmap-to-mastering-agentic-ai-design-patterns.md) -- Machine Learning Mastery, 2025. Five core agentic design patterns (ReAct, Reflection, Tool Use, Planning, Multi-Agent) as architectural vocabulary. "Start with the problem, not the pattern."
- [raw/akshay_pachaar-2041146899319971922.md](../raw/akshay_pachaar-2041146899319971922.md) -- Akshay Pachaar, Apr 2026. "The Anatomy of an Agent Harness" -- 12-component synthesis, Von Neumann analogy, 7 architectural decisions, framework-by-framework comparison.
- [raw/akshay_pachaar-2045404494641733962.md](../raw/akshay_pachaar-2045404494641733962.md) -- Akshay Pachaar, Apr 2026. Summary of UCL/MBZUAI Claude Code dissection -- the 1.6%/98.4% model-vs-infrastructure split.
- [raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md](../raw/langchain-com-blog-improving-deep-agents-with-harness-engineering.md) -- Vivek Trivedy, LangChain, Feb 2026. deepagents-cli from 52.8% to 66.5% on TerminalBench 2.0 via harness-only tuning.
- [raw/hwchase17-2042978500567609738.md](../raw/hwchase17-2042978500567609738.md) -- Harrison Chase, Apr 2026. "Your harness, your memory" -- the memory-ownership argument that anchors the Chase-Tan debate.
- [raw/arxiv-org-html-2604-14228.md](../raw/arxiv-org-html-2604-14228.md) -- Liu et al. (VILA Lab, MBZUAI & UCL), Apr 2026. "Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems" -- source-level analysis, 5 human values, 13 design principles, 7-component structure, 5-layer subsystem architecture.
