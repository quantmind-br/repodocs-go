# Architecture Patterns: LM Studio Provider Integration

**Domain:** LLM provider extension in Go CLI (repodocs)
**Researched:** 2026-04-07
**Confidence:** HIGH — analysis based on direct source code inspection

---

## Recommended Architecture: Standalone Provider Wrapping OpenAI Types

**Recommendation:** Create `internal/llm/lmstudio.go` as a standalone provider struct — but **reuse the existing OpenAI request/response types** (`openAIRequest`, `openAIResponse`, `openAIMessage`) without embedding or delegating to the `OpenAIProvider` struct itself.

This is the correct balance between code reuse and independent identity.

---

## Composition vs Standalone: Tradeoff Analysis

### Option A: Wrap/Embed OpenAIProvider (Rejected)

Embedding `*OpenAIProvider` inside an `LMStudioProvider` struct, or delegating `Complete()` to an internal `OpenAIProvider` instance.

**Why it seems appealing:** LM Studio is OpenAI-compatible, so zero duplication.

**Why it breaks down:**

1. `OpenAIProvider.Name()` returns `"openai"` — hardcoded. Overriding this requires awkward indirection or a provider-name field injected at construction, which is not how the existing providers work.
2. `OpenAIProvider.handleHTTPError()` and `LLMError` structs embed `Provider: "openai"` throughout. Wrapping means all error messages report the wrong provider name, or you intercept and rewrite every error — defeating the purpose of reuse.
3. `OpenAIProvider` always sets `Authorization: Bearer <apiKey>` header, even when `apiKey` is empty. LM Studio accepts requests with no `Authorization` header at all. Injecting an empty bearer token is technically harmless but semantically wrong and could break future LM Studio versions that enforce auth header validity.
4. The factory in `provider.go` constructs by direct type: `NewOpenAIProvider(cfg, httpClient)`. Wrapping would require an extra constructor layer that adds complexity with no user-visible benefit.

**Verdict:** Rejected. The coupling creates more problems than it solves.

### Option B: Standalone Struct + Shared Types (Recommended)

`LMStudioProvider` is its own struct, directly using `openAIRequest` / `openAIResponse` / `openAIMessage` types (package-private, same `llm` package).

**Why this is correct:**

1. `Name()` returns `"lmstudio"` — clean, unambiguous identity.
2. All `LLMError` structs use `Provider: "lmstudio"` throughout — correct error attribution.
3. Auth header is conditional: only set `Authorization: Bearer <apiKey>` when `apiKey != ""`. This is the key LM Studio UX difference from OpenAI.
4. HTTP endpoint is `p.baseURL + "/chat/completions"` — identical path to OpenAI's, so the wire format is the same without any abstraction needed.
5. Follows the exact same structural pattern as `OllamaProvider` (no-key, localhost default), which is already proven and tested.
6. The `openAIRequest`/`openAIResponse` types are not exported — they live in the same package, so `lmstudio.go` can use them directly without any coupling across package boundaries.

**Verdict:** Recommended. Maximum code reuse where it matters (JSON types), full independence where it matters (identity, auth, errors).

---

## Component Boundaries

| Component | File | Change Type | What Changes |
|-----------|------|-------------|--------------|
| Provider factory defaults | `internal/llm/provider.go` | Modify | Add `DefaultLMStudioBaseURL` constant; add `"lmstudio"` case to `DefaultBaseURL()`; add `"lmstudio"` to API-key-optional check; add `"lmstudio"` case to `NewProvider()` switch |
| LM Studio provider impl | `internal/llm/lmstudio.go` | Create new | New file with `LMStudioProvider` struct and `NewLMStudioProvider()` |
| TUI validation | `internal/tui/validation.go` | Modify | Add `"lmstudio": true` to `validProviders` map in `ValidateLLMProvider()` |
| TUI forms | `internal/tui/forms.go` | Modify | Add `huh.NewOption("LM Studio", "lmstudio")` to provider select in `CreateLLMForm()` |
| Unit tests | `internal/llm/lmstudio_test.go` | Create new | Tests mirroring `ollama_test.go` structure |
| Unit tests (provider) | `internal/llm/provider_test.go` | Modify | Add `"lmstudio"` cases to `TestNewProviderFromConfig`, `TestDefaultBaseURL` |
| TUI validation tests | `internal/tui/validation_test.go` | Modify | Add `"lmstudio"` to `validProviders` slice |

No changes needed to: `internal/domain/interfaces.go`, `internal/config/config.go`, `internal/domain/errors.go`.

---

## Data Flow

