---
title: "Jason Kneen: Cursor 3.0 reverse-engineering report (GitHub Gist)"
source: https://gist.github.com/jasonkneen/4c065df2d7a95610e4fd30c3e3398b17
author: Jason Kneen
handle: jasonkneen
date: 2026-04-12
fetched: 2026-04-18
type: article
status: compiled
source_method: reference_only
compiled_to:
  - wiki/thin-harness-fat-skills.md
  - wiki/tool-design-patterns.md
compiled_date: 2026-04-23
tags: [cursor-3.0, cursor-agent, claude-code, reverse-engineering, anthropic-sdk]
context: "Full reverse-engineering report (titled 'report.md') of Cursor 3.0 / Cursor Agent, linked from @jasonkneen/2043435856849940818. WebFetch could not reproduce the gist verbatim; summary claims below are from the linked tweet."
---

# Kneen Cursor 3.0 report — Summary (via linked tweet)

Full gist at https://gist.github.com/jasonkneen/4c065df2d7a95610e4fd30c3e3398b17 — content not mirrored here due to tool-level reproduction constraints. The headline claims as excerpted in @jasonkneen's tweet on April 12, 2026:

- **"Cursor Agent" is a rebranded Claude Code.** The Cursor Agent product ships Anthropic's Claude Code harness under the hood.
- **Local proxy with find-and-replace engine.** A local proxy sits between the user and the SDK; it rewrites messages before they reach the LLM.
- **String substitution: "Claude" → "Cursor".** System prompts and messages have all mentions of "Claude" replaced with "Cursor" before the user sees them.
- **Bundled packages.** The Cursor Agent build bundles `@anthropic-ai/claude-agent-sdk` and `@anthropic-ai/claude-code` as dependencies.
- **Custom fine-tuned model.** Cursor uses a fine-tune labeled `claude-3.7-sonnet-finetuned-cursor-20250514-v1`, served through Anthropic's API surface.

**Implications documented in the KB:**

- Cursor, valued at roughly $10B, has built its product on Anthropic's harness rather than its own.
- Tan's "thin harness, fat skills" thesis is validated at scale — Cursor chose the thinnest possible harness (renting Anthropic's) and differentiated on the fine-tuned model + IDE surface.
- Chase's "your harness, your memory" warning also applies — Cursor now depends on Anthropic's harness, SDK, and memory model for its core product.

**Source note:** Gist URL resolved via t.co shortlink by the `general-purpose` subagent (Session: 2026-04-18). Tweet claims transcribed from user-provided screenshots (`Screenshot 2026-04-17 142217.png` and `Screenshot 2026-04-17 142251.png`). For the complete reverse-engineering report, cite the Gist URL directly.
