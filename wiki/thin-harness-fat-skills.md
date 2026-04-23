---
title: "Thin Harness, Fat Skills"
type: wiki
tags:
  - thin-harness
  - fat-skills
  - fat-code
  - skills
  - resolvers
  - skillify
  - memory-ownership
  - harness-engineering
  - three-tier-architecture
sources:
  - raw/garrytan-2042925773300908103.md
  - raw/garrytan-2043198780800197025.md
  - raw/garrytan-2043198783006355747.md
  - raw/garrytan-2043565174066475472.md
  - raw/garrytan-2043566215927328955.md
  - raw/garrytan-2044479509874020852.md
  - raw/garrytan-2046876981711769720.md
  - raw/hwchase17-2042978500567609738.md
  - raw/hwchase17-2043354478074581227.md
  - raw/jasonkneen-2043435856849940818.md
  - raw/jasonkneen-cursor-3.0-gist.md
  - raw/akshay_pachaar-2045404494641733962.md
  - raw/arxiv-org-html-2604-14228.md
source_count: 13
status: draft
last_compiled: 2026-04-23
---

# Thin Harness, Fat Skills

"Thin harness, fat skills" is a three-tier agent architecture articulated by Garry Tan (YC President/CEO) in April 2026 after studying Anthropic's accidentally-published Claude Code source (512,000 lines on npm, March 31, 2026). The thesis: push fuzzy judgment into markdown *skills*, push deterministic operations into *code*, and keep the *harness* thin — a ~200 line conductor that reads files rather than owning them. The architecture has been independently validated at scale (Cursor ships Claude Code as its harness), contested on memory-ownership grounds by Harrison Chase (LangChain), and formalized into a 10-step "skillify" operational practice for converting failures into durable tests. [Source: raw/garrytan-2043566215927328955.md] [Source: raw/arxiv-org-html-2604-14228.md]

## The Three-Tier Architecture

Tan's cleanest statement of the architecture, from his April 12, 2026 self-quote-tweet:

> Push smart fuzzy operations humans do into markdown skills. **Fat skills.**
> Push must-be-perfect deterministic operations into code. **Fat code.**
> The harness? Keep it thin. [Source: raw/garrytan-2043566215927328955.md]

The layered architecture:

1. **Fat skills on top** — markdown procedures that encode judgment, process, and domain knowledge. "Where 90% of the value lives." A skill file works like a method call: same procedure, radically different capabilities depending on what you pass in. [Source: raw/garrytan-2043566215927328955.md]
2. **Thin CLI harness in the middle** — about 200 lines of code. JSON in, text out. Read-only by default. Reads the files, doesn't own them. [Source: raw/garrytan-2043566215927328955.md] [Source: raw/garrytan-2043198780800197025.md]
3. **Fat code on the bottom** — the deterministic foundation. QueryDB, ReadDoc, Search, Timeline. Same input, same output, every time. No model needed. [Source: raw/garrytan-2043566215927328955.md]

The directional principle: **push intelligence up into skills, push execution down into deterministic tooling, keep the harness thin.** When you do this, every model improvement automatically lifts every skill while the deterministic layer stays perfectly reliable. [Source: raw/garrytan-2043566215927328955.md]

## The Loop That Makes the Architecture Work

A critical observation from Tan's April 22, 2026 "Skillify" article: the three tiers aren't static — they recursively generate each other. Latent (skill) work builds deterministic (code) tooling, then the deterministic tool constrains the latent space:

> The agent used judgment (latent) to write calendar-recall.mjs. Now the skill forces the agent to run that script instead of reasoning about calendar data. The model's intelligence created the constraint that prevents the model from being stupid. [Source: raw/garrytan-2046876981711769720.md]

Tan's distinction between **latent** (work requiring judgment) and **deterministic** (work requiring precision) is the decision rule: calendar grep is deterministic, so the agent should run a script rather than spinning up reasoning and API calls. The bug isn't a wrong answer — "It's a wrong side." [Source: raw/garrytan-2046876981711769720.md]

