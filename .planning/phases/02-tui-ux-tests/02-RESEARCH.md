# Phase 2: TUI, UX & Tests - Research

**Researched:** 2026-04-07
**Domain:** Go unit/integration testing, TUI validation, connection error UX
**Confidence:** HIGH

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CONF-02 | TUI config editor shows LM Studio as a provider option in the dropdown | Already implemented in `forms.go:203` — verify no further work needed |
| CONF-04 | User sees a helpful error message when LM Studio server is not running (connection refused) | Error wrapping through `domain.LLMError` already surfaces connection errors; verify message quality |
| TEST-01 | Unit tests verify LM Studio provider sends correct request format via httptest mock | New file `internal/llm/lmstudio_test.go` needed — no lmstudio unit tests exist yet |
| TEST-02 | Unit tests verify conditional auth header behavior (present with key, absent without) | Covered by same test file as TEST-01 |
| TEST-03 | Provider factory test confirms `lmstudio` case returns valid provider | Add cases to `internal/llm/provider_test.go` — currently no lmstudio cases exist |
| TEST-04 | Integration test verifies end-to-end metadata enhancement with LM Studio (when available) | Add to `tests/integration/llm/provider_integration_test.go` |
</phase_requirements>

---

## Summary

Phase 1 delivered a functionally complete LM Studio provider implementation. Code inspection confirms that CONF-02 (TUI dropdown) is already satisfied — `forms.go` contains `huh.NewOption("LM Studio", "lmstudio")` at line 203. The `ValidateLLMProvider` function in `validation.go` also already includes `lmstudio` as a valid provider.

The remaining work for Phase 2 is almost entirely **test authoring**. The provider at `internal/llm/lmstudio.go` has 214 lines of substantive logic and zero dedicated unit test coverage. The test patterns are well-established: all sibling providers (openai, ollama, anthropic, google) use `net/http/httptest` servers in the same `package llm` (internal test package), and the factory tests live in `internal/llm/provider_test.go`.

There is one pre-existing bug (WR-01 from the Phase 1 review): `ValidateLLMProvider` is missing `"ollama"` from its valid providers map. The REQUIREMENTS.md marks fixing this as out of scope, but the test for TEST-03 should not mask this by being inconsistent — any new tests should only cover lmstudio scenarios.

CONF-04 is effectively pre-satisfied by the existing `domain.LLMError` wrapping in `lmstudio.go:88-95`: when the HTTP client fails (connection refused), it wraps the error with `Provider: "lmstudio"` and `Message: "request failed: <os-level error>"`. The UX question is whether the error message is user-friendly enough at the calling layer, which should be verified by reading the test scenario for connection-refused.

**Primary recommendation:** Write `internal/llm/lmstudio_test.go` following the `ollama_test.go` pattern, add two `lmstudio` cases to `provider_test.go`, and one integration test in `tests/integration/llm/provider_integration_test.go`.

---

## Standard Stack

### Core (all already in go.mod)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `net/http/httptest` | stdlib | Mock HTTP server for unit tests | Standard Go test pattern — used by all existing provider tests |
| `github.com/stretchr/testify` | v1.11.1 | `assert`, `require`, `ErrorAs`, `ErrorIs` | Project standard — used by every test file [VERIFIED: go.mod] |
| `context` | stdlib | Context cancellation tests | Used in all existing provider tests |

### Patterns In Use
| Pattern | Where | Notes |
|---------|-------|-------|
| Internal test package | `package llm` (same package) | Used by all `internal/llm/*_test.go` — allows access to unexported types like `openAIRequest`, `openAIResponse`, `openAIMessage` |
| External test package | `package llm_test` | Used by `tests/integration/llm/provider_integration_test.go` — public API only |
| `decodeJSON` helper | `testutil_test.go` | Available to all `package llm` tests — decodes request body for assertions |
| `testing.Short()` skip guard | All integration tests | `if testing.Short() { t.Skip(...) }` at top of each integration test |

---

## Architecture Patterns

### Test File Locations

