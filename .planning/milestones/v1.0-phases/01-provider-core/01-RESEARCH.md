# Phase 1: Provider Core - Research

**Researched:** 2026-04-07
**Domain:** Go LLM provider extension (OpenAI-compatible local server)
**Confidence:** HIGH

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROV-01 | User can configure `provider: lmstudio` and have it recognized by the provider factory | `NewProvider()` switch in `provider.go` needs a `"lmstudio"` case returning `NewLMStudioProvider()` |
| PROV-02 | LM Studio provider sends OpenAI-compatible chat completions requests to the configured endpoint | Reuse `openAIRequest`/`openAIResponse`/`openAIMessage` types; POST to `baseURL + "/chat/completions"` |
| PROV-03 | LM Studio provider defaults to `http://localhost:1234/v1` when no base_url is specified | Add `DefaultLMStudioBaseURL` constant and `"lmstudio"` case to `DefaultBaseURL()` |
| PROV-04 | LM Studio provider works without an API key (no authentication required by default) | Update `NewProviderFromConfig()` exemption guard from `!= "ollama"` to also exclude `"lmstudio"` |
| PROV-05 | LM Studio provider sends Bearer token when an API key is configured | Conditional header: `if p.apiKey != "" { httpReq.Header.Set("Authorization", "Bearer "+p.apiKey) }` |
| PROV-06 | LM Studio provider uses 300s default timeout to accommodate local model cold-start | In `NewProvider()`, detect `cfg.Provider == "lmstudio"` when timeout is 0 and set 300s instead of 60s |
| CONF-01 | User can set `provider: lmstudio` in YAML config file and have it validated | `LLMConfig.Provider` is a plain string field — no struct change needed; runtime validation in `NewProviderFromConfig` handles it |
| CONF-03 | Config validation accepts `lmstudio` as a valid provider name | Add `"lmstudio": true` to `validProviders` map in `ValidateLLMProvider()` in `internal/tui/validation.go` |
</phase_requirements>

## Summary

Phase 1 adds LM Studio as a named LLM provider in repodocs. LM Studio exposes a local HTTP server at `http://localhost:1234/v1` that is wire-format identical to the OpenAI chat completions API. The implementation requires zero new Go dependencies — all necessary types already exist in the `internal/llm` package (`openAIRequest`, `openAIResponse`, `openAIMessage`).

The correct architecture is a standalone `LMStudioProvider` struct in a new file `internal/llm/lmstudio.go`. It is structurally modelled on `OllamaProvider` (no mandatory API key, localhost default) but uses the OpenAI wire format rather than the Ollama-native format. Five existing files need targeted edits; two new files are created (provider + tests).

The single most critical pitfall is the API key validation gate in `provider.go:53`. It explicitly blocks every provider except `"ollama"` from proceeding without an API key. This gate MUST be updated to also exempt `"lmstudio"` before any other work, or zero-config use of LM Studio will fail with `ErrLLMMissingAPIKey`.

**Primary recommendation:** Create `internal/llm/lmstudio.go` as a standalone struct reusing package-private OpenAI types; update the five touch-points in `provider.go`, `validation.go`, `forms.go`, and their tests atomically.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `net/http` | stdlib | HTTP client for LM Studio requests | Already used by all providers — no new dependency |
| `encoding/json` | stdlib | Marshal/unmarshal OpenAI wire format | Already used — same JSON types reused |
| `github.com/stretchr/testify` | existing | Test assertions | Already used in all `*_test.go` files in `internal/llm/` |

[VERIFIED: direct source analysis of `internal/llm/openai.go`, `internal/llm/ollama.go`, `go.mod`]

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `net/http/httptest` | stdlib | Mock HTTP server in unit tests | Used in `ollama_test.go`, `openai_test.go` — same pattern for `lmstudio_test.go` |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Standalone struct | Wrap `*OpenAIProvider` | Wrapping causes `Name()` to return `"openai"`, all `LLMError.Provider` fields wrong, and unconditional auth header sent even when key is empty — rejected |

