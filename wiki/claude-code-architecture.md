---
title: "Claude Code Architecture"
type: wiki
tags:
  - claude-code
  - agent-architecture
  - harness-design
  - tool-design
  - progressive-disclosure
  - permission-system
sources:
  - raw/Hxlfed14-2028116431876116660.md
  - raw/trq212-2027463795355095314.md
  - raw/rohit4verse-2041548810804211936.md
  - raw/himanshustwts-2038924027411222533.md
  - raw/idoubicc-2039006326882546141.md
  - raw/code-research-claude-code.md
source_count: 6
status: draft
last_compiled: 2026-04-14
---

Claude Code is Anthropic's coding agent and one of the most extensively analyzed agent harnesses in production. After Anthropic accidentally shipped the entire source code to the npm registry on March 31, 2026 (512,000 lines across 55 directories and 331 modules), the community performed deep architectural analysis. [Source: raw/rohit4verse-2041548810804211936.md] The design reflects a consistent philosophy: keep the harness simple, give the model control, and let the scaffolding shrink as models improve. Claude Code is built on the Claude Agent SDK.

## The Core Loop

The heart of Claude Code lives in query.ts (1,729 lines of TypeScript). The most important decision is the function signature: an async generator, not a simple while loop. [Source: raw/rohit4verse-2041548810804211936.md]

Internally codenamed **nO**, it is a `while(true)` imperative state machine. Earlier external analysis described this as "the model controls the loop." [Source: raw/Hxlfed14-2028116431876116660.md] First-hand source code analysis reveals a more precise picture: **the code controls the loop, and the model's tool_use block presence is the sole continuation signal.** The code explicitly documents that `stop_reason === 'tool_use'` is unreliable and is NOT used for loop control. Instead, the harness checks whether any `ToolUseBlock` objects appeared during streaming -- if yes, continue; if no, evaluate stop hooks and potentially terminate. [Source: raw/code-research-claude-code.md]

The loop has 10+ distinct termination reasons (`completed`, `blocking_limit`, `model_error`, `aborted_streaming`, `aborted_tools`, `hook_stopped`, `stop_hook_prevented`, `prompt_too_long`, `image_error`, `max_turns`) and multiple labeled `continue` transitions for recovery paths (collapse drain retry, reactive compact retry, output token escalation, stop hook blocking). [Source: raw/code-research-claude-code.md]

The async generator form provides five properties that a simple while loop lacks: streaming (users see the model working character by character), cancellation (AbortSignal threaded through every layer), composability (REPL UI, sub-agents, and tests all consume the same generator), backpressure (production pauses when the consumer stops pulling), and in-loop error recovery. [Source: raw/rohit4verse-2041548810804211936.md]

### Five Phases Per Iteration

Each iteration runs through five phases:

1. **Setup**: Apply tool result budgets, run compaction strategies if the conversation is long, validate token counts. [Source: raw/rohit4verse-2041548810804211936.md]
2. **Model Invocation**: Call queryModelWithStreaming() through a dependency-injected interface, wrapped in a retry system. The streaming tool executor starts executing tools during this phase, before the model finishes generating. A Grep call starts running the instant its input JSON is complete in the stream. [Source: raw/rohit4verse-2041548810804211936.md]
3. **Error Recovery & Compaction**: Check for recoverable errors. prompt-too-long? Compact and retry. max_output_tokens hit? Escalate from 32K to 64K and retry. Context overflow? Run reactive compaction on media-heavy messages. These are first-class states in the loop's state machine, not edge cases in outer try-catch blocks. [Source: raw/rohit4verse-2041548810804211936.md]
4. **Tool Execution**: Tools not yet executed by the streaming executor run here. Results yield to the UI as they complete. Haiku generates tool use summaries asynchronously so the main model does not burn tokens on bookkeeping. [Source: raw/rohit4verse-2041548810804211936.md]
5. **Continuation Decision**: The model's stop_reason determines if more tool calls are needed. Turn counter checks maxTurns. Hooks can request a stop. Abort signals are checked. [Source: raw/rohit4verse-2041548810804211936.md]

