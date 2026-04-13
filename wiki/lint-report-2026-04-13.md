---
title: "Lint Report 2026-04-13"
type: lint-report
date: 2026-04-13
---

# Wiki Lint Report — 2026-04-13

## Summary

- Total articles: 9
- Errors (must fix): 0
- Warnings (should fix): 7
- Info (nice to fix): 3

## Errors

None.

## Warnings

### Low Citation Density (structural)

1. **agent-memory-and-context-management.md** — 4/17 paragraphs lack inline source citations
2. **autoresearch-and-self-improvement.md** — 4/26 paragraphs lack inline source citations
3. **long-running-agent-harnesses.md** — 4/29 paragraphs lack inline source citations
4. **openai-codex-harness.md** — 4/30 paragraphs lack inline source citations

### Unsourced Specific Claims (semantic)

5. **tool-design-patterns.md** — "A tool the model uses correctly 95% of the time beats a more powerful tool it uses correctly 70% of the time." Specific quantitative claim appears invented for illustration, not drawn from any source.
6. **claude-code-architecture.md** — "Claude Code is built on the Claude Agent SDK." Architectural claim without source attribution.
7. **agent-memory-and-context-management.md** — "If memory is core, then the harness must own decisions about what enters context..." Editorial synthesis presented as factual claim without citation.

## Info

### Tool Count Discrepancy

8. **claude-code-architecture.md** — States "~18 primitive tools" in some places, "~20 tools" in others, and the Open Questions section references "45+ built-in tools." The tool table only lists 12. The article acknowledges this in Open Questions but body text is inconsistent. (medium severity)

### Confusing Juxtaposition

9. **autoresearch-and-self-improvement.md** — Presents "$200 / 6 hours" (game maker) and "$124.70 / ~4 hours" (DAW) in the same section without clearly labeling them as different projects. Could confuse readers.

### All Articles Status: Draft

10. All 9 articles have `status: draft`. None have been promoted to `reviewed`. This is expected for a first compile but should be addressed over time.

## Knowledge Gaps

Top 5 suggested articles to fill:

1. **Agent Skills / Skill Design** — mentioned in 6+ articles, dedicated unused raw source exists (`raw/trq212-2033949937936085378.md` on skill design lessons from Anthropic). Most significant gap.
2. **Multi-Agent Coordination** — referenced across 5 articles (sub-agents, Hermes vs OpenClaw, autocontext pipeline). Unused raw source: `raw/akshay_pachaar-2033167408463069526.md` on sub-agents vs agent teams.
3. **Benchmarking and Evaluation** — cited as evidence in 3+ articles but no dedicated methodology page. Unused raw source: `raw/arxiv-org-html-2603-03329v1.md` (AutoHarness paper).
4. **Prompt Caching and Token Economics** — key engineering constraint shaping many design decisions, referenced in 3 articles but no dedicated page.
5. **Context Engineering (vs Harness Engineering)** — the foundational distinction only gets a paragraph. Unused source: `raw/Hxlfed14-2022984467380682856.md` on context engineering across companies.

### Unreferenced Raw Sources (7 of 28)

These raw sources are not consumed by any wiki article:
- `raw/akshay_pachaar-2033167408463069526.md` — Sub-agents vs agent teams
- `raw/ArtemXTech-2028330693659332615.md` — Local search engine / memory skill
- `raw/arxiv-org-html-2603-03329v1.md` — AutoHarness academic paper
- `raw/hwchase17-2042978500567609738.md` — LangGraph v0.2
- `raw/Hxlfed14-2022984467380682856.md` — Context engineering across companies
- `raw/trq212-2033949937936085378.md` — Skill design lessons
- `raw/YukerX-2038959908968919297.md` — Chinese-language Claude Code analysis
