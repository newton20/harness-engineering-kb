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
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/code-research-claude-code-2026-04-15.md
  - raw/code-research-karpathy-autoresearch-2026-04-15.md
  - raw/code-research-openclaw-openclaw-2026-04-15.md
  - raw/code-research-all-hands-ai-openhands-2026-04-15.md
  - raw/code-research-anomalyco-opencode-2026-04-15.md
  - raw/code-research-666ghj-mirofish-2026-04-15.md
  - raw/code-research-claude-code-2026-04-14.md
  - raw/code-research-openclaw-openclaw-2026-04-14.md
source_count: 15
status: draft
last_compiled: 2026-04-15
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

## Deferred Tool Loading via ToolSearchTool

Claude Code's source reveals a concrete implementation of lazy loading that goes beyond Cursor's approach. MCP tools and flagged built-ins start in a "deferred" state where the model sees only tool names via a `tool_reference` API content type -- no schemas are loaded. The model must explicitly call `ToolSearchTool` to load a tool's full schema before first use, saving approximately 11K tokens of tool schemas that would otherwise occupy context from the start. Discovered tool schemas persist via message history scanning so they survive compaction. Delta notifications keep the model updated when tool availability changes. [Source: raw/code-research-claude-code-2026-04-15.md]

The persistence mechanism is worth examining closely: once a tool schema is loaded, the harness scans message history for previously-discovered tool schemas on every turn via `extractDiscoveredToolNames`. This means the schema survives autocompaction -- the compaction process writes `compactMetadata.preCompactDiscoveredTools`, which the post-compact recovery logic uses to re-announce the previously-loaded schemas via `deferred_tools_delta` attachment messages. The model is never surprised by a tool disappearing after compaction. This is a non-obvious invariant: lazy loading normally creates a risk that compaction will evict a tool the model expects to be available; the history-scanning mechanism explicitly closes that gap. [Source: raw/code-research-claude-code-2026-04-14.md]

## Tool Concurrency Partitioning

Claude Code partitions tool execution into read-safe and write operations. Read-safe tools (Glob, Grep, Read) are batched for parallel execution -- up to 10 concurrent calls per turn -- yielding roughly 10x throughput on read-heavy turns. Write operations always run serially. Context modifiers from concurrent tools are queued and applied only after the entire batch completes. This partitioning enables fast data-gathering phases while preventing the race conditions that would arise from concurrent writes. [Source: raw/code-research-claude-code-2026-04-15.md]

## buildTool() Factory and Error-as-Context

Claude Code registers its 50+ tools through a `buildTool()` factory with fail-closed defaults -- tools must explicitly opt into capabilities rather than opt out of restrictions. All tool errors are returned as `<tool_use_error>` content in the `tool_result` message rather than throwing exceptions. The harness performs no automatic retries; instead, the model reads the error and decides how to proceed. This error-as-context pattern produces simpler harness code and smarter error handling, since the model can reason about what went wrong and adapt its approach rather than blindly retrying the same operation. [Source: raw/code-research-claude-code-2026-04-15.md]

The fail-closed default is enforced at the factory level: no individual tool author can accidentally leave a capability gate open by omitting a field. Every tool starts locked and must explicitly grant permissions via named flags. This is a security design choice as much as an engineering one -- the factory ensures that new tools added to the registry cannot inadvertently bypass the harness's permission layer. [Source: raw/code-research-claude-code-2026-04-14.md]

## Prose-as-Schema: Tool APIs in Natural Language

Karpathy's autoresearch takes an extreme position on tool design: the entire tool API is defined as natural language in a 115-line markdown file (program.md). There are no JSON schemas, no typed interfaces, no MCP integration. The agent's available tools are 6-8 shell commands (git, edit, uv run, grep, tail) described inline in numbered prose steps. Tool selection is not an LLM decision -- it is a hardcoded sequence in the loop definition, with the LLM's creativity confined to a single degree of freedom: what mutation to make to the code. This demonstrates that zero tool infrastructure is viable when the tool sequence is deterministic and the creative space is deliberately narrow. [Source: raw/code-research-karpathy-autoresearch-2026-04-15.md]

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

## Tool Descriptions as a First-Class Concern

Anthropic's multi-agent Research feature revealed that tool descriptions are far more consequential than most harness engineers realize. During development, a dedicated **tool-testing agent** was used to evaluate and rewrite tool descriptions. The rewritten descriptions resulted in a **40% decrease in task completion time** -- a dramatic improvement from changing nothing but the text the agent reads about its tools. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

