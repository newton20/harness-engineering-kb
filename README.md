# Agent Harness Engineering Knowledge Base

A [Karpathy-style](https://karpathy.ai/blog/fringe.html) LLM knowledge base covering how AI coding agents (Claude Code, Codex, Cursor, Windsurf, etc.) are built, configured, and optimized.

**61 raw sources. 15 synthesized wiki articles. 76 tags.** Every claim cited. Cross-referenced.

## Why This Exists

"Harness engineering" is the emerging discipline of configuring and extending AI coding agents to get dramatically better results. The same model scores 42% with one harness and 78% with another — the difference isn't intelligence, it's architecture. Harness-only tuning on Deep Agents took TerminalBench from 52.8% → 66.5% with no model change.

This KB collects, synthesizes, and cross-references the best thinking on the topic from Anthropic, OpenAI, LangChain, Letta, Y Combinator, arXiv, and independent practitioners. It is maintained by an LLM agent using the [kb-skills](https://github.com/newton20/kb-skills) toolkit.

## How to Use

**Read it.** Start at [wiki/_index.md](wiki/_index.md) or open any article below — each one stands alone but links to related pages.

**Query it.** Ask questions with citations via the `/kb-query` Claude Code skill, or run `npm run query` locally.

**Extend it.** Append URLs to [`raw_source_list.txt`](raw_source_list.txt), then run `/kb-ingest` followed by `/kb-compile`. The ingest script resolves tweets via X API v2, articles via direct HTTPS or headed browser, PDFs via `pdftotext` (with arxiv HTML-alt preferred). See [CLAUDE.md](CLAUDE.md) for the full fetcher chain.

## Wiki Articles

| Article | What It Covers |
|---------|---------------|
| [What Is Harness Engineering?](wiki/what-is-harness-engineering.md) | Definition, three/four-layer model, thin-harness debate, benchmark evidence |
| [Claude Code Architecture](wiki/claude-code-architecture.md) | The `while(tool_call)` loop, ~18 tools, TodoWrite, progressive disclosure, CLAUDE.md hierarchy |
| [OpenAI Codex Harness](wiki/openai-codex-harness.md) | Zero hand-written code experiment, AGENTS.md as TOC, agent legibility, garbage collection |
| [Long-Running Agent Harnesses](wiki/long-running-agent-harnesses.md) | Multi-session agents, initializer+coding agent, planner/generator/evaluator, context resets |
| [Auto Mode and Safety](wiki/auto-mode-and-safety.md) | Transcript classifier, two-stage pipeline, reasoning-blind design, deny-and-continue |
| [Agent Memory and Context Management](wiki/agent-memory-and-context-management.md) | MemGPT, filesystem > specialized tools, Context Constitution, markdown-vs-database debate, Cognee three-store |
| [Tool Design Patterns](wiki/tool-design-patterns.md) | Action space design, AskUserQuestion evolution, Vercel's 80% tool deletion, lazy loading |
| [Autoresearch and Self-Improvement](wiki/autoresearch-and-self-improvement.md) | Karpathy's autoresearch, GAN-inspired evaluators, autocontext, yes/no checklists |
| [Practical Best Practices](wiki/practical-best-practices.md) | Less is more, sycophancy handling, progressive disclosure, YOLO mode safety, multi-agent delegation |
| [Deep Research Agents](wiki/deep-research-agents.md) | Search-reason loops, orchestrator-worker patterns, convergence detection, economics |
| [Agentic Design Patterns](wiki/agentic-design-patterns.md) | ReAct, Reflection, Planning, Tool Use, Multi-Agent as formal design patterns |
| [Multi-Agent Reliability](wiki/multi-agent-reliability.md) | Credibility scoring, adversary resistance, push-based announcements, orphan recovery, writes-stay-single-threaded |
| [Thin Harness, Fat Skills](wiki/thin-harness-fat-skills.md) | Garry Tan's three-tier architecture, the Chase–Tan debate, resolvers, skill ownership |
| [Self-Evolving Agents and Skillify](wiki/self-evolving-agents.md) | Autogenesis protocol (RSPL + SEPL), skill refinement loops, version lineage, rollback |

Full index with tag navigation: [wiki/_index.md](wiki/_index.md)

## Headline Findings

Synthesized across all 61 sources:

1. **The harness matters more than the model.** Claude Opus 4.5 scores 42% with one scaffold and 78% with another — same model, same benchmark.
2. **TodoWrite is a no-op.** Claude Code's TodoWrite tool does nothing functionally. It exists to force the agent to articulate and track its plan over long trajectories.
3. **Filesystem tools beat specialized memory tools.** Letta agents using `grep` and `search_files` on gpt-4o-mini scored 74.0% on LoCoMo, beating Mem0's graph-based memory at 68.5%.
4. **OpenAI built a product with zero hand-written code.** 1M lines, 1,500 PRs, 3 engineers. Every line written by Codex. The engineers' entire job was harness engineering.
5. **AGENTS.md works as a table of contents, not an encyclopedia.** OpenAI tried one big AGENTS.md and it failed. ~100 lines pointing to structured `docs/` works.
6. **OpenClaw "dreams" to consolidate memory autonomously.** Three cron-scheduled phases (light/deep/REM) promote high-recall short-term fragments into long-term memory without agent involvement.
7. **Skills are prompt injections, not tools.** OpenClaw advertises capabilities via an XML catalog in the system prompt, then the model lazily loads SKILL.md files via read. First production implementation of Tan's "thin harness, fat skills."
8. **Writes stay single-threaded in multi-agent systems.** Parallel read, serial write is the pattern that actually ships (Walden Yan, Cognition).
9. **1.6% of Claude Code's code is agentic; 98.4% is harness.** The UCL/MBZUAI dissection of Claude Code found the LLM call sites are a tiny sliver — the rest is scaffolding.

## Sources

The [`raw/`](raw/) directory holds 61 primary sources, preserved with YAML frontmatter and fetcher provenance:

**Anthropic Engineering** — Effective harnesses for long-running agents, Claude Code auto mode, Harness design for long-running apps
**OpenAI Engineering** — Harness engineering: leveraging Codex, Unlocking the Codex harness
**LangChain** — Improving Deep Agents with harness engineering (52.8% → 66.5% on TerminalBench)
**arXiv papers** — Dive into Claude Code (the 1.6%/98.4% dissection), Autogenesis: Self-Evolving Agent Protocol, AutoHarness, Adversary-resistant multi-agent collaboration, Agentic RAG survey
**Practitioners** — Garry Tan (YC), Harrison Chase (LangChain), Sarah Wooders (Letta), Thariq (Claude Code), Walden Yan (Cognition), Akshay Pachaar, Simon Willison, and more
**Deep code research reports** — karpathy/autoresearch, Claude Code (local source), openclaw/openclaw, each via the `/kb-code-research` parallel-agent pipeline

Full list: [raw_source_list.txt](raw_source_list.txt).

## Methodology

- **Cross-pollination.** Each source touches 5–15 wiki pages, not just one. Bidirectional backlinks in every `## Related` section.
- **Inline citations.** Every factual claim cites its source: `[Source: raw/filename.md]`.
- **Authoritative fetches.** X API v2 `article.plain_text` / `note_tweet.text` before Grok fallback — Grok can hallucinate ~60% of content on recent posts. See `CLAUDE.md` for the full chain.
- **Contradiction handling.** When sources disagree (e.g., the Chase–Tan thin-harness debate), both positions are presented with attribution rather than smoothed over.
- **Supervised compilation.** For small batches, the compiler discusses takeaways before writing. For large batches, it runs delta-only.
- **Monthly lint.** `/kb-lint` runs structural checks (broken links, orphans) plus semantic checks (contradictions, staleness, gaps).

## Project Structure

```
raw/                 # 61 fetched source documents (immutable after ingest)
raw/images/          # Downloaded tweet images and diagrams
wiki/                # 15 synthesized articles + _index.md + lint reports
outputs/             # Generated reports and explorations
scripts/             # Node.js automation (ingest.js, compile.js, query.js)
docs/                # Plans, brainstorms, solutions, handoffs
CLAUDE.md            # How the pipeline works (fetcher chain, schema, fallbacks)
log.md               # Append-only operation log
ingest_manifest.json # Per-URL fetch status + source_method provenance
```

## Skills

The `/kb-*` skills live in [newton20/kb-skills](https://github.com/newton20/kb-skills):

```
/kb-ingest <url>          Fetch sources (tweets, articles, papers, GitHub repos, PDFs)
/kb-compile               Synthesize wiki articles with cross-pollination
/kb-query "question"      Answer questions with citations
/kb-lint                  Health check (contradictions, orphans, gaps)
/kb-explore               Find unexplored connections between topics
/kb-code-research <repo>  Deep code research with 4 parallel specialist agents
/kb-status                Source counts, wiki totals, pending work
```

## Recent Changes

The current line of work (PR #3 on this branch) rounded out a few gaps in the ingest pipeline and added the next batch of sources:

- **PDF handling** via `pdftotext` with an arxiv HTML-alternative tried first (`arxiv.org/html/<id>` avoids the column-interleaving problem on 2-column academic PDFs).
- **X API v2 `note_tweet.text`** fallback joins `article.plain_text` in the authoritative path, covering long regular tweets (>280 chars) that aren't X Articles.
- **`KB_ENV_FILE` whitespace tolerance** — the env-file parser now accepts `X_BEARER_TOKEN = value`.
- **9 fresh raw sources + 7 manual X transcriptions** (Tan's four-tweet thin-harness thread, Chase's "directionally correct" concession, the Akshay Pachaar anatomy synthesis, Walden Yan on multi-agents, the Autogenesis paper, and more).
- **2 new wiki articles** — `thin-harness-fat-skills` and `self-evolving-agents` — plus cross-pollination updates to 8 existing articles.

See [`log.md`](log.md) for the append-only operation history.

## License

Content is synthesized from public sources with attribution. Raw sources retain their original copyright.
