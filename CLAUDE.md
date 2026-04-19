# Agent Harness Engineering Knowledge Base

Karpathy-style LLM knowledge base focused on how AI coding agents (Claude Code, Codex, Cursor, Windsurf, etc.) are built, configured, and optimized.

## Structure

```
raw/              # Fetched source documents (markdown with frontmatter)
raw/images/       # Downloaded media (tweet images, diagrams)
wiki/             # Compiled wiki articles (synthesized from raw sources)
outputs/          # Generated reports, analyses
scripts/          # ingest.js, compile.js, query.js
```

## Frontmatter Schema

Every file in `raw/` and `wiki/` must have YAML frontmatter:

```yaml
---
title: "Human-readable title"
source: https://original-url
author: Author Name       # optional
date: 2025-01-15          # publication date
fetched: 2025-01-15       # when we fetched it
type: tweet|article|paper|github|wiki
status: raw|compiled|reviewed
source_method: scrapecreators|x_api_v2_article_plain_text|xai_grok_fallback|agent-browser|grok_fallback  # optional, which fetcher produced the file
tags: [harness, claude-code, codex]  # optional
---
```

## Commands

- `npm run ingest` — Fetch URLs into raw/
- `npm run compile` — Build wiki from raw sources
- `npm run query` — Query the knowledge base

## Source Types

The ingest script handles:
- **x.com URLs (regular tweets)** → ScrapeCreators API returns tweet text, media, and metadata. If text is URL-only or <100 chars, treats as X Article and falls through the article chain below.
- **x.com URLs (X Articles)** → Authoritative chain, tried in order:
  1. **X API v2** `GET /2/tweets/{id}` with `tweet.fields=article,note_tweet` + `expansions=author_id` → reads `article.plain_text` (authoritative, no hallucination). Recorded as `source_method: x_api_v2_article_plain_text`.
  2. **xAI Grok** with `x_search` tool → last resort only; can hallucinate on recent articles not in training data or x_search index. Recorded as `source_method: xai_grok_fallback`.

  The ScrapeCreators-outright-failure path (e.g. HTTP 402 quota) also falls through the same X API → Grok chain.
- **github.com URLs** → Fetches README.md automatically.
- **Other web URLs** → Fetches HTML, strips tags, saves as article/paper. Fallback chain for blocked sites:
  1. Direct HTTPS fetch (works for most sites).
  2. `agent-browser --headed` (solves Cloudflare challenges via real browser).
  3. xAI Grok fallback (uses x_search to find/reproduce content).

### Why X API v2 before Grok

Grok's `x_search` tool reliably finds older/popular X Articles but can hallucinate a completely unrelated tweet by the same author for recent posts (see 2026-04-17 log entry — `hwchase17/2042978500567609738` "Your harness, your memory" returned an unrelated LangGraph v0.2 post on first fetch). X API v2's `article.plain_text` field is the authoritative source and should always be tried first when credentials are available.

### Known operational state (2026-04-17)

- ScrapeCreators API is currently returning HTTP 402 (out of credits). Top up at scrapecreators.com or lean on the X API v2 path (covers articles; regular tweets still need ScrapeCreators for full media/metadata extraction).

## Key Files

- `raw_source_list.txt` — Master list of source URLs
- `ingest_manifest.json` — Tracks fetch status per URL; articles record `source_method` so fetcher provenance is auditable
- `log.md` — Append-only operation log
- `.env` — Local API keys: `SCRAPECREATORS_API_KEY`, `XAI_API_KEY`
- X API credentials (`X_BEARER_TOKEN`, `X_CONSUMER_KEY`, `X_CONSUMER_SECRET`, `X_ACCESS_TOKEN`, `X_ACCESS_TOKEN_SECRET`) can be provided via:
  - The standard `.env` file above, OR
  - A file pointed at by the `KB_ENV_FILE` environment variable (only `X_*` vars are loaded from there; other secrets are ignored). Example: `KB_ENV_FILE=$HOME/projects/knowledge_base/.env.txt npm run ingest`.

## Topic Coverage

- Agent harness design patterns (long-running agents, auto mode)
- Claude Code architecture and configuration
- OpenAI Codex harness engineering
- Cursor, Windsurf, and other AI coding agents
- Agent memory systems and benchmarking
- CLAUDE.md / rules files / system prompts
