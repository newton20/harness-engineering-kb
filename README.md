# Agent Harness Engineering Knowledge Base

A [Karpathy-style](https://karpathy.ai/blog/fringe.html) LLM knowledge base covering how AI coding agents (Claude Code, Codex, Cursor, Windsurf, etc.) are built, configured, and optimized.

29 sources. 9 synthesized wiki articles. Every claim cited. Cross-referenced.

## Why This Exists

"Harness engineering" is the emerging discipline of configuring and extending AI coding agents to get dramatically better results. The same model scores 42% with one harness and 78% with another. The difference isn't intelligence. It's architecture.

This knowledge base collects, synthesizes, and cross-references the best thinking on the topic from Anthropic, OpenAI, LangChain, Letta, Y Combinator, and independent practitioners. It's maintained by an LLM agent using the [kb-skills](https://github.com/newton20/kb-skills) toolkit.

## Wiki Articles

| Article | What It Covers |
|---------|---------------|
| [What Is Harness Engineering?](wiki/what-is-harness-engineering.md) | The definition, three/four-layer model, "thin harness fat skills", benchmark evidence |
| [Claude Code Architecture](wiki/claude-code-architecture.md) | The `while(tool_call)` loop, ~18 tools, TodoWrite, progressive disclosure, CLAUDE.md hierarchy |
| [OpenAI Codex Harness](wiki/openai-codex-harness.md) | Zero hand-written code experiment, AGENTS.md as TOC, agent legibility, garbage collection |
| [Long-Running Agent Harnesses](wiki/long-running-agent-harnesses.md) | Multi-session agents, initializer+coding agent, planner/generator/evaluator, context resets |
| [Auto Mode and Safety](wiki/auto-mode-and-safety.md) | Transcript classifier, two-stage pipeline, reasoning-blind design, deny-and-continue |
| [Agent Memory and Context Management](wiki/agent-memory-and-context-management.md) | MemGPT, filesystem > specialized tools, Context Constitution, the markdown-vs-database debate |
| [Tool Design Patterns](wiki/tool-design-patterns.md) | Action space design, AskUserQuestion evolution, Vercel's 80% tool deletion, lazy loading |
| [Autoresearch and Self-Improvement](wiki/autoresearch-and-self-improvement.md) | Karpathy's autoresearch, GAN-inspired evaluators, autocontext, yes/no checklists |
| [Practical Best Practices](wiki/practical-best-practices.md) | Less is more, sycophancy handling, progressive disclosure, YOLO mode safety, multi-agent delegation |

Browse the full index: [wiki/_index.md](wiki/_index.md)

## Sources

29 raw sources in `raw/`, including:

**Anthropic Engineering Blog**
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) (Nov 2025)
- [Claude Code auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode) (Mar 2026)
- [Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) (Mar 2026)

**OpenAI Engineering Blog**
- [Harness engineering: leveraging Codex](https://openai.com/index/harness-engineering/) (Feb 2026)
- [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/) (Feb 2026)

**Independent Practitioners**
- Garry Tan (YC) on "thin harness, fat skills"
- Harrison Chase (LangChain) on the three-layer model and continual learning
- Sarah Wooders (Letta) on memory as core harness function
- Thariq (Claude Code team) on action space design
- @systematicls on minimal harness philosophy
- @Hxlfed14 on cross-company benchmark evidence
- Simon Willison on designing agentic loops
- And 16 more tweets, articles, and papers

**Academic**
- [AutoHarness: improving LLM agents by automatically synthesizing a code harness](https://arxiv.org/html/2603.03329v1) (arxiv)

Full source list: [raw_source_list.txt](raw_source_list.txt)

## How It's Built

This KB is maintained using [kb-skills](https://github.com/newton20/kb-skills), a set of reusable Claude Code skills that implement the Karpathy second-brain methodology:

```
/kb-ingest <url>     Fetch sources (tweets, articles, papers, GitHub repos)
/kb-compile          Synthesize wiki articles with cross-pollination
/kb-query "question" Answer questions with citations
/kb-lint             Health check (contradictions, orphans, gaps)
/kb-explore          Find unexplored connections between topics
```

### Methodology

- **Cross-pollination**: Each source touches 5-15 wiki pages, not just one. Bidirectional backlinks in every `## Related` section.
- **Inline citations**: Every factual claim cites its source: `[Source: raw/filename.md]`
- **Supervised compilation**: For small batches, the AI discusses key takeaways before writing.
- **Contradiction handling**: When sources disagree, both claims are presented with attribution.
- **Monthly lint**: Structural checks (broken links, orphans) + semantic checks (contradictions, staleness, gaps).

### Ingest Pipeline

The ingest script handles multiple source types with a 3-tier fallback for blocked sites:

| Source Type | Method |
|-------------|--------|
| x.com tweets | ScrapeCreators API |
| x.com articles | xAI Grok (X Articles detected when tweet text < 100 chars) |
| GitHub repos | Raw README.md fetch |
| Web articles | Direct HTTPS fetch |
| Cloudflare-blocked sites | `agent-browser --headed` (real browser) |
| Last resort | xAI Grok search |

## Project Structure

```
raw/                 # 29 fetched source documents (immutable)
raw/images/          # Downloaded tweet images
wiki/                # 9 synthesized wiki articles + index + lint reports
outputs/             # Generated reports and explorations
scripts/             # Node.js automation (ingest.js, compile.js, query.js)
docs/plans/          # Implementation plans
docs/solutions/      # Documented learnings and bug fixes
CLAUDE.md            # Knowledge base schema
log.md               # Append-only operation log
ingest_manifest.json # Per-URL fetch status tracking
```

## Key Insights from the KB

Some of the most interesting findings synthesized across all 29 sources:

1. **The harness matters more than the model.** Claude Opus 4.5 scores 42% with one scaffold and 78% with another. Same model. Same benchmark. The only variable is the harness.

2. **TodoWrite is a no-op.** Claude Code's TodoWrite tool does nothing functionally. It exists purely to force the agent to articulate and track its plan, keeping it on course over long trajectories.

3. **Filesystem tools beat specialized memory tools.** Letta agents using simple `grep` and `search_files` on gpt-4o-mini achieved 74.0% on LoCoMo, beating Mem0's specialized graph-based memory at 68.5%.

4. **OpenAI built a product with zero hand-written code.** 1 million lines, 1,500 PRs, 3 engineers. Every line written by Codex. The engineers' job was entirely harness engineering.

5. **AGENTS.md should be a table of contents, not an encyclopedia.** OpenAI tried "one big AGENTS.md" and it failed. ~100 lines pointing to structured `docs/` works. Mechanical enforcement via linters and CI keeps it honest.

## Contributing

Add sources by appending URLs to `raw_source_list.txt` and running:

```bash
# In Claude Code
/kb-ingest raw_source_list.txt
/kb-compile
```

Or open an issue with suggested URLs.

## License

Content is synthesized from public sources with attribution. Raw sources retain their original copyright.
