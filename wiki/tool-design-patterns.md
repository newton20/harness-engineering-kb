---
title: Tool Design Patterns
type: wiki
tags:
  - tools
  - action-space
  - tool-design
  - mcp
  - agent-architecture
  - context-efficiency
  - filesystem-abstraction
  - infrastructure
sources:
  - raw/trq212-2027463795355095314.md
  - raw/garrytan-2042925773300908103.md
  - raw/Hxlfed14-2028116431876116660.md
  - raw/0xblacklight-2036534699582255329.md
  - raw/gakonst-2036560009128194052.md
  - raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md
source_count: 6
status: draft
last_compiled: 2026-04-13
---

# Tool Design Patterns

The tools available to an agent -- its action space -- are one of the most consequential design decisions in harness engineering. The accumulated evidence from Claude Code, Cursor, Vercel, and others converges on a counterintuitive finding: fewer, simpler tools consistently outperform larger, more sophisticated toolsets. The engineering challenge is matching tool abstraction level to the model's abilities, loading tools efficiently, and separating the tool interface from its execution backend.

## Action Space Design: The Math Analogy

Thariq (Claude Code team, @trq212) identifies tool design as a central challenge: "One of the hardest parts of building an agent harness is constructing its action space." [Source: raw/trq212-2027463795355095314.md]

He offers a framework: imagine giving a model a math problem. Paper would be the minimum, but the model is limited by manual calculations. A calculator is better, but the model needs to know how to operate advanced options. A computer is most powerful, but the model needs to know how to write and execute code. [Source: raw/trq212-2027463795355095314.md] Each tool level changes not just the agent's capabilities but the kind of reasoning it must perform. The right choice depends on matching the tool's abstraction level to the model's abilities: "You want to give it tools that are shaped to its own abilities. But how do you know what those abilities are? You pay attention, read its outputs, experiment. You learn to see like an agent." [Source: raw/trq212-2027463795355095314.md]

## The AskUserQuestion Tool Evolution

The Claude Code team's experience building an AskUserQuestion tool illustrates how tool design requires iteration: [Source: raw/trq212-2027463795355095314.md]

- **Attempt 1 -- Bolted onto ExitPlanTool:** Adding a user-question field to an existing tool confused Claude because it was simultaneously being asked for a plan and a set of questions about the plan. Overloading a tool with multiple responsibilities creates ambiguity the model cannot reliably resolve.
- **Attempt 2 -- Markdown format:** A separate mechanism requiring the model to format questions in specific markdown. The model's adherence was unreliable -- it would append extra sentences, omit options, or use a different format altogether.
- **Attempt 3 -- Dedicated tool:** A standalone tool with a single, clear purpose -- ask the user a question and return the answer. This worked. The model understood when and how to use it correctly.

The lesson: tools must have a single, clear purpose with an interface that matches how the model naturally wants to express intent. [Source: raw/trq212-2027463795355095314.md]

## Delete Tools to Improve Performance

Vercel's experience provides a striking data point: they deleted 80% of their agent's tools, and the agent went from failing tasks to completing them. Token usage dropped from 145,463 to 67,483, steps dropped from 100 to 19, and latency dropped from 724 to 141 seconds. [Source: raw/Hxlfed14-2028116431876116660.md]

Claude Opus 4.5 scored 42% on CORE-Bench with one scaffold and 78% with another -- same model, same benchmark, only variable is the harness. [Source: raw/Hxlfed14-2028116431876116660.md] This is counterintuitive. The instinct is to give the agent more capabilities to handle more situations. But each tool competes for the model's attention and decision-making bandwidth. A smaller, well-designed toolset frequently outperforms a larger, comprehensive one.

## Lazy Tool Loading

Cursor's approach to managing large tool sets is lazy loading: tools are loaded into context only when they become relevant to the current task. Cursor syncs tool descriptions to a folder structure and gives the agent only tool names as static context. Full definitions are fetched on-demand. In A/B testing, this cut token usage by **46.9%** (statistically significant). [Source: raw/Hxlfed14-2028116431876116660.md]

Manus takes a different approach to the same problem: rather than adding and removing tools dynamically (which invalidates the KV-cache), all ~29 tools remain permanently loaded and availability per step is controlled by constraining output token probabilities during decoding (logit masking). [Source: raw/Hxlfed14-2028116431876116660.md] Both approaches work; the right answer may depend on token economics.

## Thin Harness, Fat Skills

