---
title: "feat: Align KB skills with Karpathy second-brain principles"
type: feat
status: completed
date: 2026-04-13
---

# Align KB Skills with Karpathy Second-Brain Principles

## Overview

Gap analysis against the Karpathy "How to build a second brain" guide revealed 8 high/medium-severity gaps in our reusable `/kb-*` skills. The three highest-impact gaps all relate to the same core principle: **cross-pollination of knowledge across wiki pages** (a single source should ripple across 10-15 existing wiki pages, with backlinks). The remaining gaps cover semantic linting, interactive discussion during compile, wiki review status tracking, an explore workflow, and git initialization.

## Problem Statement

Our current KB skills create wiki articles in isolation. When a new source about "Claude Code architecture" is compiled, it creates or updates one article — but doesn't touch "tool design patterns", "practical best practices", or "agent memory" even though the source contains relevant information for all of them. The Karpathy guide explicitly says "A single source should touch 10-15 wiki pages." This cross-pollination is what makes the wiki compound over time rather than just grow linearly.

## Proposed Solution

Eight changes across 5 skill files, 1 script file, and 1 project file. Changes are grouped into 3 phases by dependency order.

---

## Phase 1: Core Cross-Pollination (Skills changes)

### 1.1 Upgrade `/kb-compile` — Add fan-out and backlinks step

**File:** `~/.claude/skills/kb-compile/SKILL.md`

**Current behavior:** Step 4 says "Write/Update Wiki Articles" — creates new articles or updates existing ones that directly match the topic. No instruction to scan OTHER articles.

**New behavior:** After Step 4, add **Step 4b: Cross-Pollinate**:

```
## Step 4b: Cross-Pollinate Across Wiki

For EACH new or updated wiki article from Step 4:

1. Read wiki/_index.md to get the full article list
2. For each OTHER existing wiki article, assess:
   - Does the new source contain information relevant to this article?
   - Would adding a paragraph, updating a section, or adding a citation improve it?
3. If yes: READ the existing article, add the relevant information with a source citation
4. Add/update the ## Related section with bidirectional links:
   - In the NEW article: add link to the existing article
   - In the EXISTING article: add link back to the new article
5. Target: each new source should touch 5-15 wiki pages total (not just 1-2)

**Backlink format in ## Related sections:**
- [Article Title](article-filename.md) — one-line description of the relationship

**Do NOT create new articles in this step.** Only update existing ones.
```

### 1.2 Upgrade `/kb-compile` — Add interactive discussion step (supervised mode)

**File:** `~/.claude/skills/kb-compile/SKILL.md`

**Current behavior:** Reads sources and writes articles without pausing.

**New behavior:** Add between Step 2 and Step 3:

```
## Step 2b: Discuss Key Takeaways (Supervised Mode)

If compiling 5 or fewer sources (supervised mode):
1. For each source, present a brief summary of key takeaways (3-5 bullet points)
2. Ask the user: "Anything to emphasize, correct, or skip?"
3. Incorporate feedback before writing wiki articles

If compiling more than 5 sources (batch mode):
- Skip this step and proceed directly to Step 3
- Print: "Batch mode: processing N sources. Run with fewer sources for supervised compilation."
```

### 1.3 Upgrade `/kb-compile` — Add wiki frontmatter fields

**File:** `~/.claude/skills/kb-compile/SKILL.md`

**Current behavior:** Wiki frontmatter has `title`, `type`, `tags`, `sources`, `last_compiled`.

**New behavior:** Update the Step 4 template to add:

```yaml
---
title: "Article Title"
type: wiki
tags: [tag1, tag2]
sources:
  - raw/file1.md
  - raw/file2.md
source_count: 2
status: draft        # draft | reviewed | needs_update
last_compiled: YYYY-MM-DD
---
```

- `source_count`: integer count of raw sources that informed this page
- `status`: review lifecycle — new articles start as `draft`, user can mark `reviewed`, lint can flag `needs_update`

---

## Phase 2: Semantic Lint (Script + Skill changes)

### 2.1 Upgrade `compile.js health` — Add semantic checks

**File:** `scripts/compile.js` → `cmdHealth()` function

**Current checks:** broken internal links, missing source refs, missing `## Sources` section.

**New checks to add:**

```javascript
// 1. Orphan pages — no inbound links from any other wiki page
// For each wiki file, check if ANY other wiki file links to it
// If not → "🟡 Orphan page: {file} — no other articles link to it"

// 2. Missing cross-references — articles that share tags but don't link to each other
// For each pair of articles sharing a tag, check if they link to each other
// If not → "🔵 Missing cross-reference: {file1} and {file2} share tag '{tag}' but don't link"

// 3. Concepts mentioned but never explained
// Scan all wiki files for [[concept]] wikilinks or bold terms
// Check if a wiki page exists for that concept
// If not → "🔵 Concept without page: '{concept}' mentioned in {file}"

// 4. Claims without source attribution
// Scan for paragraphs that make factual claims but don't contain [Source:] or (raw/) citations
// If found → "🟡 Unsourced claim in {file}: '{first 60 chars of paragraph}'"

// 5. Stale articles — last_compiled older than 30 days and status != reviewed
// If found → "🟡 Stale article: {file} — last compiled {date}"
```

**Severity levels:**
- 🔴 Error: broken links, missing sources (existing checks)
- 🟡 Warning: orphan pages, unsourced claims, stale articles
- 🔵 Info: missing cross-references, concepts without pages

### 2.2 Create `/kb-lint` skill

**File:** `~/.claude/skills/kb-lint/SKILL.md` (NEW)