## The Tool Set: Primitives Over Integrations

First-hand source analysis reveals **~50+ registered tools** across categories (the exact count depends on feature flags and environment). Earlier external analysis reported ~18-20 primitives, which reflects the core set visible to external users. The full tool registry in `tools.ts` includes ant-internal, experimental, and MCP-generated tools beyond the core set. [Source: raw/code-research-claude-code.md]

The core tools visible to users are organized in four categories:

| Category | Tools |
|---|---|
| **CLI discovery** | Bash, Glob, Grep, LS |
| **File interaction** | Read, Write, Edit, MultiEdit |
| **Web access** | WebSearch, WebFetch |
| **Orchestration** | TodoWrite, Task |

[Source: raw/Hxlfed14-2028116431876116660.md]

The philosophy is **primitives over integrations**. Anthropic chose regex (ripgrep) over vector databases for code search, reasoning that Claude's code understanding enables sophisticated regex crafting without requiring search indices. Ripgrep is fast, requires no setup, and works in any environment. [Source: raw/Hxlfed14-2028116431876116660.md]

As Thariq (@trq212), a Claude Code engineer, wrote: "You want to give it tools that are shaped to its own abilities. But how do you know what those abilities are? You pay attention, read its outputs, experiment. You learn to see like an agent." [Source: raw/trq212-2027463795355095314.md]

Claude Code currently has ~20 tools, and the team constantly asks whether all of them are needed. The bar to add a new tool is high, because this gives the model one more option to think about. [Source: raw/trq212-2027463795355095314.md]

### Tool Concurrency Classification

Claude Code classifies every tool by concurrency behavior. Read-only tools (Glob, Grep, Read, WebFetch) run concurrently, up to 10 in parallel. Write tools (Bash with mutations, Edit, Write) run serially. No race conditions. The orchestration layer in toolOrchestration.ts partitions tool calls into batches. This provides 2-5x speedup on multi-tool turns. [Source: raw/rohit4verse-2041548810804211936.md]

### Streaming Tool Executor

Most harnesses wait for the model to finish generating before executing any tools. Claude Code starts execution mid-stream. For a turn with three tool calls, this hides 2-5 seconds of latency. The model generates the description of its next step while the first tool already runs. Results yield in the original order even if tool 2 finishes before tool 1, keeping the narrative coherent. [Source: raw/rohit4verse-2041548810804211936.md]

### Tool Result Budgeting

A Bash command that dumps 1MB of logs would fill the context window if passed raw. Claude Code runs a budgeting system: each tool specifies maxResultSizeChars, results exceeding the limit persist to disk, and the model receives a file path reference plus a preview. applyToolResultBudget() runs before each API call to constrain total tool result tokens. [Source: raw/rohit4verse-2041548810804211936.md]

### Deferred Tool Loading via ToolSearchTool

First-hand source analysis reveals a sophisticated deferred loading system not documented in external write-ups. All MCP tools and flagged built-ins start as "deferred" -- the model sees only their names, not their schemas. To use a deferred tool, the model must first call `ToolSearchTool` with a query (exact name via `select:ToolA,ToolB` or keyword search). ToolSearchTool returns `tool_reference` content blocks (a beta API type) that the API expands into full schemas at inference time. [Source: raw/code-research-claude-code.md]

Discovered tools persist via message history scanning (`extractDiscoveredToolNames`), surviving even compaction through `compactMetadata.preCompactDiscoveredTools`. Delta notifications (`deferred_tools_delta` attachment messages) announce newly added/removed tools between turns without requiring a full re-scan. The entire deferred loading mechanism depends on a custom API content type (`tool_reference`) that only works with first-party Anthropic endpoints. [Source: raw/code-research-claude-code.md]