Garry Tan articulates the "thin harness, fat skills" architecture: the harness should be minimal -- about 200 lines of code doing four things (run the model in a loop, read and write files, manage context, enforce safety). The actual capabilities live in fat skill files -- reusable markdown documents that encode judgment, process, and domain knowledge. [Source: raw/garrytan-2042925773300908103.md]

The anti-pattern is a fat harness with thin skills: "40+ tool definitions eating half the context window. God-tools with 2-to-5-second MCP round-trips. REST API wrappers that turn every endpoint into a separate tool. Three times the tokens, three times the latency, three times the failure rate." [Source: raw/garrytan-2042925773300908103.md]

What you want instead is purpose-built tooling that is fast and narrow. Tan gives a concrete example: a Playwright CLI that does each browser operation in 100 milliseconds versus a Chrome MCP that takes 15 seconds for screenshot-find-click-wait-read -- 75x faster. [Source: raw/garrytan-2042925773300908103.md]

The three-layer architecture: fat skills on top (markdown procedures encoding judgment), a thin CLI harness in the middle (JSON in, text out), and deterministic application tooling on the bottom (QueryDB, ReadDoc, Search, Timeline). Push intelligence up into skills, push execution down into deterministic tooling, keep the harness thin. [Source: raw/garrytan-2042925773300908103.md]

## Primitives Over Integrations

Anthropic's tooling choices for Claude Code are instructive. For code search, they chose ripgrep -- a fast, well-known command-line tool -- over a vector database with semantic search. Claude Code has approximately 18 primitive tools in four categories: command-line discovery (Bash, Glob, Grep, LS), file interaction (Read, Write, Edit, MultiEdit), web access (WebSearch, WebFetch), and orchestration (TodoWrite, Task). [Source: raw/Hxlfed14-2028116431876116660.md]

The general principle: prefer primitive, well-known tools that the model already understands over sophisticated integrations that require the model to learn novel interfaces. A tool the model uses correctly 95% of the time beats a more powerful tool it uses correctly 70% of the time.

Simon Willison makes this case even more sharply: rather than building MCP integrations, give the agent shell commands via an AGENTS.md file. A single worked example of the right shell command is enough for agents to generalize to related tasks. This avoids the overhead of MCP tool definitions and round-trips entirely, leaning on the model's existing fluency with command-line interfaces. [Source: raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md] The implication aligns with the "thin harness, fat skills" philosophy: encoding capability as documented shell invocations in skill files rather than as formal tool definitions keeps the harness thin and the agent's action space legible.

## TodoWrite as a No-Op Planning Tool

Claude Code's TodoWrite tool does nothing functionally. It is purely a harness-level trick -- a no-op tool that forces the agent to articulate and track its plan, keeping it on course over long trajectories. LangChain's "Deep Agents" analysis calls it out explicitly. [Source: raw/Hxlfed14-2028116431876116660.md]

The model does not know that TodoWrite is a no-op. It treats it as a real tool call, which means it invests the same reasoning effort it would for any other tool. The harness exploits this to get better planning behavior. As models improved, TodoWrite was replaced by the Task Tool, which enabled subagent coordination with dependencies and shared updates -- illustrating how tool design must evolve with model capabilities. [Source: raw/trq212-2027463795355095314.md]

## Filesystem Abstraction: Separating Interface from Execution

Kyle Mistele (@0xblacklight) argues that the real insight is that agents do not actually need a filesystem -- they just need something that looks like a filesystem. Agents do not see the POSIX APIs; they just see tokens in and tokens out. [Source: raw/0xblacklight-2036534699582255329.md]

This means you can separate the tool **interface** from the tool **execution**: "your tool can look like a normal FS read tool to the agent, but you can use whatever backend you want for the execution logic" -- S3, Postgres, Chroma, durable streams, whatever. [Source: raw/0xblacklight-2036534699582255329.md] The problem with off-the-shelf harnesses is that they marry you to the actual filesystem, forcing FUSE or NFS hacks or parallel MCP tools. If you own the harness, you own the control flow. [Source: raw/0xblacklight-2036534699582255329.md]

## Postgres as Coordinator

Georgios Konstantopoulos (@gakonst) describes a production agent architecture where Postgres is the coordinator: "our internal agent is: 1. a postgres that coordinates everything & persists msgs, 2. an API that spawns docker containers piping container stdin to amp's stdin and streaming out stdout as SSE, 3. 150+ API keys / tool integrations that the agent can call (standard REST API), 4. a firewall that ensures containers have no secrets and all secrets get injected on the fly when they leave the container + audit logs + alerts, 5. a slackbot interface." [Source: raw/gakonst-2036560009128194052.md]