```markdown
---
name: kb-lint
description: Run full health check on the knowledge base wiki. Use when asked to
  "lint wiki", "health check", "kb lint", or "check wiki quality".
---

# Lint Knowledge Base

## Step 1: Run Structural Checks

```bash
node scripts/compile.js health
```

## Step 2: Run Semantic Checks (LLM-powered)

Read wiki/_index.md, then for each wiki article:

1. **Contradiction scan**: Read each article. Flag claims that contradict
   claims in other articles you've read.
   Format: > 🔴 CONTRADICTION: {article1} claims X. {article2} claims Y.

2. **Staleness scan**: For articles with status: needs_update or
   last_compiled older than 30 days, check if newer raw sources exist
   that should update them.

3. **Gap analysis**: Based on all wiki content, identify 3-5 topics that
   are mentioned frequently but have no dedicated wiki page.

## Step 3: Generate Report

Write report to `wiki/lint-report-YYYY-MM-DD.md` with:
- 🔴 Errors (must fix)
- 🟡 Warnings (should fix)
- 🔵 Info (nice to fix)
- Top 3 suggested articles to fill knowledge gaps

## Step 4: Summary

Print: "Lint complete. N errors, M warnings, K info items.
Report saved to wiki/lint-report-YYYY-MM-DD.md"
```

---

## Phase 3: Explore + Init improvements

### 3.1 Create `/kb-explore` skill

**File:** `~/.claude/skills/kb-explore/SKILL.md` (NEW)

```markdown
---
name: kb-explore
description: Discover unexplored connections in the knowledge base. Use when asked
  to "explore KB", "find connections", "kb explore", or "what's interesting".
---

# Explore Knowledge Base

## Step 1: Load Context

Read wiki/_index.md, then read ALL wiki articles to build a mental model.

## Step 2: Find Connections

Identify the 5 most interesting unexplored connections between existing
topics. For each:

1. Name the two topics/concepts being connected
2. Explain what insight the connection might reveal
3. Suggest what source (URL, paper, or research) would help confirm it
4. Rate confidence: high/medium/low

## Step 3: Offer Actions

For each connection, ask: "Want me to:
a) Create a new wiki page exploring this connection
b) Add a note to the relevant existing articles
c) Skip"

If the user selects (a) or (b), write the content with
`origin: generated` in frontmatter and update the index.

Save the full exploration to outputs/explore-YYYY-MM-DD.md
```

### 3.2 Add `git init` to `/kb-init`

**File:** `~/.claude/skills/kb-init/SKILL.md`

**Add after Step 8 (Create empty files), before Step 9 (Print success):**

```
8b. **Initialize git repository:**
    ```bash
    git init
    git add -A
    git commit -m "Initialize knowledge base for [topic]"
    ```
    This enables version control from day one — full history, branching,
    and the ability to undo any AI-generated changes.
```

### 3.3 Update CLAUDE.md template in `/kb-init`

**File:** `~/.claude/skills/kb-init/SKILL.md` → Step 7 (Create CLAUDE.md)

Add these sections to the CLAUDE.md template that kb-init generates:

```markdown
## Wiki Conventions
- Every wiki file starts with YAML frontmatter (see Frontmatter Schema)
- After frontmatter, a one-paragraph summary
- Every factual claim cites its source: [Source: filename.md]
- When new info contradicts existing content, flag explicitly:
  > CONTRADICTION: [old claim] vs [new claim] from [source]
- Use ## Related section for bidirectional links between wiki pages

## Focus Areas
[List 3-5 topics this knowledge base covers]
```

---

## Acceptance Criteria

### Phase 1: Cross-Pollination
- [ ] `/kb-compile` has Step 4b that updates 5-15 existing wiki pages per new source
- [ ] `/kb-compile` adds bidirectional links in `## Related` sections
- [ ] `/kb-compile` has Step 2b for supervised discussion of key takeaways
- [ ] Wiki frontmatter includes `source_count` and `status` fields

### Phase 2: Semantic Lint
- [ ] `compile.js health` detects orphan pages, missing cross-references, stale articles
- [ ] `/kb-lint` skill exists and runs both structural + LLM-powered semantic checks
- [ ] Lint report uses 🔴🟡🔵 severity levels
- [ ] Lint suggests 3 articles to fill knowledge gaps

### Phase 3: Explore + Init
- [ ] `/kb-explore` skill exists and finds 5 unexplored connections
- [ ] `/kb-init` runs `git init` and makes initial commit
- [ ] CLAUDE.md template includes Wiki Conventions and Focus Areas sections

## Implementation Notes

- All skill changes are at `~/.claude/skills/kb-*/SKILL.md` — these are **shared across all KBs**, not project-specific
- `compile.js` changes are **per-project** in `scripts/compile.js` — but should be copied to the trading KB afterward
- New skills (`kb-lint`, `kb-explore`) need new directories under `~/.claude/skills/`
- The cross-pollination step (Phase 1) is the highest-impact change — it's what makes the wiki compound

## Files to Change

| File | Action | Phase |
|------|--------|-------|
| `~/.claude/skills/kb-compile/SKILL.md` | Edit (add Steps 2b, 4b, update frontmatter template) | 1 |
| `scripts/compile.js` | Edit (add semantic health checks) | 2 |
| `~/.claude/skills/kb-lint/SKILL.md` | Create | 2 |
| `~/.claude/skills/kb-explore/SKILL.md` | Create | 3 |
| `~/.claude/skills/kb-init/SKILL.md` | Edit (add git init, update CLAUDE.md template) | 3 |
