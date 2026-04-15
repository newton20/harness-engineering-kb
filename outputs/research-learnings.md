# Research Learnings

Accumulated observations from /kb-code-research runs. Human reviews every 3-5 repos and updates dimension prompts based on findings here.

**Format per entry:**
- Repo name, date, relevance score
- What worked, what didn't, what was missed
- Prompt improvement suggestions

---

### karpathy-autoresearch — 2026-04-14

**Research goal:** extract autoresearch loop patterns for trading strategy self-improvement
**Relevance score:** 8/10
**Dimensions analyzed:** Architecture, Memory, Tools (Dim 4 skipped — single-agent)
**Evidence validation:** 75% verified (6/8 files; 2 unverified are runtime artifacts)

**What worked well:**
- All three dimension agents produced rich, evidence-grounded reports with specific file:line citations
- The Dim 1 agent correctly identified the dual-loop architecture (prose outer loop + Python inner loop)
- The Dim 2 agent found the git-as-experiment-database pattern and the deliberate results.tsv gitignore
- The Dim 3 agent mapped the complete action space and identified the tamper-proof evaluation pattern
- Cross-cutting synthesis revealed 2 flows that no individual agent captured

**What didn't work:**
- Hot path builder found only 1 file (program.md) via standard patterns — too sparse for a tiny repo
- Dimension 4 correctly gated off (7 < 10 signals), saving ~$2-5
- The 2 unverified evidence paths (results.tsv, run.log) are runtime artifacts — expected for a static clone

**Missed findings (discovered during synthesis):**
- The multi-objective optimization (val_bpb + simplicity criterion) was noted by Dim 1 but not cross-referenced with the tool constraints in Dim 3
- The GC management pattern (gc.freeze + gc.disable) in train.py was noted by Dim 1 but could be relevant to Dim 3's tool analysis

**Prompt improvement suggestions:**
- Hot path builder should also glob for *train*, *eval*, *test*, *experiment*, *research* files — these are the actual key files in ML repos
- Dimension 1 should search for "experiment" and "trial" patterns, not just standard loop keywords
- For tiny repos (<5 source files), skip the hot path builder entirely and pass ALL source files to every dimension agent
- The "Surprising Observations" section (renamed from "Novelty Assessment") worked well — agents correctly identified code-level surprises without speculating about cross-repo novelty

---

### openclaw-openclaw — 2026-04-14

**Research goal:** analyze Pi harness, SOUL.md, skill system, clawhub plugin architecture for extensible harness design
**Relevance score:** 9/10
**Dimensions analyzed:** Architecture, Memory, Tools, Multi-Agent (all 4)
**Evidence validation:** 100% verified (50/50 key paths)

**What worked well:**
- All 4 dimension agents produced exceptionally deep, evidence-grounded reports with specific file:line citations
- Dim 1 agent navigated the complex two-level loop and identified all 7 termination conditions
- Dim 2 agent discovered the dreaming system (light/deep/REM) which is genuinely novel — not in any prior research
- Dim 3 agent found the streaming JSON argument repair pattern and per-provider schema normalization
- Dim 4 agent documented the sessions_yield cooperative abort and frozen result capture (frozenResultText)
- Hot path builder produced a rich 40-file index with 7 categories — worked well for this large repo
- 100% evidence verification rate — all 50 cited paths exist
- Cross-cutting synthesis revealed 3 flows spanning all dimensions

**What didn't work:**
- Context pressure: each Sonnet dimension agent used ~120-130K tokens on this large repo (~6-9 min per wave)
- Wave 2 took longer than Wave 1 (9 min vs 6 min) — the multi-agent dimension has deeper file chains to follow
- The agents cited some file paths with full Windows paths instead of repo-relative paths (harmless but inconsistent)

**Missed findings (discovered during synthesis):**
- The skill system as a meta-layer above tools AND memory — no individual agent framed this as a cross-cutting orchestration pattern
- The AGENTS.md progressive disclosure architecture (14 AGENTS.md files scattered across subdirectories) — Dim 1 noted it but didn't map the full hierarchy
- The ACP (Agent Communication Protocol) as a bridge to external coding agents was found by Dim 4 but its deeper implications for harness extensibility weren't connected to the plugin architecture from Dim 3

**Prompt improvement suggestions:**
- For large repos with subagent/multi-agent systems, add budget awareness: "Do not read test files unless the source file is unclear"
- The cross-cutting question "What patterns in your dimension also appear in other dimensions?" worked well — agents noted overlaps but didn't speculate beyond evidence
- Hot path builder should also glob for *dream*, *consolidat*, *promote* files for memory-heavy repos
- Dim 3 agent could benefit from explicit instruction to check for skill-related code (it found skills-as-not-tools, which was one of the strongest findings)

---

### claude-code — 2026-04-14

**Research goal:** reverse-engineer Claude Code harness architecture for pattern extraction
**Relevance score:** 10/10
**Dimensions analyzed:** Architecture, Memory, Tools, Multi-Agent (all 4)
**Evidence validation:** 100% verified (20/20 key paths)

**What worked well:**
- All 4 dimension agents produced exceptionally deep, evidence-grounded reports
- Dim 1 agent navigated the ~1,500-line query.ts loop and identified all termination conditions
- Dim 2 agent found all 5 memory systems including the extractMemories RAG pipeline
- Dim 3 agent cataloged 50+ tools and identified the deferred loading mechanism
- Dim 4 agent documented 3 multi-agent modes (standard, fork, swarm) with coordination protocols
- Hot path builder produced a rich 40-file index with 10 categories — excellent for large repos
- 100% evidence verification rate

**What didn't work:**
- Context pressure: each Sonnet dimension agent used ~100-130K tokens on this large repo
- Wave 1 agents took ~8 minutes each (vs ~1 min for autoresearch). Large repos need patience.