## The Claude Code Dissection: 1.6% Model, 98.4% Harness

A UCL/MBZUAI research paper (arXiv:2604.14228, "Dive into Claude Code") reverse-engineered the leaked Claude Code v2.1.88 source and empirically validated the thin-harness thesis. The headline finding, as summarized by Akshay Pachaar: **only 1.6% of the codebase is AI decision logic; the other 98.4% is operational infrastructure** — permission gates, tool routing, context compaction, recovery logic, session persistence. [Source: raw/akshay_pachaar-2045404494641733962.md]

The paper's own framing agrees: "The core of the system is a simple while-loop that calls the model, runs tools, and repeats. Most of the code, however, lives in the systems around this loop: a permission system with seven modes and an ML-based classifier, a five-layer compaction pipeline for context management, four extensibility mechanisms (MCP, plugins, skills, and hooks), a subagent delegation and orchestration mechanism, and append-oriented session storage." [Source: raw/arxiv-org-html-2604-14228.md]

Crucial nuance: Anthropic gives the model **maximum decision latitude inside a rich deterministic harness**. This inverts the pattern most agent frameworks follow. LangGraph routes model outputs through explicit state machines. Devin bolts heavy planners onto operational scaffolding. Claude Code bets that as frontier models converge on raw capability, the quality of the harness — not the model — becomes the differentiator. [Source: raw/akshay_pachaar-2045404494641733962.md]

## Resolvers: The Routing Table for Intelligence

Tan's most underrated contribution is the **resolver** — a ~200-line markdown routing table that replaced his 20,000-line CLAUDE.md. A resolver maps task types to context: "When task type X appears, load document Y first." Invisible when it works, catastrophic when it doesn't. [Source: raw/garrytan-2044479509874020852.md]

### The 20,000-Line Confession

Tan's CLAUDE.md grew to 20,000 lines as he accumulated every quirk, pattern, lesson, and convention from daily Claude Code use. The model's attention degraded. Responses got slower and less precise. "Claude Code literally told me to cut it back. That's when you know you've gone too far — the AI is telling you to stop talking." [Source: raw/garrytan-2044479509874020852.md]

The fix was ~200 lines: a numbered decision tree with pointers. Person? → `/people/`. Company? → `/companies/`. Policy analysis? → `/civic/`. Twenty thousand lines of knowledge remained accessible on demand, without polluting the context window. The system immediately got faster, more accurate, and hallucinated less — "not because the model got smarter. Because I stopped blinding it with noise." [Source: raw/garrytan-2044479509874020852.md]

### Why Every Skill Must Consult the Resolver

Tan audited 13 skills that wrote to his knowledge base. Only 3 referenced the resolver. The other 10 had hardcoded paths — idea-ingest defaulted to `sources/`, PDF-ingest to `originals/`, meeting-ingest to `meetings/`. Each was a misfiling waiting to happen. The fix: a `_brain-filing-rules.md` document plus a mandate that every brain-writing skill reads `RESOLVER.md` before creating any page. One rule, ten skills fixed. "Zero misfilings since." [Source: raw/garrytan-2044479509874020852.md]

### The Invisible-Skill Problem

After a month of building, Tan had 40+ skills — some user-authored, some spawned by sub-agent crons. Skills were being born but not registered in the resolver. He built **check-resolvable**, a meta-skill that walks the entire chain (AGENTS.md → skill file → code) and finds dead links. First run found 6 unreachable skills out of 40+. **Fifteen percent of the system's capabilities were dark.** A flight tracker nobody could invoke by asking about flights. A content-ideas generator that only ran on cron. A citation fixer listed nowhere in the resolver. "Worse than not having the skill at all, because you think the system handles it." Now check-resolvable runs weekly. [Source: raw/garrytan-2044479509874020852.md] [Source: raw/garrytan-2046876981711769720.md]

