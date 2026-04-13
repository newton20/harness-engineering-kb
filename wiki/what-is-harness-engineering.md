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
  - raw/rohit4verse-2041548810804211936.md
  - raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md
source_count: 6
status: draft
last_compiled: 2026-04-13
---

Harness engineering is the discipline of configuring, extending, and structuring the integration points of an existing AI agent to maximize its effectiveness for a particular task or team. The term was coined by @dexhorthy in a November 4, 2025 thread on X, where he observed that the people getting dramatically better outcomes from coding agents were not using better models -- they were engineering the wrapper around the model more effectively. [Source: raw/Hxlfed14-2028116431876116660.md] The concept has since been validated by benchmark evidence, independent practitioners, and both Anthropic and OpenAI, establishing it as one of the central organizing ideas in AI agent development.

## Definition and Origin

In dexhorthy's original formulation, harness engineering is distinct from context engineering. Context engineering concerns how context -- long or short, agentic or not -- is passed to an LLM to get the best results, and is primarily about agent design and construction. Harness engineering concerns how a user or team configures, extends, and instruments an existing agent to maximize its effectiveness. As dexhorthy clarified: "No, context engineering was all about agent design and construction. This is about how you *use* the agent." [Source: raw/Hxlfed14-2028116431876116660.md]

Harrison Chase (@hwchase17) of LangChain provided the sharpest distinction between frameworks and harnesses: "A framework is abstractions... pretty unopinionated. Harnesses are batteries included." [Source: raw/Hxlfed14-2028116431876116660.md]

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

As Himanshu put it: "The model is the engine. The harness is the car. Nobody buys an engine." [Source: raw/Hxlfed14-2028116431876116660.md]

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

## Industry Convergence and Disagreement

Near-consensus across teams: the single flat while(tool_call) loop wins over complex orchestration; file system as extended memory; errors preserved not cleaned; fake planning tools (TodoWrite, todo.md) for coherence; primitives (bash, grep, filesystem) over custom integrations. [Source: raw/Hxlfed14-2028116431876116660.md]

Active disagreement exists on how to handle tool overload. Manus uses logit masking (all ~29 tools permanently loaded, availability controlled via output token probability constraints). Cursor uses lazy loading (tool descriptions fetched on demand). Opposite strategies; both work. [Source: raw/Hxlfed14-2028116431876116660.md]

The pattern worth noting: the teams shipping the best agents keep simplifying. Manus: five rewrites, each one removed things. Anthropic designs Claude Code's scaffold to shrink as models improve. Replit went from one agent to three, but each individual agent got simpler. Over-engineering is the default failure mode. [Source: raw/Hxlfed14-2028116431876116660.md]

Dex Horthy (creator of "12 Factor Agents") puts the threshold at 40% of the model's input capacity: push past that and you enter the "dumb zone" -- signal-to-noise degrades, attention fragments, and agents start making mistakes that look like reasoning failures but are actually information overload. [Source: raw/Hxlfed14-2028116431876116660.md]

## Related

- [Claude Code Architecture](claude-code-architecture.md) -- the most analyzed production harness
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- multi-session patterns that extend harness engineering across context windows
- [OpenAI Codex Harness](openai-codex-harness.md) -- the zero-hand-written-code case study
- [Auto Mode and Safety](auto-mode-and-safety.md) -- harness-level safety via classifiers
- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- memory as core harness function, context curation, and compaction strategies
- [Practical Best Practices](practical-best-practices.md) -- the concrete "how" of harness engineering: progressive disclosure, simplification, sycophancy handling
- [Tool Design Patterns](tool-design-patterns.md) -- "thin harness, fat skills" architecture and action space design principles

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
