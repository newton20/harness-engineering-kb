---
title: Autoresearch and Self-Improvement
type: wiki
tags:
  - autoresearch
  - self-improvement
  - evaluation
  - iteration
  - automated-optimization
  - skill-refinement
  - multi-agent
  - autocontext
sources:
  - raw/itsolelehmann-2033919415771713715.md
  - raw/anthropic-com-engineering-harness-design-long-running-apps.md
  - raw/JayScambler-2033971974284714355.md
  - raw/arxiv-org-html-2512-20491v1.md
  - raw/arxiv-org-html-2501-09136v4.md
  - raw/code-research-karpathy-autoresearch.md
source_count: 6
status: draft
last_compiled: 2026-04-14
---

# Autoresearch and Self-Improvement

One of the most powerful applications of agent harnesses is turning agents on themselves -- using automated loops to iteratively improve agent skills, prompts, and outputs without human intervention. Karpathy's autoresearch concept, adapted for agent skills by Ole Lehmann, demonstrated that a landing page copy skill could go from 56% to 92% pass rate with zero manual work. Anthropic's GAN-inspired evaluator architecture and Jay Scambler's autocontext system extend the same idea into multi-agent evaluation loops with persistent knowledge accumulation.

## The Autoresearch Concept

Andrej Karpathy (co-founder of OpenAI, former head of AI at Tesla) released a method called autoresearch: an automated loop where a system tries a small change, checks if the result got better, keeps it if it did, throws it out if it did not, and repeats. [Source: raw/itsolelehmann-2033919415771713715.md]

The key insight is that this loop does not require sophisticated reinforcement learning or training infrastructure. It operates at the prompt and skill level, using the same models that power the agent to also evaluate and improve the agent's behavior. Karpathy originally used it for machine learning code, but the method works on anything that can be measured and improved. [Source: raw/itsolelehmann-2033919415771713715.md]

## First-Hand Code Analysis: karpathy/autoresearch

First-hand source code analysis of Karpathy's autoresearch repo reveals implementation patterns not visible from external descriptions. The repo is deliberately minimal: 2 Python files (train.py at 630 LOC, prepare.py at 389 LOC) + program.md (115 lines). [Source: raw/code-research-karpathy-autoresearch.md]

### Prose-as-Control-Flow

The entire agent harness is `program.md` -- a markdown file that IS the agent's system prompt. There is no Python orchestrator, no scheduler, no state machine in code. The LLM reads the numbered steps ("LOOP FOREVER: 1. Look at the git state... 2. Tune train.py... 3. git commit... 4. uv run train.py > run.log 2>&1...") and executes them as its own decision procedure. This is the simplest possible harness: the system prompt IS the harness. [Source: raw/code-research-karpathy-autoresearch.md]

### Git-as-Experiment-Database

Every `git commit` before a run is a hypothesis insertion into a content-addressed database. `git reset` is the rollback/discard operation. The branch HEAD always points to the current best-performing hypothesis. `results.tsv` is deliberately gitignored to separate outcome-memory (what was tried and how it scored) from code-state (what worked). This creates clean git history (only kept experiments as commits) alongside a comprehensive experiment log (all attempts including failures). [Source: raw/code-research-karpathy-autoresearch.md]

### Fixed-Budget Evaluation

Every experiment trains for exactly 5 minutes (`TIME_BUDGET = 300` in prepare.py, immutable). This makes val_bpb directly comparable across all experiments without controlling for compute cost. The 10-step warmup exclusion (`if step > 10 and total_training_time >= TIME_BUDGET: break`) surgically removes torch.compile overhead from the budget. This is a convergence proxy, not convergence detection -- the outer loop has no stopping criterion and runs until human interruption. [Source: raw/code-research-karpathy-autoresearch.md]

### Minimal-Signal Extraction

Training output is redirected to run.log (`> run.log 2>&1` with the explicit instruction "do NOT use tee or let output flood your context"). Only 2 scalars are extracted via `grep "^val_bpb:\|^peak_vram_mb:" run.log`. Each experiment adds ~3-5 lines to the agent's context instead of 630 lines of training output. This is an explicit anti-context-bloat pattern. [Source: raw/code-research-karpathy-autoresearch.md]

### Structural Immutability

The boundary between mutable code (train.py -- "the file you modify") and immutable evaluation (prepare.py -- "do not modify") is enforced by file separation + natural language contract. The evaluation oracle (`evaluate_bpb` in prepare.py) is structurally protected from the agent. The simplicity criterion adds a second optimization objective: "A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not worth it." This makes the loop multi-objective (metric + simplicity). [Source: raw/code-research-karpathy-autoresearch.md]

### Meta-Learning, Not Model Training