### Resolvers Decay: Trigger Evals Against Context Rot

Resolvers rot. Day 1 every trigger is accurate. Day 30, three new skills aren't registered. Day 60, two trigger descriptions don't match how users actually phrase things — skill handles "track this flight" but users say "is my flight delayed?" Day 90, the resolver is a historical document. [Source: raw/garrytan-2044479509874020852.md]

Tan's fix is **resolver trigger evals** — a test suite of 50+ sample inputs with expected skill mappings. Two failure modes: false negative (skill should fire but doesn't, trigger description wrong) and false positive (wrong skill fires, triggers overlap). Both fixable by editing markdown. "If you can't prove the right skill fires for the right input, you don't have a system. You have a collection of skills and a prayer." [Source: raw/garrytan-2044479509874020852.md]

Forward-looking: a CTO in YC office hours asked whether an RLM could solve context rot around resolvers. The idea — observe every task dispatch, periodically rewrite the resolver based on which skills fired, which didn't, and which tasks had no match. Claude Code's **AutoDream** memory consolidation is a primitive version. Apply that principle to the resolver and you get a routing table that improves with use. [Source: raw/garrytan-2044479509874020852.md]

### Resolvers Are Fractal

Resolvers compose. They exist at every layer. The **skill resolver** (AGENTS.md) maps task types to skill files. The **filing resolver** (RESOLVER.md) maps content types to directories. The **context resolver** lives inside each skill as sub-routing. Claude Code's built-in resolver uses the `description` field of every skill — the model matches user intent to skill descriptions automatically. "It's resolvers all the way down." [Source: raw/garrytan-2044479509874020852.md]

## Skillify: The 10-Step Operational Practice

Tan's April 22, 2026 article formalized "skillify" — a verb and a practice for converting every agent failure into a permanent structural fix. "Every failure becomes a skill. Every skill has tests. The bug becomes structurally impossible to repeat." [Source: raw/garrytan-2046876981711769720.md]

The 10-step checklist, in order:

