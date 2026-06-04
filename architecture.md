---
title: "ggcoder Architecture — 14-Package Monorepo, Goal System, Tool Surface"
description: "Inside KenKaiii/gg-framework: the TypeScript monorepo layout, the UI re-architecture of May 2026, the Goal system maturation, the 16-tool LLM surface, and what got added then removed."
---

# Architecture

Window observed: 2026-05-25 → 2026-06-03 (ggcoder 4.3.217 → 4.3.243).

## The 14-package monorepo

`KenKaiii/gg-framework` is a pnpm-workspace monorepo. As of 4.3.243:

| Package | Role |
|---------|------|
| **`gg-ai`** | LLM provider abstraction layer. Direct HTTP clients for 9 providers, no SDK dependencies. Shared error / SSE / schema helpers (refactored 2026-06-01 in commit `bd5635d`). |
| **`gg-agent`** | Agent loop core. `agent-loop.ts` runs the tool-call cycle. |
| **`ggcoder`** | The headline coding-agent CLI. TUI (Ink), session manager, model registry, tools, skills, MCP, ACP modes. |
| **`ggcoder-eyes`** | Vision/screenshot extension. |
| **`gg-boss`** | Orchestrator that runs multiple ggcoder workers (one per project), assigns tasks, monitors progress. Stand-alone CLI. |
| **`gg-voice`** | Realtime voice orchestration (added 2026-05-19 in commit `41327da`). OpenAI Realtime + Codex Realtime providers, bridges to ggcoder via RPC. Mobile-safe package root. |
| **`gg-editor`** | Video/editor extension (Premiere Pro panel exists in a sibling package). |
| **`gg-pixel-*`** (6 packages) | Pixel-art helpers in Go / Python / Ruby / Rust / Swift / Server — utility experiments outside the coding-agent scope. |

The coding-agent stack you'd use day-to-day is **`ggcoder` + `gg-ai` + `gg-agent`**. Everything else is optional layers.

## Recent major work (2026-05-25 → 2026-06-03)

In the last 10 days, ~75 commits and ~12 releases (4.3.217 → 4.3.243). The work splits into 4 themes by line count:

| Theme | Lines churned | What it is |
|------|---------------|-----------|
| **UI re-architecture** | ~14,200 | App.tsx monolith decomposed into 25+ extracted hooks/components/services |
| **Goal system maturation** | ~3,000 | New worktree integration, worker orchestration, plan upserts, evidence flow |
| **New tools added** | ~5,000 | screenshot, file checkpoint/rewind, html-extract, pdf-extract, ideal-review hook, plan mode (`enter-plan`/`exit-plan`) |
| **Provider + quota work** | ~1,300 | Video input (commit `bd5d60b`), hard quota error classification, system prompt slimming |

### UI re-architecture (the biggest single effort)

Before this window: `App.tsx` was a large file holding most rendering, layout decisions, and event handling. Between 2026-05-26 and 2026-05-31, it was decomposed into:

```
packages/ggcoder/src/ui/
├── App.tsx                                ← still exists, now coordinator
├── components/
│   ├── ChatScreen.tsx                     ← extracted main screen (+302 lines)
│   ├── SessionSummary.tsx                 ← exit-summary panel (+146)
│   ├── SlashStyledSelectList.tsx          ← slash command picker (+126)
│   ├── InputArea.tsx                      ← input box
│   ├── RewindOverlay.tsx                  ← file-rewind UI (+81)
│   ├── IdealHookMessage.tsx               ← agent-loop hook UI (+45)
│   └── AssistantMessage.test.tsx
├── hooks/
│   ├── useChatLayoutMeasurements.ts       ← layout measuring (+133)
│   ├── useTranscriptHistory.ts            ← history hook (+158)
│   └── useAgentLoop.ts                    ← agent integration hook
├── transcript/
│   ├── TranscriptRenderer.tsx             ← (+194)
│   ├── MiscRows.tsx                       ← (+205)
│   ├── presentation.ts                    ← (+212)
│   ├── spacing.ts                         ← (+188 logic + tests)
│   └── spacing.test.ts
├── utils/
│   └── terminal-graphics.ts               ← inline image rendering (+80)
├── layout-decisions.ts                    ← pure-logic layout rules (+465)
├── prompt-routing.ts                      ← input → action routing (+190)
├── submit-prompt-command.ts               ← submit pipeline (+160)
├── terminal-history.ts                    ← history state mgmt (+771 over 9 commits)
├── terminal-history-format.ts             ← formatting (+176)
├── terminal-history-status-renderers.ts   ← status renderers (+223)
├── app-items.ts                           ← App-level item model (+290)
├── goal-progress.ts                       ← Goal progress display (+236)
├── tool-group-summary.ts                  ← tool output summarization (+212)
└── session-summary.ts                     ← exit summary data collection (+139)
```