The underlying problem: bad tool descriptions send agents down completely wrong paths. When an agent encounters a tool with a vague or misleading description, it may select the wrong tool entirely, use the right tool with incorrect parameters, or waste tokens on exploratory tool calls trying to figure out what a tool does. In multi-agent systems where subagents encounter unseen MCP tools with varying description quality, this problem compounds -- each subagent independently misinterprets the same bad description. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

The heuristic that emerged: agents should examine all available tools first, match tool usage to user intent, and prefer specialized tools over generic ones. This is not just prompt engineering advice -- it is a structural design decision about how the harness presents tools to the agent. If you have both a generic "web_search" tool and a specialized "academic_search" tool, the descriptions must make the specialization clear enough that the agent reliably chooses the right one for the task. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

This finding reinforces the broader principle that tool design is iterative (see "The AskUserQuestion Tool Evolution" above), but adds a specific lever: if your agent is underperforming, rewrite your tool descriptions before adding new tools or changing the agent's prompt.

## Per-Provider Schema Normalization

OpenClaw reveals a production-scale challenge not addressed by simpler harnesses: tool schemas must be adapted per LLM provider. Tools are defined once in TypeBox (`@sinclair/typebox`), but a `normalizeToolParameterSchema()` function normalizes schemas at call time for each provider -- flattening `anyOf`/`oneOf` unions for OpenAI strict mode, stripping unsupported keywords for Gemini and xAI, enforcing `additionalProperties: false` for OpenAI strict mode, and validating compatibility before enabling strict mode. If any tool in the inventory fails the `isStrictOpenAIJsonSchemaCompatible()` check, the harness silently falls back to non-strict mode. MCP tools pass through the same normalization pipeline, receiving identical treatment to core tools. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

This is a provider portability abstraction: tool authors write schemas once, and the normalization layer handles all provider quirks. The practical lesson for harness engineers: if you support multiple LLM providers, plan for a normalization layer between your tool definitions and the API calls.

The TypeBox-first approach has a concrete advantage over JSON Schema-first: TypeBox schemas are TypeScript types at compile time and JSON Schema at runtime, so the same definition drives both static type checking and runtime validation without duplication. The `normalizeToolParameterSchema()` function operates on the JSON Schema output of TypeBox, making it provider-agnostic -- it does not need to understand TypeBox internals. New providers require only a new normalization branch in that function, not changes to any tool definition. This write-once, run-anywhere approach was identified as a key architectural choice in the original code research. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## Streaming JSON Argument Repair

A related finding from OpenClaw: provider-specific bugs in tool call argument streaming are common enough to warrant dedicated repair logic. The harness includes a stream wrapper that tracks partial JSON buffers per tool call content index, attempts `JSON.parse` on each partial, and falls back to `extractBalancedJsonPrefix` when parsing fails. Specific repairs target named providers -- Kimi (JSON with leading/trailing garbage) and xAI (HTML entity encoding in arguments). The repair runs in the streaming pipeline, fixing partial events before they reach the agent core. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

Additionally, tool name normalization goes 4 levels deep: exact match, case-insensitive match, structured segment matching (splitting on `.` and `/`), and tool-call-ID inference (extracting names from IDs like `functions.tool_name.1`). An unknown-tool loop guard detects when the model repeatedly calls a hallucinated tool name and rewrites the assistant's message to a self-corrective instruction. This depth of fallback reflects the reality of multi-provider deployments where model output quality varies significantly. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md] [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## Skills as Prompt Injections (Not Tools)

OpenClaw draws a sharp architectural distinction between tools and skills. Skills are NOT registered as callable tools. Instead, the system prompt injects an XML `<available_skills>` catalog containing name, description, and file location for each available skill. The model is instructed to read the SKILL.md file via its `read` tool only when a skill clearly applies -- never preloading multiple skill files. This means skills are prompt injections that instruct the model to use its existing tools in domain-specific ways, not additions to the tool set. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

This aligns with Garry Tan's "thin harness, fat skills" architecture but makes the mechanism concrete: skills live as markdown files in the workspace, are advertised cheaply via a catalog, and are loaded lazily via the model's own tools. The harness stays thin (no skill-specific tools), the skills stay fat (full procedural instructions), and the model mediates the loading decision. This is the first concrete implementation we've documented of the thin-harness/fat-skills pattern at production scale. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

## Security Risk as Tool Parameter

