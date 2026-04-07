# Technology Stack: LM Studio Provider

**Project:** repodocs — LM Studio LLM provider integration
**Milestone:** Add LM Studio as a dedicated provider
**Researched:** 2026-04-07
**Confidence:** HIGH (API details verified from official LM Studio docs)

---

## LM Studio API Specification

### Endpoint and URL

Default base URL: `http://localhost:1234/v1`

LM Studio runs a local HTTP server. The default port is 1234, configurable in the app's Developer page. The `/v1` path prefix is required — all OpenAI-compatible endpoints live under it.

Relevant endpoint for repodocs: `POST /v1/chat/completions`

### OpenAI Compatibility Level

LM Studio implements the OpenAI chat completions API with high fidelity. The request and response wire format is identical to OpenAI's:

**Request** (POST `/v1/chat/completions`):
```json
{
  "model": "model-identifier",
  "messages": [{"role": "user", "content": "..."}],
  "max_tokens": 4096,
  "temperature": 0.7
}
```

**Response** is the standard OpenAI shape with `choices[0].message.content`, `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens`, and `finish_reason`.

The existing `openAIRequest` / `openAIResponse` structs in `internal/llm/openai.go` are fully compatible. No new wire types are needed.

### Authentication Behavior

This is the critical difference from OpenAI:

- **By default: no authentication required.** Requests to localhost succeed without an `Authorization` header.
- **When auth is enabled** (opt-in toggle in LM Studio's Developer settings): requests must include `Authorization: Bearer <token>`.
- **Empty Bearer token causes errors** in some HTTP clients. Sending a dummy non-empty string (e.g., `"lm-studio"`) satisfies strict SDK constructors without failing LM Studio's auth-disabled mode.

**Pattern to follow:** Ollama (`ollama.go`) — no API key stored, no `Authorization` header sent. But LM Studio needs a twist: if an API key is provided (user enabled auth in LM Studio), send the `Authorization: Bearer` header. If no key is provided, omit the header entirely.

This is distinct from both:
- OpenAI: always sends the header, errors without a key
- Ollama: never sends the header

### Differences from Standard OpenAI API

| Aspect | OpenAI | LM Studio |
|--------|--------|-----------|
| Auth header | Required | Optional (omit when not configured) |
| Model names | `gpt-4`, `gpt-4o`, etc. | Local model file identifiers |
| Base URL | `https://api.openai.com/v1` | `http://localhost:1234/v1` |
| Rate limiting | Yes (cloud enforced) | No (local, effectively unlimited) |
| Response structure | Identical | Identical |
| `finish_reason` values | `stop`, `length`, etc. | Same values |
| Usage tokens | Accurate | Present, model-dependent accuracy |

There are no undocumented fields, extra headers, or response schema deviations to account for. The `/v1/chat/completions` contract is identical.

---

## Recommended Implementation Stack

### No New Dependencies

Do not add any third-party LM Studio or OpenAI Go SDK packages. The existing implementation uses raw `net/http` with manual JSON marshaling. This is correct for this codebase — adding `sashabaranov/go-openai` or similar would be over-engineering a ~100-line provider file.

The entire implementation uses only:
- `net/http` (stdlib)
- `encoding/json` (stdlib)
- `context` (stdlib)
- existing domain types from `internal/domain`

### New File: `internal/llm/lmstudio.go`

Structure mirrors `openai.go` with two differences:

1. `Name()` returns `"lmstudio"`
2. Auth header is conditional: only set `Authorization: Bearer` if `apiKey` is non-empty

```go
// Conditional auth — the only behavioral difference from openai.go
if p.apiKey != "" {
    httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
}
```

Reuse `openAIRequest` and `openAIResponse` types directly from `openai.go` (they are in the same `llm` package). No duplication needed.

### Changes to `internal/llm/provider.go`

Two additions:

1. New constant:
```go
DefaultLMStudioBaseURL = "http://localhost:1234/v1"
```

2. In `DefaultBaseURL()` switch: add `"lmstudio"` case returning the constant.

3. In `NewProviderFromConfig()`: add `"lmstudio"` to the no-API-key-required check alongside `"ollama"`:
```go
if cfg.APIKey == "" && cfg.Provider != "ollama" && cfg.Provider != "lmstudio" {
    return nil, domain.ErrLLMMissingAPIKey
}
```

4. In `NewProvider()` switch: add `"lmstudio"` case calling `NewLMStudioProvider`.

### Changes to `internal/config/` (None Required)

`LLMConfig` already has `Provider`, `APIKey`, `BaseURL`, `Model` fields. No struct changes needed. Config validation in `config.go` does not whitelist provider names — any string is accepted, validation happens in `NewProviderFromConfig`. No config changes needed.

---

## What NOT to Do

**Do not fork `openai.go` wholesale.** The only difference is the auth header and the `Name()` return value. Duplicating 200 lines of identical HTTP handling creates maintenance debt. The right approach is a thin `LMStudioProvider` struct that either:
- Embeds or delegates to `OpenAIProvider` with the `Name()` override, or
- Copies only the struct and constructor, reusing `openAIRequest`/`openAIResponse` types, with the single conditional auth line

Given the codebase's explicit per-provider files pattern, a standalone file with reused types is the idiomatic approach here.

**Do not send a dummy API key.** Some guides suggest passing `"lm-studio"` as a placeholder. This is a workaround for SDKs that reject empty strings. Since repodocs constructs HTTP requests manually, it can simply omit the header entirely when no key is configured. Sending a fake key is misleading and would break LM Studio setups where auth is enabled (the fake key would be rejected).

**Do not use the `/api/v1/` native LM Studio endpoints.** LM Studio also exposes a native REST API at `/api/v1/chat`. This requires the LM Studio SDK or different request/response shapes. The OpenAI-compatible `/v1/chat/completions` endpoint is the correct integration target — it's stable, documented, and wire-compatible with existing code.

**Do not add model listing/auto-detection.** LM Studio exposes `GET /v1/models` but the PROJECT.md explicitly excludes model auto-detection from scope. Users specify model names manually.

---

## Test Strategy

Follow `ollama_test.go` exactly:

- `TestNewLMStudioProvider` — construction with/without API key
- `TestLMStudioProvider_Complete_Success` — verify URL path is `/v1/chat/completions`, verify `Authorization` header is absent when no key set
- `TestLMStudioProvider_Complete_WithAPIKey` — verify `Authorization: Bearer token` header is present when key is configured
- `TestLMStudioProvider_Complete_APIError` — error response handling
- `TestLMStudioProvider_Complete_RateLimit` — 429 maps to `ErrLLMRateLimited`
- Standard context cancellation and JSON error tests

The httptest server pattern used throughout `*_test.go` files handles all of this without a live LM Studio instance.

---

## Sources

- [LM Studio OpenAI Compatibility Endpoints](https://lmstudio.ai/docs/developer/openai-compat) — HIGH confidence
- [LM Studio Authentication Docs](https://lmstudio.ai/docs/developer/core/authentication) — HIGH confidence
- [LM Studio API Overview](https://lmstudio.ai/docs/api/openai-api) — HIGH confidence
- [LM Studio OpenAI Compatible Provider (Vercel AI SDK)](https://ai-sdk.dev/providers/openai-compatible-providers/lmstudio) — MEDIUM confidence (third-party but confirms behavior)
