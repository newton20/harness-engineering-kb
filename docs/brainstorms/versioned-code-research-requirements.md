# Versioned Code Research Files + Additive Compile Merge

**Date:** 2026-04-15
**Status:** Ready for planning
**Scope:** Standard — touches SKILL.md, compile.js, and /kb-compile skill

## Problem

When `/kb-code-research` is re-run on a repo (e.g., with tuned prompts or a different research goal), it overwrites the previous `raw/code-research-{repo}.md` file. This destroys findings from the earlier run that the re-run didn't cover.

**Measured impact from the last session:**
- OpenClaw re-run: lost 203 lines, gained only 136. Dreaming system details, streaming JSON repair patterns, and cooperative abort findings were permanently deleted.
- Claude Code re-run: lost 107 lines, gained 190. Net gain, but first-run findings about the main loop, 5 memory systems, and 50+ tool catalog were replaced with compaction/permission focus.
- The overwritten content is only recoverable via `git diff` — not discoverable by the compile pipeline.

## Solution

### 1. Timestamped versioned files in raw/

Each research run produces a **new file** with a date suffix:

```
raw/code-research-claude-code-2026-04-14.md   (run 1: architecture)
raw/code-research-claude-code-2026-04-15.md   (run 2: compaction + permissions)
raw/code-research-claude-code-2026-04-20.md   (run 3: future run)
```

**Naming convention:** `code-research-{REPO_NAME}-{YYYY-MM-DD}.md`

If two runs happen on the same day, append a counter: `-2026-04-15-2.md`

Each file is self-contained with full frontmatter including `research_goal`, `date`, `dimensions_analyzed`, and a new `run_number` field.

### 2. Additive compile merge strategy

When `/kb-compile` processes code research files, it:

1. **Groups by repo:** Discovers all `code-research-{REPO_NAME}-*.md` files for a given repo using glob.
2. **Reads all versions:** Reads every version chronologically (oldest first).
3. **Merges findings additively:**
   - A finding present in run 1 but absent in run 2 is **kept** in the wiki. Absence means "not re-investigated," not "wrong."
   - A finding present in run 2 but not run 1 is **added** to the wiki.
   - A finding present in both is **updated** to the latest version's wording (it may have more detail or better evidence).
4. **Handles contradictions explicitly:**
   - If run 2 explicitly contradicts run 1 (e.g., "Dimension 1 previously reported X, but code analysis shows Y"), the wiki uses run 2's corrected version with a note: "Earlier research reported X; re-analysis found Y."
   - The original finding is NOT silently deleted — the correction is documented.
5. **Preserves all evidence paths:** The union of all evidence indexes across runs. An evidence path from any run stays in the wiki.
6. **Novelty table is cumulative:** NOVEL/VARIANT/KNOWN assessments from all runs are unioned.

### 3. Changes required

#### SKILL.md changes
- **Step 0d:** Add `RUN_DATE` variable (YYYY-MM-DD)
- **Step 1b:** Change from "will overwrite" to "previous runs found: {list}. This run will create a new versioned file."
- **Step 6:** Write to `raw/code-research-{REPO_NAME}-{RUN_DATE}.md` instead of `raw/code-research-{REPO_NAME}.md`
- **Step 6 outputs/ copy:** Also write to `outputs/code-research-{REPO_NAME}-{RUN_DATE}.md` (versioned identically to raw/). Outputs/ is archival; it mirrors raw/ naming.
- **Step 6 frontmatter:** Add `run_number: N` field (auto-incremented based on existing files for this repo)
- **Step 8 (learnings):** Reference the specific versioned file, not the unversioned name

#### compile.js changes
- **`delta` command:** Update the uncompiled-file scanner to handle versioned filenames. A file is uncompiled if `status: raw` in its frontmatter, regardless of whether other versions of the same repo are already compiled.
- **New helper:** `groupCodeResearchByRepo()` — groups `code-research-*` files by REPO_NAME, returns `Map<string, VersionedFile[]>` sorted by date. Parse using regex: `/^code-research-(.+)-(\d{4}-\d{2}-\d{2})(-\d+)?\.md$/` where group 1 is REPO_NAME (may contain hyphens), group 2 is date, optional group 3 is same-day counter.

#### /kb-compile skill changes
- **Step 2 (Read Sources):** When reading code-research files, detect multi-version repos. Read all versions and present a merged view to the compile step.
- **Step 2b (Context window management):** If a repo has 5+ versions, read full content for the latest 2 versions and only the Executive Summary + Key Patterns sections from older versions. This prevents context overflow (~10-18K chars per full report).
- **Step 4 (Write Wiki):** Use the additive merge strategy described above. Cite the specific versioned file as the source (e.g., `[Source: raw/code-research-claude-code-2026-04-15.md]`).
- **Step 5 (Update Status):** Mark each versioned file individually as compiled. Each version's `compiled_to` list reflects the articles it contributed to at the time of its compilation — not retroactively updated when later versions also contribute to the same articles.

### 4. Migration

For existing files without date suffixes:
- Rename `raw/code-research-{repo}.md` to `raw/code-research-{repo}-{date-from-frontmatter}.md`
- Run once manually or add a migration step to compile.js

For the 3 files that were overwritten in the last session:
- Recover the original versions from git history (`git show c8d951f:raw/code-research-claude-code.md`)
- Save them as `raw/code-research-{repo}-2026-04-14.md` (the original run date)
- The current files become `raw/code-research-{repo}-2026-04-15.md`

## Non-goals

- **Version diffing UI** — We don't need a UI to compare runs. Git diff serves this purpose.
- **Automatic deduplication of findings** — The compile step doesn't need to deduplicate identical findings across runs. The wiki author (the LLM in /kb-compile) handles this naturally.
- **Run linking** — We don't need formal "this run supersedes that run" metadata. Chronological order is sufficient.

## Success criteria

1. Re-running `/kb-code-research` on a repo never destroys previous findings
2. `node scripts/compile.js delta` correctly identifies uncompiled versioned files
3. `/kb-compile` produces wiki articles that contain the union of findings from all runs
4. Existing unversioned files are migrated without data loss
5. The 3 overwritten files from the last session are recovered from git history