**Installation:**
No new packages. Zero dependency changes.

## Architecture Patterns

### Recommended Project Structure
```
internal/llm/
├── lmstudio.go          # NEW: LMStudioProvider struct
├── lmstudio_test.go     # NEW: unit tests (mirrors ollama_test.go)
├── provider.go          # MODIFY: constant, DefaultBaseURL, API key guard, NewProvider switch
├── openai.go            # READ-ONLY: source of openAIRequest/openAIResponse types
├── ollama.go            # READ-ONLY: structural pattern reference
internal/tui/
├── validation.go        # MODIFY: add "lmstudio" to validProviders map
├── forms.go             # MODIFY: add LM Studio option to provider select
├── validation_test.go   # MODIFY: add "lmstudio" to valid providers test slice
```

### Pattern 1: Standalone Provider Struct

**What:** `LMStudioProvider` is its own struct with its own `Name()`, `Complete()`, `Close()`, and `handleHTTPError()` methods. It directly uses the package-private `openAIRequest`, `openAIResponse`, and `openAIMessage` types from the same `llm` package.

**When to use:** Any time a provider shares wire format with OpenAI but differs in authentication, identity, error attribution, or defaults.

**Example:**
```go
// Source: direct codebase analysis — mirrors internal/llm/openai.go structure
type LMStudioProvider struct {
    httpClient  *http.Client
    apiKey      string   // optional, may be empty
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

func NewLMStudioProvider(cfg ProviderConfig, httpClient *http.Client) (*LMStudioProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    return &LMStudioProvider{
        httpClient:  httpClient,
        apiKey:      cfg.APIKey,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   cfg.MaxTokens,
        temperature: cfg.Temperature,
    }, nil
}

func (p *LMStudioProvider) Name() string {
    return "lmstudio"
}
```

### Pattern 2: Conditional Authorization Header

**What:** Only set `Authorization: Bearer` header when the API key is non-empty. This is the key behavioral difference from `OpenAIProvider`, which unconditionally sets the header.

**When to use:** Any provider where authentication is optional.

**Example:**
```go
// Source: direct codebase analysis — contrast with openai.go line 114
httpReq.Header.Set("Content-Type", "application/json")
if p.apiKey != "" {
    httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
}
```

### Pattern 3: Provider-Specific Timeout Override

**What:** In `NewProvider()`, detect `"lmstudio"` provider and apply a 300s default timeout instead of the global 60s default. This handles cold-start model loading.

**When to use:** Any local provider where first-request latency is structurally higher than cloud providers.

**Example:**
```go
// Source: direct codebase analysis of provider.go:83-91
timeout := cfg.Timeout
if timeout == 0 {
    if cfg.Provider == "lmstudio" {
        timeout = 300 * time.Second
    } else {
        timeout = 60 * time.Second
    }
}
```

### Pattern 4: API Key Exemption Guard Update

**What:** The current guard at `provider.go:53` exempts only `"ollama"`. Adding `"lmstudio"` uses the same explicit inequality pattern — do not refactor to a slice or set.

**When to use:** Every time a new keyless provider is added.

**Example:**
```go
// Source: direct codebase analysis of provider.go:53 (current code)
// CURRENT:
if cfg.APIKey == "" && cfg.Provider != "ollama" {
    return nil, domain.ErrLLMMissingAPIKey
}
// REQUIRED:
if cfg.APIKey == "" && cfg.Provider != "ollama" && cfg.Provider != "lmstudio" {
    return nil, domain.ErrLLMMissingAPIKey
}
```

### Pattern 5: Factory Switch Registration

**What:** Both `DefaultBaseURL()` and `NewProvider()` switches must be updated atomically. Missing one causes a different error (missing base URL vs. invalid provider).

**Example:**
```go
// In DefaultBaseURL():
const DefaultLMStudioBaseURL = "http://localhost:1234/v1"

case "lmstudio":
    return DefaultLMStudioBaseURL

// In NewProvider() switch:
case "lmstudio":
    return NewLMStudioProvider(cfg, httpClient)
```

