---
title: "feat: Versioned code research files with additive compile merge"
type: feat
status: active
date: 2026-04-15
origin: docs/brainstorms/versioned-code-research-requirements.md
---

# feat: Versioned code research files with additive compile merge

## Overview

Change `/kb-code-research` to produce timestamped versioned files instead of overwriting, and update the compile pipeline to merge findings additively across all versions for a given repo. Also recover 3 files overwritten in the last session from git history.

## Problem Frame

When `/kb-code-research` is re-run on a repo with different prompts or research goals, it overwrites `raw/code-research-{repo}.md`. This destroys findings from the earlier run that the re-run didn't cover. Measured loss: OpenClaw lost 203 lines of dreaming/streaming JSON findings; Claude Code lost 107 lines of loop/memory/tool catalog findings. The content is only recoverable via git archaeology, not by the compile pipeline. (see origin: `docs/brainstorms/versioned-code-research-requirements.md`)

## Requirements Trace

- R1. Re-running `/kb-code-research` on a repo never destroys previous findings
- R2. `node scripts/compile.js delta` correctly identifies uncompiled versioned files
- R3. `/kb-compile` produces wiki articles containing the union of findings from all runs
- R4. Existing unversioned files are migrated without data loss
- R5. The 3 overwritten files from the last session are recovered from git history

## Scope Boundaries

- No version diffing UI — git diff serves this purpose
- No automatic deduplication of findings — the LLM in /kb-compile handles this naturally
- No formal "this run supersedes that run" metadata — chronological order is sufficient
- No changes to non-code-research raw files (tweets, articles, papers keep their current naming)

## Context & Research

### Relevant Code and Patterns

- `scripts/compile.js:78-108` — `cmdDelta()` scans all `.md` files in raw/, checks `status` frontmatter field. Currently does no grouping by repo.
- `.claude/skills/kb-code-research/SKILL.md:79-91` — Step 0d sets `REPO_NAME`. Step 1b at line 143-151 checks for existing file and prints "will overwrite."
- `.claude/skills/kb-code-research/SKILL.md:519-626` — Step 6 writes report to both `raw/` and `outputs/`.
- `~/.claude/skills/kb-compile/SKILL.md` — Step 1 calls `compile.js delta`, Step 2 reads uncompiled files, Steps 4-4b write/update wiki articles.
- Existing raw code-research files: `raw/code-research-{repo}.md` × 6 repos (autoresearch, claude-code, openclaw, openhands, opencode, mirofish).

### Institutional Learnings

- REPO_NAME can contain hyphens (e.g., `all-hands-ai-openhands`, `anomalyco-opencode`). The date suffix parser must use the `YYYY-MM-DD` pattern to disambiguate from repo-name hyphens.
- Each research run's frontmatter already contains `date`, `research_goal`, and `dimensions_analyzed` — sufficient to identify and sort versions.

## Key Technical Decisions

- **Timestamped suffix, not goal-slugged:** Dates are deterministic, sortable, and collision-free (with same-day counter). Goal slugs would be freeform and hard to normalize. (see origin)
- **Additive merge, not latest-wins:** Absence of a finding in a re-run means "not re-investigated," not "finding retracted." Only explicit contradictions trigger corrections. (see origin)
- **Regex-based REPO_NAME extraction:** `/^code-research-(.+)-(\d{4}-\d{2}-\d{2})(-\d+)?\.md$/` — group 1 is REPO_NAME, group 2 is date, optional group 3 is same-day counter. This handles hyphens in repo names correctly.
- **No changes to compile.js index/health commands:** Only `delta` needs awareness of versioned files. The index and health commands already work per-wiki-article, not per-raw-file.

## Open Questions

### Resolved During Planning

- **Should outputs/ be versioned too?** Yes — mirror raw/ naming exactly. (see origin)
- **What about `compiled_to` on older versions?** Each version's `compiled_to` reflects articles it contributed to at its compilation time. Not retroactively updated. (see origin)
- **Context window limits with many versions?** For 5+ versions, /kb-compile reads full content for latest 2 and executive summary + key patterns only from older ones. (see origin)

### Deferred to Implementation

- **Exact migration script behavior:** Whether to add a `migrate` subcommand to compile.js or handle as a one-time manual script. The approach is clear; the integration point is flexible.

## Implementation Units

- [ ] **Unit 1: Recover overwritten files from git history**

**Goal:** Restore the 3 original research reports that were overwritten by re-runs, so no data is permanently lost.

**Requirements:** R5

**Dependencies:** None — do this first as a standalone recovery.

**Files:**
- Create: `raw/code-research-claude-code-2026-04-14.md`
- Create: `raw/code-research-karpathy-autoresearch-2026-04-14.md`
- Create: `raw/code-research-openclaw-openclaw-2026-04-14.md`
- Create: `outputs/code-research-claude-code-2026-04-14.md`
- Create: `outputs/code-research-karpathy-autoresearch-2026-04-14.md`
- Create: `outputs/code-research-openclaw-openclaw-2026-04-14.md`