```
internal/llm/
├── lmstudio.go            # implementation (214 lines, no unit tests yet)
├── lmstudio_test.go       # NEW — unit tests (package llm)
├── provider_test.go       # EDIT — add lmstudio cases
├── testutil_test.go       # decodeJSON helper (shared, package llm)
├── ollama_test.go         # reference pattern for no-auth provider
└── openai_test.go         # reference pattern for auth-required provider

tests/integration/llm/
└── provider_integration_test.go  # EDIT — add lmstudio integration test
```

### Pattern 1: httptest Unit Test (no-auth provider — matches lmstudio)

The Ollama tests are the closest model for LM Studio because both are local providers with no mandatory API key.

```go
// Source: internal/llm/ollama_test.go (adapted)
package llm

func TestLMStudioProvider_Complete_Success(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Verify request format
        assert.Equal(t, "POST", r.Method)
        assert.Equal(t, "/chat/completions", r.URL.Path)
        assert.Equal(t, "application/json", r.Header.Get("Content-Type"))
        assert.Empty(t, r.Header.Get("Authorization")) // no key case

        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"id":"test","model":"lmstudio-model","choices":[{"message":{"role":"assistant","content":"Hello"},"finish_reason":"stop"}],"usage":{"prompt_tokens":5,"completion_tokens":3,"total_tokens":8}}`))
    }))
    defer server.Close()

    provider, err := NewLMStudioProvider(ProviderConfig{
        BaseURL: server.URL,
        Model:   "lmstudio-model",
    }, server.Client())
    require.NoError(t, err)
    // ...
}
```

### Pattern 2: Auth Header Conditional Test

```go
// TEST-02: auth header present with key, absent without key
func TestLMStudioProvider_Complete_WithAPIKey(t *testing.T) {
    var receivedAuth string
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        receivedAuth = r.Header.Get("Authorization")
        // return valid response...
    }))
    defer server.Close()

    provider, err := NewLMStudioProvider(ProviderConfig{
        BaseURL: server.URL,
        Model:   "model",
        APIKey:  "my-token",
    }, server.Client())
    // ...
    assert.Equal(t, "Bearer my-token", receivedAuth)
}

