---
title: "Archon vs openab vs gg-framework — Three-Project Deep Comparison (2026-06)"
description: "Three-layer AI-coding stack compared: Archon (workflow engine, process layer), openab (ACP transport + Discord bridge), gg-framework (agent layer). Maintainership models, sprint themes, response to Claude Code Workflows, six-month staying-power scoring."
---

# Archon vs openab vs gg-framework — three-project comparison

Observation window: 2026-06-04. Sources: public GitHub APIs (commits, PRs, issues,
releases), upstream repo trees, public statements from the maintainers.

This document zooms out from the earlier
[vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md) comparison (which kept the
focus on coding agents themselves) and adds a third dimension: **process-level
orchestration**, represented by [coleam00/Archon](https://github.com/coleam00/Archon).

## TL;DR — one line each

| Project | One-line definition |
|---------|---------------------|
| **coleam00/Archon** | YAML-DAG workflow engine that lets a team version-control AI-coding pipelines and trigger them from Slack/GitHub/Telegram |
| **openabdev/openab** | Three-layer stack: an emerging **multi-agent cognitive protocol (ACAS)** at the top, multi-platform chat gateways (Discord / LINE / Telegram / WeCom / Google Chat / Teams / Slack) in the middle, **AWS-native ECS Control Plane + k8s Operator** at the bottom |
| **KenKaiii/gg-framework** | TypeScript monorepo of 14 packages, 9 LLM providers, powering the ggcoder CLI and gg-boss |

They all sit near "AI coding agents" but solve **non-overlapping** problems:

- Archon — **process layer** (which steps run in what order)
- openab — **three sub-layers**: cognitive protocol (ACAS) → multi-platform chat gateway → cloud-native deployment
- gg-framework — **agent layer** (the agent itself, plus providers and tools)

Understanding that the three are stacked, not substitutable, is the prerequisite
for any honest comparison.

> **Revision (2026-06-05):** an earlier version of this document framed openab as
> simply a "transport layer" for ACP. That description was incomplete. Reading
> openab's source surfaced an in-flight **Agent Cognitive Architecture
> Specification (ACAS)** ADR plus a fully-coded **ECS Control Plane** (`oabctl`).
> Sections 3, 7, and 10 have been updated to reflect the three-sub-layer reality.

## 1. Quick facts

| Metric | Archon | openab | gg-framework |
|--------|-------:|-------:|-------------:|
| Stars | **22,150** | 551 | 22 |
| Forks | 3,344 | 141 | 20 |
| Star:fork ratio | 6.6 : 1 | **3.9 : 1** | 1.1 : 1 |
| Primary language | TypeScript | **Rust** | TypeScript |
| Repo size | 19 MB | 15 MB | 5 MB |
| Created | 2025-02-07 | 2026-04-03 | 2026-03-04 |
| Project age | ~16 months | ~2 months | ~3 months |
| Open issues | 191 | 27 | 1 |
| Closed issues | 550 | 281 | 5 |
| **Issue backlog ratio** | 26% | **9.6%** | 17% |
| Open PRs | 103 | 13 | 14 |
| Recent release cadence | ~weekly | multi/day | 13/day (npm) |
| License | MIT | MIT | MIT |

Two ratios worth pausing on:

**Star-to-fork ratio.** Lower means more "people fork to use" relative to "people
star to bookmark." openab at 3.9:1 implies users are forking to deploy their own
Discord bot — this is consistent with the project being a deployment-ready
template, not a library. gg-framework at 1.1:1 is a tiny but committed audience.

**Issue backlog ratio** (open / total). openab's 9.6% means triage is unusually
disciplined — Pahud closes issues fast. Archon's 26% reflects a long tail of
community reports accumulating as the project grows past its founder's bandwidth.

## 2. Maintainership models

The three projects show three completely different "who runs this?" patterns.

### Archon — team with founder handoff in progress

PR authors over the last 28 days:

| Author | PRs | Notes |
|--------|----:|-------|
| Wirasm | 44 | Rasmus Widing — de facto lead maintainer |
| coleam00 | 21 | Cole Medin — founder (YouTube creator) |
| kagura-agent | 14 | bot |
| truffle-dev | 8 | community |
| bluedevilcollectibles | 7 | community |

Of the most recent 20 merged PRs, **all 20 were merged by Wirasm**, none by Cole.
Cole's commit count dropped from a peak of 41/week in early April to 1–4/week
through May. This is an intentional founder → lead-maintainer handoff that took
place over roughly four weeks, predating the Claude Code Workflows announcement
(2026-05-28) by several weeks.

### openab — solo founder + self-hosted agent fleet

Commit authors over the last 14 days:

| Author | Commits | Notes |
|--------|--------:|-------|
| thepagent | 29 | Pahud Hsieh — GitHub handle |
| Pahud Hsieh | 29 | Pahud Hsieh — commit signoff |
| chaodu-agent | 15 | the project's own AI agent (self-hosting) |
| 超渡法師 | 12 | Pahud Hsieh — Chinese alias |
| openab-app[bot] | 8 | release bot |
| chaodu-agent[bot] | 3 | bot variant |

Combining the three Pahud identities, Pahud personally authored ~70 commits in
the last two weeks. The `chaodu-agent` and bots account for ~30. **The project
literally uses itself to develop itself** — an openab instance opens PRs against
the openab repo. This is OpenAB's main narrative and the strongest evidence for
the platform's self-hosting story.

### gg-framework — pure solo

Commit authors over the last 14 days:

| Author | Commits |
|--------|--------:|
| kenkaiii | 100 |
| anyone else | 0 |

100% solo. All commits go direct to `main`. Community PRs sit open (14 currently
open, vast majority unmerged). Ken Kai stated explicitly in his 2026-05-25 Skool
demo that the project does not do peer review — "we move fast, we don't review,
we ship." This is single-maintainer indie velocity, not collaborative OSS.

### Risk comparison

| Risk axis | Archon | openab | gg-framework |
|-----------|--------|--------|--------------|
| Single point of failure | Medium (Wirasm) | High (Pahud) | Extreme (Ken Kai) |
| Founder-coder gap | Wide and intentional | None — Pahud writes code daily | None — Ken Kai writes code daily |
| Bus factor | ~2 plus community | ~1 plus agent fleet | 1 |

## 3. Architecture philosophy

### Archon — "process is the moat"

- YAML DAGs in `.archon/workflows/` capture team processes
- 20 bundled workflows ship out of the box (`archon-fix-github-issue`,
  `archon-piv-loop`, `archon-comprehensive-pr-review`, …)
- Triggers: CLI, Web UI, Slack, Telegram, GitHub webhooks
- Each workflow run gets its own `git worktree` for isolation
- Node types: `bash:`, `prompt:`, `loop: { until: … }`, `interactive: true`

Core claim: LLM output is non-deterministic, but the process around it can be.
Encoding the process as YAML in the repo means every run executes the same
steps in the same order regardless of which engineer triggers it.

### openab — protocol + transport + control plane, all three at once

The repo's top-level reveals the scope:

```
Dockerfile.claude         Dockerfile.codex          Dockerfile.copilot
Dockerfile.cursor         Dockerfile.gemini         Dockerfile.grok
Dockerfile.hermes         Dockerfile.native         Dockerfile.opencode
Dockerfile.pi             Dockerfile.antigravity
gateway/                  agy-acp/                  openab-agent/
operator/                 openab-auth-proxy/        openshell/
charts/                   k8s/                      docs/
src/
```

This is more than a Rust binary. Reading the source surfaces **three distinct
sub-layers**, each a different bet on where the moat sits.

**Sub-layer 1 — Agent Cognitive Architecture Specification (ACAS).** In
`adr-001-agent-cognitive-architecture` (proposed 2026-04-23 by Pahud + the
`chaodu-agent` AI reviewer fleet), openab drafts what it calls "a generic,
platform-agnostic specification that any agent implementation can follow." The
spec defines three pillars — Self-Identity (YAML-defined `name`, `uid`, `persona`,
`capabilities`, `boundaries`), Social Awareness (Peer Registry + Discovery
Handshake), and Knowledge (`/recall`, `/remember`, `/reflect` commands backed by
SQLite). It declares three conformance levels (Level 1 = identity + recall;
Level 2 = full knowledge; Level 3 = shared knowledge across peers). The framing
is explicit: *"OpenAB is a multi-bot, agent-agnostic, vendor-agnostic platform.
It bridges multiple coding CLIs... where multiple bots/agents coexist in the
same chatroom."*

This is openab's highest-stakes bet. ACAS is positioned as a **standard above
ACP** — where ACP solves "how one editor talks to one agent," ACAS attempts
"how many agents coexist in one room with shared identity, memory, and social
graph." No other project in this report attempts this layer. Whether ACAS gets
adopted by any external implementation determines whether openab becomes a
protocol body or remains a tool.

**Sub-layer 2 — Multi-platform chat gateway.** The `gateway/` subtree handles
Discord, LINE, Telegram, WeCom (企业微信), Google Chat, Teams, and Slack (planned).
Recent sprint PRs add support for inbound LINE images (#991), WeCom adapter
(#769), Google Chat attachments (#762), and a generic `[[reply_to]]` directive
across platforms (#783). The pattern is breadth-first: openab keeps adding chat
surfaces, betting that the next user is always on a chat platform you haven't
adapted yet. WeCom in particular signals an explicit move into the
Chinese-enterprise segment that Microsoft Teams cannot enter.

**Sub-layer 3 — AWS-native control plane.** In `operator/`, a CLI called
`oabctl` provisions OpenAB agents directly on **Amazon ECS Fargate** through a
YAML-manifest + reconciler pattern that mirrors a Kubernetes Operator but runs
on ECS. The ADR (`docs/adr/ecs-control-plane.md`, 2026-05-18) calls it
*"GitOps for ECS — pushing a YAML change triggers the controller to converge
the cluster to the new desired state."* This is Pahud Hsieh's AWS Container
expertise translated directly into product. openab now offers two deployment
paths: Helm chart on k8s (existing) **and** native ECS Fargate via `oabctl`
(new). Plus `openshell/` ships a sandboxed container image
(`ghcr.io/openabdev/openab-native-sandbox`) for safer agent execution.

Core claim, restated: all major coding CLIs converge on ACP. openab places three
bets on top of that convergence — a protocol for multi-agent coexistence, a
gateway that follows users to whichever chat platform they already use, and a
deployment model that meets enterprises where their infrastructure already
lives.

### gg-framework — "the agent IS the product"

- 14 packages in a pnpm monorepo
- 9 LLM providers in `gg-ai` (the description's "Four providers" is outdated)
- TUI in Ink + React, 16 built-in tools
- Goal system + plan mode + checkpointing + skills + MCP
- No external transport — runs as a personal CLI on the developer's machine

Core claim: a well-designed agent loop with hand-rolled provider HTTP clients
beats heavy abstractions and unnecessary orchestration.

### Where each puts AI in the system

| Project | AI's role | User-facing surface |
|---------|-----------|--------------------:|
| Archon | a node in a pipeline | YAML / Web UI / Slack bot |
| openab | a deployable unit | Discord channels / LINE groups |
| gg-framework | the CLI itself | terminal TUI |

## 4. Recent sprint themes (2026-05-21 → 2026-06-04)

### Archon — workflow engine hardening + multi-tenant auth

Highlight PRs:

| PR | Description |
|----|-------------|
| #1790 | feat(workflows): persist per-node provider sessions across re-runs |
| #1812 | fix(workflows): fail dag node on idle-timeout with zero output |
| #1830 | fix(core): concurrency-safe workflow resume/cancel (CAS guards) |
| #1799 | fix(workflows): stop silently dropping workflow-level effort/thinking/fallbackModel/betas/sandbox in loader |
| #1791 | **Release 0.4.0** (2026-05-28) |
| #1793 | Release 0.4.1 (2026-05-28) |
| #1788 | feat(core): GitHub App multi-installation routing |
| #1841 | feat(auth): opt-in web login via Better Auth + user-identity seam |

Theme: harden the workflow engine, add per-user auth surfaces.

### openab — agy-acp integration sprint

Commits per day:

```
2026-05-31  17 commits
2026-06-01  15 commits
2026-06-02  23 commits   ← openab-0.8.4 release
2026-06-03  30 commits   ← openab-0.8.5-beta.1 release
```

Of 85 commits over those four days, the majority sit under `agy-acp/`. Sample
messages:

- `test(agy-acp): multi-turn, session continuity, error path e2e tests`
- `fix(ci): use ~/.gemini/.env for GEMINI_API_KEY (per official docs)`
- `chore(agy-acp): remove deprecated auth seed script`
- `docs(agy-acp): add README with build, test, and e2e instructions`

Theme: bring Gemini Code Assist (Google's subscription-OAuth CLI) into the ACP
fleet by wrapping it.

### gg-framework — TUI polish + provider abstraction + plan-mode iteration

Notable commits 2026-05-31 → 2026-06-03:

- `Refactor gg-ai providers to share json, SSE, usage, error, and schema helpers`
- `Add provider video input and hard billing/quota error handling, plus prompt-bench`
- `Fix plan-step tracking with heading synonyms and shimmer the Plan Steps activity`
- `Add per-run vital-signs status line and fix gg-boss CLI version resolution`
- `Update GG banner art across all commands with shared logo renderer`
- `Update package versions to 4.3.243`

Theme: continue the patterns already documented in
[architecture.md](./architecture.md) and [providers.md](./providers.md) — UI
polish, provider unification, plan-mode iteration.

## 5. Response to Claude Code Workflows (2026-05-28)

Anthropic shipped "dynamic workflows" in Claude Code on 2026-05-28. Each project
reacted differently:

| Project | 2026-05-28 activity | Interpretation |
|---------|--------------------|----------------|
| **Archon** | Released v0.4.0 *and* v0.4.1 the same day, 18 commits (highest in the 14-day window), multiple workflow-related PRs | Direct response — same-day release timing reads as deliberate |
| **openab** | 9 commits, none workflow-related; continued agy-acp sprint | Ignored — different layer of the stack |
| **gg-framework** | 13 commits; continued video input and provider refactor work | Ignored — different layer of the stack |

Only Archon competes head-on with Claude Code Workflows. openab and gg-framework
sit one layer away (transport / agent) and are not directly threatened.

## 6. Six-month staying-power assessment

Subjective scoring on a 0–100 scale for "still being actively developed and
adopted in six months":

| Risk axis | Archon | openab | gg-framework |
|-----------|:------:|:------:|:------------:|
| Single-maintainer risk | Medium | High | Extreme |
| Direct Anthropic competition | **High** (Workflows ships in Claude Code) | Low (ACP transport not on Anthropic's roadmap) | Low (it's a client of Anthropic) |
| Business model clarity | Unclear | Self-hosting (Discord deploys) | Unclear |
| Community contribution acceptance | High | Moderate (strict CI bar) | Low |
| **Estimated six-month probability** | ~70% | ~80% | ~60% |

Why openab scores highest despite the smallest team: it sits in the transport
layer. As more coding CLIs add ACP support, openab's value grows passively. The
problem it solves ("how do I run multiple coding agents inside Discord and LINE
with k8s deployments") is not on Anthropic's roadmap and is unlikely to be
absorbed.

Why gg-framework scores lowest: Ken Kai's personal productivity is exceptional,
but the project's differentiation collapses if Anthropic ships the features ggcoder
currently leads on (video, subscription OAuth for ChatGPT Plus, plan mode). Most
of those are already in Claude Code in some form. What remains unique — running
non-Anthropic providers like Kimi/MiniMax/GLM from the same CLI — appeals to
power users but not to a broad audience.

## 7. The three-layer stack, visualized

```
        ┌────────────────────────────────────────────────┐
        │  Archon                  process layer         │
        │  (workflow, DAG, multi-platform triggers)      │
        └────────────────────────┬───────────────────────┘
                                 │ orchestrates
                                 ▼
        ┌────────────────────────────────────────────────┐
        │  ggcoder / Claude Code / Codex / Pi / Cursor   │
        │  (agent loop, providers, tools, TUI)           │
        │                                                │
        │  ← gg-framework lives here                     │
        └────────────────────────┬───────────────────────┘
                                 │ ACP
                                 ▼
        ┌────────────────────────────────────────────────┐
        │  openab — three sub-layers                     │
        │                                                │
        │   ┌──────────────────────────────────────────┐ │
        │   │  ACAS (Agent Cognitive Architecture)     │ │
        │   │  identity / social / knowledge spec      │ │
        │   └──────────────────┬───────────────────────┘ │
        │                      │                         │
        │   ┌──────────────────▼───────────────────────┐ │
        │   │  Multi-platform chat gateway             │ │
        │   │  Discord / LINE / Telegram / WeCom /     │ │
        │   │  Google Chat / Teams / Slack             │ │
        │   └──────────────────┬───────────────────────┘ │
        │                      │                         │
        │   ┌──────────────────▼───────────────────────┐ │
        │   │  Cloud-native control plane              │ │
        │   │  oabctl → ECS Fargate                    │ │
        │   │  Helm chart → k8s                        │ │
        │   │  OpenShell sandbox container             │ │
        │   └──────────────────────────────────────────┘ │
        └────────────────────────────────────────────────┘
```

The openab "transport" box that earlier versions of this document drew as a
single tier is actually three. ACAS at the top is the bet most likely to be
adopted externally and most likely to fail; the chat gateway in the middle is
the part already in production today; the AWS-native control plane at the
bottom is Pahud Hsieh's domain expertise made tangible.

A team could meaningfully run all three at once: Archon orchestrates a workflow,
each workflow node invokes ggcoder (or Claude Code, or Codex), and the human
endpoint sits in Discord channels backed by openab. Nothing about that stack is
contradictory.

## 8. Picking one — situation matrix

| You want to… | Pick |
|--------------|------|
| Run repeatable PR review / fix pipelines across a team | **Archon** |
| Deploy a coding-agent bot into Slack or Discord channels | **openab** |
| Hand a developer a CLI that supports many LLM providers | **gg-framework (ggcoder)** |
| Study a clean agent-loop implementation | **gg-framework** (smallest, focused) |
| Study multi-agent orchestration | **Archon** (plus Pi's `examples/extensions/`) |
| Study a production ACP implementation in Rust | **openab** |
| Build a single source of process truth that lives in your git repo | **Archon** |
| Avoid sitting in Anthropic's competitive crosshairs | **openab** |

## 9. Founder profiles — why they decide each project's fate

| | Cole Medin (Archon) | Pahud Hsieh (openab) | Ken Kai (gg-framework) |
|-|---|---|---|
| Background | YouTube AI content creator | AWS Container expert | Independent dev + Skool community |
| Audience source | Several hundred thousand YouTube subscribers | Engineer networks + Discord | Skool group (not GitHub) |
| Star quality | Mixed — flow from the channel | Engineering peers, signal-dense | True fans, small but loyal |
| Day-to-day code involvement | Diminishing | Maximum | Maximum |
| Likely five-year outcome | Hinges on commercialization | Hinges on ACP ecosystem adoption | Hinges on Ken Kai personally |

Each founder type produces a project shape that matches their distribution
strategy. Cole's mass-audience channel translates into raw stars; Pahud's
engineer-network translates into signal-dense forks and self-hosted deployments;
Ken Kai's Skool community translates into a tight, opinionated user base.

## 10. Takeaways

1. The three projects **are not substitutes** — they sit at three different
   layers of the AI coding stack.
2. The only one in **direct competition with Claude Code Workflows** is Archon,
   and it is hardening its differentiators (multi-platform triggers,
   determinism, isolated worktrees per run).
3. **openab is the most under-priced** of the three, and the case is stronger
   than "Rust ACP transport." The project is making three simultaneous bets:
   (a) **ACAS**, a draft specification for multi-agent coexistence — identity,
   social registry, shared knowledge — positioned one layer above ACP, which no
   other project in this report attempts; (b) a chat gateway that keeps adding
   platforms (Discord, LINE, Telegram, WeCom, Google Chat, Teams, Slack
   planned) on the bet that every new chat surface is another audience; (c) an
   **ECS Control Plane** (`oabctl` CLI + reconciler) that brings AWS-native
   deployment to the project alongside the existing k8s path. Any one of these
   three bets, if it lands, makes openab structurally important. If ACAS in
   particular gets external adoption, openab transitions from "tool" to
   "protocol body" — a category change.
4. **gg-framework is the most pure single-developer artifact** — fascinating to
   study, risky to depend on, unlikely to scale beyond Ken Kai.
5. The single biggest external risk surfaced between this document's first
   draft and the 2026-06-05 revision is **Devin Desktop** (the rebranded
   Windsurf, now owned by Cognition) adding official **ACP support** with
   bundled Codex / Claude Agent / Gemini CLI / OpenCode / Junie integrations
   via a `~/.windsurf/acp/registry.json` registry. That move occupies the
   IDE-side ACP fleet niche commercially. openab survives this only because
   Devin Desktop targets a single user at a desktop, while openab targets
   teams in chat — the customer is different even when the protocol is the
   same. The two products co-exist if openab keeps its chat + multi-agent
   advantage; the question for openab is whether it can ship ACAS to external
   adopters before Cognition decides to push Devin Desktop into team-chat
   surfaces.
6. A betting order on five-year staying power: **openab > Archon >
   gg-framework**. Same ranking as the first draft, different reasons. openab's
   case used to be "transport layer accrues value passively"; the case now is
   "three simultaneous bets, one of which is protocol-level, all driven by a
   founder with deep AWS infrastructure background."

## See also

- [architecture.md](./architecture.md) — gg-framework code structure
- [providers.md](./providers.md) — gg-framework LLM provider details
- [vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md) — narrower comparison
  (Pi vs ggcoder vs openab-agent, agent layer only)
- [engineering-notes.md](./engineering-notes.md) — Ken Kai's velocity and
  prompting principles
- [coleam00/Archon](https://github.com/coleam00/Archon) — Archon repo
- [openabdev/openab](https://github.com/openabdev/openab) — openab repo
- [KenKaiii/gg-framework](https://github.com/KenKaiii/gg-framework) — gg-framework repo
