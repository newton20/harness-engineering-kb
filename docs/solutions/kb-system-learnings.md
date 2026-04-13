---
title: "KB System Learnings: Building a Karpathy-Style LLM Knowledge Base"
date: 2026-04-13
type: solution
---

# KB System Learnings: Building a Karpathy-Style LLM Knowledge Base

Lessons learned from building and operating a Karpathy-style second-brain knowledge base for agent harness engineering. Each section documents a problem encountered, the root cause, and the solution applied.

---

## 1. Ingestion: X Article Detection

### Problem

ScrapeCreators API returns 23-character `t.co` shortened links for X Articles (long-form posts), not the actual article content. The initial content-length threshold of 10 characters treated these as valid fetches, so X Articles were silently stored as URL-only stubs.

### Root Cause

X Articles are not inline tweets. When ScrapeCreators processes an X Article URL, it returns only the `t.co` redirect link because the article body lives behind a separate rendering path that the tweet-scraping API does not traverse.

### Solution

1. **Raised the minimum content threshold from 10 to 100 characters.** Any ScrapeCreators response shorter than 100 chars is now treated as a failed fetch requiring fallback.
2. **Added URL-only regex detection.** If the returned body matches a pattern of bare URLs with no surrounding prose, it is flagged as insufficient content regardless of length.
3. **Grok fallback for X Articles.** Detected X Articles are re-ingested via the xAI Grok API (`x_search` tool), which can read and reproduce the full article text.

### Impact

Of the 28 initial sources, 17 were X posts. On the first ingest pass, all X Articles came back as stubs. The second pass (with Grok fallback) successfully fetched all 17 with full content.

---

## 2. Ingestion: Cloudflare Bypass and the 3-Tier Fallback Chain

### Problem

Direct HTTPS fetch returns HTTP 403 for sites protected by Cloudflare JS challenges, including openai.com. The two OpenAI harness engineering articles failed on the initial ingest.

### Root Cause

Cloudflare's bot protection requires executing JavaScript challenges in a real browser environment. A plain `https.get` request from Node.js has no JS engine and fails the challenge immediately.

### Solution

A 3-tier fallback chain in `ingest.js`:

| Tier | Method | When It Works |
|------|--------|---------------|
| 1 | Direct HTTPS fetch | Most sites (Anthropic, arxiv, Letta, Simon Willison) |
| 2 | `agent-browser --headed` | Cloudflare-protected sites (runs a real Chromium instance) |
| 3 | xAI Grok API | Last resort when even headed browser fails |

**Critical detail: headless mode does NOT solve Cloudflare challenges.** Only `--headed` mode works because Cloudflare detects headless browser signatures. The OpenAI articles were ultimately ingested via Grok fallback (tier 3) on the first attempt, then later re-ingested via `agent-browser --headed` (tier 2) to get the actual page content rather than Grok's reproduction.

### Impact

All 28 sources were successfully ingested across two passes. The fallback chain handles the three major categories of web content: open sites, Cloudflare-protected sites, and walled-garden content (X Articles).

---

## 3. Compilation: Cross-Pollination Is the Number One Karpathy Gap

### Problem

Without explicit instruction, LLMs create wiki articles in isolation. When compiling a source about "Claude Code architecture," the model creates or updates one article but does not touch "tool design patterns," "practical best practices," or "agent memory" -- even when the source contains relevant information for all of them.

### Root Cause

The default LLM behavior is to map one source to one output. The Karpathy second-brain guide explicitly states that a single source should touch 10-15 wiki pages. This fan-out is what makes a wiki compound over time rather than grow linearly.

### Evidence