This is a concrete example of putting deterministic infrastructure (database, firewall, container orchestration) below the agent while keeping the agent's tool interface simple -- the agent calls standard REST APIs; the infrastructure handles security, persistence, and coordination.

## Design Principles Summary

The accumulated wisdom on tool design converges on several principles:

1. **Fewer tools are better.** Every tool is a decision the model must make. Minimize the decision space. [Source: raw/Hxlfed14-2028116431876116660.md]
2. **Single responsibility.** Each tool should do one thing with a clear interface. Overloaded tools confuse models. [Source: raw/trq212-2027463795355095314.md]
3. **Match the model's training.** Tools that resemble common programming interfaces (file I/O, shell commands, HTTP requests) work better than novel abstractions. [Source: raw/Hxlfed14-2028116431876116660.md]
4. **Load lazily.** Do not pay the context cost of tools the agent is not currently using. [Source: raw/Hxlfed14-2028116431876116660.md]
5. **Prefer speed.** Fast tools enable tight feedback loops. Slow tools (multi-second MCP round-trips) break the agent's flow. [Source: raw/garrytan-2042925773300908103.md]
6. **Use no-op tools for reasoning.** Tools do not have to perform actions to be valuable. Structuring the agent's thought process is a legitimate tool purpose. [Source: raw/Hxlfed14-2028116431876116660.md]
7. **Iterate on the interface.** The first design of a tool is rarely right. Expect to revise based on how the model actually uses it. As model capabilities change, tools that were once necessary may become constraining. [Source: raw/trq212-2027463795355095314.md]
8. **Separate interface from execution.** The tool's appearance to the agent and its backend implementation are independent concerns. [Source: raw/0xblacklight-2036534699582255329.md]

## Related

- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- Filesystem tools as memory tools, the filesystem-vs-database debate
- [Claude Code Architecture](claude-code-architecture.md) -- Claude Code's ~18 primitive tools, TodoWrite/Task evolution, progressive disclosure
- [Practical Best Practices](practical-best-practices.md) -- Progressive disclosure, simplest solution first, model generation rethinking
- [What Is Harness Engineering?](what-is-harness-engineering.md) -- The harness as the product, the core agent loop

## Open Questions

- How to handle tool overload remains an active disagreement: Manus uses logit masking (all tools loaded, constrain outputs), Cursor uses lazy loading (load tool definitions on demand). Opposite strategies, both work. [Source: raw/Hxlfed14-2028116431876116660.md]
- When should you build your own harness to control the tool abstraction layer vs. use an off-the-shelf harness? Owning the harness gives you control over the interface-execution separation, but at significant engineering cost. [Source: raw/0xblacklight-2036534699582255329.md]
- No standard benchmarks exist for comparing harness/tool designs head-to-head. When to share sub-agent state vs. isolate it is still purely empirical. [Source: raw/Hxlfed14-2028116431876116660.md]

## Sources

- [raw/trq212-2027463795355095314.md](../raw/trq212-2027463795355095314.md) -- Thariq (Claude Code team) on action space design, the math analogy, AskUserQuestion evolution, TodoWrite to Task transition, progressive disclosure.
- [raw/garrytan-2042925773300908103.md](../raw/garrytan-2042925773300908103.md) -- Garry Tan on thin harness / fat skills architecture, skill files as method calls, resolvers, latent vs. deterministic work, 75x speed difference with purpose-built tools.
- [raw/Hxlfed14-2028116431876116660.md](../raw/Hxlfed14-2028116431876116660.md) -- Himanshu's comprehensive survey of harness architectures. Vercel 80% tool deletion, Cursor lazy loading (46.9%), CORE-Bench scaffold comparison, progressive disclosure across systems.
- [raw/0xblacklight-2036534699582255329.md](../raw/0xblacklight-2036534699582255329.md) -- Kyle Mistele on filesystem abstraction, separating tool interface from execution, FUSE-based approaches, owning the harness for control flow.
- [raw/gakonst-2036560009128194052.md](../raw/gakonst-2036560009128194052.md) -- Georgios Konstantopoulos on production agent architecture with Postgres as coordinator, Docker containers, 150+ API integrations, firewall-based secret injection.
- [raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md](../raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md) -- Simon Willison, Sep 2025. Shell commands over MCP, AGENTS.md with worked examples, agents generalize from a single example.
