# Feature Landscape: LM Studio Provider Integration

**Domain:** Local LLM provider for Go CLI metadata enhancement
**Researched:** 2026-04-07
**Confidence:** HIGH — based on LM Studio official docs + direct codebase analysis

---

## Table Stakes

Features users expect. Missing = provider feels broken or incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| `lmstudio` recognized as valid provider name | Every other provider has a named entry in the factory switch; "unknown provider" error is jarring | Low | Add to `NewProvider` switch + `DefaultBaseURL` |
| Default base URL `http://localhost:1234/v1` | LM Studio's documented default; users expect zero-config for localhost | Low | Constant in `provider.go`, same pattern as `DefaultOllamaBaseURL` |
| No API key required | LM Studio has no auth by default; requiring a key breaks zero-config UX | Low | Mirror the Ollama exemption in `NewProviderFromConfig` (line 53) |
| OpenAI-compatible chat completions format | LM Studio's REST API IS the OpenAI format at `/v1/chat/completions` | Low | Reuse `openAIRequest`/`openAIResponse` types or duplicate them as `lmstudioRequest` |
| `Authorization: Bearer` header sent only when API key is set | Some users enable LM Studio auth; the header must be omitted (not empty) when no key is provided | Low | Conditional header: `if p.apiKey != "" { set header }` |
| Config validation accepts `lmstudio` as valid provider | `ValidateLLMProvider` in `tui/validation.go` currently rejects it — TUI config save fails | Low | Add `"lmstudio": true` to the `validProviders` map |
| TUI form lists LM Studio as selectable provider option | Without it, users cannot configure the provider through the TUI | Low | Add `huh.NewOption("LM Studio (local)", "lmstudio")` to `CreateLLMForm` |
| `provider.Name()` returns `"lmstudio"` | Error messages and logs use `Name()` to identify the provider; must be consistent | Low | Return `"lmstudio"` not `"openai"` |
| Unit tests covering provider creation and `Name()` | All existing providers have unit tests; missing tests blocks CI | Low | Follow `ollama_test.go` pattern with httptest server |

---

## Differentiators

Features that improve UX but users won't notice if absent.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Helpful error message when LM Studio is unreachable | Connection refused to localhost is a common failure mode; a hint like "Is LM Studio running?" saves troubleshooting time | Low | Detect `connection refused` in `net.Error` and wrap with a hint message |
| Default model placeholder in TUI form | LM Studio model names are unusual (e.g., `lmstudio-community/Meta-Llama-3.1-8B-Instruct-GGUF`); a useful placeholder reduces friction | Low | Set TUI placeholder to e.g. `lmstudio-community/...` when provider is lmstudio |
| Documentation in `README` or config comments | First-time users need to know LM Studio must be running before repodocs starts | Low | YAML config comment explaining the dependency |

---

## Anti-Features

Features to deliberately NOT build for this milestone.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Model listing / auto-detection via `GET /v1/models` | Out of scope per PROJECT.md; adds complexity and a network call at startup | Users specify model name manually, same as every other provider |
| Streaming responses | Not used by the metadata enhancement pipeline; the pipeline calls `Complete()` which expects a single response | Leave `stream: false` as implicit (no field needed; OpenAI default) |
| LM Studio process management (start/stop) | Scope creep; repodocs does not manage Ollama either | Document that users must start LM Studio manually |
| Custom LM Studio SDK types | LM Studio's API is a strict superset of OpenAI chat completions; maintaining a separate type set is pure overhead | Reuse `openAIRequest`/`openAIResponse` internally or share them |
| Token usage reporting for LM Studio | LM Studio does populate `usage` in responses, but the existing pipeline ignores usage data; adding it now is out of scope | Pass `LLMUsage` from the response same as OpenAI provider does |
| Rate limiting tuned for local server | Local inference is not rate-limited; the existing `RateLimitConfig` already applies generically if enabled | No LM Studio-specific rate limit logic needed |

---

## Feature Dependencies

```
lmstudio in DefaultBaseURL(provider.go)
  → lmstudio in NewProvider switch (provider.go)
    → lmstudio in NewProviderFromConfig API-key exemption (provider.go)
      → lmstudio in ValidateLLMProvider (tui/validation.go)
        → lmstudio option in CreateLLMForm (tui/forms.go)

LMStudioProvider struct (new file: lmstudio.go)
  → reuses openAIRequest / openAIResponse types (openai.go or shared)
  → Name() returns "lmstudio"
  → Complete() calls /chat/completions, omits Authorization header when apiKey == ""

Unit tests (lmstudio_test.go)
  → depends on LMStudioProvider implementation
  → depends on provider.go factory entry
```

---

## MVP Recommendation

All table stakes features are the MVP. They are all Low complexity and form a single coherent unit — partial implementation (e.g., provider registered in factory but not in TUI validation) results in silent failures or confusing error messages.

Build in this order:

1. `provider.go` — constant, `DefaultBaseURL`, `NewProvider` switch, API-key exemption
2. `lmstudio.go` — provider struct and `Complete()` implementation
3. `tui/validation.go` — add `lmstudio` to valid providers
4. `tui/forms.go` — add `lmstudio` option to provider select
5. `lmstudio_test.go` — unit tests

Differentiators can be added without blocking delivery:
- "Is LM Studio running?" error hint is a single `strings.Contains` check; add it opportunistically
- TUI placeholder is a one-liner; add it with the forms change

---

## Implementation Notes

### API key handling
The current `NewProviderFromConfig` exemption is a string comparison: `cfg.Provider != "ollama"`. After adding LM Studio, expand to: `cfg.Provider != "ollama" && cfg.Provider != "lmstudio"`. This is the only place the exemption lives.

### Optional API key with Bearer token
When `apiKey != ""`, send `Authorization: Bearer <key>`. When empty, omit the header entirely. LM Studio with auth enabled will reject requests without the header; LM Studio without auth ignores it. The OpenAI provider always sets the header — the LM Studio provider must be conditional.

### Request/response types
LM Studio's `/v1/chat/completions` accepts and returns the same JSON as OpenAI. Options:
- Simplest: define `lmstudioRequest` / `lmstudioResponse` as type aliases pointing to the openAI types (avoids duplication)
- Cleaner long-term: copy the structs into `lmstudio.go` (keeps files self-contained, same pattern as Ollama)
- Recommended: copy structs (matches existing pattern; avoids cross-file type coupling)

### Error handling
LM Studio returns errors in OpenAI error envelope format `{"error": {"message": "...", "type": "...", "code": "..."}}`. Handle 401 as `ErrLLMAuthFailed` and connection errors with a localhost hint. No 429 rate limiting expected from a local server.

---

## Sources

- [LM Studio OpenAI Compatibility Endpoints](https://lmstudio.ai/docs/developer/openai-compat) — HIGH confidence
- [LM Studio Authentication](https://lmstudio.ai/docs/developer/core/authentication) — HIGH confidence
- [LM Studio Local Server Docs](https://lmstudio.ai/docs/developer/core/server) — HIGH confidence
- Codebase analysis: `internal/llm/provider.go`, `internal/llm/ollama.go`, `internal/llm/openai.go`, `internal/tui/forms.go`, `internal/tui/validation.go` — HIGH confidence