The deferred loading system reduces the baseline tool schema cost from ~11K tokens to near-zero for unused tools -- critical when the full tool set exceeds 50 tools with MCP servers connected. [Source: raw/code-research-claude-code.md]

### Tool Evolution: From TodoWrite to Task

**TodoWrite** was created early because the model needed a todo list to stay on track. Even with this tool, Claude would forget what it had to do, so the team inserted system reminders every 5 turns that reminded Claude of its goal. [Source: raw/trq212-2027463795355095314.md]

As models improved, the reminders became counterproductive -- they made Claude think it had to stick rigidly to the list instead of adapting. Claude also got better at using subagents, but subagents could not coordinate on a shared todo list. [Source: raw/trq212-2027463795355095314.md]

This led to the **Task tool**, which replaced TodoWrite. Where todos were about keeping the model on track, tasks are about helping agents communicate with each other. Tasks include dependencies, share updates across subagents, and the model can alter and delete them. [Source: raw/trq212-2027463795355095314.md]

The lesson: as model capabilities increase, the tools your models once needed might now be constraining them. It is important to constantly revisit previous assumptions about what tools are needed. [Source: raw/trq212-2027463795355095314.md]

### TodoWrite as a No-Op Anchoring Tool

Despite its eventual replacement, TodoWrite represents one of the most underrated patterns in harness design. LangChain's "Deep Agents" analysis calls it out explicitly: **TodoWrite does nothing functionally**. It is purely a harness-level trick -- a no-op tool that forces the agent to articulate and track its plan, keeping it on course over long trajectories. The tool's value is in what it forces the model to do: externalize its plan into a structured format. This acts as a "progress anchor" -- after errors or distractions, the model consults its todo list to know where it was. Manus independently converged on the same idea with their todo.md rewriting pattern. [Source: raw/Hxlfed14-2028116431876116660.md]

## Information Layering

Claude Code loads **six layers** of information at session start: organization policies, project-level CLAUDE.md, user settings, auto-learned MEMORY.md, session history, and git state. [Source: raw/Hxlfed14-2028116431876116660.md]

### The System Prompt

The system prompt comprises approximately **110+ conditional strings** with a core of ~2,896 tokens. The prompt is split by a SYSTEM_PROMPT_DYNAMIC_BOUNDARY marker into two zones. Everything above it (~80% of the prompt) is identical across all users and sessions, hitting the prompt cache at the API level globally. Below the boundary, sections are memoized (computed once per session) or volatile (recomputed every turn). Volatile sections are minimized because each change breaks the cache for everything after it. [Source: raw/rohit4verse-2041548810804211936.md]

User context (git status, CLAUDE.md contents, current date) is injected as the first user message wrapped in `<system-reminder>` tags rather than in the system prompt. This keeps the system prompt cache-stable turn after turn. [Source: raw/rohit4verse-2041548810804211936.md]

### Tool Result Injection

A critical but underappreciated harness pattern: **tool results carry injected system reminders** -- fixed text appended after every tool execution. This achieves higher behavioral adherence than system-prompt-only instructions because it repeats with every call, keeping instructions in the model's recent attention span. [Source: raw/Hxlfed14-2028116431876116660.md]

This pattern directly addresses the "lost in the middle" problem identified by Liu et al. (TACL 2024), which demonstrated that LLM performance follows a U-shaped curve -- highest when relevant information is at the beginning or end of the input, degraded in the middle. By injecting reminders after every tool result, Claude Code keeps critical instructions in the high-attention zone at the end of the context. [Source: raw/Hxlfed14-2028116431876116660.md]

## The 4-Level CLAUDE.md Hierarchy

A four-level instruction hierarchy acts as composable memory with RBAC semantics:

1. **Enterprise** (`/etc/claude-code/CLAUDE.md`): Organization-wide policies, integrates with MDM for enforcement.
2. **Project** (`.claude/CLAUDE.md`): Per-project conventions and instructions.
3. **User** (`~/.claude/CLAUDE.md`): Personal preferences and configuration.
4. **Local** (`CLAUDE.local.md`): Private developer overrides, kept out of version control.