- **Initial compile (no cross-pollination):** 28 sources produced 9 articles with 5 orphan pages (no inbound links from other articles).
- **After adding Step 4b (cross-pollination) to kb-compile:** A single new source (Simon Willison's "Designing Agentic Loops") touched 4 existing articles (`practical-best-practices`, `tool-design-patterns`, `auto-mode-and-safety`, `what-is-harness-engineering`).
- **Full recompile with cross-pollination:** 28 sources into 9 articles, 0 orphan pages, bidirectional backlinks throughout.

### Solution

Added Step 4b ("Cross-Pollinate Across Wiki") to the `/kb-compile` skill:

1. After writing/updating the primary article, read the full wiki index.
2. For each other existing article, assess whether the new source contains relevant information.
3. If yes, read the existing article and add the relevant information with source citations.
4. Add bidirectional links in `## Related` sections.
5. Target: each new source should touch 5-15 wiki pages total.

The instruction "Do NOT create new articles in this step -- only update existing ones" prevents scope explosion.

---

## 4. Compilation: YAML Tag Parser Only Handled Inline Arrays

### Problem

All wiki article tags displayed as "uncategorized" in the index, even though every article had tags in its frontmatter.

### Root Cause

The `compile.js` tag parser only handled inline YAML arrays (`tags: [tag1, tag2]`). It did not handle the YAML block list format:

```yaml
tags:
  - tag1
  - tag2
```

When wiki articles were generated with block-style tags (the default YAML serialization for longer lists), the parser returned an empty array and the index builder fell back to "uncategorized."

### Solution

Updated the tag parser to handle both formats:
- Inline: `tags: [tag1, tag2]` -- parsed via regex extracting comma-separated values inside brackets.
- Block list: `tags:\n  - tag1\n  - tag2` -- parsed by scanning subsequent lines matching the `  - value` pattern.

### Lesson

When parsing YAML frontmatter with regex instead of a full YAML parser, always account for both inline and block representations of the same data type. Better yet, use a proper YAML parser (`js-yaml`) for frontmatter extraction.

---

## 5. Performance: Grok API Latency in Batch Operations

### Problem

The Grok fallback pass for 17 X Articles took approximately 15 minutes total. Some individual articles took 60+ seconds.

### Observed Timings

| Source | Approximate Time |
|--------|-----------------|
| Typical X Article | 20-40 seconds |
| `systematicls-2028814227004395561` | 60+ seconds |
| `Hxlfed14-2028116431876116660` | 60+ seconds |
| Full batch (17 articles) | ~15 minutes |

### Root Cause

Each Grok API call uses the `x_search` tool to find and reproduce the article content. This involves a search query, retrieval of the X post, and generation of the full article text. The API processes these sequentially and there is significant variance in response time depending on article length and search complexity.

### Recommendations

1. **Plan for 1 minute per article when budgeting batch Grok operations.** The average is 30-40 seconds but outliers reach 60+.
2. **Run Grok fallback as a separate pass** (already implemented). The first pass handles fast sources (direct HTTPS, ScrapeCreators), and only the failures go to Grok.
3. **Log timestamps per article** (already implemented in `log.md`) so slow sources can be identified.
4. **Consider parallelism with care.** The xAI API may have rate limits; parallel requests could trigger throttling. Sequential processing is safer for reliability.

---

## 6. Compilation: Supervised vs. Batch Mode Matters

### Problem

The Karpathy guide recommends discussing takeaways with the user for the first sources added to a knowledge base. Batch-processing 28 sources on the first compile misses the opportunity to establish patterns, correct misinterpretations, and guide the wiki's editorial voice.

### The Karpathy Principle

For the first ~10 sources, supervised compilation (reading together, discussing key takeaways, deciding what matters) builds a shared mental model between the human and the system. After patterns are established, batch compilation is fine.

### Solution

Added Step 2b ("Discuss Key Takeaways") to `/kb-compile` with a threshold:

- **5 or fewer sources (supervised mode):** For each source, present 3-5 key takeaways and ask "Anything to emphasize, correct, or skip?" Incorporate feedback before writing articles.
- **More than 5 sources (batch mode):** Skip discussion and proceed directly. Print: "Batch mode: processing N sources. Run with fewer sources for supervised compilation."

### Evidence

The supervised compile of Simon Willison's "Designing Agentic Loops" article (1 source) resulted in a discussion of key takeaways before writing. The resulting updates to 4 existing articles were more precisely targeted than the batch-compiled articles from the initial 28-source run.

---

## 7. Linting: Unreferenced Sources Reveal Coverage Gaps

### Problem

After the initial compile, it was unclear whether all raw sources had been incorporated into the wiki. Manual checking of 28 sources against 9 articles is error-prone.

### Solution

The `/kb-lint` skill (and `compile.js health` command) now checks for unreferenced raw sources -- files in `raw/` that are not cited by any wiki article.

### Findings from First Lint (2026-04-13)

7 of 28 raw sources were never consumed by any wiki article:

| Unreferenced Source | Topic |
|---------------------|-------|
| `akshay_pachaar-2033167408463069526.md` | Sub-agents vs agent teams |
| `ArtemXTech-2028330693659332615.md` | Local search engine / memory skill |
| `arxiv-org-html-2603-03329v1.md` | AutoHarness academic paper |
| `hwchase17-2042978500567609738.md` | LangGraph v0.2 |
| `Hxlfed14-2022984467380682856.md` | Context engineering across companies |
| `trq212-2033949937936085378.md` | Skill design lessons |
| `YukerX-2038959908968919297.md` | Chinese-language Claude Code analysis |

These unreferenced sources directly informed the lint report's "Knowledge Gaps" section, suggesting 5 new articles: Agent Skills, Multi-Agent Coordination, Benchmarking and Evaluation, Prompt Caching and Token Economics, and Context Engineering. This demonstrates the lint-then-compile feedback loop: lint identifies gaps, compile fills them.

### Additional Lint Findings

- 4 articles had low citation density (paragraphs without inline source citations).
- 3 unsourced specific claims were flagged, including a quantitative claim ("95% vs 70%") that appeared invented for illustration rather than drawn from a source.
- Tool count discrepancy in `claude-code-architecture.md` (references to 12, 18, 20, and 45+ tools in different sections).
- All 9 articles remained in `status: draft` -- expected for a first compile but tracked for future review cycles.

---

## Summary: Operational Patterns

### What Worked Well

1. **Append-only log (`log.md`)** -- every ingest and compile operation is timestamped, making it easy to reconstruct what happened and when.
2. **Manifest-based idempotency (`ingest_manifest.json`)** -- re-running ingest skips already-fetched URLs, enabling safe re-runs after partial failures.
3. **Fallback chains over single-method ingestion** -- no single fetch method works for all sources. The 3-tier chain handles the real-world diversity of web content.
4. **Lint as a compile feedback loop** -- running lint after compile identifies both quality issues and coverage gaps that inform the next compile cycle.

### What To Do Differently Next Time

1. **Use a proper YAML parser from day one.** Regex-based frontmatter parsing creates an ongoing maintenance burden as edge cases accumulate.
2. **Start with supervised compilation** for the first 5-10 sources before switching to batch mode. The editorial patterns established early propagate through all future compiles.
3. **Budget 1 minute per Grok API call** when planning batch operations involving X Articles.
4. **Set the X Article content threshold aggressively high** (100+ chars). False negatives (re-fetching a valid short tweet) are cheap; false positives (accepting a t.co stub as content) corrupt the knowledge base.
5. **Add cross-pollination instructions from the start.** It is the single highest-impact alignment with the Karpathy guide and transforms the wiki from a collection of isolated articles into a connected knowledge graph.