1. **SKILL.md** — the contract (name, triggers, rules)
2. **Deterministic code** — scripts/*.mjs (no LLM for what code can do)
3. **Unit tests** — vitest, pure functions, fixture data
4. **Integration tests** — live endpoints, real data
5. **LLM evals** — quality + correctness judged by LLM-as-judge
6. **Resolver trigger** — entry in AGENTS.md
7. **Resolver eval** — verify the trigger actually routes
8. **Check-resolvable + DRY audit** — reachability and overlap
9. **E2E smoke test** — full pipeline end-to-end
10. **Brain filing rules** — where outputs live

"A feature that doesn't pass all ten is not a skill. It's just code that happens to work today." [Source: raw/garrytan-2046876981711769720.md]

### Skillify as Verb: "We Built It, Now Make It Permanent"

The pattern in daily use: prototype in conversation, see it work, say "skillify." The ad-hoc session becomes a durable skill with tests, resolver entry, and documentation. Tan's examples:

- "hot damn it worked. can you remember this as a webhook skill and skillify it..."
- "whenever anything in openclaw needs a headless browser... skillify it!"
- "whenever you send me a link you have to curl it yourself to make sure... skillify it!"

"One sentence. Code, skill, tests, resolver entry, reachability audit. The whole 10-step checklist in one breath." [Source: raw/garrytan-2046876981711769720.md]

### Two Concrete Failures That Shaped the Practice

**Failure 1 (calendar-recall):** Tan asked his agent about an old business trip. The agent tried the live calendar API (blocked for old events), tried email search (noisy), tried the API again — five minutes wasted before finally grepping the local knowledge base and finding the answer instantly. The answer had been one grep away all along. The fix: a `calendar-recall` skill whose hard rule is "Live calendar APIs are ONLY for events in the FUTURE or the LAST 48 HOURS. Everything historical goes through the local knowledge base first." The agent then wrote the deterministic script itself. [Source: raw/garrytan-2046876981711769720.md]

**Failure 2 (context-now):** Agent said "your next meeting is in 28 minutes." Actual answer: 88 minutes. The agent had done UTC→PT timezone math mentally and was off by exactly an hour. A `context-now.mjs` script already existed that produced correct JSON in 50ms. The agent just didn't run it. Fix: a skill whose rule is "ALWAYS-ON discipline: run context-now.mjs before making ANY time-sensitive claim. Never do UTC→PT conversion in your head." [Source: raw/garrytan-2046876981711769720.md]

### Why Hermes Agent Isn't Enough

Nous Research's Hermes Agent has a `skill_manage` tool that lets the agent itself create, patch, and delete skills — procedural memory the agent earns on its own. Progressive disclosure (skill index first, full SKILL.md on select). Bounded memory (MEMORY.md capped at 2,200 chars). Smart design. But Hermes doesn't test its skills: no unit tests, no resolver evals, no check-resolvable, no DRY audit, no daily health check. "Hermes handles creation beautifully. GBrain handles verification. You need both." [Source: raw/garrytan-2046876981711769720.md]

## The Chase–Tan Debate: Memory Ownership

Harrison Chase (LangChain) published "Your harness, your memory" (April 11, 2026, 10,578 bookmarks, 1.88M impressions) arguing that agent harnesses and memory are inseparable, and that using a closed harness behind a proprietary API means yielding ownership of your memory — the flywheel, the proprietary dataset, the personalization substrate. [Source: raw/hwchase17-2042978500567609738.md]

Chase's key claims:

- **Harnesses aren't going away.** Evidence: Claude Code's leaked source is 512k lines. "Even the makers of the best model in the world are investing heavily in harnesses." [Source: raw/hwchase17-2042978500567609738.md]
- **Memory is just a form of context, and harnesses control context.** Quoting Sarah Wooders (Letta CTO): "Asking to plug memory into an agent harness is like asking to plug driving into a car. Managing context, and therefore memory, is a core capability and responsibility of the agent harness." [Source: raw/hwchase17-2042978500567609738.md]
- **Three degrees of memory lock-in, worst to best:**
  - *Worst:* Whole harness (including long-term memory) behind an API — Anthropic's Claude Managed Agents puts literally everything behind an API.
  - *Bad:* Closed harness (like Claude Agent SDK) — interacts with memory in unknown ways.
  - *Mildly bad:* Stateful API (OpenAI Responses API, Anthropic server-side compaction) — can't swap models and resume threads.
- **Even open-source Codex generates encrypted compaction summaries unusable outside OpenAI's ecosystem.** [Source: raw/hwchase17-2042978500567609738.md]

### Tan's Counterpoint: "Memory Is Markdown"

Tan quote-tweeted Chase the same day with the debate's sharpest line:

> If your memory dies when your harness dies, you built the harness too thick.
> Memory is markdown. Skills are markdown. Brain is a git repo. The harness is a thin conductor — it reads the files, it doesn't own them. [Source: raw/garrytan-2043198780800197025.md]

Then self-quoted his own "Thin Harness, Fat Skills" article as a counterpoint "after 3 months of releasing open source used by tens of thousands of agentic engineers per day." [Source: raw/garrytan-2043198783006355747.md]

### Chase's Concession

Chase replied the next day: **"Directionally correct. Open memory standards will need to emerge. But it's so early right now. We're still just figuring out what best practices are. And so a lot is at the mercy of harnesses. Agents.md and skills are great start though."** [Source: raw/hwchase17-2043354478074581227.md]

The debate resolves on timing rather than principle. Both agree memory should be open. Chase argues that best-practice memory abstractions haven't been discovered yet, so pragmatically the harness has to own memory today. Tan argues that the open-markdown-file substrate (AGENTS.md, CLAUDE.md, skills, brain-as-git-repo) is already the right primitive, and every harness that hoards state in opaque internal structures is building itself too thick.

## Independent Validation: Cursor Agent Is Claude Code

On April 12, 2026, Jason Kneen reverse-engineered Cursor 3.0 and reported the headline finding:

> "Cursor Agent" is a rebranded Claude Code running behind a local proxy with a find-and-replace on messages engine that swaps "Claude" → "Cursor" in system prompts and messages. They bundle the full @anthropic-ai/claude-agent-sdk and @anthropic-ai/claude-code packages, plus a custom fine-tuned model `claude-3.7-sonnet-finetuned-cursor-20250514-v1`. [Source: raw/jasonkneen-2043435856849940818.md]

Cursor — valued at roughly $10B — chose the thinnest possible harness (renting Anthropic's) and differentiated on the fine-tuned model plus IDE surface. Tan's three-word response captured the implication: **"Thin harness fat skill confirmed."** [Source: raw/garrytan-2043565174066475472.md]

The Kneen finding simultaneously validates both sides of the Chase-Tan debate:

- **Tan's thesis at scale:** Cursor proved a thin-harness strategy works economically — build on top of someone else's harness, differentiate above.
- **Chase's warning made concrete:** Cursor now depends on Anthropic's harness, SDK, and memory model for its core product. The thinnest harness outsources the most. [Source: raw/jasonkneen-cursor-3.0-gist.md]

## Why the Thin Harness Is Skeptical of Frameworks

Tan's April 22 article takes direct aim at LangChain's $160M/3-year ecosystem: "LangChain gives you testing tools. It never tells you what to test, in what order, or when you're done. There's no opinionated workflow that says, in order: this failure happened → now write a skill → now write the deterministic code → now write unit tests → now write LLM evals → now add a resolver trigger..." [Source: raw/garrytan-2046876981711769720.md]

The critique isn't against the primitives — LangSmith has trajectory evals, trace-to-dataset pipelines, LLM-as-judge, regression suites. "Pieces aren't a practice." The missing piece is the workflow: the moment where a human says "that worked, now make it permanent" and the system knows exactly what "permanent" means. That's skillify. [Source: raw/garrytan-2046876981711769720.md]

## The Management Analogy

Tan's final framing reinterprets the entire architecture as organizational design:

- **Skills are employees** — each has a capability.
- **Resolvers are the org chart** — who handles what, how tasks route, what escalation looks like.
- **Filing rules are internal process** — where information lives, how decisions get recorded.
- **check-resolvable is audit and compliance** — can the system actually do what it claims?
- **Trigger evals are performance reviews** — given real input, does the right part of the org respond?

"The problem isn't that models aren't smart enough. The problem is that we've been building organizations with no management layer. Just a pile of talented employees and a vague hope they'll coordinate. Resolvers are that missing layer." [Source: raw/garrytan-2044479509874020852.md]

## Related

- [Claude Code Architecture](claude-code-architecture.md) — the production system whose 1.6%/98.4% split validates the thesis
- [What Is Harness Engineering?](what-is-harness-engineering.md) — the broader discipline; Tan's three-tier model is one of its defining architectures
- [Agent Memory and Context Management](agent-memory-and-context-management.md) — the memory-ownership debate at the heart of the Chase-Tan exchange
- [Tool Design Patterns](tool-design-patterns.md) — "thin harness" translates to action-space design: primitives over integrations, ~200-line harness, deterministic backend
- [Self-Evolving Agents and Skillify](self-evolving-agents.md) — the skillify 10-step loop and resolver learning as mechanisms for agent self-improvement
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) — resolvers and skillify as patterns for multi-session continuity
- [Practical Best Practices](practical-best-practices.md) — concrete operational practices that flow from the three-tier architecture
- [Autoresearch and Self-Improvement](autoresearch-and-self-improvement.md) — self-healing resolvers as a continuous-improvement loop

## Open Questions

- Can RLM-based self-healing resolvers actually converge in production? Tan identifies this as the endgame but notes "we haven't fully built it." Claude Code's AutoDream is a primitive version. [Source: raw/garrytan-2044479509874020852.md]
- At what scale does "thin harness" break down? Tan reports 200 inputs/day, 25,000 files, 40+ skills in his personal OpenClaw. No public data on thin-harness economics at enterprise scale.
- How do teams without GBrain/GStack adopt skillify? Tan ships a reference implementation but the pattern hasn't been independently replicated in other stacks. [Source: raw/garrytan-2046876981711769720.md]
- Chase's "open memory standards will need to emerge" — what would such a standard look like beyond agents.md and skills? Undefined as of April 2026. [Source: raw/hwchase17-2043354478074581227.md]

## Sources

- [raw/garrytan-2042925773300908103.md](../raw/garrytan-2042925773300908103.md) — Garry Tan, April 2026. Original "Thin Harness, Fat Skills" article with the three-tier architecture.
- [raw/garrytan-2043198780800197025.md](../raw/garrytan-2043198780800197025.md) — Garry Tan, April 11, 2026. Quote-tweet of Chase: "If your memory dies when your harness dies, you built the harness too thick."
- [raw/garrytan-2043198783006355747.md](../raw/garrytan-2043198783006355747.md) — Garry Tan, April 11, 2026. "Here is my counterpoint after 3 months" — self-quotes his article as counterpoint to Chase.
- [raw/garrytan-2043565174066475472.md](../raw/garrytan-2043565174066475472.md) — Garry Tan, April 12, 2026. "Thin harness fat skill confirmed" in response to Kneen's Cursor 3.0 finding.
- [raw/garrytan-2043566215927328955.md](../raw/garrytan-2043566215927328955.md) — Garry Tan, April 12, 2026. The cleanest three-tier statement: "Fat skills. Fat code. Thin harness."
- [raw/garrytan-2044479509874020852.md](../raw/garrytan-2044479509874020852.md) — Garry Tan, April 15, 2026. "Resolvers: The Routing Table for Intelligence" — the 20,000-line CLAUDE.md confession, check-resolvable, the management analogy.
- [raw/garrytan-2046876981711769720.md](../raw/garrytan-2046876981711769720.md) — Garry Tan, April 22, 2026. "How to really stop your agents from making the same mistakes" — the skillify 10-step practice, calendar-recall, context-now, Hermes critique.
- [raw/hwchase17-2042978500567609738.md](../raw/hwchase17-2042978500567609738.md) — Harrison Chase, April 11, 2026. "Your harness, your memory" — the memory-ownership argument and three-tier lock-in taxonomy.
- [raw/hwchase17-2043354478074581227.md](../raw/hwchase17-2043354478074581227.md) — Harrison Chase, April 12, 2026. "Directionally correct" — the concession that resolves the debate on timing.
- [raw/jasonkneen-2043435856849940818.md](../raw/jasonkneen-2043435856849940818.md) — Jason Kneen, April 12, 2026. Cursor 3.0 reverse-engineering — "Cursor Agent is a rebranded Claude Code."
- [raw/jasonkneen-cursor-3.0-gist.md](../raw/jasonkneen-cursor-3.0-gist.md) — Jason Kneen, April 12, 2026. Full reverse-engineering report summary.
- [raw/akshay_pachaar-2045404494641733962.md](../raw/akshay_pachaar-2045404494641733962.md) — Akshay Pachaar, April 18, 2026. Summary of UCL/MBZUAI Claude Code dissection — the 1.6%/98.4% finding.
- [raw/arxiv-org-html-2604-14228.md](../raw/arxiv-org-html-2604-14228.md) — Liu et al. (VILA Lab, MBZUAI & UCL), April 2026. "Dive into Claude Code" — the paper behind the 1.6%/98.4% claim.
