# Engineering notes — Ken Kai's style + the gg-framework operating pattern

Observations on the engineering pattern behind gg-framework, not the code itself. Useful if you're deciding whether to depend on ggcoder, fork it, or just observe.

## Velocity

Sample window: 2026-05-25 → 2026-06-03 (10 days).

| Metric | Value |
|--------|-------|
| Total commits | ~75 |
| Releases (npm) | ~12 (4.3.217 → 4.3.243) |
| Lines churned | ~25,000 |
| New source files added | 70+ |
| Lines removed | ~9,000 |
| Net repo growth | +16,000 lines |
| Distinct themes worked on | UI re-arch, Goal system, plan mode, video input, quota classification, gg-voice fixes, exit summary, screenshot, checkpoint/rewind, web-fetch/web-search rewrites, ideal-review hook |
| Primary committer | crynta / Ken Kai (~95%+) |
| Distinct outside contributors in window | ~0-1 |

That's ~7-8 commits per day on average, with multiple npm releases per week. **This is single-maintainer indie-product velocity, not collaborative open-source velocity.**

## Code review pattern

There essentially isn't one for community contributions. Ken Kai works alone, commits direct to main, releases when he feels it's stable. The CI on the repo runs tests but isn't a contributor-review gate.

This is fine for the current scale — but it shapes what "stable" means. **A 4.3.x release is not a peer-reviewed milestone; it's a "Ken thinks this is ok" snapshot**, often shipped same-day as the work.

## Trade-off: what you get vs. what you accept

**What you get:**
- Features ship in days, not quarters (plan mode, exit summary, video — all within 7 days of conception)
- Bugs you report often get fixed within 24h
- Direct line to the maintainer on Discord
- No corporate roadmap politics

**What you accept:**
- Features get removed almost as fast as they're added (see "Things that got removed" below)
- API stability is best-effort, not contractual
- Single point of failure — if Ken Kai stops, the project stops
- Code review is essentially "Ken self-reviews"
- Documentation lags implementation (commit messages are often the only docs for new features)

## Things that got added then removed in the last 30 days

