# Research Summary: LM Studio Provider

**Date:** 2026-04-07
**Confidence:** HIGH

## Executive Summary

Adding LM Studio as a named LLM provider in repodocs. LM Studio runs a local HTTP server at `http://localhost:1234/v1` implementing the OpenAI chat completions wire format. The correct approach is a standalone `LMStudioProvider` struct in `internal/llm/lmstudio.go` that reuses existing `openAIRequest`/`openAIResponse` types from the same package. No new dependencies required.

## Stack

- Zero new dependencies — stdlib only (`net/http`, `encoding/json`) plus existing domain types
- `openAIRequest`/`openAIResponse`/`openAIMessage` types reusable directly by `lmstudio.go`
- Default base URL: `http://localhost:1234/v1`

## Table Stakes Features

- `lmstudio` recognized in provider factory switch and `DefaultBaseURL`
- API key exemption in `NewProviderFromConfig` (same pattern as `"ollama"`)
- Conditional `Authorization: Bearer` header — only when `apiKey != ""`
- `provider.Name()` returns `"lmstudio"`
- `lmstudio` added to `ValidateLLMProvider` in `tui/validation.go`
- `lmstudio` option in TUI provider select form (`tui/forms.go`)
- Unit tests with httptest server (following `ollama_test.go`)

## Architecture

- Standalone `LMStudioProvider` struct — not a wrapper of `OpenAIProvider`
- Build order: `lmstudio.go` -> `provider.go` updates -> TUI files -> tests
- 5-7 files touched, all Low complexity, all following established patterns

## Critical Pitfalls

1. **API key validation gate** — `provider.go:53` must exempt `lmstudio` alongside `ollama`
2. **Empty Bearer token** — must conditionally omit header when no key configured
3. **60s HTTP timeout too short** — LM Studio cold-starts take 30-120+ seconds; default to 300s
4. **Both factory switches** — `DefaultBaseURL` and `NewProvider` must both add `lmstudio` atomically
5. **Flat-string 503 errors** — LM Studio may return non-OpenAI error format on 503

## Watch Out For

- LM Studio silently ignores `model` field when only one model loaded — don't validate response model matches request
- Pre-existing bug: `ValidateLLMProvider` in TUI doesn't include `ollama` — add `lmstudio` but don't fix `ollama` gap in this milestone
- Circuit breaker may open during cold-start if timeout is too short