Higher levels override lower ones. An @include directive enables composition. Enterprise admins enforce coding standards organization-wide; users set personal preferences; projects define conventions; developers keep private overrides out of version control. Conflicts resolve deterministically. [Source: raw/rohit4verse-2041548810804211936.md]

## Memory Architecture

First-hand source code analysis reveals **five distinct memory systems operating in parallel**, beyond what external analysis documented: [Source: raw/code-research-claude-code.md]

1. **CLAUDE.md hierarchy** (6 priority levels: managed → user → project → local → AutoMem → TeamMem). Files closer to CWD have higher priority. Supports `@include` directives (depth 5) and `paths:` frontmatter glob-scoping. Max 40,000 chars per file. [Source: raw/code-research-claude-code.md]
2. **Memdir / Auto Memory** (`~/.claude/projects/<sanitized-git-root>/memory/`). MEMORY.md (max 200 lines / 25KB) as index + individual topic files with YAML frontmatter (types: `user`, `feedback`, `project`, `reference`). Always loaded at session start. [Source: raw/code-research-claude-code.md]
3. **extractMemories background agent** -- a forked subagent that fires after each query loop turn (fire-and-forget). Uses `sideQuery` with Sonnet to rank relevance of existing memories (a mini RAG pipeline). Up to 5 memories injected per turn via `<system-reminder>`. Mutual exclusion with main agent: if the main agent already wrote to memdir, the fork is skipped. [Source: raw/code-research-claude-code.md]
4. **Session Memory** (gated behind `tengu_session_memory`). Running notes at `~/.claude/<session-id>/session-memory/`. 8-section structured template (Current State, Task Spec, Files and Functions, Workflow, Errors, Learnings, Key Results, Worklog). Max 12,000 tokens total. Fires as a forked subagent when token + tool-call thresholds are both met. [Source: raw/code-research-claude-code.md]
5. **Prompt History** (`~/.claude/history.jsonl`). Shell-style up-arrow recall. Max 100 entries per project. Large pastes stored by SHA-256 hash. [Source: raw/code-research-claude-code.md]

Earlier external analysis identified the design principles behind this system: [Source: raw/himanshustwts-2038924027411222533.md]