### Pattern 6: TUI Validation Map Update

**What:** `ValidateLLMProvider()` uses a `map[string]bool` allow-list. Add `"lmstudio"` to it. The error message string must also be updated.

**Example:**
```go
// Source: direct codebase analysis of tui/validation.go:135-143
validProviders := map[string]bool{
    "openai":    true,
    "anthropic": true,
    "google":    true,
    "lmstudio":  true,   // ADD THIS
}
// Update error message:
return fmt.Errorf("invalid LLM provider: must be openai, anthropic, google, or lmstudio")
```

### Pattern 7: LM Studio-Specific Error Handling

**What:** On non-200 responses, attempt to parse the OpenAI error envelope. If it fails and status is 503, inspect raw body for "no model" text and return a descriptive error.

**Example:**
```go
// Source: pitfall analysis — LM Studio may return flat string errors on 503
if resp.StatusCode != http.StatusOK {
    // Try OpenAI envelope first
    if openAIResp.Error != nil {
        // ... standard error handling
    }
    // Fallback: check for LM Studio-specific 503 pattern
    if resp.StatusCode == http.StatusServiceUnavailable {
        bodyStr := string(respBody)
        if strings.Contains(strings.ToLower(bodyStr), "no model") {
            return nil, &domain.LLMError{
                Provider:   "lmstudio",
                StatusCode: resp.StatusCode,
                Message:    "no model is loaded in LM Studio — load a model and retry",
                Err:        domain.ErrLLMRequestFailed,
            }
        }
    }
    return nil, p.handleHTTPError(resp.StatusCode, respBody)
}
```

### Anti-Patterns to Avoid

- **Wrapping OpenAIProvider:** `Name()` returns `"openai"`, unconditional auth header always sent even with empty key — causes confusing errors.
- **Sending `Authorization: Bearer ` with empty token:** Some clients validate this header client-side and raise errors before reaching LM Studio.
- **Updating only one factory switch:** `DefaultBaseURL` and `NewProvider` must both add `"lmstudio"` — missing one causes a different opaque error.
- **Using 60s timeout for LM Studio:** Cold-start for 7B+ models takes 30-120+ seconds; 60s causes timeout cascades that open the circuit breaker.
- **Validating that `response.Model == config.Model`:** LM Studio ignores the `model` field when a single model is loaded and returns whatever is active.
- **Fixing the pre-existing Ollama TUI validation gap:** `ValidateLLMProvider` currently omits `"ollama"` — add `"lmstudio"` only, do not fix the Ollama gap (out of scope).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JSON request/response types | New `lmstudioRequest` struct | `openAIRequest`/`openAIResponse` from same package | LM Studio is wire-identical to OpenAI chat completions; duplication adds maintenance burden |
| HTTP client with timeout | Custom transport | `&http.Client{Timeout: timeout}` same as all other providers | Consistent with codebase pattern |
| Error types | New error structs | `domain.LLMError{Provider: "lmstudio", ...}` | All providers use this type; consistent unwrapping and error attribution |

**Key insight:** The entire HTTP interaction pattern is already proven by both `OpenAIProvider` and `OllamaProvider`. The new provider is structurally a combination of their authentication model (Ollama's optional key) and wire format (OpenAI's JSON types).

## Common Pitfalls

### Pitfall 1: API Key Validation Gate (CRITICAL — must fix first)
**What goes wrong:** `provider.go:53` has `cfg.Provider != "ollama"` — adding `"lmstudio"` to the factory without updating this guard causes `ErrLLMMissingAPIKey` for any zero-config LM Studio setup.
**Why it happens:** The exemption is an explicit allow-list; new providers are not exempt by default.
**How to avoid:** Update the guard condition BEFORE or IN THE SAME COMMIT as factory registration. Use the exact negation pattern: `cfg.Provider != "ollama" && cfg.Provider != "lmstudio"`.
**Warning signs:** `TestNewProviderFromConfig` with `provider: lmstudio` and empty `api_key` returns `ErrLLMMissingAPIKey`.