```
config.LLMConfig{Provider: "lmstudio", APIKey: "", Model: "..."}
  |
  v
NewProviderFromConfig()
  - APIKey empty check: skipped because provider == "lmstudio" (same as ollama branch)
  - DefaultBaseURL("lmstudio") -> "http://localhost:1234/v1"
  |
  v
NewProvider(ProviderConfig{...})
  - switch "lmstudio" -> NewLMStudioProvider(cfg, httpClient)
  |
  v
LMStudioProvider.Complete(ctx, req)
  - Builds openAIRequest{} (reuses package-private type)
  - POST to baseURL + "/chat/completions"
  - Sets "Authorization: Bearer <apiKey>" ONLY IF apiKey != ""
  - Parses openAIResponse{} (reuses package-private type)
  - Returns domain.LLMResponse with Provider: "lmstudio" in errors
```

---

## LMStudioProvider Struct

```go
type LMStudioProvider struct {
    httpClient  *http.Client
    apiKey      string   // optional, may be empty
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

func (p *LMStudioProvider) Name() string {
    return "lmstudio"
}
```

The `Complete()` method is structurally identical to `OpenAIProvider.Complete()` with one conditional difference:

```go
if p.apiKey != "" {
    httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
}
```

All `LLMError` structs use `Provider: "lmstudio"`.

---

## Key Differences from OpenAI Provider

| Aspect | OpenAI | LM Studio |
|--------|--------|-----------|
| `Name()` | `"openai"` | `"lmstudio"` |
| Default base URL | `https://api.openai.com/v1` | `http://localhost:1234/v1` |
| API key required | Yes | No (optional) |
| Auth header | Always set | Only when apiKey non-empty |
| Wire format | OpenAI chat completions | Identical (OpenAI-compatible) |
| Error Provider field | `"openai"` | `"lmstudio"` |

---

## Key Differences from Ollama Provider

| Aspect | Ollama | LM Studio |
|--------|--------|-----------|
| Wire format | Ollama-native (`/api/chat`, `ollamaRequest`) | OpenAI-compatible (`/chat/completions`, `openAIRequest`) |
| API key | Not supported | Optional |
| Response parsing | `ollamaResponse` with `done` field | `openAIResponse` with `choices` array |

LM Studio is structurally closer to OpenAI in wire format, and closer to Ollama in authentication model. The standalone approach captures both.

---

## Build Order

The implementation has no novel dependencies — everything needed exists. Recommended build sequence:

1. **`internal/llm/lmstudio.go`** — core provider, no external dependencies beyond existing package types
2. **`internal/llm/provider.go`** — add constant, update `DefaultBaseURL()`, update API-key guard, update `NewProvider()` switch
3. **`internal/tui/validation.go`** — add `"lmstudio"` to valid providers map
4. **`internal/tui/forms.go`** — add LM Studio option to select
5. **`internal/llm/lmstudio_test.go`** — unit tests
6. **`internal/llm/provider_test.go`** — extend existing provider tests
7. **`internal/tui/validation_test.go`** — extend TUI validation tests

Each step is independently mergeable. Steps 1-2 form a functional unit (provider works). Steps 3-4 form a functional unit (TUI works). Steps 5-7 are test coverage.

---

## Pitfalls to Avoid

**Do not skip the API-key guard update in `NewProviderFromConfig()`.** The current check is:

```go
if cfg.APIKey == "" && cfg.Provider != "ollama" {
    return nil, domain.ErrLLMMissingAPIKey
}
```

This must become:

```go
if cfg.APIKey == "" && cfg.Provider != "ollama" && cfg.Provider != "lmstudio" {
    return nil, domain.ErrLLMMissingAPIKey
}
```

Missing this causes `lmstudio` configs without an API key to return `ErrLLMMissingAPIKey` before even reaching the factory switch — the most likely integration bug.

**Do not use a string list for the no-key providers.** The current pattern (explicit `!= "ollama"` check) should be maintained for consistency with the existing code style rather than refactoring to a set. Refactoring scope is out of bounds for this milestone.

**TUI validation is separate from factory validation.** `ValidateLLMProvider()` in `internal/tui/validation.go` currently lists `openai`, `anthropic`, `google` — it omits `ollama` even though `ollama` works at runtime. This is a pre-existing bug. For LM Studio, add it to both the runtime factory (provider.go) and the TUI validation (validation.go) so the TUI does not reject valid input.

---

## Sources

- Direct source analysis: `internal/llm/provider.go`, `internal/llm/openai.go`, `internal/llm/ollama.go`
- Direct source analysis: `internal/domain/interfaces.go`, `internal/config/config.go`
- Direct source analysis: `internal/tui/validation.go`, `internal/tui/forms.go`
- Confidence: HIGH — all findings from current codebase, no external sources required