OpenHands inverts the conventional model for safety checks: rather than the harness classifying each action, the LLM self-labels it. Every executable action in OpenHands carries a mandatory `security_risk: LOW|MEDIUM|HIGH` field that the model must fill in before the action executes. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

The classification is then passed to a pluggable `SecurityAnalyzer` interface -- implementations include Invariant, GraySwan, and a generic LLM-based analyzer -- which can override the model's self-assessment and block the action. This produces two complementary safety signals: the model's first-person judgment about what it is about to do, and an independent classifier's judgment. Either can veto the action. The arrangement is novel because the agent becomes a participant in its own oversight rather than a passive subject of external review. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

## MCP stdio-over-HTTP Proxy in Sandbox

OpenHands solves a distribution problem that affects any sandboxed harness attempting to use MCP: stdio-based MCP servers must run inside the sandbox, but the outer coordinator (which manages the conversation loop) lives outside. OpenHands runs stdio MCP servers inside the Docker action execution server, then wraps them with a FastMCP proxy that exposes them as an HTTP endpoint reachable by the outer coordinator. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

This makes sandboxed MCP servers transparently accessible without breaking the isolation boundary. The pattern generalizes to any architecture where execution is sandboxed but tool interfaces must be accessible from an outer orchestration layer.

## Action-as-Typed-Dataclass with Runtime Reflection

OpenHands defines all agent actions as Python dataclasses, which gives them typed fields, default values, and serialization for free. Dispatch is done by reflection: `getattr(self, action_type)(action)`. This is clean and concise, but it creates a coupling constraint -- the string values in the action-type enum must stay synchronized with the method names on the handler object at runtime. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

The pattern is a good default for harnesses where the action space is stable, but becomes a maintenance hazard in harnesses where actions are added frequently, since a typo or renaming in either the enum or the method silently routes actions to a missing handler.

## Context Condensation as Agent-Callable Tool

OpenHands exposes a `request_condensation` tool that the agent can call proactively to trigger memory compression. Rather than waiting for the harness to detect context overflow and act, the agent can observe its own context load and request compression when it judges it appropriate. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]