### Pitfall 2: Empty Bearer Token Sent
**What goes wrong:** Copying OpenAI's `httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)` unconditionally sends `Authorization: Bearer ` (with trailing space) when key is empty.
**Why it happens:** OpenAI provider assumes a key is always present — it is the first consumer of this code path where key is legitimately absent.
**How to avoid:** Wrap the header set in `if p.apiKey != "" { ... }`.
**Warning signs:** Test asserting `assert.Empty(t, r.Header.Get("Authorization"))` when no key configured fails.

### Pitfall 3: 60s Timeout Too Short for Cold Start
**What goes wrong:** Default 60s timeout in `NewProvider()` is insufficient for local model JIT loading (30-120s). All retries burn through their full timeout; circuit breaker opens after 5 failures.
**Why it happens:** Cloud providers respond in milliseconds; local models have a fundamentally different latency profile.
**How to avoid:** Apply a 300s default specifically for `"lmstudio"` when `cfg.Timeout == 0`.
**Warning signs:** Consecutive timeout errors on first run, followed by `ErrLLMCircuitOpen`.

### Pitfall 4: DefaultBaseURL Switch Not Updated
**What goes wrong:** If `NewProvider` switch adds `"lmstudio"` but `DefaultBaseURL` does not, configs without an explicit `base_url` field return `ErrLLMMissingBaseURL`.
**Why it happens:** Two separate switch statements must both be updated; easy to forget one.
**How to avoid:** Update both in the same commit. Unit test `TestDefaultBaseURL` must have a `"lmstudio"` case.
**Warning signs:** `TestDefaultBaseURL("lmstudio")` returns empty string.

### Pitfall 5: Wrong Provider Name in Errors
**What goes wrong:** Wrapping `OpenAIProvider` causes all `LLMError.Provider` fields to read `"openai"` — confusing when the user is using LM Studio.
**Why it happens:** `OpenAIProvider.Name()` is hardcoded.
**How to avoid:** Standalone struct with `Name()` returning `"lmstudio"` and all `LLMError{Provider: "lmstudio"}` literals.
**Warning signs:** Error message says "openai error (HTTP 503)" when using LM Studio.

### Pitfall 6: TUI Validation Rejects Valid Config
**What goes wrong:** `ValidateLLMProvider` in `tui/validation.go` does not include `"lmstudio"` — TUI rejects it even though the runtime factory accepts it.
**Why it happens:** Separate validation layer; the TUI and runtime factory must both be updated.
**How to avoid:** Add `"lmstudio": true` to the `validProviders` map AND update the error message string.
**Warning signs:** TUI form shows validation error for `lmstudio` selection.

## Code Examples

Verified patterns from official sources (direct codebase analysis):

### Complete LMStudioProvider struct skeleton
```go
// Source: mirrors internal/llm/openai.go with conditional auth
package llm

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strings"

    "github.com/quantmind-br/repodocs/internal/domain"
)

type LMStudioProvider struct {
    httpClient  *http.Client
    apiKey      string
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

func NewLMStudioProvider(cfg ProviderConfig, httpClient *http.Client) (*LMStudioProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    return &LMStudioProvider{
        httpClient:  httpClient,
        apiKey:      cfg.APIKey,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   cfg.MaxTokens,
        temperature: cfg.Temperature,
    }, nil
}

func (p *LMStudioProvider) Name() string { return "lmstudio" }
func (p *LMStudioProvider) Close() error { return nil }
```

