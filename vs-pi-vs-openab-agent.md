# ggcoder vs Pi vs openab-agent

Three coding-agent projects that share design DNA but serve fundamentally different customers.

## TL;DR — the three categories

| | **ggcoder** | **Pi** | **openab-agent** |
|---|---|---|---|
| **Customer** | Human at a terminal | Human at a terminal + extension authors | **Another system** (OpenAB orchestrator) |
| **Stack** | TypeScript / Ink / pnpm monorepo | TypeScript / Ink / pnpm monorepo | **Rust** / single binary |
| **Source size** | 14 packages | 4 packages | 1 binary, 6 files, ~80 KB |
| **Memory / session** | ~200 MB+ | ~200 MB | **7 MB** |
| **Tool count exposed to LLM** | ~16 + MCP | ~8 (built-in) | **4** |
| **TUI** | ✅ Ink + React | ✅ Ink + React | ❌ |
| **Subagents** | ✅ (parallel since 2026-05-29) | ❌ in core (extension example: `handoff.ts`) | ❌ |
| **MCP** | ✅ deep | ❌ in core (extension) | ❌ (PR #955 in flight) |
| **Skills system** | ✅ | ❌ in core (extension) | 🟡 PR #955 lands soon |
| **Plan mode** | ✅ (`enter-plan`, `exit-plan`) | ❌ in core | ❌ |
| **Session branching** | ✅ | ✅ | ❌ (flat history, hard message cap) |
| **Compaction** | ✅ | ✅ | ❌ (just drop oldest when over cap) |
| **OAuth subscription auth** | ✅ Codex + Claude Pro | ✅ provider-extension | ✅ Codex (Anthropic + Codex only) |
| **MiniMax / Kimi / GLM** | ✅ all of them | provider-extensions | ❌ Anthropic + Codex only |
| **License** | MIT | MIT | MIT |

## What each one is actually for

### ggcoder — batteries-included personal coding agent

Closest to Claude Code or Cursor in product position. Targets a developer at a keyboard who wants:

- Rich TUI with live transcript, plan overlay, goal pane
- Many tools out of the box, no extension hunting required
- Provider switching mid-session (`/model`)
- Branching session trees
- Skills + MCP for power users
- Voice / video input on supported models

**You pick ggcoder when:** you want one CLI to install and have a productive day, with reasonable defaults for everything.

### Pi — extension-first coding agent toolkit

The `pi-coding-agent` package is smaller in core but ships with **30+ example extensions** in `packages/coding-agent/examples/extensions/`. Things ggcoder treats as built-in (plan mode, subagent / handoff, skills, MCP, dynamic tools) are **opt-in extensions** in Pi.

**Pi's "minimalism" is architectural**, not code-size. The repo is 47 MB. What's minimal is:

- The default agent loop runs with ~8 tools and a short system prompt
- Adding features means installing extensions
- The user controls the surface area

**You pick Pi when:** you want to build something on top of an agent loop, not consume an agent product. You're going to write or install extensions to shape it.

### openab-agent — ACP-only fleet worker

A **single Rust binary**, 6 source files, ~80 KB. **No TUI**, no terminal interaction, no subagents, no skills, no MCP (yet). The only way in or out is ACP (JSON-RPC over stdio).

Its design intent, per OpenAB maintainer Pahud's Discord:

> *"lead bot 我會繼續用 Kiro，但其他 blot fleet 僅僅需要執行或 review PR，用 native agent 即可，速度絕倫"*

Translation: "I keep using Kiro/ggcoder for the lead bot. But the rest of the fleet, which only needs to execute or review PRs, the native agent is enough — speed is unmatched."

**The 28× memory difference between ggcoder and openab-agent isn't optimization — it's product category.** openab-agent runs in containers, dozens in parallel, killed when done. ggcoder runs in a developer's terminal, one or two at a time.

**You pick openab-agent when:** you're building infrastructure that orchestrates many concurrent agents (Discord/Slack bot, CI runner, scheduled tasks). You don't want a human-facing UI; you want an ACP-speaking subprocess that does narrow tasks fast.

## Pi `handoff.ts` vs Claude Code `Task` — same name, different design

Both get called "subagent," but they solve different problems. Worth understanding because the design choices contrast nicely.

### Pi `handoff` (extension, 191 lines)

```
[User runs /handoff <goal>]
  → Current conversation history serialized
  → LLM as one-shot text-compressor (NOT agent loop)
  → Generated context-transfer prompt opens in editor
  → User reviews + edits
  → New session created with that prompt pre-filled
```

| Property | Value |
|---------|-------|
| Trigger | **User** via slash command |
| Concurrency | Serial — one new session at a time |
| LLM role | Compressor, not agent — just text-to-text |
| Human in loop | **Required** — must review prompt |
| Parent fate | Ends or becomes parent reference |
| Solves | Context bloat without lossy compaction |

### Claude Code `Task` / `Agent` tool

```
[Parent LLM during a turn decides to fan out]
  → Calls Agent({prompt, subagent_type}) tool
  → Sub-agent process starts autonomously
  → Sub runs with separate context, separate tool list
  → Returns one final message to parent
  → Parent continues

Many subagents can run in parallel.
```

| Property | Value |
|---------|-------|
| Trigger | **LLM** chooses to invoke |
| Concurrency | Parallel — many at once |
| LLM role | Full agent — tools, multi-turn, etc. |
| Human in loop | Not required (autonomous) |
| Parent fate | Suspends, gets sub's return value, continues |
| Solves | Fan-out + synthesis of independent sub-tasks |

### Why this matters

These are **incompatible mental models** despite sharing the "subagent" label:

- Pi's pattern: *"I (the user) want to restart with focused context."*
- Claude Code's: *"I (the parent agent) want to delegate independent work."*

If you read someone saying "we use subagents in our agent system" without specifying which model, **assume miscommunication** until you see the trigger pattern. The differences in trust model, concurrency, and human-in-loop are real and consequential.

## Footprint as positioning signal

A reflection I wrote elsewhere applies here directly: **when two coding agents have a 10×-100× memory difference, that's not optimization — it's different product categories**.

| Agent | Per-session footprint | Run mode |
|-------|----------------------|----------|
| openab-agent | 7 MB | Spawn → narrow task → exit (often dozens in parallel) |
| Pi | ~200 MB | Long-lived TUI for one human |
| ggcoder | ~200 MB+ | Same |
| Claude Code | Several hundred MB | Same + Anthropic-cloud features |

The test: *"If I ran 100 instances in parallel, which one bankrupts me first?"*

If the answer is obvious, the two are **co-existing categories**, not substitutes.

## Which one should you use?

| Situation | Choice |
|-----------|--------|
| Daily coding work, one terminal, want it to just work | **ggcoder** or Claude Code |
| Want to build a custom agent product, will write extensions | **Pi** |
| Building a fleet of bots (Discord, Slack, CI) that need lightweight ACP workers | **openab-agent** |
| Want a research toy / minimal reference | Pi or openab-agent |
| Need MCP + skills + voice + video + everything | **ggcoder** (most feature-complete OSS option today) |
| Need to ship to a non-technical user | Probably none of these yet — they all assume CLI comfort |

## See also

- **[architecture.md](./architecture.md)** — ggcoder's package layout & UI re-architecture
- **[providers.md](./providers.md)** — ggcoder's LLM provider system
- [openabdev/openab](https://github.com/openabdev/openab) — openab-agent source (single Rust binary)
- [earendil-works/pi](https://github.com/earendil-works/pi) — Pi source + examples/extensions/
