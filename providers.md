---
title: "ggcoder Provider System — 9 LLMs, Video Input, Subscription OAuth via Identity Spoofing, Quota Classifier"
description: "How KenKaiii/gg-framework wires 9 LLM providers — official Anthropic and OpenAI SDKs as transport, custom OAuth handling for subscription auth, and Claude Code identity spoofing to unlock Claude Pro/Max without API billing. Plus three video-input wire formats and the hard-vs-transient quota classifier."
---

# Provider system & video input

Window observed: 2026-06-02 → 2026-06-03 (after the video-input commit `bd5d60b`).

> **Corrigendum (2026-06-04):** an earlier version of this document claimed gg-ai
> shipped "no SDK dependencies." That was wrong. `packages/gg-ai/package.json`
> depends on `@anthropic-ai/sdk@^0.94.0` and `openai@^6.34.0`. What gg-ai *does*
> bypass is the **auth abstraction** inside those SDKs — see "Subscription OAuth"
> below. This section has been rewritten to reflect what the source actually does.

## 9 LLM providers, two transport strategies

`packages/gg-ai/src/providers/` wires 9 providers, but the transport choice splits in two:

- **Anthropic** and **OpenAI** use the **official vendor SDKs** (`@anthropic-ai/sdk@0.94.0` and `openai@6.34.0`) as the HTTP/SSE transport layer. ggcoder then bypasses the SDKs' auth abstractions to inject OAuth tokens directly.
- **Gemini, Codex, GLM, Moonshot, MiniMax, DeepSeek, OpenRouter** are **hand-rolled** with `fetch` plus a shared SSE / JSON parser in `gg-ai/src/providers/utils/`. None of them have an official TypeScript SDK that fits ggcoder's needs (or in Codex's case, the endpoint is internal and unpublished).