Each training run starts from random initialization (`torch.manual_seed(42)`). No model checkpoints are saved. The "progress" accumulated is improvement in code quality (the training recipe), not trained model weights. This is a meta-learning setup: the agent learns a better training algorithm, not a better model. [Source: raw/code-research-karpathy-autoresearch.md]

## Adapting Autoresearch for Agent Skills

@itsolelehmann adapted Karpathy's autoresearch concept for Claude skills, demonstrating the approach with a concrete workflow: "run autoresearch on my landing page skill." [Source: raw/itsolelehmann-2033919415771713715.md] The process:

1. **Pick a skill to improve.** Choose the one where output quality is inconsistent.
2. **Define scoring criteria** as a binary yes/no checklist (the only human input required).
3. **Establish a baseline** by running the skill and scoring the output.
4. **Enter the loop:** The agent analyzes what is failing, makes one small change to the skill prompt, tests again, and keeps the change if the score goes up or undoes it if it goes down.
5. **Repeat** until the score stabilizes at 95%+ three times in a row, or until stopped.

The improved skill is saved as a separate file so the original stays untouched, and a changelog records every change attempted, why the agent tried it, and whether it helped. [Source: raw/itsolelehmann-2033919415771713715.md]

## Scoring: Yes/No Checklists, Not Vague Scales

A critical implementation detail is how improvement is measured. The autoresearch approach uses binary yes/no checklists rather than vague 1-10 scales. Each quality criterion is phrased as a concrete, verifiable question: [Source: raw/itsolelehmann-2033919415771713715.md]

- "Does the headline include a specific number or result?" (catches vague headlines like "Grow Your Business")
- "Is the copy free of buzzwords like 'revolutionary,' 'synergy,' 'cutting-edge,' 'next-level'?"
- "Does the CTA use a specific verb phrase?" (catches weak CTAs like "Learn More")
- "Does the first line call out a specific pain point?" (catches generic openers)

Binary criteria are more reliable than scalar ratings because they reduce evaluator subjectivity. A model asked "rate this output 1-10 on quality" will give inconsistent scores across runs. A model asked a concrete yes/no question will give consistent answers. [Source: raw/itsolelehmann-2033919415771713715.md]

The sweet spot is 3-6 checklist questions. More than that and the skill starts "gaming the checklist" -- like a student who memorizes answers without understanding the material. [Source: raw/itsolelehmann-2033919415771713715.md]

## Results: 56% to 92%

The results from applying autoresearch to a landing page copy skill were striking: the skill's pass rate went from **56% to 92%** in 4 rounds of changes (3 kept, 1 undone), with zero manual work. [Source: raw/itsolelehmann-2033919415771713715.md]

Specific changes the agent made:
- Added a rule for the most common failure: "Your headline must include a specific number or result. Never use vague promises like 'Transform Your Business.'"
- Added a banned buzzwords list.
- Added a worked example of a strong landing page section so the skill could see what good looks like.
- Tried a tighter word count, undid it because the copy got too thin and the CTA suffered. [Source: raw/itsolelehmann-2033919415771713715.md]

The method generalizes beyond skills: one person ran it on page load time and went from 1100ms to 67ms in 67 rounds. It works on cold outreach, newsletter intros, or any prompt used repeatedly -- "if you can score it, you can autoresearch it." [Source: raw/itsolelehmann-2033919415771713715.md]

## The GAN-Inspired Evaluator

Anthropic's approach to evaluation in long-running agent harnesses draws inspiration from Generative Adversarial Networks (GANs). The insight is that self-evaluation fails: "When asked to evaluate work they've produced, agents tend to respond by confidently praising the work -- even when, to a human observer, the quality is obviously mediocre." [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