### provider.go changes
```go
// Source: internal/llm/provider.go — exact lines to modify
const DefaultLMStudioBaseURL = "http://localhost:1234/v1"

// In DefaultBaseURL():
case "lmstudio":
    return DefaultLMStudioBaseURL

// In NewProviderFromConfig() — update the API key guard:
if cfg.APIKey == "" && cfg.Provider != "ollama" && cfg.Provider != "lmstudio" {
    return nil, domain.ErrLLMMissingAPIKey
}

// In NewProvider() — timeout logic:
if timeout == 0 {
    if cfg.Provider == "lmstudio" {
        timeout = 300 * time.Second
    } else {
        timeout = 60 * time.Second
    }
}

// In NewProvider() switch:
case "lmstudio":
    return NewLMStudioProvider(cfg, httpClient)
```

### Test pattern (mirrors ollama_test.go)
```go
// Source: internal/llm/ollama_test.go — same httptest pattern
func TestNewLMStudioProvider(t *testing.T) {
    cfg := ProviderConfig{
        BaseURL:   "http://localhost:1234/v1",
        Model:     "meta-llama/Llama-3.2-3B-Instruct",
        MaxTokens: 512,
    }
    provider, err := NewLMStudioProvider(cfg, &http.Client{Timeout: 300 * time.Second})
    require.NoError(t, err)
    assert.Equal(t, "lmstudio", provider.Name())
}

func TestLMStudioProvider_Complete_NoAuthHeader(t *testing.T) {
    // Key test: verifies auth header absent when no API key
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Empty(t, r.Header.Get("Authorization"))  // MUST be absent
        assert.Equal(t, "/chat/completions", r.URL.Path) // OpenAI path, not /api/chat
        // ... return valid openAI response
    }))
    defer server.Close()
    // ...
}
```

### TUI validation update
```go
// Source: internal/tui/validation.go:135
validProviders := map[string]bool{
    "openai":    true,
    "anthropic": true,
    "google":    true,
    "lmstudio":  true,
}
if !validProviders[strings.ToLower(s)] {
    return fmt.Errorf("invalid LLM provider: must be openai, anthropic, google, or lmstudio")
}
```

