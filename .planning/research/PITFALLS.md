# Domain Pitfalls: LM Studio Provider Integration

**Domain:** Local LLM provider (OpenAI-compatible API)
**Researched:** 2026-04-07
**Codebase context:** repodocs `internal/llm/` package

---

## Critical Pitfalls

Mistakes that cause silent failures, integration breakage, or require rewrites.

---

### Pitfall 1: API Key Validation Gate Blocks LM Studio

**What goes wrong:**
`provider.go` line 53 explicitly guards `ErrLLMMissingAPIKey` for every provider except `"ollama"`:

```go
if cfg.APIKey == "" && cfg.Provider != "ollama" {
    return nil, domain.ErrLLMMissingAPIKey
}
```

If `lmstudio` is added to the switch in `NewProvider` without also adding it to this exemption, users get a hard error requiring an API key — even though LM Studio runs locally with no auth by default.

**Why it happens:** The exemption is an explicit allow-list. New providers are not exempt by default.

**Consequences:** `repodocs` refuses to start metadata enhancement with `provider: lmstudio` unless the user supplies a dummy key, defeating the zero-config UX goal.

**Prevention:** The exemption condition must be updated to allow both `"ollama"` and `"lmstudio"` to pass with an empty API key:

```go
if cfg.APIKey == "" && cfg.Provider != "ollama" && cfg.Provider != "lmstudio" {
```

**Detection:** Unit test `TestNewProviderFromConfig` with `provider: lmstudio` and empty `api_key` must not return `ErrLLMMissingAPIKey`.

**Phase:** Implementation — day 1 before any other work.

---

### Pitfall 2: Authorization Header Sent with Empty Bearer Token

**What goes wrong:**
`openai.go` unconditionally sets the Authorization header:

```go
httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
```

When `apiKey` is empty string this sends `Authorization: Bearer ` (with a trailing space and no token). The OpenAI Go SDK and some proxies validate this header client-side and raise an error before the request reaches LM Studio. The `openai-python` library has a filed issue for exactly this behaviour (openai/openai-python#961).

**Why it happens:** The OpenAI provider assumes a key is always present. LM Studio is the first consumer of this code path where the key is legitimately absent.

**Consequences:** Requests fail with an SDK or HTTP validation error rather than reaching LM Studio. Error message is confusing ("missing API key") not "LM Studio unavailable."

**Prevention:** The LM Studio provider implementation must conditionally set the Authorization header only when `apiKey != ""`:

```go
if p.apiKey != "" {
    httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
}
```

This means `lmstudio.go` cannot simply delegate to `NewOpenAIProvider` — it needs its own `Complete` method (or the OpenAI provider needs a conditional header path).

**Detection:** Integration test against a mock HTTP server that asserts the Authorization header is absent when no key is configured.

**Phase:** Implementation — part of the provider struct definition.

---

### Pitfall 3: Default HTTP Timeout (60s) Too Short for Local Model Cold Start

**What goes wrong:**
`provider.go` sets a 60-second default HTTP timeout:

```go
if timeout == 0 {
    timeout = 60 * time.Second
}
```

LM Studio's Just-In-Time (JIT) loading feature loads models on first inference request. A 7B+ model on CPU can take 30–120+ seconds to load. The first request after a long idle period (or after LM Studio's TTL-based auto-eviction unloads the model) silently times out with a generic network error, not a helpful "model is loading" message.

