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
tags: [harness, claude-code, codex]  # optional
---
```

## Commands

- `npm run ingest` — Fetch URLs into raw/
- `npm run compile` — Build wiki from raw sources
- `npm run query` — Query the knowledge base

## Source Types

The ingest script handles:
- **x.com URLs** → ScrapeCreators API (tweets) + xAI Grok fallback (X Articles, threshold: <100 chars or URL-only)
- **github.com URLs** → Fetches README.md automatically
- **Other web URLs** → Fetches HTML, strips tags, saves as article/paper. Fallback chain for blocked sites:
  1. Direct HTTPS fetch (works for most sites)
  2. `agent-browser --headed` (solves Cloudflare challenges via real browser)
  3. xAI Grok fallback (uses x_search to find/reproduce content)

## Key Files

- `raw_source_list.txt` — Master list of source URLs
- `ingest_manifest.json` — Tracks fetch status per URL
- `log.md` — Append-only operation log
- `.env` — API keys (SCRAPECREATORS_API_KEY, XAI_API_KEY)

## Topic Coverage

- Agent harness design patterns (long-running agents, auto mode)
- Claude Code architecture and configuration
- OpenAI Codex harness engineering
- Cursor, Windsurf, and other AI coding agents
- Agent memory systems and benchmarking
- CLAUDE.md / rules files / system prompts