### TUI forms update
```go
// Source: internal/tui/forms.go:196-204
Options(
    huh.NewOption("None (disabled)", ""),
    huh.NewOption("OpenAI", "openai"),
    huh.NewOption("Anthropic", "anthropic"),
    huh.NewOption("Google", "google"),
    huh.NewOption("Ollama", "ollama"),
    huh.NewOption("LM Studio", "lmstudio"),   // ADD THIS
).
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `cfg.Provider != "ollama"` exemption | `cfg.Provider != "ollama" && cfg.Provider != "lmstudio"` | Phase 1 | Enables zero-config LM Studio use |
| 60s global timeout | 300s for `"lmstudio"`, 60s for others | Phase 1 | Prevents cold-start timeout cascade |
| 3-provider TUI select | 4-provider TUI select (+ LM Studio) | Phase 1 | Users can configure via TUI |

**Deprecated/outdated:**
- Nothing deprecated; this is additive change only.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | LM Studio's `/chat/completions` endpoint is wire-identical to OpenAI's (uses same JSON schema) | Standard Stack, Code Examples | Requests fail silently; response parsing may return empty results |
| A2 | 300s is a sufficient default timeout for local model cold-start | Architecture Patterns (Pattern 3) | Users with very large models on slow hardware may still timeout on first request |
| A3 | LM Studio accepts requests with no `Authorization` header (not just empty bearer) | Architecture Patterns (Pattern 2) | If LM Studio requires the header even when empty, conditional omission would fail auth |

Note: A1 and A3 are supported by the project-level research (SUMMARY.md, PITFALLS.md) which cites official LM Studio docs. A2 is also from project research citing lmstudio-ai/lmstudio-bug-tracker#944.

## Open Questions

1. **Max tokens default for LM Studio**
   - What we know: `openAIRequest.MaxTokens` uses `omitempty`; if 0, field is omitted entirely. LM Studio without `max_tokens` generates until context window fills (potentially thousands of tokens).
   - What's unclear: Should the provider constructor enforce a minimum `maxTokens` (e.g. 512) when `cfg.MaxTokens == 0`? The `config/defaults.go` already sets `DefaultLLMMaxTokens = 4096` globally, so in practice the field should always be non-zero for LM Studio.
   - Recommendation: Check at runtime in `NewLMStudioProvider` — if `cfg.MaxTokens == 0`, set to 512 as a local-model-appropriate safe default.

2. **Circuit breaker thresholds for local providers**
   - What we know: `DefaultCircuitBreakerFailureThreshold = 5` and `DefaultCircuitBreakerResetTimeout = 30s`. Cold-start timeouts can trigger 5 consecutive failures before the model finishes loading.
   - What's unclear: Whether Phase 1 scope includes overriding circuit breaker config for LM Studio at the factory level, or whether this is deferred.
   - Recommendation: Defer to Phase 2 (configuration UX); document in provider help text that users with slow hardware should increase circuit breaker failure threshold.

## Environment Availability

Step 2.6: SKIPPED — this phase is pure Go source code changes. No external CLI tools, databases, or services are required at implementation time. LM Studio itself is tested by integration tests (TEST-04, deferred to Phase 2), not Phase 1.

## Project Constraints (from CLAUDE.md)

| Directive | Source | Impact on Phase |
|-----------|--------|-----------------|
| `make test` runs unit tests with `-short` flag | CLAUDE.md Quick Commands | All new tests must pass under `-short` (no external calls; use httptest) |
| `make lint` runs golangci-lint v2 | CLAUDE.md Quick Commands | Code must be lint-clean before commit |
| `make fmt` / `make vet` required | CLAUDE.md Quick Commands | Format and vet after every file change |
| Strategy pattern: `Name() string`, `CanHandle()`, `Execute()` | CLAUDE.md Architecture | `LLMProvider` interface is separate — `Name()` and `Complete()` and `Close()` as defined in `internal/domain/interfaces.go` |
| Detection order: LLMS → PkgGo → Sitemap → Git → Crawler | CLAUDE.md Architecture | Not relevant to LLM provider changes |

## Sources

### Primary (HIGH confidence)
- Direct source analysis: `internal/llm/provider.go` — exact API key guard location (line 53), factory switch, DefaultBaseURL switch
- Direct source analysis: `internal/llm/openai.go` — `openAIRequest`/`openAIResponse`/`openAIMessage` types, unconditional auth header pattern
- Direct source analysis: `internal/llm/ollama.go` — standalone struct pattern, no-key auth model, structural template
- Direct source analysis: `internal/tui/validation.go` — `validProviders` map, pre-existing Ollama gap confirmed
- Direct source analysis: `internal/tui/forms.go` — `CreateLLMForm()` provider select, exact `huh.NewOption` call signature
- Direct source analysis: `internal/llm/ollama_test.go` — httptest mock pattern to replicate
- Direct source analysis: `internal/llm/provider_test.go` — `TestDefaultBaseURL` table structure, `TestNewProviderFromConfig` cases to extend
- Direct source analysis: `internal/config/defaults.go` — `DefaultLLMMaxTokens = 4096`, `DefaultCircuitBreakerFailureThreshold = 5`
- Project research: `.planning/research/SUMMARY.md`, `.planning/research/ARCHITECTURE.md`, `.planning/research/PITFALLS.md` — LM Studio behavior, timeout requirements, error format issues

### Secondary (MEDIUM confidence)
- `.planning/research/PITFALLS.md` — cites lmstudio-ai/lmstudio-bug-tracker#619, #944; openai/openai-python#961; Kilo-Org/kilocode#1681
- `.planning/research/ARCHITECTURE.md` — cites LM Studio official docs for OpenAI compatibility and authentication

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — zero new dependencies; all types and patterns verified in current source
- Architecture: HIGH — all patterns verified by direct source code inspection; no assumptions about external behavior required for Phase 1 scope
- Pitfalls: HIGH — all critical pitfalls verified against actual code (exact line numbers, exact guard conditions)

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable Go codebase; no fast-moving dependencies)