This is a concrete implementation of the principle that context management can be a deliberate LLM action rather than purely an infrastructure concern (compare with Claude Code's compaction, which is harness-triggered). It treats the agent as a self-aware participant in its own resource management.

## Fuzzy Edit Tolerance: The Nine-Strategy Cascade

OpenCode implements the most sophisticated fuzzy edit matching found in any open harness, using a nine-strategy cascade to apply LLM-generated edits even when they don't exactly match the file contents. The strategies run in order until one succeeds: [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

1. Exact match
2. Line-trimmed match
3. Block-anchor match (Levenshtein similarity)
4. Whitespace-normalized match
5. Indentation-flexible match
6. Escape-normalized match
7. Trimmed-boundary match
8. Context-aware match
9. Multi-occurrence match

This cascade makes the edit tool extremely tolerant of LLM formatting drift -- the model does not need to reproduce file content character-for-character to have its edits applied correctly. The practical benefit is fewer failed edits and fewer retry loops, at the cost of more complex harness code.

## Tree-Sitter Bash AST for Permission Detection

When the agent requests permission to run shell commands, OpenCode does not parse the command string with regular expressions. It uses tree-sitter-bash to build an AST of the command, then walks the AST to extract file paths from file-modifying operations, and converts those paths to glob patterns for the permission request. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

Structural parsing produces more reliable path extraction than string matching -- it handles quoting, variable expansion, and compound commands correctly. The resulting glob patterns give users a precise, readable summary of what files an approved command will touch, rather than requiring them to parse raw shell syntax.

## Model-Gated Tool Selection

OpenCode's tool registry is model-aware: when the active model is GPT-4 family, the harness automatically substitutes `apply_patch` for the standard `edit` and `write` tools. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

This is a concrete implementation of the observation that different models have different tool aptitudes. Rather than exposing a single tool interface and hoping all models use it equally well, the registry swaps tools based on model identity at registration time. The pattern generalizes: harnesses that support multiple providers should consider maintaining per-model tool variants for operations where model aptitude varies significantly.

## Description-as-Template from .txt Sidecar Files

OpenCode externalizes tool descriptions into separate `.txt` sidecar files with `${variable}` placeholders for runtime substitution. At registration time, the harness reads the sidecar, performs variable substitution, and attaches the result as the tool's description. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

This separates prompt engineering from code. Tool descriptions can be edited, versioned, and A/B tested without touching the implementation. It also makes the codebase legible: engineers working on tool logic and engineers working on tool descriptions can operate independently. The approach is a lightweight alternative to a full prompt management system for harnesses of moderate complexity.

## The `invalid` Tool as First-Class Error Handler

Both OpenHands and OpenCode register an `invalid` tool as a real, callable tool. When the model calls a tool that does not exist, or when a tool call's arguments fail schema validation, the call is routed to `invalid` rather than raising an exception. `invalid` returns a structured result explaining what went wrong. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

This converts tool-call failures from control-flow exceptions into ordinary tool results that the model can read and respond to. The model sees the same message-result structure it sees for successful tool calls, which means it can reason about the error using the same inference process it uses for everything else. The pattern extends the error-as-context principle (see "buildTool() Factory and Error-as-Context" above) to the case where the error occurs before the tool even executes.

## Uniform Truncation Middleware

OpenCode wraps every tool with a `Tool.wrap()` middleware that applies output truncation uniformly. When a tool's output exceeds the limit, the truncation hint is agent-aware: if a subagent (explore) is available, it suggests delegating to the subagent; otherwise it suggests using Grep or Read with an offset to access the remaining content. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

The agent-aware hint is significant. A generic "output truncated" message leaves the model to figure out how to get the rest of the content. A hint that names the specific tool and parameter (e.g., `Read` with `offset: 150`) gives the model an actionable next step, reducing the turn cost of recovering from truncation.

## LSP Integration as a Tool

OpenCode (experimental) exposes nine language server protocol operations as a single tool: `goToDefinition`, `findReferences`, `hover`, `callHierarchy`, `rename`, `codeAction`, `completion`, `diagnostic`, and `format`. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

LSP integration gives the agent structural code navigation that would otherwise require either shell invocations of analysis tools or direct file reading and AST construction. The fact that it is a single tool with an operation parameter rather than nine separate tools preserves the "fewer tools are better" principle while still exposing the full LSP surface.

## Remote Skill CDN Discovery

OpenCode implements a marketplace-style skill distribution mechanism. `Discovery.pull(url)` fetches an `index.json` from a remote URL listing available skills, caches the index locally, and makes the skills available for loading. This is the CDN model applied to agent capabilities: skill publishers host an index, consumers pull and cache on demand. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

This extends the skills-as-prompt-injections pattern (see "Skills as Prompt Injections" above) to a distributed setting. The harness stays thin (no built-in domain knowledge), skills stay fat (full procedural instructions), and skill distribution is handled by a simple HTTP pull from a remote index rather than by shipping updated harness versions.

## Minimum Tool-Call Enforcement

MiroFish's ReportAgent enforces a hard floor before accepting a Final Answer: at least 3 tool calls must be made per reporting section, with a maximum of 5. Unused-tool hints are injected into the agent's context to actively nudge it toward tool diversity — steering it away from reusing the same tool repeatedly. This is a quality guardrail that forces evidence breadth before synthesis: the agent cannot shortcut to a conclusion without demonstrating it has gathered sufficient evidence from multiple angles. [Source: raw/code-research-666ghj-mirofish-2026-04-15.md]

## Asymmetric Tool Clearing in Compaction

Claude Code's microcompaction applies asymmetric clearing to read and write tools. Read-tool results (Bash, Glob, Grep, FileRead) are cleared from context during compaction because they can always be re-fetched. Write-tool history (FileEdit, FileWrite, NotebookEdit) is never cleared, because losing those records would make it impossible to audit what changes were made to the codebase. This asymmetry reflects a key invariant: reads are reproducible, writes are not. The practical consequence is that compaction can safely shrink context from read-heavy exploration phases without sacrificing the immutable audit trail of mutations. [Source: raw/code-research-claude-code-2026-04-15.md]

## Permission Rule DSL

Claude Code's permission system uses a `ToolName(content)` grammar where parentheses in content are escaped. Shell matching operates at three tiers: exact literal match, prefix match (e.g., `npm:*` matches any npm subcommand), and wildcard match (e.g., `git *` compiles to a regex with dotAll mode to handle heredoc arguments spanning multiple lines). The system includes shadowed rule detection: if a broad deny rule would make a specific allow rule unreachable, a warning is emitted at load time. This prevents silent misconfiguration where a developer adds a targeted carve-out that a broader deny rule silently overrides. [Source: raw/code-research-claude-code-2026-04-15.md]

## Head+Tail Truncation with 30% Tail Budget

OpenClaw's tool result truncation uses a head+tail strategy rather than simple head truncation. When a result exceeds the size limit, the harness preserves the beginning of the output at 70% of the budget and the end at 30%. The tail budget exists specifically to preserve error output and stack traces, which typically appear at the end of command output. A naive head-only truncation would silently discard the error that explains why the tool call failed -- the worst possible outcome for a model trying to reason about what went wrong. The 30% tail allocation is a deliberate judgment about where diagnostic information lives in typical tool output. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## Tool Policy Pipeline

OpenClaw applies tool availability through a multi-step filtering pipeline rather than a single permission check. The pipeline has four stages: profile-level filtering (based on the active configuration profile), provider-level filtering (tools unavailable for the current LLM provider are removed), agent-level filtering (tools restricted to specific agent types), and group-level filtering (fine-grained access control by user or team group). Each stage operates independently and can remove tools without affecting the others. This means a tool can be excluded for four independent reasons, and adding a tool to one stage does not implicitly grant access at other stages. The pipeline approach makes access control auditable: you can inspect each stage's output to understand exactly why a tool is or is not available for a given request. [Source: raw/code-research-openclaw-openclaw-2026-04-14.md]

## Plugin Security Scanning

OpenClaw gates every plugin install through 4 scan functions before the plugin code is allowed to run. Results are cached in an mtime-keyed file cache with a 5000-entry capacity, making re-checks on unchanged files nearly free. Prompt injection detection uses 13 patterns covering classic attack vectors. The mtime key invalidates cache entries automatically whenever a plugin file is modified, so the security scan is never skipped on updated plugins while still avoiding redundant scans on unchanged ones. [Source: raw/code-research-openclaw-openclaw-2026-04-15.md]

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
9. **Make agents participants in their own oversight.** Security checks and context management can be deliberate agent actions -- self-labeling risk, requesting condensation -- rather than purely external constraints. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]
10. **Build for formatting drift.** LLMs do not reproduce file content character-for-character. Edit tools need fuzzy matching cascades, not exact-match assumptions. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]
11. **Route errors to the model, not the exception handler.** An `invalid` tool that catches schema mismatches and unknown tool names produces recoverable, model-readable failures instead of crashes. [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Related

- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- Filesystem tools as memory tools, the filesystem-vs-database debate
- [Claude Code Architecture](claude-code-architecture.md) -- Claude Code's ~18 primitive tools, TodoWrite/Task evolution, progressive disclosure
- [Practical Best Practices](practical-best-practices.md) -- Progressive disclosure, simplest solution first, model generation rethinking
- [What Is Harness Engineering?](what-is-harness-engineering.md) -- The harness as the product, the core agent loop
- [Deep Research Agents](deep-research-agents.md) -- tool-testing agents and parallel tool calling as key deep research optimizations
- [Agentic Design Patterns](agentic-design-patterns.md) -- Tool Use as one of the five core agentic design patterns
- [Multi-Agent Reliability](multi-agent-reliability.md) -- how tool selection interacts with credibility scoring in adversarial multi-agent settings
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) -- prose-as-schema and deterministic tool sequences in autonomous research loops

