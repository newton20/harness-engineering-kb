---
title: "Long-Running Agent Harnesses"
type: wiki
tags:
  - long-running-agents
  - multi-context-window
  - agent-architecture
  - harness-design
  - context-management
  - autocontext
sources:
  - raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md
  - raw/anthropic-com-engineering-harness-design-long-running-apps.md
  - raw/JayScambler-2033971974284714355.md
  - raw/anthropic-com-engineering-multi-agent-research-system.md
  - raw/code-research-claude-code.md
  - raw/code-research-karpathy-autoresearch.md
  - raw/code-research-openclaw-openclaw.md
source_count: 7
status: draft
last_compiled: 2026-04-14
---

Complex software projects cannot be completed within a single context window. Building agents that work effectively across multiple context windows -- spanning hours or days of autonomous operation -- requires deliberate harness design that solves for state handoff, incremental progress, and quality evaluation. Anthropic published two major research posts on this problem (November 2025 and March 2026), each introducing distinct architectural patterns, while the open-source community has developed complementary approaches to persistent learning across agent runs.

## The Core Problem

When an agent must work across multiple context windows, three failure modes dominate:

1. **One-shotting**: The agent tries to do too much at once, runs out of context mid-implementation, and leaves the next session to start with a half-implemented, undocumented feature. The next agent must guess what happened and spend substantial time recovering a working state. This happens even with compaction, which does not always pass perfectly clear instructions to the next agent. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

2. **Premature victory**: After some features have been built, a later agent instance looks around, sees that progress has been made, and declares the job done -- even when critical work remains. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

3. **Context anxiety**: Some models (notably Claude Sonnet 4.5) begin wrapping up work prematurely as they approach what they believe is their context limit. This manifests as the agent rushing to finish or cutting corners, even when substantial context capacity remains. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

Anthropic's analogy: imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of what happened on the previous shift. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

## Solution 1: Initializer + Coding Agent (November 2025)

Published in "Effective harnesses for long-running agents" (November 26, 2025), this approach decomposes the problem into two specialized agent roles running on the Claude Agent SDK. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

### The Initializer Agent

The very first session uses a specialized prompt that asks the model to set up the initial environment:

- **init.sh**: A script that can run the development server, install dependencies, and execute basic end-to-end tests. Every subsequent coding agent runs this first to confirm the app is in a working state before starting new work. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- **feature_list.json**: A comprehensive, structured JSON file of feature requirements expanding on the user's initial prompt. In the claude.ai clone example, this meant over 200 features (e.g., "a user can open a new chat, type in a query, press enter, and see an AI response"). All features are initially marked as `"passes": false`. The model is instructed with strongly-worded rules like "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality." JSON was chosen over Markdown because the model is less likely to inappropriately change or overwrite JSON files. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- **claude-progress.txt**: A log file where agents write summaries of what they accomplished and what remains. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- **Initial git commit**: Establishing a version-controlled baseline. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

### The Coding Agent

Every subsequent session follows a structured startup sequence:

1. Run `pwd` to confirm the working directory.
2. Read git logs and `claude-progress.txt` to understand recent work.
3. Read `feature_list.json` and choose the highest-priority incomplete feature.
4. Run `init.sh` to start the development server.
5. Run a basic end-to-end test to verify the app is not in a broken state.
6. Work on exactly one feature at a time.
7. Test the feature end-to-end (using browser automation tools like Puppeteer MCP when building web apps).
8. Commit progress to git with descriptive messages.
9. Update `claude-progress.txt` with a session summary.

[Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

The key insight was finding a way for agents to quickly understand the state of work when starting with a fresh context window. The combination of `claude-progress.txt` alongside git history provides this. Inspiration came from observing what effective software engineers do every day. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

### Key Artifacts

| Artifact | Purpose |
|---|---|
| `init.sh` | Reproducible environment setup; every session starts by running it |
| `feature_list.json` | Structured task list with pass/fail status; prevents premature victory |
| `claude-progress.txt` | Session-to-session handoff notes; reduces ramp-up time |
| Git commits | Descriptive messages enable recovery and context reconstruction |

[Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]

## Solution 2: Three-Agent Architecture (March 2026)

Published in "Harness design for long-running application development" (March 24, 2026), this approach takes inspiration from Generative Adversarial Networks (GANs) to create a multi-agent system with specialized roles. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

### Architecture

The system uses three agent personas:

**Planner**: Takes a simple 1-4 sentence prompt and expands it into a full product spec. Prompted to be ambitious about scope and to stay focused on product context and high-level technical design rather than detailed implementation. The rationale: if the planner specifies granular technical details upfront and gets something wrong, errors cascade into downstream implementation. Better to constrain the agents on deliverables and let them figure out the path. The planner also looks for opportunities to weave AI features into the product spec. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

**Generator**: Works in sprints, picking up one feature at a time from the spec. Each sprint implements using a specific stack (React, Vite, FastAPI, SQLite/PostgreSQL in the examples). The generator self-evaluates at the end of each sprint before handing off to QA. Has git for version control. Before each sprint, the generator and evaluator negotiate a **sprint contract** -- agreeing on what "done" looks like before any code is written. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

**Evaluator**: Uses the **Playwright MCP** to click through the running application the way a user would, testing UI features, API endpoints, and database states. Grades each sprint against bugs found and a set of criteria covering product depth, functionality, visual design, and code quality. Each criterion has a hard threshold; if any falls below it, the sprint fails and the generator gets detailed feedback. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

Communication between agents is handled via files: one agent writes a file, another reads and responds within that file or with a new file. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

### The Self-Evaluation Problem

A critical insight driving the three-agent design: when asked to evaluate work they have produced, agents tend to confidently praise it -- even when quality is obviously mediocre. This is especially pronounced for subjective tasks like design, where there is no binary check equivalent to a verifiable software test. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

Out of the box, Claude is a poor QA agent. In early runs, it would identify legitimate issues, then talk itself into deciding they were not a big deal and approve the work anyway. It also tended to test superficially, so subtle bugs slipped through. The tuning loop was to read the evaluator's logs, find examples where its judgment diverged from the human's, and update the QA prompt to address those issues. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

Separating the agent doing the work from the agent judging it proves to be a strong lever. The separation does not immediately eliminate leniency, but tuning a standalone evaluator to be skeptical is far more tractable than making a generator critical of its own work. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

### Results

Comparing the harness to a solo agent on the same prompt ("Create a 2D retro game maker"):

| Metric | Solo Agent | Full Harness |
|---|---|---|
| Duration | 20 min | 6 hr |
| Cost | $9 | $200 |
| Core feature working | No (game was broken) | Yes (playable) |

The solo run produced a visually plausible but fundamentally broken application -- entities appeared on screen but nothing responded to input. The harness run produced a functional game maker with working sprite editor, level editor, AI-assisted generation, and playable test mode. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Context Resets vs. Compaction

Two approaches exist for managing context growth in long-running agents:

**Compaction** summarizes earlier parts of the conversation in place so the same agent can keep going on a shortened history. Preserves continuity but does not give the agent a clean slate. Context anxiety can persist because the agent still perceives itself as being deep into a long session. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

**Context resets** clear the context window entirely and start a fresh agent, combined with a structured handoff that carries the previous agent's state and next steps. Provides a clean slate at the cost of requiring handoff artifacts to carry enough state for the next agent to pick up work cleanly. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

In Anthropic's earlier testing, Claude Sonnet 4.5 exhibited context anxiety strongly enough that compaction alone was insufficient, making context resets essential. The March 2026 work notes that **Opus 4.5 largely removed the context anxiety behavior on its own**, allowing the harness to drop context resets entirely. Agents ran as one continuous session with automatic compaction handling context growth. When Opus 4.6 arrived, it provided further motivation to reduce harness complexity -- it "plans more carefully, sustains agentic tasks for longer, can operate more reliably in larger codebases, and has better code review and debugging skills." [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

This illustrates a general principle: every component in a harness encodes an assumption about what the model cannot do on its own, and those assumptions are worth stress-testing as models improve. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Solution 4: Claude Code's 4-Layer Compaction Cascade

Source code analysis of Claude Code reveals the most sophisticated compaction system in any open agent harness, designed to enable unlimited-length sessions. The cascade has four layers that fire in order of increasing cost: (1) snip (drop oldest messages), (2) microcompact (clear old tool_result bodies), (3) API-level context management (server-side clearing), and (4) autocompact (a forked agent that produces a structured 9-section summary). All four layers can fire in the same loop iteration. A circuit breaker suppresses autocompact after 3 consecutive failures to prevent cascading overhead. [Source: raw/code-research-claude-code.md]

The autocompact layer uses a forked sub-agent -- a child process that inherits the parent's full conversation history and byte-identical system prompt, sharing the parent's prompt cache prefix so the summarization API call is nearly free. After compaction completes, up to 5 recently-read files (within a 50K token budget, 5K per file) are re-injected as attachments to restore active working context. Skill content intentionally survives compaction and is never cleared. This post-compact file re-injection is what enables coding agents to resume coherent work after context reduction. [Source: raw/code-research-claude-code.md]

The forked agent pattern extends beyond compaction: Claude Code uses it for background memory extraction (extractMemories), mid-session note-taking (SessionMemory), and sub-agent status polling. Each fork shares the parent's cached prompt prefix, making the pattern economically viable for frequent background tasks. This represents a fundamental building block for long-running agent work -- any background operation that needs the parent's context can be forked cheaply. [Source: raw/code-research-claude-code.md]

## Solution 5: Autoresearch's "NEVER STOP" Loop

Karpathy's autoresearch takes the opposite approach to structured session management: the outer experiment loop has no termination criterion whatsoever. The system prompt contains the directive "NEVER STOP... You are autonomous," and the loop runs until the human physically interrupts the process. There is no convergence detection, no budget cutoff, no coverage checklist. The design choice is deliberate -- the system cannot detect when it has exhausted useful mutations, so human judgment is the only reliable termination signal. Combined with git-as-state-machine (where branch HEAD always points to the current best experiment), the system can be interrupted and resumed at any point without losing progress. [Source: raw/code-research-karpathy-autoresearch.md]

## Solution 3: Orchestrator-Worker Pattern (Anthropic Research, April 2026)

Anthropic's multi-agent Research feature introduces a third long-running architecture: the **orchestrator-worker** pattern. A lead research agent (the orchestrator) receives a user query, creates a plan, and delegates subtasks to multiple subagents (workers) that run in parallel. This is distinct from the earlier planner-generator-evaluator architecture in that the orchestrator actively coordinates a variable number of workers rather than running a fixed pipeline. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

Several design decisions in this system are directly relevant to long-running agent harness design:

- **Subagent output to filesystem:** Rather than passing research results back through the orchestrator's context (a "game of telephone" that loses fidelity), subagents write their outputs directly to the filesystem. The lead agent reads these files when synthesizing the final report. This avoids token overhead and information loss that accumulate when results are relayed through intermediate context windows. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

- **Rainbow deployments:** Anthropic uses rainbow deployments to avoid disrupting running agents during software updates. Because research sessions can run for extended periods (using 15x the tokens of a standard chat turn), deploying a new version mid-session could break an in-progress agent. Rainbow deployments route new sessions to updated code while letting existing sessions complete on the version they started with -- a critical infrastructure pattern for any long-running agent system. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

- **Checkpoints and resume-from-failure:** Long research sessions are inherently fragile. The system saves the research plan to persistent memory because context may be truncated at 200K tokens. If a subagent fails or the session is interrupted, the orchestrator can resume from the saved plan rather than starting over. This makes checkpoint-and-resume a non-negotiable requirement for any agent session measured in hours rather than minutes. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

- **Token usage as performance predictor:** Anthropic found that token usage explains 80% of performance variance in their research system. This provides a practical proxy metric for harness engineers: if the agent is not using enough tokens (not doing enough work), the output quality will suffer regardless of other design choices. [Source: raw/anthropic-com-engineering-multi-agent-research-system.md]

## Iterating on the Harness

The March 2026 work documents a progressive simplification. After the initial three-agent harness proved effective but expensive:

1. The **sprint construct was removed** entirely. Opus 4.6 could handle coherent work over 2+ hours without needing decomposition into explicit sprints. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
2. The **evaluator moved to a single pass** at the end of the run rather than grading per sprint. Its usefulness became task-dependent: for work within the model's native capability, it was unnecessary overhead; for work at the edge of capability, it still provided real lift. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
3. The result: a DAW (Digital Audio Workstation) built from a one-sentence prompt in ~4 hours at $124.70, with a working arrangement view, mixer, transport, and an AI agent that could compose simple songs. The builder ran coherently for over two hours without sprint decomposition. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

The broader developer community has converged on similar patterns. The **"Ralph Wiggum" method** uses hooks or scripts to keep agents in continuous iteration cycles -- a community-developed approach to the same multi-session challenge. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Autocontext: Persistent Learning Across Runs

Jay Scambler (@JayScambler) published autocontext, an open-source system that addresses the persistent memory gap in long-running agents. The core problem: when the context window fills and compacts, the agent forgets what it figured out. Every invocation starts cold. The context window is "a goldfish brain -- useful for the current conversation, useless for institutional knowledge." [Source: raw/JayScambler-2033971974284714355.md]

Autocontext runs an agent against a task repeatedly. After each run, a multi-agent team analyzes the results and updates a shared knowledge base. The next run inherits everything the previous run learned. The pipeline consists of: [Source: raw/JayScambler-2033971974284714355.md]

- **Competitor**: Generates a strategy for the task
- **Translator**: Converts raw output into validated, structured JSON
- **Analyst**: Examines what happened (findings, root causes, recommendations)
- **Coach**: Distills analysis into a playbook update and competitor hints
- **Architect**: Proposes and generates tooling improvements
- **Curator**: Gates quality of playbook updates and consolidates lessons
- **Orchestrator**: Sequences the entire pipeline, handling parallel execution and retry logic

The **playbook** is the key abstraction -- a living document that grows across runs, containing strategies that worked (with scores), strategies that failed (with specific failure modes), tier-specific rules, and generated tools. When run against a grid capture-the-flag scenario, it accumulated 33 distinct lessons across 2 generations, growing to 5,870 characters of actionable operational knowledge. An agent starting fresh with a populated playbook outperforms one starting cold. [Source: raw/JayScambler-2033971974284714355.md]

Autocontext also includes a frontier-to-local distillation pipeline: run discovery with a frontier model (Claude Opus 4.6, GPT-5.4), export training data, train a small local model via MLX on Apple Silicon, and route future runs through the local model when strong enough, falling back to frontier when weak. [Source: raw/JayScambler-2033971974284714355.md]

## OpenClaw: Two-Level Loops and Lane-Based Serialization

OpenClaw's Pi harness introduces two production patterns for long-running agent reliability not seen in the harnesses described above.

**Two-level loop architecture.** Rather than a single while-tool-call loop, OpenClaw separates the agent loop into two levels: an outer `while (true)` retry loop (`runEmbeddedPiAgent`) that manages failover, compaction, auth-profile rotation, and model switching across "attempts," and an inner model-driven tool-calling loop (`activeSession.prompt()`) delegated entirely to the Pi SDK. The outer loop has a dynamic iteration cap computed as `BASE(24) + authProfiles × 8`, capped at 160 -- meaning more API key profiles unlock more retries. Termination conditions include successful completion, cap exhaustion, timeout, abort signal, strict-agentic planning-only blocks, and the `sessions_yield` cooperative abort. The clean separation means harness-level retry bugs cannot corrupt the tool loop, and the SDK can be upgraded independently. [Source: raw/code-research-openclaw-openclaw.md]

**Lane-based session serialization.** Each OpenClaw session gets its own command queue (`session:${key}` lane), preventing concurrent agent runs on the same session without blocking other sessions. Cron jobs get a separate `Cron` lane, and nested operations (subagent runs triggered from within a cron job) use a `Nested` lane to avoid deadlock. This per-session serialization is critical for multi-channel deployments where messages from different channels (WhatsApp, Slack, Discord) may arrive simultaneously for the same user session. [Source: raw/code-research-openclaw-openclaw.md]

**SOUL.md as persona identity file.** OpenClaw introduces a `SOUL.md` workspace file for defining the agent's personality. The template reads: "You're not a chatbot. You're becoming someone." The system prompt injects a single line: "If SOUL.md is present, embody its persona and tone." This makes personality a user-controlled, file-based property that survives harness upgrades -- distinct from CLAUDE.md (code instructions) and AGENTS.md (agent guidelines). [Source: raw/code-research-openclaw-openclaw.md]

## Lessons

- Decomposing work into tractable chunks and using structured artifacts to hand off context between sessions are the two core load-bearing patterns for long-running agents. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- The evaluator/QA agent is worth the cost when the task sits beyond what the current model does reliably solo. As models improve, that boundary moves outward. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
- The space of interesting harness combinations does not shrink as models improve; instead it moves, and the work is to keep finding the next novel combination. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
- When a new model lands, re-examine the harness: strip away pieces that are no longer load-bearing and add new pieces to achieve greater capability that was not possible before. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
- Knowledge accumulation across runs is as important as knowledge management within a single run. The playbook pattern provides a mechanism for this that pure compaction cannot. [Source: raw/JayScambler-2033971974284714355.md]

## Related

- [What Is Harness Engineering?](what-is-harness-engineering.md) -- foundational concepts and definitions
- [Claude Code Architecture](claude-code-architecture.md) -- the core loop and compaction strategies these harnesses build on
- [OpenAI Codex Harness](openai-codex-harness.md) -- Codex regularly runs 6+ hour tasks with similar multi-session patterns
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) -- the GAN-inspired evaluator pattern and persistent learning loops across agent runs
- [Practical Best Practices](practical-best-practices.md) -- actionable tips for multi-session work: git as recovery, progressive disclosure, context management
- [Deep Research Agents](deep-research-agents.md) -- orchestrator-worker patterns, convergence detection, and economics of long-running research sessions
- [Agentic Design Patterns](agentic-design-patterns.md) -- ReAct, Reflection, Planning, Tool Use, Multi-Agent as foundational patterns underlying these architectures
- [Multi-Agent Reliability](multi-agent-reliability.md) -- credibility scoring and adversary-resistant coordination for multi-agent systems

