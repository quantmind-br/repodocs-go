---
phase: 01-provider-core
reviewed: 2026-04-07T00:00:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - internal/llm/lmstudio.go
  - internal/llm/provider.go
  - internal/tui/validation.go
  - internal/tui/forms.go
  - internal/tui/validation_test.go
findings:
  critical: 0
  warning: 1
  info: 2
  total: 3
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-04-07
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 01 adds LM Studio as an LLM provider and updates the TUI to include it in the provider dropdown and validation. The `LMStudioProvider` implementation in `lmstudio.go` closely follows the established `OpenAIProvider` pattern, reusing the same wire-format types. The provider factory in `provider.go` correctly exempts LM Studio from API key requirements (like Ollama) and uses a sensible 300-second default timeout for local model inference. The TUI validation and forms are updated to include LM Studio.

One notable bug was found: the `ValidateLLMProvider` function is missing "ollama" from its valid providers map, meaning Ollama -- which IS present in the TUI dropdown -- would fail validation if validated through this function.

## Warnings

### WR-01: ValidateLLMProvider missing "ollama" from valid providers

**File:** `internal/tui/validation.go:135-144`
**Issue:** The `ValidateLLMProvider` function's `validProviders` map includes `openai`, `anthropic`, `google`, and `lmstudio`, but omits `ollama`. However, `forms.go:204` includes "Ollama" as a selectable option in the LLM provider dropdown. If a user selects Ollama and the value passes through `ValidateLLMProvider`, it will be rejected as invalid. The test at `validation_test.go:198` also does not include `"ollama"` in its valid providers list, masking this bug.
**Fix:**
```go
validProviders := map[string]bool{
    "openai":    true,
    "anthropic": true,
    "google":    true,
    "ollama":    true,
    "lmstudio":  true,
}
```
Also update the error message to include "ollama":
```go
return fmt.Errorf("invalid LLM provider: must be openai, anthropic, google, ollama, or lmstudio")
```
And add `"ollama"` to the test's `validProviders` slice in `validation_test.go:198`.

## Info

### IN-01: Substantial code duplication between LMStudioProvider and OpenAIProvider

**File:** `internal/llm/lmstudio.go:46-168`
**Issue:** The `Complete` method is nearly line-for-line identical to `OpenAIProvider.Complete` in `openai.go:76-186`. The only meaningful differences are: (1) conditional Authorization header (line 84-86), and (2) special handling for 503 "no model" responses (lines 106-117). This is consistent with the project's existing pattern where each provider has its own implementation, but the duplication is significant (~120 lines). Consider extracting a shared `openAICompatibleComplete` helper if more OpenAI-compatible providers are added in the future.
**Fix:** No immediate action required. This follows the established project pattern. If a third OpenAI-compatible provider is added, refactoring to a shared base would reduce maintenance burden.

### IN-02: Redundant error handling between Complete method and handleHTTPError

**File:** `internal/llm/lmstudio.go:120-148` and `internal/llm/lmstudio.go:175-213`
**Issue:** The `Complete` method handles 401, 429, and 503 status codes inline (lines 120-143) when `openAIResp.Error` is not nil. The `handleHTTPError` method (lines 175-213) also handles these same status codes. In practice, `handleHTTPError` is only reached when JSON parsing succeeds but there is no `error` field in the response, meaning the inline handling in `Complete` takes precedence for error responses. The 401 and 429 cases in `handleHTTPError` are effectively dead code when the server returns a well-formed OpenAI error response. This mirrors the same pattern in `openai.go` and is not a bug, but it increases maintenance surface.
**Fix:** No immediate action required. The redundancy serves as a safety net for edge cases where the server returns a non-200 status without a JSON error field. Same pattern exists in `openai.go`.

---

_Reviewed: 2026-04-07_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
