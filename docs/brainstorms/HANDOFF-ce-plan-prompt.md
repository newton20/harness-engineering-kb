# Handoff Prompt — Copy Everything Below This Line

---

/compound-engineering:ce-plan

## Context

I have a completed requirements document for building a Claude Code skill called `/kb-code-research` — a reusable tool that deploys parallel specialist agents to deeply analyze open-source code repos, extracting agent harness patterns and reusable components.

The full requirements doc is at:
`docs/brainstorms/code-repo-deep-research-requirements.md`

Read that document fully before planning.

## What Was Decided in Brainstorming

**Approach:** Layered Research Protocol (Approach C) — two-pass system:
- **Pass 1 (Triage):** Quick 5-min scorecard per repo — size, relevance, maintenance activity. Low threshold (score < 3 skips) to avoid filtering out important findings.
- **Pass 2 (Deep Dive):** 3-4 parallel specialist agents per repo, 2-wave parallelism (2 agents per wave to stay within Claude Code limits), orchestrator handles synthesis and KB integration directly (no nested subagents for those).

**v1 Core Dimensions (4):** Architecture & Loop Design, Memory & State Management, Tool & Action Space Design, Multi-Agent Coordination.

**v2 Deferred:** Safety & Error Recovery, Self-Improvement & Eval Loops, Reusable Components Extraction, Domain Relevance Mapping (pluggable via `--domain` flag).

**Key review findings already incorporated:**
1. Token budget: ~900K-1M per repo. Sonnet for dimension agents, Opus for synthesis only. Estimated $8-20/repo.
2. Parallelism: 2 agents per wave (not 8 simultaneous). Total 2 waves for 4 dimensions.
3. Self-improvement deferred to v2 — v1 uses "learnings accumulation" (append to outputs/research-learnings.md, human reviews every 3-5 repos).
4. KB integration: orchestrator writes raw/ files directly (bypasses ingest.js), calls `node scripts/compile.js` via Bash. No nested agent for KB integration.
5. Evidence validation: file-existence checks on all cited paths before synthesis.
6. Dimension 8 (Trading Relevance) deferred to v2 as pluggable `--domain` parameter.
7. Triage gate prevents wasting full research budget on irrelevant/dead repos.

**Research queue (7 repos):**
- P0: karpathy/autoresearch, NousResearch/hermes-agent, openclaw/openclaw, local Claude Code source (C:\Users\dunliu\Downloads\Claude Code\src), langchain-ai/deepagents
- P1: anomalyco/opencode, 666ghj/MiroFish

**Ultimate goal:** This skill feeds findings into a harness engineering KB (C:\Users\dunliu\projects\knowledge_base\agents\harness_engineering) which informs the design of an auto-agentic Polymarket trading system harness (design doc at C:\Users\dunliu\projects\knowledge_base\agents\trading\Auto-Agentic Polymarket Trading System.md).

## What I Need From the Plan

1. The skill file structure and where it lives (project-level, not ~/.claude/)
2. Detailed implementation steps for each component: orchestrator, triage, 4 dimension agent prompts, synthesis, KB integration, learnings accumulation
3. Phased delivery: what ships first, how to test incrementally
4. The actual dimension agent prompts (or at least their structure and key instructions)
5. How the output report is structured (frontmatter, sections, evidence format)
6. Error handling and edge cases (tiny repos, huge repos, auth failures, dimension N/A)

The existing KB has 13 wiki articles and 36 raw sources about agent harness engineering patterns. The KB infrastructure uses node scripts (ingest.js, compile.js) and /kb-* skills for ingestion and compilation.