Multiple integrations have filed this as a bug (lmstudio-ai/lmstudio-bug-tracker#944, Kilo-Org/kilocode#1681).

**Why it happens:** Cloud providers (OpenAI, Anthropic) respond in milliseconds. Local model cold start is fundamentally different.

**Consequences:** Users see a cryptic timeout error. The circuit breaker records failures and eventually opens, blocking all subsequent requests — even after the model finishes loading. This is particularly bad because `retry.go`'s `IsRetryableError` considers `url.Error` timeouts retryable, so all 3 retries will consume the full 60s each = 3+ minutes of hang time before surfacing a failure.

**Prevention:**
- LM Studio provider should default to a much longer timeout (e.g. 300s) or make it configurable independently of the global `timeout` config field.
- Document in config that `lmstudio` benefits from a longer timeout setting.
- Consider a descriptive error message: "lmstudio: request timed out — model may still be loading. Consider increasing timeout or pre-loading the model in LM Studio."

**Detection:** Warning sign: consecutive timeout errors on first use. Test with a slow mock server (100ms+ sleep).

**Phase:** Implementation — set provider-specific timeout default. Config/docs phase for user guidance.

---

## Moderate Pitfalls

---

### Pitfall 4: `DefaultBaseURL` Switch Falls Through for `lmstudio`

**What goes wrong:**
`DefaultBaseURL()` in `provider.go` returns `""` for any unrecognised provider string:

```go
default:
    return ""
```

If `lmstudio` is added to the `NewProvider` switch but not to `DefaultBaseURL`, any config without an explicit `base_url` field triggers `ErrLLMMissingBaseURL` — even though the correct default (`http://localhost:1234/v1`) is well-known.

**Why it happens:** Both the factory switch and the default URL switch must be updated. It is easy to update one and forget the other.

**Consequences:** Users with a minimal config (`provider: lmstudio` + `model: ...`) get an unhelpful "base URL is required" error instead of working out of the box.

**Prevention:** Add `"lmstudio": "http://localhost:1234/v1"` to `DefaultBaseURL` in the same commit that adds the provider case to `NewProvider`. A test `TestDefaultBaseURL` should cover the `lmstudio` case.

**Detection:** Unit test calling `DefaultBaseURL("lmstudio")` must not return empty string.

**Phase:** Implementation — atomic change with factory registration.

---

### Pitfall 5: Silent Model Mismatch — Wrong Model Used Without Error

**What goes wrong:**
When exactly one model is loaded in LM Studio, the server ignores the `model` field in the request and uses whatever is loaded, regardless of what was specified. The response `model` field reflects the loaded model, not what was requested. This is a confirmed LM Studio behaviour (lmstudio-ai/lmstudio-bug-tracker#619).

For repodocs, the `domain.LLMResponse.Model` field will return a different value than what was configured. If there is any downstream logic that validates or logs the model used, it will see unexpected values.

**Why it happens:** LM Studio prioritises availability over strict model routing when a single model is running.

**Consequences:** Users believe they are running `meta-llama/Llama-3.2-3B-Instruct` but are actually running whatever model was last loaded. Metadata quality may differ from expectations. No error is surfaced.

**Prevention:**
- Document in provider help text / TUI that the model name must match what is loaded in LM Studio.
- Do not add validation that asserts `response.Model == config.Model` — this will always fail for LM Studio users with JIT enabled.

**Detection:** Warning sign: `response.Model` in logs doesn't match configured model name.

**Phase:** Documentation / TUI label — not a code change.

---

### Pitfall 6: Error Response Format Differs Pre-/Post-v0.3.18

**What goes wrong:**
LM Studio's changelog records that "errors returned from streaming endpoints now follow the correct format expected by OpenAI clients" was fixed in v0.3.18. Before this version, error JSON from LM Studio did not match the `openAIResponse.Error` struct shape that `openai.go` expects.

For non-streaming chat completions (which repodocs uses), the format appears consistent. However, LM Studio's error for "no model loaded" or server startup failures may return a plain string body or a 503 with no JSON body, not the `{"error": {"message": ..., "type": ..., "code": ...}}` envelope.

**Why it happens:** OpenAI-compatible does not mean error-format-compatible. LM Studio was developed incrementally toward full compatibility.

**Consequences:**
- `json.Unmarshal(respBody, &openAIResp)` succeeds (JSON is valid).
- `openAIResp.Error` is nil because the shape doesn't match.
- `resp.StatusCode` is non-200 (e.g. 503).
- Execution falls to `handleHTTPError` which returns a generic `string(body)` message.
- The user sees an opaque error string rather than "no model is loaded in LM Studio."

**Prevention:**
- The LM Studio provider's `handleHTTPError` (or the error handling block after JSON parse) should check for known LM Studio error patterns in the raw body when the parsed error struct is nil and status is 503/500.
- Specifically: if status 503 and body contains "no model" → return a descriptive `LLMError` with guidance.

**Detection:** Integration test with a mock that returns `{"error": "No model loaded"}` (flat string) with a 503 status.

**Phase:** Implementation — error handling branch in `lmstudio.go`.

---

## Minor Pitfalls

---

### Pitfall 7: `max_tokens: 0` Sends No Token Limit to LM Studio

**What goes wrong:**
`openai.go` uses `omitempty` on `MaxTokens`:

```go
MaxTokens int `json:"max_tokens,omitempty"`
```

When `MaxTokens` is 0 (the zero value), the field is omitted from the JSON payload entirely. For OpenAI this is fine — the API applies a model-specific default. For local models through LM Studio, omitting `max_tokens` means the model will generate until it hits the full context window (potentially thousands of tokens), making metadata enhancement requests very slow and potentially returning truncated/excessive output.

**Why it happens:** Cloud API defaults are conservative. Local model defaults are unconstrained.

**Prevention:** Always set a sensible `MaxTokens` default for LM Studio. The provider constructor should set `maxTokens` to a reasonable value (e.g. 512 or 1024) if the user's config is 0 and the provider is `lmstudio`. Document the recommended range.

**Detection:** Warning sign: inference taking >60 seconds for short prompts suggests unconstrained generation.

**Phase:** Implementation — provider constructor default.

---

### Pitfall 8: Circuit Breaker Opens on Cold-Start Timeout Cascade

**What goes wrong:**
The `RateLimitedProvider` wrapper's `DefaultRateLimitedProviderConfig` has `FailureThreshold: 5`. If the first 5 requests all time out during model loading (which can happen if repodocs is run immediately after LM Studio starts), the circuit breaker opens. All subsequent requests are rejected with `ErrLLMCircuitOpen` without even attempting the network call — even after LM Studio finishes loading the model.

**Why it happens:** The circuit breaker treats local model cold-start timeouts as service failures, which is correct for cloud APIs but misleading for local servers where a long initial load is expected behaviour.

**Consequences:** User runs repodocs, all requests fail with "circuit breaker is open," user restarts repodocs, problem goes away. Hard to diagnose.

**Prevention:**
- Set a higher `FailureThreshold` for local providers (e.g. 10 vs 5) or a longer `ResetTimeout` (e.g. 60s vs 30s) in the LM Studio provider's wrapper config.
- Log a user-friendly message when the circuit opens: "lmstudio: circuit breaker opened — LM Studio may be loading a model. Requests will retry after 30s."

**Detection:** `ErrLLMCircuitOpen` appearing in logs shortly after startup.

**Phase:** Implementation — pass custom `RateLimitedProviderConfig` for lmstudio in the factory.

---

### Pitfall 9: `Name()` Method Returns Wrong Provider Name in Errors

**What goes wrong:**
If the LM Studio implementation is built by wrapping `NewOpenAIProvider`, the wrapped provider's `Name()` returns `"openai"` — not `"lmstudio"`. This propagates into all `LLMError.Provider` fields and log output.

**Why it happens:** Code reuse temptation: "LM Studio is just OpenAI-compatible, so why not instantiate OpenAIProvider directly?" The `Name()` return value is hardcoded in `OpenAIProvider`.

**Consequences:** Errors say `"openai error (HTTP 503): ..."` when the user is not using OpenAI at all. Support burden and confusion.

**Prevention:** Implement a separate `LMStudioProvider` struct (even if it shares most logic with OpenAI via a helper or embedding). `Name()` must return `"lmstudio"`. All `LLMError{Provider: ...}` calls must use `"lmstudio"`.

**Detection:** Check that `provider.Name()` returns `"lmstudio"` in the unit test for the constructor.

**Phase:** Implementation — structural decision made at the start.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Provider factory registration | Missing `lmstudio` in `DefaultBaseURL` switch | Update both switches atomically, cover with `TestDefaultBaseURL` |
| API key exemption | `ErrLLMMissingAPIKey` blocks zero-config use | Update exemption condition alongside provider registration |
| HTTP client instantiation | 60s timeout causes cold-start failures | Override default timeout in `NewLMStudioProvider` or use a named constant |
| Authorization header | `Bearer ` with empty string breaks some clients | Conditional header set; never send an empty bearer token |
| Error handling | Flat-string 503 errors not parsed correctly | Inspect raw body when `openAIResp.Error == nil` and status >= 500 |
| Token generation | Unconstrained output without `max_tokens` | Provider constructor must set a sensible default if config value is 0 |
| Circuit breaker | Opens prematurely on cold-start timeout cascade | Use higher thresholds for local providers |
| Provider naming | `Name()` returning `"openai"` in error messages | Dedicated struct, not OpenAI alias |

---

## Sources

- LM Studio OpenAI Compatibility Docs: https://lmstudio.ai/docs/developer/openai-compat
- LM Studio API Key / Auth Docs: https://lmstudio.ai/docs/developer/core/authentication
- LM Studio Bug: API ignores model name when one model loaded: https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/619
- LM Studio Bug: 300s timeout with local server: https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/944
- openai-python: empty bearer token causes error: https://github.com/openai/openai-python/issues/961
- langchain4j: `stream: false` causes socket close: https://github.com/langchain4j/langchain4j/issues/2882
- Kilocode: hardcoded 300s timeout unusable for local LLMs: https://github.com/Kilo-Org/kilocode/issues/1681
- LM Studio TTL/Auto-Evict docs: https://lmstudio.ai/docs/developer/core/ttl-and-auto-evict
- LM Studio Changelog (error format fix v0.3.18): https://lmstudio.ai/changelog