**Approach:**
- Use `git show c8d951f:raw/code-research-claude-code.md` to extract the pre-overwrite versions
- Save to the versioned filename with `2026-04-14` suffix (the original run date from frontmatter)
- Update frontmatter `status: compiled` and add `compiled_date: 2026-04-14` since these were already compiled into wiki articles

**Patterns to follow:**
- Existing frontmatter schema in current raw files

**Test scenarios:**
- Happy path: Each recovered file matches the content from git commit c8d951f
- Happy path: Frontmatter `date` field matches `2026-04-14` in all recovered files
- Edge case: Recovered file does not have a date suffix collision with the current (2026-04-15) files

**Verification:**
- 6 new files exist (3 raw + 3 outputs)
- `git show c8d951f:raw/code-research-claude-code.md` matches `raw/code-research-claude-code-2026-04-14.md` content (excluding updated frontmatter fields)

---

- [ ] **Unit 2: Rename existing unversioned files to versioned format**

**Goal:** Migrate the 6 existing `raw/code-research-{repo}.md` files (including the 3 current re-run files) to the versioned naming convention.

**Requirements:** R1, R4

**Dependencies:** Unit 1 (recovered files must exist before renaming to avoid collisions)

**Files:**
- Rename: `raw/code-research-claude-code.md` → `raw/code-research-claude-code-2026-04-15.md`
- Rename: `raw/code-research-karpathy-autoresearch.md` → `raw/code-research-karpathy-autoresearch-2026-04-15.md`
- Rename: `raw/code-research-openclaw-openclaw.md` → `raw/code-research-openclaw-openclaw-2026-04-15.md`
- Rename: `raw/code-research-all-hands-ai-openhands.md` → `raw/code-research-all-hands-ai-openhands-2026-04-15.md`
- Rename: `raw/code-research-anomalyco-opencode.md` → `raw/code-research-anomalyco-opencode-2026-04-15.md`
- Rename: `raw/code-research-666ghj-mirofish.md` → `raw/code-research-666ghj-mirofish-2026-04-15.md`
- Same renames in `outputs/` directory

**Approach:**
- Use `git mv` for tracked files to preserve history
- Extract the date from each file's frontmatter `date:` field for the suffix
- Update any `compiled_to` paths in other raw files that reference the old filenames (check wiki article source lists too)

**Patterns to follow:**
- The naming convention: `code-research-{REPO_NAME}-{YYYY-MM-DD}.md`

**Test scenarios:**
- Happy path: All 6 raw files renamed, no unversioned `code-research-*.md` files remain in raw/
- Happy path: `git log --follow` works on renamed files
- Edge case: OpenHands and OpenCode files get the correct date from their frontmatter (2026-04-15)

**Verification:**
- `ls raw/code-research-*.md` shows only files with date suffixes
- `node scripts/compile.js delta` still reports 0 uncompiled (all files already compiled)

---

- [ ] **Unit 3: Update compile.js to handle versioned filenames**

**Goal:** Make `cmdDelta()` work correctly with versioned filenames and add a `groupCodeResearchByRepo()` helper for the /kb-compile skill.

**Requirements:** R2

**Dependencies:** Unit 2 (files must be in versioned format)

**Files:**
- Modify: `scripts/compile.js`

**Approach:**
- Add `groupCodeResearchByRepo()` function that:
  - Filters raw files matching the code-research pattern
  - Parses each filename with regex `/^code-research-(.+)-(\d{4}-\d{2}-\d{2})(-\d+)?\.md$/`
  - Groups into `Map<repoName, Array<{file, date, counter}>>` sorted by date ascending
  - Falls back gracefully for any legacy unversioned files (treat as undated)
- Add a `group` subcommand: `node scripts/compile.js group` that prints the grouped view
- No changes needed to `cmdDelta()` itself — it already checks `status` in frontmatter which is sufficient. The grouping is consumed by /kb-compile, not by delta.

**Patterns to follow:**
- Existing `parseFrontmatter()` function at `scripts/compile.js:30-69`
- Existing `cmdDelta()` scan pattern at `scripts/compile.js:78-108`

**Test scenarios:**
- Happy path: `node scripts/compile.js group` correctly groups `claude-code-2026-04-14.md` and `claude-code-2026-04-15.md` under `claude-code`
- Happy path: `all-hands-ai-openhands-2026-04-15.md` parses REPO_NAME as `all-hands-ai-openhands` (not `all-hands-ai-openhands-2026`)
- Edge case: A same-day counter file `claude-code-2026-04-15-2.md` sorts after `claude-code-2026-04-15.md`
- Edge case: A legacy unversioned file (if any remain) is handled gracefully
- Happy path: `node scripts/compile.js delta` still correctly reports uncompiled files

**Verification:**
- `node scripts/compile.js group` produces correct grouping for all 9 research files (6 repos, 3 with 2 versions each)

---

- [ ] **Unit 4: Update SKILL.md for versioned output**

**Goal:** Change `/kb-code-research` to produce timestamped versioned files instead of overwriting.

**Requirements:** R1

**Dependencies:** Unit 3 (compile.js must support the new naming convention)

**Files:**
- Modify: `.claude/skills/kb-code-research/SKILL.md`