## Open Questions

- Whether a single general-purpose coding agent performs best across contexts, or if specialized multi-agent architectures (testing agent, QA agent, cleanup agent) consistently outperform. The March 2026 work provides evidence that the answer is task-dependent. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- How to generalize these findings beyond full-stack web app development to domains like scientific research or financial modeling. [Source: raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md]
- Whether autocontext-style persistent learning can be integrated directly into production agent harnesses like Claude Code, or if it remains an external layer. [Source: raw/JayScambler-2033971974284714355.md]

## Sources

- [raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md](../raw/anthropic-com-engineering-effective-harnesses-for-long-running-agents.md) -- Anthropic, Nov 2025. Initializer + coding agent pattern for multi-context-window work.
- [raw/anthropic-com-engineering-harness-design-long-running-apps.md](../raw/anthropic-com-engineering-harness-design-long-running-apps.md) -- Anthropic (Prithvi Rajasekaran), Mar 2026. Three-agent architecture with planner, generator, and evaluator.
- [raw/JayScambler-2033971974284714355.md](../raw/JayScambler-2033971974284714355.md) -- Jay Scambler (@JayScambler), Mar 2026. Autocontext: persistent learning and playbook accumulation across agent runs.
- [raw/anthropic-com-engineering-multi-agent-research-system.md](../raw/anthropic-com-engineering-multi-agent-research-system.md) -- Anthropic, Apr 2026. Multi-agent Research feature: orchestrator-worker pattern, rainbow deployments, subagent output to filesystem, token usage as performance predictor.
- [raw/code-research-claude-code.md](../raw/code-research-claude-code.md) -- Code research, Apr 2026. 4-layer compaction cascade, forked agent pattern for background work, post-compact file re-injection, circuit breaker on autocompact failures.
- [raw/code-research-karpathy-autoresearch.md](../raw/code-research-karpathy-autoresearch.md) -- Code research, Apr 2026. "NEVER STOP" directive with human interruption as only termination, git-as-state-machine for interrupt-safe progress.