**Missed findings (discovered during synthesis):**
- The prompt cache-first architecture is the single most important cross-cutting pattern, but no individual dimension agent framed it as the organizing principle
- The forked agent pattern appears in D1 (compaction), D2 (extractMemories), and D4 (sub-agents) but none connected them as instances of the same pattern

**Prompt improvement suggestions:**
- For large repos (50K+ LOC), add budget awareness: "Prioritize breadth of checklist coverage over exhaustive reading"
- Add a cross-cutting question: "What patterns in your dimension also appear in other dimensions?"
- Hot path builder should include *query*, *bootstrap*, *services* directory listings for TypeScript repos

---

### all-hands-ai-openhands — 2026-04-14

**Research goal:** analyze event-driven agent loop, runtime sandboxing, and micro-agent delegation patterns
**Relevance score:** 10/10
**Dimensions analyzed:** Architecture, Memory, Tools, Multi-Agent (all 4)
**Evidence validation:** 100% verified (49/49 key paths)

**What worked well:**
- All 4 dimension agents produced exceptionally rich reports with specific file:line citations
- Dim 1 agent correctly identified the event-driven (callback) nature of the loop — not deceived by the misleading `loop.py` file name
- Dim 2 agent thoroughly documented all 9 condenser implementations and the pipeline composability pattern
- Dim 3 agent found the security-risk-as-parameter pattern and the MCP stdio-over-HTTP proxy — both genuinely novel
- Dim 4 agent correctly distinguished microagents (prompt injection) from true delegation (AgentDelegateAction)
- Hot path builder produced 40 files across 7 categories — excellent coverage for this large Python repo
- 100% evidence verification rate on 49 paths — agents used repo-relative paths correctly
- Cross-cutting question worked well again — Dim 1 and Dim 3 both noted the condensation dual-path pattern independently

**What didn't work:**
- Context pressure: Sonnet agents used ~85-112K tokens each; Wave 2 Dim 3 was the longest at ~415K ms
- The V0 legacy status permeated every file, leading all 4 agents to note it — useful but repetitive across reports
- No agent connected the "condensation as tool" finding (Dim 3) with the "condenser-returns-action" finding (Dim 1) as the same architectural pattern viewed from different angles — synthesis step caught this

**Missed findings (discovered during synthesis):**
- The three-layer condensation defense (agent-requested + error-triggered + stuck-loop detection) was not framed as a unified defense-in-depth pattern by any individual agent
- The EventStream as the single integration point for ALL subsystems (memory, tools, delegation, audit) was noted by each agent for their dimension but none captured the full scope of "one stream for everything"
- The security assessment dual-layer (model self-labeling + harness SecurityAnalyzer) was not identified as a cross-cutting pattern — Dim 1 and Dim 3 each saw their half

**Prompt improvement suggestions:**
- Consider adding an explicit instruction: "Note any deprecated/legacy markers and their implications for the research goal"
- For Python repos, hot path builder should also glob for *critic*, *linter*, *security*, *resolver* patterns — these were relevant in OpenHands
- The cross-cutting question could be more specific: "What patterns in your dimension connect to patterns another specialist might find in the memory/tool/coordination dimension?"
- For repos with many condenser/strategy implementations, Dim 2 agent could benefit from a budget note: "Read the abstract base class first, then sample 3-4 implementations, don't read all"

---

### anomalyco-opencode — 2026-04-14

**Research goal:** Analyze alternative harness design — compare with Claude Code. Look for novel patterns in tool design or context management.
**Relevance score:** 9/10
**Dimensions analyzed:** Architecture, Memory, Tools, Multi-Agent (all 4)
**Evidence validation:** 100% verified (36/36 key paths)

**What worked well:**
- All 4 dimension agents produced excellent reports with specific file:line evidence
- Dim 1 agent correctly identified the while(true) loop and all 5 termination paths, including the novel max-steps-as-assistant-prefill pattern
- Dim 2 agent found the triple-storage architecture and the snapshot-per-step git time-travel revert — genuinely novel
- Dim 3 agent identified the 9-strategy fuzzy edit cascade and tree-sitter bash AST parsing — the strongest findings of this research run
- Dim 4 agent correctly distinguished ACP (external protocol) from internal multi-agent coordination
- Hot path builder adapted well to the monorepo structure (focused on packages/opencode/src/)
- Fastest Wave 1 of any large repo (~2.5 min per agent vs ~4.5 min for OpenHands) — well-structured TypeScript is easier for agents to navigate

**What didn't work:**
- The multi-agent signal count (128) was above the threshold but many matches were for process spawning (cross-spawn, Bun.spawn) not actual multi-agent patterns. Could have been a Dim 4 skip. However, the actual multi-agent system (task tool, subtask queue, plan agent) is genuinely rich, so dispatching was correct.
- No agent noticed the AGENTS.md file at the repo root, which contains strict style guidelines and repo conventions — could be relevant to how the harness enforces coding standards

**Missed findings (discovered during synthesis):**
- The three-stage tool output lifecycle (truncate on generation → prune on aging → compact on overflow) is a unified pattern that no individual agent framed as such
- The permission system as the harness's central control plane (gating tool execution, loop termination, AND delegation scope) was noted piecemeal by each dimension but not unified
- Agent mode switching is multi-modal (tool-call, queue-based, config-based) — this was scattered across Dims 1, 3, and 4

**Prompt improvement suggestions:**
- For TypeScript monorepos, hot path builder should also glob for packages/*/src/ patterns to find the core package faster
- Dim 3 should be explicitly asked to look for fuzzy matching or tolerance strategies in edit tools — this was the strongest finding and could have been highlighted more
- For repos with AGENTS.md at root, Dim 1 should read it as part of the system prompt analysis

---
