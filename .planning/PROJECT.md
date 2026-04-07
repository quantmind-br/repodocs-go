# LM Studio Provider for repodocs

## What This Is

First-class LM Studio support in repodocs as a dedicated LLM provider. LM Studio runs local models and exposes an OpenAI-compatible API. Users get a local, free alternative for metadata enhancement without needing cloud API keys. Shipped in v1.0 with zero-config defaults, full test coverage, and TUI integration.

## Core Value

Users can run repodocs metadata enhancement with local LLM models via LM Studio, with zero-config defaults (no API key required, auto-detected localhost URL).

## Requirements

### Validated

- ✓ OpenAI provider with custom base URL support — existing
- ✓ Ollama provider with no-key-required pattern — existing
- ✓ Provider factory pattern in `internal/llm/provider.go` — existing
- ✓ LLM config structure with provider/api_key/base_url/model fields — existing
- ✓ Metadata enhancement pipeline using LLM providers — existing
- ✓ TUI config editor with provider categories — existing
- ✓ Config validation and defaults system — existing
- ✓ Dedicated `lmstudio` provider recognized by the provider factory — v1.0
- ✓ Default base URL `http://localhost:1234/v1` for LM Studio — v1.0
- ✓ API key is optional (like Ollama) — works without authentication — v1.0
- ✓ Optional API key support for secured LM Studio setups — v1.0
- ✓ LM Studio provider uses OpenAI-compatible request/response format — v1.0
- ✓ Config file support: `provider: lmstudio` in YAML config — v1.0
- ✓ TUI config editor includes LM Studio as a provider option — v1.0
- ✓ Config validation accepts `lmstudio` as valid provider — v1.0
- ✓ Unit tests for LM Studio provider (14 unit + 4 factory tests) — v1.0
- ✓ Integration test for LM Studio provider (httptest mock server) — v1.0

### Active

(None — next milestone requirements TBD)

### Out of Scope

- Streaming responses — not used by current metadata enhancement pipeline
- Model listing/auto-detection from LM Studio — users specify model name
- LM Studio installation or setup automation — users install separately
- Broader LLM use cases beyond metadata enhancement — same scope as existing providers

## Context

Shipped v1.0 with 1,039 lines of Go added across 9 files.
Tech stack: Go 1.24.1, Cobra, Viper, Charmbracelet TUI, httptest.
Provider system: `internal/llm/` now has 5 providers — OpenAI, Anthropic, Google, Ollama, and LM Studio.
LM Studio reuses OpenAI wire format types with its own provider entry, zero-config defaults (localhost:1234/v1, no API key), and 300s timeout for cold-start tolerance.
Test coverage: 14 unit tests + 4 factory tests + 1 integration test for LM Studio provider.

## Constraints

- **API compatibility**: Must use OpenAI-compatible chat completions format (LM Studio's API)
- **Existing patterns**: Must follow the established provider implementation pattern (see `openai.go`, `ollama.go`)
- **Config structure**: Must integrate with existing `LLMConfig` struct and Viper config loading
- **No breaking changes**: Adding a new provider must not affect existing provider behavior

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Dedicated provider vs reuse OpenAI | First-class UX with sensible defaults (no key, local URL) is better than documenting workarounds | ✓ Good — shipped v1.0 |
| Reuse OpenAI request/response types | LM Studio is OpenAI-compatible; avoids code duplication | ✓ Good — zero duplication |
| Optional API key (like Ollama) | LM Studio runs locally, auth is optional | ✓ Good — conditional auth works |
| 300s default timeout | Local models have cold-start loading time | ✓ Good — accommodates slow first inference |
| LM Studio-specific 503 error handling | LM Studio returns plain-text 503 when no model loaded | ✓ Good — clear UX error message |
| Skip ollama TUI validation fix | Pre-existing gap, out of scope for this milestone | — Deferred to future milestone |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? -> Move to Out of Scope with reason
2. Requirements validated? -> Move to Validated with phase reference
3. New requirements emerged? -> Add to Active
4. Decisions to log? -> Add to Key Decisions
5. "What This Is" still accurate? -> Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check -- still the right priority?
3. Audit Out of Scope -- reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-07 after v1.0 milestone*