| `--provider` flag | Backend | Transport | Auth modes |
|-------------------|---------|-----------|------------|
| `anthropic` | `api.anthropic.com` | `@anthropic-ai/sdk` | API key + **OAuth (Claude Pro/Max) via Claude Code identity spoof** |
| `openai` | `api.openai.com/v1` | `openai` SDK | API key |
| `openai-codex` (via `--provider codex`) | `chatgpt.com/backend-api/codex/responses` | hand-rolled `fetch` | **OAuth subscription** (ChatGPT Plus/Pro/Team) — see [my standalone notes](https://github.com/Joseph19820124/codex-oauth-client) |
| `gemini` | `cloudcode-pa.googleapis.com/v1internal:streamGenerateContent` (Code Assist) + `generativelanguage.googleapis.com` (AI Studio) | hand-rolled `fetch` | OAuth subscription (Code Assist) + API key (AI Studio) |
| `glm` | Z.AI backend | hand-rolled `fetch` | API key + MCP tools (Z.AI has provider-specific MCP servers) |
| `moonshot` | Moonshot platform (also serves Kimi K2.6) | hand-rolled `fetch` | API key |
| `minimax` | MiniMax M3 — **rides Anthropic transport, see below** | `@anthropic-ai/sdk` (yes, really) | API key |
| `deepseek` | DeepSeek platform | hand-rolled `fetch` | API key |
| `openrouter` | Aggregator | hand-rolled `fetch` | API key (one key → 100+ models) |
| `openai-codex` (via `--provider codex`) | `chatgpt.com/backend-api/codex/responses` | **OAuth subscription** (ChatGPT Plus/Pro/Team) — see [my standalone notes](https://github.com/Joseph19820124/codex-oauth-client) |
| `gemini` | `cloudcode-pa.googleapis.com/v1internal:streamGenerateContent` (Code Assist) + `generativelanguage.googleapis.com` (AI Studio) | OAuth subscription (Code Assist) + API key (AI Studio) |
| `glm` | Z.AI backend | API key + MCP tools (Z.AI has provider-specific MCP servers) |
| `moonshot` | Moonshot platform (also serves Kimi K2.6) | API key |
| `minimax` | MiniMax M3 — **rides Anthropic transport, see below** | API key |
| `deepseek` | DeepSeek platform | API key |
| `openrouter` | Aggregator | API key (one key → 100+ models) |

Plus an emerging `xiaomi` provider (added per current `--help` output but lightly populated).

After commit `bd5635d` (2026-06-01), providers share a common helpers module (`gg-ai/src/providers/utils/` — JSON, SSE, usage, error, schema). Each provider is essentially:

```typescript
async function* runStream(options: StreamOptions): AsyncGenerator<StreamEvent, StreamResponse> {
  const downgradedImages = downgradeUnsupportedImages(options.messages, options.supportsImages);
  const downgradedMessages = downgradeUnsupportedVideos(downgradedImages, options.supportsVideo);
  // ... provider-specific wire format ...
  // ... POST + parse SSE ...
}
```

The contract is the same; the wire format differs.

## Subscription OAuth — both Claude.ai and ChatGPT Plus

Two paths that let a `$20/month` subscriber use Claude Pro or ChatGPT Plus without API billing:

### Anthropic (Claude Pro / Max)

- Auth file at `~/.kenkaiii/auth.json`
- Browser-based PKCE flow (default) or device-code flow (headless servers)
- **Endpoint is still `api.anthropic.com`** — same host as API-key mode. The subscription is unlocked by the auth header and identity, not by hitting a different URL.

The interesting part is what ggcoder does to make `api.anthropic.com` accept the OAuth token. In `gg-ai/src/providers/anthropic.ts`:

```typescript
const isOAuth = options.apiKey?.startsWith("sk-ant-oat");   // OAuth token prefix
return new Anthropic({
  ...(isOAuth
    ? { apiKey: null, authToken: options.apiKey }           // Bearer-style auth
    : { apiKey: options.apiKey }),                          // x-api-key style auth
  ...(isOAuth
    ? {
        defaultHeaders: {
          "user-agent": "claude-cli/2.1.75 (external, cli)",  // ← spoof Claude Code CLI
          "x-app": "cli",
        },
      }
    : {}),
});
```

And two more pieces are mandatory when in OAuth mode:

1. **System prompt prefix** — the first system block must be exactly
   `"You are Claude Code, Anthropic's official CLI for Claude."`. ggcoder
   prepends this when `isOAuth` is true, before any user-supplied system prompt.
2. **Beta header** — the request must declare `claude-code-20250219,oauth-2025-04-20`
   in the `anthropic-beta` list. ggcoder adds these only when `isOAuth`.

Without all three (user-agent, system prompt prefix, beta header) Anthropic's
OAuth edge rejects the call. This is identity spoofing: the request to
`api.anthropic.com` must look indistinguishable from one made by the real
Claude Code CLI binary, because that's the only client Anthropic intends to
accept OAuth tokens from.

**Implication:** this works today and is unsafe to depend on long-term. Anthropic
can tighten any of the three checks (rotate the accepted user-agent string, change
the required beta header, validate the system prompt content) and ggcoder users
lose subscription access until the project ships a new spoof. By contrast, the
official `claude-agent-sdk` route — `Python SDK → spawn claude CLI → api.anthropic.com`
— is the path Anthropic actively supports and won't deliberately break.

See also: [Joseph19820124/claude-sdk-subscription-demo](https://github.com/Joseph19820124/claude-sdk-subscription-demo)
— my Python demo of the **official** subscription path via `claude-agent-sdk`.
That repo and ggcoder's anthropic provider solve the same end goal (use Claude
Pro instead of API billing) by completely different mechanisms.

### OpenAI Codex (ChatGPT Plus / Pro / Team)

- Auth file at `~/.kenkaiii/codex-auth.json`
- Browser-based PKCE flow with the OpenAI Codex CLI's published OAuth client ID
- Uses `chatgpt.com/backend-api/codex/responses` (the internal Codex API) — NOT the public `api.openai.com`
- Same protocol I documented in [codex-oauth-client](https://github.com/Joseph19820124/codex-oauth-client). ggcoder's `gg-ai/src/auth/codex-oauth.ts` and my repo's `src/oauth.ts` solve the same problem in TypeScript independently.

Login flow:

```bash
ggcoder login
# → interactive picker:
#   1. Anthropic (API key)
#   2. Anthropic (Claude Pro/Max subscription)
#   3. OpenAI Codex (ChatGPT Plus subscription)
#   4. Gemini (Code Assist)
#   ...
```

## Video input (added 2026-06-02, commit `bd5d60b`)

### Schema

```typescript
// packages/gg-ai/src/types.ts
export interface VideoContent {
  type: "video";
  mediaType: string;  // e.g. "video/mp4"
  data: string;       // base64-encoded
}
```

User messages now accept `(TextContent | ImageContent | VideoContent)[]`. The `StreamOptions` got a new `supportsVideo?: boolean` flag (defaults to `false`).

### Which models support it

Per `packages/ggcoder/src/core/model-registry.ts`:

| Provider | Model ID | Display name |
|---------|---------|--------------|
| `gemini` | `gemini-3.5-flash` | Gemini 3.5 Flash |
| `gemini` | `gemini-3.1-flash-lite-preview` | Gemini 3.1 Flash Lite Preview |
| `moonshot` | `kimi-k2.6` | Kimi K2.6 |
| `minimax` | `MiniMax-M3` | MiniMax M3 |

**Anthropic Claude models all have `supportsVideo: false`** — no native Claude video as of this commit. **OpenAI GPT/Codex models also `supportsVideo: false`**.

### Three different wire formats per provider

This is the architectural curiosity. Same `VideoContent` type, three serializations:

**1. Gemini (native, same as images)**

```typescript
// gg-ai/src/providers/gemini.ts (toSystemAndContents)
{ inlineData: { mimeType: part.mediaType, data: part.data } }
```

Gemini's API has had multimodal `inlineData` for a while; video just rides the same shape as images.

**2. Kimi / Moonshot (OpenAI-compatible extension)**

```typescript
// gg-ai/src/providers/transform.ts (toOpenAIMessages)
{
  type: "video_url",
  video_url: {
    url: `data:${part.mediaType};base64,${part.data}`,
  },
}
```

Moonshot supports a `video_url` content part — an OpenAI-shape extension that OpenAI itself doesn't have.

**3. MiniMax M3 (rides Anthropic transport!)**

```typescript
// gg-ai/src/providers/transform.ts (toAnthropicMessages)
{
  type: "video",
  source: {
    type: "base64",
    media_type: part.mediaType,
    data: part.data,
  },
}
```

**Code comment from the source**:

> *"MiniMax-M3 rides the Anthropic transport and accepts native video blocks. Non-video models never reach here — video is downgraded to text by downgradeUnsupportedVideos first."*

In other words: MiniMax M3 speaks the Anthropic message-schema dialect (probably for SDK compatibility / shared tooling), but Anthropic's own Claude models don't accept this video block. The Anthropic provider code has a `video` branch that's dead code for actual Claude but live code for MiniMax-via-Anthropic-transport.

### Graceful degradation

If you have a video in your message and switch to a non-video model mid-session, `downgradeUnsupportedVideos()` in `transform.ts` replaces the video block with:

```
(video omitted: model does not support video)
```

…before the message goes on the wire. No crash, no silently-dropped attachment. The placeholder also dedupes consecutive instances.

### What's missing in v1 of video input

- ❌ Video URL input — only inline base64 (so practical limit is ~5-10 MB)
- ❌ Anthropic Claude native video — they don't support it; ggcoder can't fake it
- ❌ Video as tool output — only as user input
- ❌ Multi-video correlation — model sees them as independent parts

## Hard / transient quota classification (same commit, often missed)

Commit `bd5d60b` also added a `classifyOpenAICompatLimit()` function. This is **separate from the video work but very useful**.

```typescript
// gg-ai/src/providers/openai.ts
function classifyOpenAICompatLimit(args: {
  status: number | undefined;
  code: string | undefined;
  type: string | undefined;
  message: string;
}): "hard" | "transient" | null {
  const isHard =
    args.status === 402
    || `${args.code ?? ""} ${args.type ?? ""}`.toLowerCase().includes("insufficient_quota")
    || isHardBillingMessage(args.message);
  if (isHard) return "hard";       // ← DON'T retry; surface immediately
  if (args.status === 429
      || `${args.code ?? ""} ${args.type ?? ""}`.toLowerCase().includes("rate_limit_exceeded")
      || `${args.code ?? ""} ${args.type ?? ""}`.toLowerCase().includes("too_many_requests")) {
    return "transient";              // ← retry with backoff
  }
  return null;
}
```

**Why this matters**: API providers return `429` for both "you hit the per-minute rate limit, wait 20s" and "you exhausted your monthly quota, top up your account." Treating them the same wastes retry budget on the second case and confuses the user with "Rate limited — retrying" when they should see "Your plan is exhausted." This classifier splits them based on:

- **Hard**: HTTP 402, codes like `insufficient_quota`, balance/credit messages
- **Transient**: HTTP 429, codes like `rate_limit_exceeded` / `too_many_requests`

The agent loop in `gg-agent/src/agent-loop.ts` uses this signal to decide whether to retry.

A nearly-identical helper exists for Gemini in `gg-ai/src/providers/gemini.ts` (`parseGeminiQuota`), which checks for the presence of `RetryInfo.retryDelay` in the response body — when present, it's a per-minute throttle; when absent, it's a daily/billing exhaustion.

This is one of the most directly-portable bits of engineering in ggcoder for any other tool building against the same providers.

## Provider-specific MCP tools (GLM only)

`agent-session.ts` has logic to reconnect MCP servers when switching to/from GLM:

```typescript
const glmInvolved = this.provider === "glm" || prevProvider === "glm";
if (this.mcpManager && glmInvolved) {
  // Tear down & reconnect Z.AI-specific MCP tools
}
```

→ GLM ships its own MCP tools (Z.AI native); other providers just use the user's configured MCP servers. The reconnect happens only when GLM is involved on either side — skipping the dispose/reconnect when not needed avoids killing a live stdio child (e.g. `kencode-search`).

## See also

- **[architecture.md](./architecture.md)** — package layout + Goal system + UI architecture
- **[engineering-notes.md](./engineering-notes.md)** — what got added and removed
- [Joseph19820124/codex-oauth-client](https://github.com/Joseph19820124/codex-oauth-client) — my TypeScript port of the ChatGPT subscription OAuth (same protocol ggcoder's auth.rs uses)
- [Joseph19820124/gemini-codeassist-client](https://github.com/Joseph19820124/gemini-codeassist-client) — my TypeScript port of Google Code Assist OAuth