**61 brand-new source files in 3 days** (2026-05-26 → 2026-05-28). The pattern: extract pure logic to `.ts`, presentational concerns to `components/`, reusable behaviour to `hooks/`.

**Senior-eye observation**: this is preparing for community contributions. A 5000-line `App.tsx` is hostile to PRs; a modular layout invites them. No obvious external contributors have arrived yet (Ken Kai still authors ~95% of commits).

### The Goal system

A new orchestration primitive that became the headline feature. From the commit log, Goals work like this:

1. User defines a Goal (multi-step task with success criteria)
2. ggcoder spawns Goal "workers" — each worker is a focused sub-session in a git **worktree**
3. Workers commit their changes to worktree branches
4. Goal system merges back to main when done, with evidence (test output, file diffs)
5. Mid-stream pause / resume / cancel

Files of note (in `packages/ggcoder/src/core/`):
- `goal-overhead-harness.ts` (+372 lines) — telemetry / efficiency tracking
- `goal-worktree.ts` (+124) + test — worktree lifecycle
- `goal-lifecycle-orchestration.test.ts` (+178) — black-box test of the orchestration

**~17 of the last 34 commits touch Goal** — it's the central feature investment right now.

### Tools added in the last 10 days

| Tool | Where | Added in |
|------|-------|----------|
| **Plan mode** (`enter-plan`, `exit-plan`) | `tools/enter-plan.ts`, `tools/exit-plan.ts` | `e787f20` (5/27) |
| **File checkpoint/rewind** | `core/checkpoint-store.ts` (+241) | `a1f321b` (5/29) |
| **Screenshot tool** | `tools/screenshot.ts` (+197) | `a1f321b` (5/29) |
| **Inline terminal image rendering** | `ui/utils/terminal-graphics.ts` (+80) | `a1f321b` (5/29) |
| **HTML extraction** | `tools/html-extract.ts` (+182) | `af918e3` (5/31) |
| **PDF extraction** | `tools/pdf-extract.ts` (+51) | `af918e3` (5/31) |
| **Ideal-review hook** | `core/ideal-review.ts` (+82) | `af918e3` (5/31) |
| **Web fetch upgrade** | `tools/web-fetch.ts` (+437, rewrite) | `af918e3` (5/31) |
| **Web search upgrade** | `tools/web-search.ts` (+368, rewrite) | `af918e3` (5/31) |
| **Read-only bash** | `tools/read-only-bash.ts` | (during plan-mode work) |

The current tool surface exposed to the LLM is approximately:

```
read, write, edit, edit-diff, bash, find, grep, ls,
screenshot, html-extract, pdf-extract, web-fetch, web-search,
enter-plan, exit-plan, read-only-bash
+ subagent (parallelized as of 5/29, commit cfa773e)
+ goal-related tools
+ MCP-loaded tools (varies)
```

— roughly **16 built-in + N MCP tools**. For comparison, see [vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md).

## Session model

Sessions are persisted, branchable, and resumable.

| Capability | Where |
|-----------|-------|
| Branch from any turn | core/session-manager.ts |
| Compaction (lossy summarize) | core/compaction/ |
| **Handoff-style new session with context distillation** | implemented in agent-session-services |
| Continue replay | `--continue` flag |
| Resume by session ID | `--resume <id>` |
| Auto-checkpoint of file edits | `core/checkpoint-store.ts` |
| Exit summary | `ui/session-summary.ts` (added 2026-05-27) |

## Things removed in this window

- **Dynamic repo map** (`4122092`, 2026-05-27) — added ~10 days earlier, removed wholesale. -2,594 lines, +25 lines. The replacement is static workspace info in system prompt. Reason not stated in commit message; preceding commits show "Fix repo map hangs" type fixes, so likely performance / reliability.

## What's NOT here

- **No GUI / desktop app** (TUI only). The `gg-editor-premiere-panel` exists but is a Premiere Pro plugin, not a ggcoder GUI.
- **No web client.** The `serve` subcommand exists per CLI help but isn't a polished web UI.
- **No subagent / MCP / SKILL support in `openab-agent`** (the sibling Rust project) — those are ggcoder-only.

## See also

- **[providers.md](./providers.md)** — the LLM-facing wire-format details
- **[engineering-notes.md](./engineering-notes.md)** — Ken Kai's velocity + the dropped features
- **[vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md)** — positioning