Highlights of feature churn — illustrating both the upside (fast iteration) and the risk (don't depend on what just landed).

### Dynamic repo map — added ~mid-May, removed 2026-05-27 (commit `4122092`)

```
Remove dynamic repo map support
+25  / -2,569 lines
```

Stayed in the codebase for **~10 days**. Preceding commits showed "Fix repo map hangs" type fixes. The replacement is a static workspace info block in the system prompt. **Reason not stated in commit message**; probable performance/context-budget issue.

### Sequential subagents (default behaviour) — fixed to parallel in `cfa773e` (2026-05-29, 6 lines)

```
Fix subagent tool to run in parallel instead of sequential
+6 / -1 lines
```

A six-line change with potentially significant performance impact. Easy to miss; bumps the version automatically.

### Codex thinking-off vs reasoning leaks — multiple iterations

Several commits in the May 19 → June 2 window iterate on how to suppress reasoning blocks from Codex output when thinking is disabled. The protocol changed several times during the period.

### gg-voice — landed 2026-05-19, stabilized over a week

```
41327da  Add gg-voice package and repo map focus context (2026-05-19)
2d3cc53  Fix gg-voice lockfile (2026-05-19)
```

Adding a new package mid-monorepo is itself a velocity signal. The package is "Provider-agnostic realtime voice orchestration" — its README is one of the cleaner docs in the repo.

## What does NOT churn (the stable core)

Despite the rapid feature churn, these have been stable across the window:

- The 4-tool baseline: `read`, `write`, `edit`, `bash`
- The agent-loop core in `gg-agent/src/agent-loop.ts` (gets edited but the shape stays)
- ACP / RPC / JSON / Print modes (the four non-interactive surfaces)
- Provider abstraction layer (the contract per provider is stable; what changes is which providers are added)
- Slash commands `/help`, `/model`, `/quit`, `/compact`, `/session`, `/new`, `/settings`

If you're building something dependent on ggcoder, **build against the stable surface** (the 4 tools + agent loop contract + RPC mode), **not against the latest feature** (Goal system, ideal-review hook, etc., are likely to evolve).

## Ken Kai's prompting principles (from Skool community video, 2026-05-25)

In a 126-minute live demo, Ken Kai articulated several principles that show up in ggcoder's design:

1. **Imperfect English is fine.** *"LLMs are very capable. It doesn't have to be perfect English, doesn't have to be like human-to-human."*
2. **Don't over-optimize PRDs.** *"Some people are stuck on the first thing for the next day, just optimizing a PRD file. We could have jumped into it right now."*
3. **Don't justify yourself to the LLM.** *"Justify themselves to the LLM and it's just noise. You only need to command it."*
4. **CLAUDE.md / AGENTS.md updates continuously.** Add `start the dev server on the device, device ID is X` when the agent loves to use the iPhone simulator.
5. **Reasoning tokens aren't for you.** *"OpenAI removes filler words in reasoning because it's not for you to read. It's reasoning with itself."*

These match the design of his system prompts in `packages/ggcoder/src/system-prompt.ts` — short, command-style, no filler. Compare to Claude Code's longer system prompts.

### Standout principle: "Simulate at scale, don't iterate manually"

Ken Kai's most striking story (from same video):

> *"Took over a developer team — 5 guys, 6 months optimizing prompts manually. What do I do? Simulate it. Thousands of iterations, $50 in an hour, rubric scoring system. Client over the moon — 'My God, what have I been doing for 10 months?'"*

The takeaway: **for prompt optimization, write a simulator + rubric, run many iterations cheap. Don't do it by hand.** This shows up in `experiments/prompt-bench/` (added in commit `bd5d60b`, 2026-06-02) — a small harness for batch-running prompts against multiple models with comparison.

## The Goal system as a strategic bet

If you read the commit log post-2026-05-20 the central effort is **Goal**. 17 of 34 commits in that window touch it. It's the new headline feature.

**My read:** Goal is Ken Kai's bet on multi-step, worktree-isolated, evidence-gathering workflows — the kind of thing you'd want for autonomous PR generation, multi-file refactors, or "implement this spec from start to finish" tasks.

It's structurally similar to OpenAB's bot-orchestration model but lives inside a single ggcoder process (vs. OpenAB which spawns multiple agent processes).

**Risk if you're depending on Goal:** the API surface is still moving. Several commits per week tweak orchestration semantics, evidence flow, worktree merge strategy. Don't ship a Goal-dependent product until 4.4 or later.

## What watching this project teaches you

If you're reading the commit log of gg-framework regularly (which is a lot), you're observing:

1. **A senior dev's iteration loop in public.** Many of the small commits (e.g. the 6-line subagent parallelism fix) are micro-optimizations he found while using his own tool. Worth studying.

2. **An interesting language choice debate, lived out.** Ken Kai writes ggcoder in TypeScript but writes openab-agent in Rust. The contrast is visible per-commit. The decision criteria are real (footprint, distribution, performance) — see [vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md).

3. **A bet on "many providers, no SDKs."** Every provider integration in gg-ai is hand-written HTTP. This is unusual — most projects use the official SDK. The trade-off: more maintenance, more control, smaller dependency tree. Worth observing whether the bet pays off as providers evolve their APIs.

## Should you build on top?

| If you want to... | Recommendation |
|-------------------|----------------|
| Use ggcoder as your daily CLI | **Yes.** Install, use, expect occasional breakage; you can pin a version. |
| Write a ggcoder extension / plugin | **Wait.** Extension API isn't really documented; reverse-engineer at your own risk. |
| Wrap ggcoder programmatically | Use `--rpc` mode + the JSON-RPC contract. Reasonably stable. |
| Fork ggcoder for your product | **Yes, but pin a SHA**, don't track `main`. |
| Vendor pieces of `gg-ai/` (the provider abstraction) | **Yes, with attribution.** This is one of the cleanest provider-abstraction layers in OSS today. Copying it is legitimate. |
| Depend on Goal system in a customer-facing product | **Not yet.** Wait for 4.4 or later. |

## See also

- **[architecture.md](./architecture.md)** — code structure
- **[providers.md](./providers.md)** — LLM provider details
- **[vs-pi-vs-openab-agent.md](./vs-pi-vs-openab-agent.md)** — positioning
- [The Fathom recording of Ken Kai's 2026-05-25 Skool demo](https://www.skool.com/systems-to-scale-9517) — primary source for prompting principles section
