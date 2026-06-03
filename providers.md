# Provider system & video input

Window observed: 2026-06-02 → 2026-06-03 (after the video-input commit `bd5d60b`).

## 9 LLM providers, no SDK dependencies

`packages/gg-ai/src/providers/` ships direct HTTP clients for these 9 providers. There are **no `@anthropic-ai/sdk`, `openai`, `@google/genai` SDK dependencies in production code** — Ken Kai writes the API calls directly with `fetch` and SSE parsers.

| `--provider` flag | Backend | Auth modes supported |
|-------------------|---------|---------------------|
| `anthropic` | `api.anthropic.com` (API) + `claude.ai/v1/chat/completions` (subscription) | API key + OAuth (Claude Pro/Max) |
| `openai` | `api.openai.com/v1` | API key |
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
- The `anthropic` provider uses these tokens against Anthropic's subscription-bound endpoint when present, otherwise falls back to API key.

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