The solution is a separate evaluator agent. Anthropic designed four grading criteria given to both generator and evaluator agents: [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

- **Design quality:** Does the design feel like a coherent whole rather than a collection of parts?
- **Originality:** Is there evidence of custom decisions, or is this template layouts and AI-generated patterns? Explicitly penalizes "AI slop" patterns like purple gradients over white cards.
- **Craft:** Technical execution -- typography hierarchy, spacing consistency, color harmony, contrast ratios.
- **Functionality:** Usability independent of aesthetics.

The evaluator was calibrated with few-shot examples with detailed score breakdowns. In practice, the evaluator navigated the page on its own using Playwright MCP, screenshotting and studying the implementation before producing its assessment. Feedback flowed back to the generator as input for the next iteration. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## The Evaluator Must Be Separate and Tunable

A key architectural insight: the evaluator must be a separate agent from the generator. Models are lenient self-evaluators. The separation does not immediately eliminate leniency -- the evaluator is still an LLM inclined to be generous toward LLM-generated outputs. But tuning a standalone evaluator to be skeptical turns out to be "far more tractable than making a generator critical of its own work." [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

Tuning the evaluator required reading the evaluator's logs, finding examples where its judgment diverged from the developer's, and updating the QA's prompt to solve for those issues. It took several rounds before the evaluator was grading reasonably. The evaluator's findings were specific enough to act on: for example, identifying that a fillRectangle function exists but is not triggered properly on mouseUp, or that a PUT route defined after a parameterized route causes a 422 error. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## Iteration Counts and Runtime

Practical autoresearch/evaluation runs operate at a specific scale: [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

- **5-15 iterations per generation:** Each iteration typically pushes the generator in a more distinctive direction.
- **Full runs up to 4 hours:** Anthropic's updated harness for a DAW application ran for approximately 3 hours 50 minutes at $124.70 total cost across planner, multiple build rounds, and QA rounds.
- **Cost comparison:** A solo agent run took 20 minutes and $9; the full harness took 6 hours and $200 -- over 20x more expensive, but with dramatically better output quality. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]

## autocontext: Multi-Agent Learning System

Jay Scambler's autocontext extends autoresearch into a full multi-agent optimization system with persistent knowledge accumulation. The core problem it addresses: "Agents have no persistent memory across runs. No mechanism to carry forward what worked and what didn't. Every invocation starts cold." [Source: raw/JayScambler-2033971974284714355.md]

The autocontext loop uses a team of specialized agents: [Source: raw/JayScambler-2033971974284714355.md]
- **Competitor:** Generates a strategy for the task.
- **Translator:** Converts raw output into validated, structured JSON strategy.
- **Analyst:** Examines what happened -- findings, root causes, recommendations.
- **Coach:** Distills analysis into a playbook update and competitor hints.
- **Architect:** Proposes and generates tooling improvements.
- **Curator:** Gates quality of playbook updates and consolidates lessons.
- **Orchestrator:** Sequences the entire pipeline with parallel execution and retry logic.

Each generation runs through tournament matches (for game scenarios) or LLM judge evaluation (for agent tasks) with Elo-based progression gating. Strategies that do not improve get rolled back. Knowledge only persists when validated. [Source: raw/JayScambler-2033971974284714355.md]

The **playbook** is the key abstraction -- a living document that grows across runs containing strategies that worked (with scores proving they worked), strategies that failed (with specific failure modes to avoid), tier-specific rules, and generated tools. When run against a grid capture-the-flag scenario, it accumulated 33 distinct lessons across 2 generations, producing 5,870 characters of actionable operational knowledge. [Source: raw/JayScambler-2033971974284714355.md]

## Evaluation Quality Is the Bottleneck

Across all three sources, a consistent theme: the quality of the evaluator determines whether the loop improves or just churns.

autocontext's judge subsystem uses multi-dimensional rubrics where each task defines specific dimensions (accuracy, clarity, actionability). The judge scores each dimension independently so revisions can target the weakest dimension. It includes a 4-tier fallback parser for handling messy LLM outputs, and adversarial testing showed the system handles vague rubrics, contradictory criteria, and hallucination-prone tasks. [Source: raw/JayScambler-2033971974284714355.md]

The harness engineer's role shifts from directly writing optimal prompts to designing good evaluation criteria and improvement loops. The quality of the evaluator becomes the bottleneck -- if your evaluator cannot distinguish good from bad, the autoresearch loop will not converge on improvement.

## Deep Research as Scaled Autoresearch

Deep research represents the natural evolution of autoresearch: instead of iterating on a single skill prompt, the agent iterates on search-reason loops at scale, progressively gathering and synthesizing information across dozens of sources. Where autoresearch improves a prompt through trial-and-error, deep research improves understanding through iterative search, retrieval, and synthesis.

### Step-DeepResearch: Training Atomic Capabilities

Step-DeepResearch (Ren et al., 2025) demonstrates that a 32B open-source model can rival closed-source deep research systems by training individual capabilities separately rather than end-to-end. The "Atomic Capabilities" training strategy decomposes deep research into four distinct skills -- planning, information gathering, reflection, and report writing -- and trains each one independently before combining them. [Source: raw/arxiv-org-html-2512-20491v1.md]

The evaluation method is itself an autoresearch innovation: a **Checklist-style Judger** generates a checklist of criteria from the research question, then scores the output against each criterion independently. This is structurally identical to the yes/no checklist scoring that makes autoresearch work (see "Scoring: Yes/No Checklists, Not Vague Scales" above), but applied to research output rather than skill output. Step-DeepResearch achieved **61.4% on Scale AI Research Rubrics**, competitive with proprietary systems while using a fraction of the compute. [Source: raw/arxiv-org-html-2512-20491v1.md]

The implication for harness engineering: decomposing a complex agent capability into atomic sub-capabilities and training/optimizing each one separately may be more effective than trying to optimize the whole system at once. This mirrors the broader pattern of breaking autoresearch loops into targeted improvement cycles.

### Agentic RAG as Self-Improving Retrieval

Agentic RAG (Singh et al., 2025) represents another evolution of the autoresearch concept, applied specifically to retrieval. Traditional RAG uses a static retrieve-then-generate pipeline. Agentic RAG embeds autonomous agents into the retrieval process: agents dynamically decide what to retrieve, evaluate whether retrieved information is sufficient, reformulate queries when initial results are poor, and iteratively refine their understanding before generating output. [Source: raw/arxiv-org-html-2501-09136v4.md]

This is autoresearch applied to the retrieval step itself: the agent runs a loop of query-retrieve-evaluate-refine until the retrieved context meets a quality threshold, then generates its response. The agent is simultaneously the researcher and the evaluator of its own research quality -- the same self-improvement loop, but operating at the retrieval level rather than the prompt level. [Source: raw/arxiv-org-html-2501-09136v4.md]

## Frontier-to-Local Distillation

autocontext implements a pipeline for distilling strategies from frontier models into local models: run discovery with a frontier model (Claude Opus 4.6, GPT-5.4), export training data from the run database, train a small local model via MLX on Apple Silicon, and route future runs through the local model when strong enough, falling back to the frontier model when weak. [Source: raw/JayScambler-2033971974284714355.md]

## Related

- [Practical Best Practices](practical-best-practices.md) -- Handling sycophancy with adversarial agents, iterative improvement philosophy
- [Agent Memory and Context Management](agent-memory-and-context-management.md) -- Knowledge persistence across runs, playbook as long-term memory
- [Long-Running Agent Harnesses](long-running-agent-harnesses.md) -- Multi-agent architecture, planner-generator-evaluator patterns
- [Tool Design Patterns](tool-design-patterns.md) -- No-op planning tools, evaluation as a tool design concern
- [Deep Research Agents](deep-research-agents.md) -- deep research as scaled autoresearch, convergence detection, Step-DeepResearch architecture
- [Agentic Design Patterns](agentic-design-patterns.md) -- Reflection pattern as the theoretical foundation of autoresearch loops
- [Multi-Agent Reliability](multi-agent-reliability.md) -- adversary-resistant evaluation and credibility scoring for multi-agent self-improvement

## Open Questions

- How to prevent overfitting to the checklist -- the "student who memorizes answers without understanding the material" problem. At what point does autoresearch produce skills that pass their checklist but fail on edge cases not covered by the criteria? [Source: raw/itsolelehmann-2033919415771713715.md]
- How does the evaluator-generator dynamic change as models improve? Anthropic found that on Opus 4.6, the evaluator was less load-bearing for tasks within the model's native capability, but still critical for edge-of-capability work. [Source: raw/anthropic-com-engineering-harness-design-long-running-apps.md]
- Can autocontext-style knowledge accumulation transfer across fundamentally different task domains, or does the playbook become domain-specific?

## Sources

- [raw/itsolelehmann-2033919415771713715.md](../raw/itsolelehmann-2033919415771713715.md) -- Ole Lehmann on adapting Karpathy's autoresearch for Claude skills. 56% to 92% pass rate on landing page copy skill. Yes/no checklists, the loop mechanics, generalization to other domains.
- [raw/anthropic-com-engineering-harness-design-long-running-apps.md](../raw/anthropic-com-engineering-harness-design-long-running-apps.md) -- Anthropic's Prithvi Rajasekaran on GAN-inspired generator-evaluator architecture for frontend design and full-stack coding. Design quality criteria, evaluator tuning, iteration counts, cost comparison, sprint contracts.
- [raw/JayScambler-2033971974284714355.md](../raw/JayScambler-2033971974284714355.md) -- Jay Scambler on autocontext: multi-agent learning system with persistent knowledge accumulation, playbook abstraction, Elo-based progression gating, frontier-to-local distillation, multi-dimensional rubric judging.
- [raw/arxiv-org-html-2512-20491v1.md](../raw/arxiv-org-html-2512-20491v1.md) -- Ren et al., 2025. Step-DeepResearch: 32B open-source model trained with Atomic Capabilities strategy (planning, info gathering, reflection, report writing) and Checklist-style Judger rewards. 61.4% on Scale AI Research Rubrics.
- [raw/arxiv-org-html-2501-09136v4.md](../raw/arxiv-org-html-2501-09136v4.md) -- Singh et al., 2025. Agentic RAG survey: evolution from static RAG to agent-driven retrieval with dynamic query reformulation, sufficiency evaluation, and iterative refinement.
- [raw/code-research-karpathy-autoresearch.md](../raw/code-research-karpathy-autoresearch.md) -- First-hand source code analysis via /kb-code-research skill, Apr 2026. Deep 3-dimension analysis of the actual autoresearch codebase: prose-as-control-flow, git-as-experiment-database, fixed-budget evaluation, structural immutability, meta-learning pattern.