**Approach:**
- **Step 0d:** Add `RUN_DATE` variable set to today's date (YYYY-MM-DD)
- **Step 1b:** Change logic from "will overwrite" to: discover existing versions via glob `raw/code-research-{REPO_NAME}-*.md`, list them, print "Previous runs found: {list}. This run will create a new versioned file." Also compute `RUN_NUMBER` (count of existing versions + 1). If same-day collision, append counter.
- **Step 6:** Change write path from `raw/code-research-{REPO_NAME}.md` to `raw/code-research-{REPO_NAME}-{RUN_DATE}.md`. Add `run_number: N` to frontmatter. Same change for outputs/ copy.
- **Step 7c & Step 9:** Update all references to use the versioned filename.
- **Step 8 (learnings):** Reference the versioned file.

**Patterns to follow:**
- Existing SKILL.md variable convention (REPO_NAME, REPO_PATH, etc.)

**Test scenarios:**
- Happy path: First run on a new repo creates `code-research-newrepo-2026-04-15.md`
- Happy path: Second run on same repo creates `code-research-newrepo-2026-04-15-2.md` (same day) or `code-research-newrepo-2026-04-16.md` (different day)
- Happy path: Previous runs listed correctly when starting a new run
- Edge case: `run_number: 1` for first run, `run_number: 2` for second

**Verification:**
- Running `/kb-code-research` on a repo that already has research produces a NEW file, not an overwrite
- The old file remains unchanged

---

- [ ] **Unit 5: Update /kb-compile skill for additive merge**

**Goal:** Add multi-version awareness and additive merge strategy to the compile pipeline.

**Requirements:** R3

**Dependencies:** Unit 3 (compile.js grouping helper), Unit 4 (versioned filenames)

**Files:**
- Modify: `~/.claude/skills/kb-compile/SKILL.md`

**Approach:**
- **Step 2 (Read Sources):** After getting the delta list, check if any uncompiled files are code-research type. If so, use `node scripts/compile.js group` to discover all versions for that repo. Read all versions:
  - Latest 2: full content
  - Older versions (if 5+ total): executive summary + key patterns sections only
- **Step 2b adjustment:** Count unique code-research repos (not individual version files) for the supervised/batch mode threshold.
- **Step 4 (Write Wiki):** Add explicit merge instructions:
  - "When compiling code-research files with multiple versions for the same repo, merge findings ADDITIVELY. A finding present in an earlier version but absent in a later version stays in the wiki — absence means 'not re-investigated.' Only explicit contradictions trigger corrections, documented as: 'Earlier research reported X; re-analysis found Y.' Cite the specific versioned file as the source."
- **Step 5 (Update Status):** Mark each compiled versioned file individually. Each version's `compiled_to` reflects the articles it contributed to at compilation time.

**Patterns to follow:**
- Existing Step 4 write/update pattern in `~/.claude/skills/kb-compile/SKILL.md`

**Test scenarios:**
- Happy path: Compiling 2 versions of claude-code produces wiki articles with findings from BOTH runs
- Happy path: A finding from run 1 (e.g., "5 memory systems") absent from run 2 still appears in wiki
- Happy path: A finding present in both runs uses the latest version's wording
- Integration: `[Source: raw/code-research-claude-code-2026-04-14.md]` citations appear alongside `[Source: raw/code-research-claude-code-2026-04-15.md]` citations in the same wiki article
- Edge case: A repo with only 1 version compiles normally (no merge needed)

**Verification:**
- After running `/kb-compile`, wiki articles for repos with multiple versions contain the union of findings from all runs
- Each compiled version file has `status: compiled` in frontmatter

## System-Wide Impact

- **Interaction graph:** SKILL.md → writes raw/ + outputs/ files → compile.js delta reads raw/ → /kb-compile reads raw/ and writes wiki/. The naming convention change flows through all three.
- **Error propagation:** If the regex in compile.js fails to parse a versioned filename, it falls back to treating it as a regular (ungrouped) file. No data loss, just no multi-version merge.
- **State lifecycle risks:** The migration (Units 1-2) must complete before new runs use the versioned naming. If a new run happens between Unit 1 and Unit 2, it would create an unversioned file alongside versioned ones — harmless but messy.
- **Unchanged invariants:** Non-code-research raw files (tweets, articles, papers) are completely unaffected. The wiki article format is unchanged. The compile.js `index` and `health` commands are unchanged.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Regex fails on unusual REPO_NAME patterns | Test with `all-hands-ai-openhands` (4 hyphens) and `666ghj-mirofish` (numeric prefix) |
| Wiki article sources lists reference old unversioned filenames | Update during migration (Unit 2) |
| Context window pressure with many versions during compile | Cap at full-content for latest 2, summary-only for older (Unit 5) |

## Sources & References

- **Origin document:** [versioned-code-research-requirements.md](../brainstorms/versioned-code-research-requirements.md)
- Related code: `scripts/compile.js`, `.claude/skills/kb-code-research/SKILL.md`, `~/.claude/skills/kb-compile/SKILL.md`
- Git recovery point: commit `c8d951f` (pre-overwrite state of 3 research files)