## Open Questions

- How to handle tool overload remains an active disagreement: Manus uses logit masking (all tools loaded, constrain outputs), Cursor uses lazy loading (load tool definitions on demand). Opposite strategies, both work. [Source: raw/Hxlfed14-2028116431876116660.md]
- When should you build your own harness to control the tool abstraction layer vs. use an off-the-shelf harness? Owning the harness gives you control over the interface-execution separation, but at significant engineering cost. [Source: raw/0xblacklight-2036534699582255329.md]
- No standard benchmarks exist for comparing harness/tool designs head-to-head. When to share sub-agent state vs. isolate it is still purely empirical. [Source: raw/Hxlfed14-2028116431876116660.md]
- Agent self-labeling of security risk (OpenHands) vs. external classification: which produces fewer false negatives? The inverted model is novel but unvalidated at scale. [Source: raw/code-research-all-hands-ai-openhands-2026-04-15.md]
- At what point does a fuzzy edit cascade (OpenCode's nine strategies) create more problems than it solves by silently applying edits that don't match intent? [Source: raw/code-research-anomalyco-opencode-2026-04-15.md]

## Sources

- [raw/trq212-2027463795355095314.md](../raw/trq212-2027463795355095314.md) -- Thariq (Claude Code team) on action space design, the math analogy, AskUserQuestion evolution, TodoWrite to Task transition, progressive disclosure.
- [raw/garrytan-2042925773300908103.md](../raw/garrytan-2042925773300908103.md) -- Garry Tan on thin harness / fat skills architecture, skill files as method calls, resolvers, latent vs. deterministic work, 75x speed difference with purpose-built tools.
- [raw/Hxlfed14-2028116431876116660.md](../raw/Hxlfed14-2028116431876116660.md) -- Himanshu's comprehensive survey of harness architectures. Vercel 80% tool deletion, Cursor lazy loading (46.9%), CORE-Bench scaffold comparison, progressive disclosure across systems.
- [raw/0xblacklight-2036534699582255329.md](../raw/0xblacklight-2036534699582255329.md) -- Kyle Mistele on filesystem abstraction, separating tool interface from execution, FUSE-based approaches, owning the harness for control flow.
- [raw/gakonst-2036560009128194052.md](../raw/gakonst-2036560009128194052.md) -- Georgios Konstantopoulos on production agent architecture with Postgres as coordinator, Docker containers, 150+ API integrations, firewall-based secret injection.
- [raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md](../raw/simonwillison-net-2025-sep-30-designing-agentic-loops.md) -- Simon Willison, Sep 2025. Shell commands over MCP, AGENTS.md with worked examples, agents generalize from a single example.
- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) -- Anthropic, Apr 2026. Tool-testing agent that rewrote descriptions (40% task time decrease), tool selection heuristics for multi-agent systems.
- [raw/code-research-claude-code-2026-04-15.md](../raw/code-research-claude-code-2026-04-15.md) -- Code research, Apr 2026. Deferred tool loading via ToolSearchTool + tool_reference API type, concurrency partitioning, buildTool() factory, error-as-context pattern, cache-stable tool ordering.
- [raw/code-research-karpathy-autoresearch-2026-04-15.md](../raw/code-research-karpathy-autoresearch-2026-04-15.md) -- Code research, Apr 2026. Prose-as-schema tool definition in natural language, deterministic tool sequence, shell commands as tool API.
- [raw/code-research-openclaw-openclaw-2026-04-15.md](../raw/code-research-openclaw-openclaw-2026-04-15.md) -- Code research, Apr 2026. Per-provider schema normalization (TypeBox → Gemini/OpenAI/xAI), streaming JSON argument repair, 4-level tool name normalization, skills as prompt injections (not tools), MCP tools as first-class with same normalization pipeline.
- [raw/code-research-all-hands-ai-openhands-2026-04-15.md](../raw/code-research-all-hands-ai-openhands-2026-04-15.md) -- Code research, Apr 2026. Security-risk-as-parameter (LLM self-labels risk, pluggable SecurityAnalyzer override), MCP stdio-over-HTTP proxy in Docker sandbox, action-as-typed-dataclass with reflection dispatch, request_condensation as agent-callable context management tool.
- [raw/code-research-anomalyco-opencode-2026-04-15.md](../raw/code-research-anomalyco-opencode-2026-04-15.md) -- Code research, Apr 2026. Nine-strategy fuzzy edit replacer cascade, tree-sitter bash AST for permission detection, model-gated tool selection (GPT-4 → apply_patch), description-as-template from .txt sidecar files, invalid tool as first-class error handler, uniform truncation middleware with agent-aware hints, LSP integration as a single multi-operation tool, remote skill CDN discovery.
- [raw/code-research-666ghj-mirofish-2026-04-15.md](../raw/code-research-666ghj-mirofish-2026-04-15.md) -- Code research, Apr 2026. Minimum tool-call enforcement (3-call floor, 5-call ceiling, unused-tool hints), plugin security scanning with mtime-keyed cache.
- [raw/code-research-claude-code-2026-04-14.md](../raw/code-research-claude-code-2026-04-14.md) -- Code research, Apr 2026 (original run). Recovered findings: buildTool() fail-closed factory enforcement, deferred tool schema persistence via message history scanning, assembleToolPool() cache-breakpoint ordering.
- [raw/code-research-openclaw-openclaw-2026-04-14.md](../raw/code-research-openclaw-openclaw-2026-04-14.md) -- Code research, Apr 2026 (original run). Recovered findings: TypeBox write-once/run-anywhere schema approach, 4-level tool name normalization with loop guard, head+tail truncation with 30% tail budget for error preservation, 4-stage tool policy pipeline (profile → provider → agent → group).