- **Memory as index, not storage**: MEMORY.md is always loaded, but it is just pointers (~150 chars/line). Actual knowledge lives outside, fetched only when needed. [Source: raw/himanshustwts-2038924027411222533.md]
- **3-layer bandwidth-aware design**: Index (always loaded), topic files (on-demand), transcripts (never read, only grep'd). [Source: raw/himanshustwts-2038924027411222533.md]
- **Strict write discipline**: Write to file, then update index. Never dump content into the index. Prevents entropy and context pollution. [Source: raw/himanshustwts-2038924027411222533.md]
- **Background memory rewriting (autoDream)**: Merges, dedupes, removes contradictions, converts vague to absolute, aggressively prunes. Memory is continuously edited, not appended. [Source: raw/himanshustwts-2038924027411222533.md]
- **Staleness is first-class**: If memory does not match reality, memory is wrong. Code-derived facts are never stored. Index is forcibly truncated. [Source: raw/himanshustwts-2038924027411222533.md]
- **Isolation**: Consolidation runs in a forked subagent with limited tools to prevent corruption of main context. [Source: raw/himanshustwts-2038924027411222533.md]
- **Skeptical retrieval**: Memory is a hint, not truth. The model must verify before using. [Source: raw/himanshustwts-2038924027411222533.md]
- **What they don't store is the real insight**: No debugging logs, no code structure, no PR history. If it is derivable, don't persist it. [Source: raw/himanshustwts-2038924027411222533.md]

## Context Window Management: The 4-Layer Compaction Cascade

Claude Code supports unlimited conversation length through a cascade of compaction strategies that can ALL fire in the same loop iteration. Ordered cheapest to most expensive: [Source: raw/rohit4verse-2041548810804211936.md] [Source: raw/code-research-claude-code.md]

1. **Snip** (`HISTORY_SNIP`): Client-side. Drops oldest messages above a token threshold. Fast, lossless from the model's perspective. [Source: raw/code-research-claude-code.md]
2. **Microcompact** (`microCompact.ts`): Client-side. Truncates/clears old `tool_result` content inline for compactable tools (Read, Bash, Grep, Glob, WebFetch, WebSearch, FileEdit, FileWrite). Preserves message structure, empties body. Threshold: ~180K tokens, keeps last ~40K. [Source: raw/code-research-claude-code.md]
3. **API-level microcompact** (`apiMicrocompact.ts`): Server-side. Passes `context_management` config to the API (`clear_tool_uses_20250919`, `clear_thinking_20251015`). The API server edits the context directly. [Source: raw/code-research-claude-code.md]
4. **Auto-compact** (`autoCompact.ts`): Full conversation summary via a **forked agent** sharing the parent's prompt cache prefix. Fires at ~87-91% context fill (context window minus 20K output reserve minus 13K buffer). The fork produces a structured 9-section summary (Primary Request, Key Technical Concepts, Files/Code, Errors, Problem Solving, All User Messages, Pending Tasks, Current Work, Optional Next Step). Circuit breaker: 3 consecutive autocompact failures suppresses further attempts. [Source: raw/code-research-claude-code.md]

**Post-compact recovery**: After autocompact, up to 5 recently-read files (50K token budget, 5K/file) are re-injected as attachments via `createPostCompactFileAttachments`. Invoked skill content survives compaction intentionally (never cleared). Deferred tool schemas and agent/MCP listings are re-announced via delta messages. The model is told the full transcript path so it can grep `.jsonl` files as a "last resort" to recover details lost in compaction. [Source: raw/code-research-claude-code.md]

**Reactive compact** (experimental): Triggered AFTER a 413 "prompt too long" API error rather than proactively. Same mechanism but as recovery rather than prevention. [Source: raw/code-research-claude-code.md]

The hierarchy matters: cheapest strategy runs first, most expensive fires only when nothing else works. Microcompact and snip handle a large percentage of cases with zero model calls. [Source: raw/rohit4verse-2041548810804211936.md]

## Progressive Disclosure

Progressive disclosure -- borrowed from UI/UX design (John Carroll, IBM Research, 1980s; popularized by Jakob Nielsen in the 1990s) -- is the principle of showing only what is needed now and revealing complexity on demand. [Source: raw/Hxlfed14-2028116431876116660.md]

### Skills as On-Demand Loading

Skills are stored as `.claude/skills/` files and are **not preloaded** into every conversation. Unlike CLAUDE.md (loaded every session), skills load only when Claude detects relevance. The system prompt describes this as "on-demand loading." This prevents context bloat when a project has dozens of skills. Skills have five sources: bundled, project, user, plugin, MCP. Path-based discovery means a skill specifying `paths: ["*.tsx"]` only activates when the agent touches matching files. [Source: raw/Hxlfed14-2028116431876116660.md] [Source: raw/rohit4verse-2041548810804211936.md]

### Search as Context Building

When Claude Code first launched, it used a RAG vector database to find context. While RAG was powerful and fast, it required indexing and setup, could be fragile across environments, and -- critically -- the model was given context instead of finding it itself. [Source: raw/trq212-2027463795355095314.md]

The team replaced this with the **Grep tool**, letting Claude search for files and build context on its own. As Thariq describes: "This is a pattern we've seen as Claude gets smarter -- it becomes increasingly good at building its context if it's given the right tools." [Source: raw/trq212-2027463795355095314.md]

With the introduction of Agent Skills, this became formalized. Claude could read skill files and those files could reference other files the model could read recursively. Over the course of a year, Claude went from not being able to build its own context to performing nested search across several layers of files to find exactly what it needed. [Source: raw/trq212-2027463795355095314.md]

### The Claude Code Guide Subagent

When Anthropic wanted Claude to answer questions about Claude Code itself, they faced a choice: stuff all documentation into the system prompt (adding context rot and interfering with the primary task of writing code), or use progressive disclosure. They chose progressive disclosure. First they gave Claude a link to its docs, but found it would load excessive results. So they built the **Claude Code Guide subagent** -- a specialized subagent with extensive instructions on how to search docs efficiently and what to return. This added capability to Claude's action space without adding a tool. [Source: raw/trq212-2027463795355095314.md]

## The Permission Pipeline

Claude Code runs a seven-stage permission pipeline rather than a binary allow/deny toggle. Rules use glob-like pattern matching on tool name and input. Permission modes create progressive trust: new users start in default (approving each action); as confidence builds, they move to acceptEdits or bypassPermissions. [Source: raw/rohit4verse-2041548810804211936.md]

Hooks serve as the escape hatch: a script receives tool call details and returns `{"decision": "approve"}` or `{"decision": "block"}`. Organizations build custom guardrails: block destructive operations, post to Slack on completion, run linters after every file write. No source modifications required. [Source: raw/rohit4verse-2041548810804211936.md]

## The Error Recovery System

The retry system (services/api/withRetry.ts, 823 lines) handles every error class with specific recovery paths: [Source: raw/rohit4verse-2041548810804211936.md]

- **429 (Rate Limited)**: Check Retry-After header. Under 20 seconds? Retry. Over 20? Enter 30-minute cooldown. overage-disabled header? Permanently disable fast mode.
- **529 (Server Overloaded)**: Track consecutive counts. Three in a row with fallback model? Switch models. Background task? Bail. Foreground? Retry with backoff.
- **400 (Context Overflow)**: Parse error to extract actual vs limit token counts. Recalculate with 1,000-token safety buffer and 3,000 minimum floor.
- **401/403 (Auth)**: Clear API key cache. Force-refresh OAuth tokens.
- **Network Errors**: Disable keep-alive socket pooling. Retry with new connection.

Backoff formula: `delay = min(500ms * 2^attempt, 32s) + random(0, 0.25 * baseDelay)`. For unattended sessions, persistent retry mode retries 429 and 529 errors indefinitely with a maximum 5-minute backoff and 6-hour reset cap. [Source: raw/rohit4verse-2041548810804211936.md]

## Multi-Agent Architecture

First-hand source analysis reveals **three distinct multi-agent modes** coexisting in Claude Code: [Source: raw/code-research-claude-code.md]

### Standard Subagents (AgentTool)
Sub-agents start with zero parent context -- a fresh `[createUserMessage({content: prompt})]` is the entire conversation history. System prompt is rebuilt from scratch. Tools are assembled independently. This means the parent must write self-contained prompts with specific file paths and line numbers. The coordinator mode prompt explicitly warns: "Workers can't see your conversation. Every prompt must be self-contained." and forbids "Based on your findings, fix the bug" (delegating understanding). [Source: raw/code-research-claude-code.md]

Sub-agents that modify code get their own **git worktree** -- one agent, one worktree, parallel agents on separate branches. Changes merge when verified. [Source: raw/rohit4verse-2041548810804211936.md]

### Fork Subagents (experimental, `FORK_SUBAGENT` gate)
Fork children **inherit the parent's full conversation history** and byte-identical system prompt. The fork builds an API request prefix that maximizes prompt cache hits -- making the fork nearly free for short tasks. Fork children cannot themselves fork (recursive prevention via both context-object check and message-history scan). Communication back to the parent is via tool_result (sync) or `<task-notification>` XML injected as a user-role message (async). [Source: raw/code-research-claude-code.md]

The forked agent pattern is Claude Code's fundamental building block for background work: autocompact forks a summarizer, extractMemories forks a background memory writer, SessionMemory forks a note-taker, and sub-agents can fork with full parent context. All share the parent's prompt cache prefix. [Source: raw/code-research-claude-code.md]

### Swarm Teammates (experimental, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`)
Long-lived peer agents running in separate OS processes (tmux panes or iTerm2 split panes). Communication via file-based mailbox at `~/.claude/teams/{team}/inboxes/{name}.json` with proper-lockfile locking. Shared task list at `~/.claude/tasks/{team-name}/` with ownership via `TaskUpdate({owner: name})`. [Source: raw/code-research-claude-code.md]

### Coordinator Mode
A dedicated `CLAUDE_CODE_COORDINATOR_MODE=1` replaces the default system prompt with a structured Research -> Synthesis -> Implementation -> Verification workflow. The coordinator sees only 4 tools (AgentTool, TaskStopTool, SendMessageTool, SyntheticOutputTool) -- pure orchestration with no direct file/shell access. All spawns are forced async. [Source: raw/code-research-claude-code.md]

### Auto-Background Escalation
Sync agents exceeding 120 seconds are automatically promoted to background via `Promise.race()`. The agent continues uninterrupted; only the parent's waiting behavior changes. [Source: raw/code-research-claude-code.md]

Three spawn backends: in-process (direct Node.js, fastest, shared memory), tmux pane (terminal multiplexer isolation, each agent visible in its own tab), remote (CCR environment, full machine isolation). Task coordination uses a disk-backed task list with file-based locking at `~/.claude/tasks/<taskListId>/<taskId>.json`. [Source: raw/rohit4verse-2041548810804211936.md]

## Extensibility: Four Mechanisms

Claude Code has four extension mechanisms, none requiring source code modification:

1. **Skills**: Markdown files with YAML frontmatter. Five sources: bundled, project, user, plugin, MCP. Path-based discovery. [Source: raw/rohit4verse-2041548810804211936.md]
2. **Hooks**: Six types (shell commands, LLM evaluation, agentic verification, HTTP endpoints, TypeScript callbacks, in-memory functions). Fires on PreToolUse, PostToolUse, SessionStart, FileChanged, Stop. [Source: raw/rohit4verse-2041548810804211936.md]
3. **MCP**: Five transport types (stdio, SSE, HTTP streaming, WebSocket, in-process). Configured at three levels (enterprise managed, project, user). [Source: raw/rohit4verse-2041548810804211936.md]
4. **Plugins**: Directories containing skills, agents, hooks, and configuration. The top-level composition mechanism. [Source: raw/rohit4verse-2041548810804211936.md]

All four follow the same principle: composition over modification. Extend by adding, not changing. [Source: raw/rohit4verse-2041548810804211936.md]

## Open Source SDK Derivatives

The source code leak enabled derivative projects. Idoubi (@idoubicc) built **open-agent-sdk** by extracting all logic from the Claude Code sourcemap, creating an open-source replacement for claude-agent-sdk. The motivation: claude-agent-sdk depends on Claude Code (a black box), requires creating Claude Code processes (high overhead for cloud-scale use), and cannot be customized. Open-agent-sdk provides function-level calls without CLI process overhead, fully open source with MIT license. [Source: raw/idoubicc-2039006326882546141.md]

## Design Philosophy

Several principles emerge from Claude Code's architecture:

1. **The bar to add a new tool is high.** Every additional tool gives the model one more option to consider. [Source: raw/trq212-2027463795355095314.md]
2. **Designing tools is an art, not a science.** It depends heavily on the model, the goal of the agent, and the environment. "Experiment often, read your outputs, try new things. See like an agent." [Source: raw/trq212-2027463795355095314.md]
3. **What works for one model may not work for another.** Cursor tunes its harness specifically for every frontier model based on internal evals -- different models get different tool names, prompt instructions, and behavioral guidance. [Source: raw/Hxlfed14-2028116431876116660.md]
4. **The teams shipping the best agents keep simplifying.** Manus went through five rewrites, each removing things. Anthropic designs Claude Code's scaffold to shrink as models improve. Over-engineering is the default failure mode. [Source: raw/Hxlfed14-2028116431876116660.md]
5. **Layer 4 (infrastructure) from day one.** Where does state live across sessions? How do permissions scale to teams? How does coordination work with parallelism? Retrofitting infrastructure is harder than designing for it by an order of magnitude. [Source: raw/rohit4verse-2041548810804211936.md]

## Related

- [What Is Harness Engineering?](what-is-harness-engineering.md) -- the broader discipline this architecture exemplifies
- [Auto Mode and Safety](auto-mode-and-safety.md) -- the classifier pipeline that gates Claude Code's tool calls
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- multi-session patterns built on this architecture
- [OpenAI Codex Harness](openai-codex-harness.md) -- a different production harness for comparison
- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- the multi-level memory hierarchy and compaction strategies built into Claude Code
- [Practical Best Practices](practical-best-practices.md) -- actionable practices derived from Claude Code's design patterns
- [Tool Design Patterns](tool-design-patterns.md) -- Claude Code's ~18 primitive tools, action space design, and the thin-harness-fat-skills philosophy

## Open Questions

- **RESOLVED**: The 50+ tools vs. ~18-20 primitives discrepancy is explained by feature flags and environment. First-hand analysis confirms ~50+ registered tools in `tools.ts`, but many are ant-internal-only, experimental, or MCP-generated. The ~18-20 count in external write-ups reflects the user-visible core set. [Source: raw/code-research-claude-code.md]
- How the async generator pattern performs at scale compared to simpler while-loop architectures in other production agents. [Source: raw/rohit4verse-2041548810804211936.md]
- Whether the memory architecture's "if it's derivable, don't persist it" principle generalizes to non-coding agent domains. [Source: raw/himanshustwts-2038924027411222533.md]
- How the `tool_reference` beta API type for deferred tool loading will evolve -- currently only works with first-party Anthropic endpoints. [Source: raw/code-research-claude-code.md]
- Whether the extractMemories background agent's RAG pipeline (sideQuery ranking of existing memories) is effective in practice or creates noise. [Source: raw/code-research-claude-code.md]

## Sources

- [raw/Hxlfed14-2028116431876116660.md](../raw/Hxlfed14-2028116431876116660.md) -- Himanshu (@Hxlfed14), Mar 2026. Cross-company harness analysis with Claude Code's core loop, tool philosophy, and progressive disclosure patterns.
- [raw/trq212-2027463795355095314.md](../raw/trq212-2027463795355095314.md) -- Thariq (@trq212), Feb 2026. Claude Code engineer on tool design philosophy, TodoWrite-to-Task evolution, search interface design, and progressive disclosure.
- [raw/rohit4verse-2041548810804211936.md](../raw/rohit4verse-2041548810804211936.md) -- Rohit (@rohit4verse), Apr 2026. Deep source code analysis of Claude Code's 331-module architecture: async generator loop, compaction, permissions, sub-agents, extensibility.
- [raw/himanshustwts-2038924027411222533.md](../raw/himanshustwts-2038924027411222533.md) -- Himanshu (@himanshustwts), Mar 2026. Memory architecture analysis: index-not-storage, autoDream, staleness, skeptical retrieval.
- [raw/idoubicc-2039006326882546141.md](../raw/idoubicc-2039006326882546141.md) -- Idoubi (@idoubicc), Mar 2026. Open-agent-sdk: open-source Claude Code logic extraction for cloud-scale agent deployment.
- [raw/code-research-claude-code.md](../raw/code-research-claude-code.md) -- First-hand source code analysis via /kb-code-research skill, Apr 2026. Deep 4-dimension analysis: imperative loop architecture, 5 memory systems, 50+ tools with deferred loading, 3 multi-agent modes, prompt cache-first design.