func TestLMStudioProvider_Complete_WithoutAPIKey(t *testing.T) {
    // same but APIKey is "" and assert.Empty(t, receivedAuth)
}
```

### Pattern 3: Factory Test (provider_test.go additions)

```go
// TEST-03: in TestNewProviderFromConfig
{
    name: "valid lmstudio config",
    cfg: &config.LLMConfig{
        Provider: "lmstudio",
        BaseURL:  "http://localhost:1234/v1",
        Model:    "local-model",
        // No APIKey — that's the point
    },
    wantErr: nil,
},
{
    name: "lmstudio without api key",
    cfg: &config.LLMConfig{
        Provider: "lmstudio",
        Model:    "local-model",
    },
    wantErr: nil,
},
// Also in TestDefaultBaseURL:
{"lmstudio", DefaultLMStudioBaseURL},
// Also in TestNewProvider:
{
    name: "valid lmstudio",
    cfg:  ProviderConfig{Provider: "lmstudio", BaseURL: "http://localhost:1234/v1", Model: "x"},
    wantErr: false,
},
```

### Pattern 4: Connection Refused Test (CONF-04)

```go
// Connection refused — LM Studio server not running
func TestLMStudioProvider_Complete_ConnectionRefused(t *testing.T) {
    provider, err := NewLMStudioProvider(ProviderConfig{
        BaseURL: "http://localhost:19999", // nothing listening
        Model:   "model",
    }, &http.Client{Timeout: 1 * time.Second})
    require.NoError(t, err)

    _, err = provider.Complete(context.Background(), &domain.LLMRequest{
        Messages: []domain.LLMMessage{{Role: domain.RoleUser, Content: "Hi"}},
    })

    require.Error(t, err)
    var llmErr *domain.LLMError
    require.ErrorAs(t, err, &llmErr)
    assert.Equal(t, "lmstudio", llmErr.Provider)
    assert.Contains(t, llmErr.Message, "request failed")
    // The message wraps the OS error: "connect: connection refused"
    // This is acceptable for CONF-04 — the error is surfaced clearly
}
```

### Pattern 5: Integration Test Addition (TEST-04)

```go
// Source: tests/integration/llm/provider_integration_test.go pattern
func TestLMStudioProvider_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test in short mode")
    }
    // Uses httptest server — no live LM Studio required
    server := httptest.NewServer(/* success handler */)
    defer server.Close()

    cfg := &config.LLMConfig{
        Provider: "lmstudio",
        BaseURL:  server.URL,
        Model:    "test-model",
        // No APIKey
    }
    provider, err := llm.NewProviderFromConfig(cfg)
    require.NoError(t, err)
    assert.Equal(t, "lmstudio", provider.Name())

    resp, err := provider.Complete(context.Background(), &domain.LLMRequest{
        Messages: []domain.LLMMessage{{Role: domain.RoleUser, Content: "Summarize this"}},
    })
    require.NoError(t, err)
    assert.NotEmpty(t, resp.Content)
    assert.NoError(t, provider.Close())
}
```

### Anti-Patterns to Avoid

- **Fixing the Ollama TUI validation bug (WR-01)**: Out of scope per REQUIREMENTS.md. Do not add `"ollama"` to `ValidateLLMProvider` in this phase.
- **Testing live LM Studio**: TEST-04 should use `httptest` like the existing integration tests. The requirement says "when available" — use a mock server to make it always runnable.
- **Using `package llm_test` for unit tests**: Unit tests in `lmstudio_test.go` must use `package llm` (internal) to access `openAIRequest`, `openAIResponse`, `decodeJSON`, etc. Only the tests in `tests/integration/` use external packages.
- **Forgetting `testing.Short()` guard in integration tests**: All integration tests in `tests/integration/` have this guard.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Mock HTTP server | Custom TCP listener | `net/http/httptest.NewServer` | Standard Go stdlib, handles port allocation, cleanup |
| JSON body parsing in tests | Custom decoder | `decodeJSON` helper from `testutil_test.go` | Already available in `package llm` scope |
| Error type assertions | `errors.Is` chains | `require.ErrorAs(t, err, &llmErr)` then `assert.ErrorIs` | Testify pattern matches existing tests |

---

## Common Pitfalls

### Pitfall 1: Wrong package declaration for unit tests

**What goes wrong:** Declaring `package llm_test` in `lmstudio_test.go` — this prevents access to `openAIRequest`, `openAIResponse`, `openAIMessage`, and `decodeJSON` which are unexported.
**Why it happens:** External test package is the Go convention for integration-style tests.
**How to avoid:** Use `package llm` (matching all other `*_test.go` files in `internal/llm/`).
**Warning signs:** Compiler error "undefined: openAIRequest" or "undefined: decodeJSON".

### Pitfall 2: Missing lmstudio cases in TestDefaultBaseURL

**What goes wrong:** Only adding lmstudio cases to `TestNewProviderFromConfig` and `TestNewProvider` but forgetting `TestDefaultBaseURL`.
**Why it happens:** `TestDefaultBaseURL` is a small table test at the bottom of `provider_test.go` that is easy to miss.
**How to avoid:** Read all test tables in `provider_test.go` before writing. The `DefaultLMStudioBaseURL` constant already exists — add `{"lmstudio", DefaultLMStudioBaseURL}` to the table.

### Pitfall 3: CONF-02 is already done — do not re-implement

**What goes wrong:** Writing code to add "LM Studio" to the TUI dropdown, not noticing it was already done in Phase 1.
**Evidence:** `forms.go:203` — `huh.NewOption("LM Studio", "lmstudio")` is present and tested. `validation_test.go` already includes `"lmstudio"` in valid providers.
**How to avoid:** Verify with `grep "lmstudio" internal/tui/forms.go` before writing any TUI code.

### Pitfall 4: CONF-04 error message quality

**What goes wrong:** Assuming the connection-refused error message is user-friendly. The current message is: `lmstudio error: request failed: Post "http://localhost:1234/v1/chat/completions": dial tcp [::1]:1234: connect: connection refused`
**Assessment:** This is acceptable — it tells the user which address LM Studio should be at. CONF-04 says "helpful error message" — the address + connection refused is helpful. No change required unless tested and found confusing.
**How to avoid:** Write the connection-refused test first; if the message contains the address and "connection refused", the requirement is satisfied.

### Pitfall 5: openAIResponse type is in openai.go — share carefully

**What goes wrong:** Trying to re-declare `openAIRequest`/`openAIResponse` in `lmstudio_test.go`.
**Why it happens:** The lmstudio implementation reuses types from `openai.go`. These are package-level unexported types, already accessible since `lmstudio.go` and `openai.go` are in the same package.
**How to avoid:** In tests, construct the response JSON as a raw string literal (as `ollama_test.go` does) rather than using the response struct directly — this avoids coupling tests to internal struct layout.

---

## State of the Art

| Old Approach | Current Approach | Notes |
|--------------|------------------|-------|
| Provider tests using external test package | All `internal/llm/*_test.go` use `package llm` | Must match for access to unexported types |
| Writing response JSON via `json.NewEncoder` | Mix: some tests use `json.NewEncoder`, some use raw `w.Write([]byte(...))` | Either is fine; ollama_test.go uses raw string literals for compact tests |

---

## Current Coverage Gap (the work)

| File | Status | What Needs Writing |
|------|--------|-------------------|
| `internal/llm/lmstudio_test.go` | Does not exist | Full test suite: constructor, success, auth with key, auth without key, API error (401/429/503), empty choices, connection refused, context cancellation, invalid JSON response |
| `internal/llm/provider_test.go` | Exists, lmstudio missing | Add: `TestNewProviderFromConfig` cases (lmstudio with/without key, default base URL), `TestNewProvider` case, `TestDefaultBaseURL` case |
| `tests/integration/llm/provider_integration_test.go` | Exists | Add: `TestLMStudioProvider_Integration` with httptest server |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | CONF-02 is fully satisfied by Phase 1 work (no TUI changes needed in Phase 2) | Common Pitfalls #3 | If somehow forms.go was not committed, planner would need a TUI task — verify with grep before planning |
| A2 | CONF-04 is satisfied by existing error wrapping (connection-refused error is user-friendly enough) | Common Pitfalls #4 | If the calling layer suppresses the error message, planner would need a UX improvement task |

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Go toolchain | All tests | Yes | 1.26.1 | — |
| `net/http/httptest` | Unit/integration tests | Yes (stdlib) | — | — |
| `github.com/stretchr/testify` | All tests | Yes (go.mod) | v1.11.1 | — |
| Live LM Studio server | TEST-04 (optional) | Unknown | — | Use httptest mock (recommended) |

**TEST-04 does not require a live LM Studio server.** All existing integration tests use httptest mocks for provider tests. TEST-04 should follow this pattern — the "when available" clause in REQUIREMENTS.md is addressed by using a mock server that is always available.

---

## Sources

### Primary (HIGH confidence)

- `internal/llm/lmstudio.go` — verified implementation, conditional auth header at lines 84-86 [VERIFIED: codebase grep]
- `internal/llm/provider.go` — verified factory registration, `DefaultLMStudioBaseURL` constant, API key exemption [VERIFIED: codebase grep]
- `internal/tui/forms.go:203` — verified `huh.NewOption("LM Studio", "lmstudio")` present [VERIFIED: file read]
- `internal/tui/validation.go:139` — verified `"lmstudio": true` in ValidateLLMProvider [VERIFIED: file read]
- `internal/llm/ollama_test.go` — reference test pattern for no-auth local provider [VERIFIED: file read]
- `internal/llm/provider_test.go` — confirmed no lmstudio cases exist [VERIFIED: file read]
- `tests/integration/llm/provider_integration_test.go` — confirmed no lmstudio test exists [VERIFIED: file read]
- `tests/integration/llm/provider_integration_test.go` — confirmed test structure with `testing.Short()` guard [VERIFIED: file read]

### Secondary (MEDIUM confidence)

- Phase 1 VERIFICATION.md — confirms all CONF-02, CONF-03 requirements satisfied by Phase 1 [VERIFIED: file read]
- Phase 1 REVIEW.md WR-01 — Ollama bug in ValidateLLMProvider confirmed out of scope [VERIFIED: file read]

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already in project, versions verified from go.mod
- Architecture: HIGH — test patterns read directly from sibling test files
- Pitfalls: HIGH — grounded in direct code inspection of the current implementation
- CONF-02 status: HIGH — grep-verified in forms.go
- TEST gap inventory: HIGH — confirmed no lmstudio_test.go exists

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable domain — test patterns don't change)
