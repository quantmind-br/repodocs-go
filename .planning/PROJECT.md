# LM Studio Provider for repodocs

## What This Is

Adding first-class LM Studio support to repodocs as a dedicated LLM provider. LM Studio runs local models and exposes an OpenAI-compatible API. This gives users a local, free alternative for metadata enhancement without needing cloud API keys.

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

### Active

- [ ] Dedicated `lmstudio` provider recognized by the provider factory
- [ ] Default base URL `http://localhost:1234/v1` for LM Studio
- [ ] API key is optional (like Ollama) — works without authentication
- [ ] Optional API key support for secured LM Studio setups
- [ ] LM Studio provider uses OpenAI-compatible request/response format
- [ ] Config file support: `provider: lmstudio` in YAML config
- [ ] TUI config editor includes LM Studio as a provider option
- [ ] Config validation accepts `lmstudio` as valid provider
- [ ] Unit tests for LM Studio provider
- [ ] Integration test for LM Studio provider (when available)

### Out of Scope

- Streaming responses — not used by current metadata enhancement pipeline
- Model listing/auto-detection from LM Studio — users specify model name
- LM Studio installation or setup automation — users install separately
- Broader LLM use cases beyond metadata enhancement — same scope as existing providers

## Context

repodocs already has a well-structured LLM provider system (`internal/llm/`) with OpenAI, Anthropic, Google, and Ollama implementations. The provider factory in `provider.go` uses a switch statement to instantiate providers by name. LM Studio exposes an OpenAI-compatible API at `localhost:1234/v1`, so the implementation can reuse the OpenAI request/response types while having its own provider entry with appropriate defaults (no required API key, local base URL).

The Ollama provider already establishes the pattern for local providers: no API key required, localhost default URL. LM Studio follows the same pattern but uses the OpenAI API format rather than Ollama's native API.

## Constraints

- **API compatibility**: Must use OpenAI-compatible chat completions format (LM Studio's API)
- **Existing patterns**: Must follow the established provider implementation pattern (see `openai.go`, `ollama.go`)
- **Config structure**: Must integrate with existing `LLMConfig` struct and Viper config loading
- **No breaking changes**: Adding a new provider must not affect existing provider behavior

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Dedicated provider vs reuse OpenAI | First-class UX with sensible defaults (no key, local URL) is better than documenting workarounds | -- Pending |
| Reuse OpenAI request/response types | LM Studio is OpenAI-compatible; avoids code duplication | -- Pending |
| Optional API key (like Ollama) | LM Studio runs locally, auth is optional | -- Pending |

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
*Last updated: 2026-04-07 after initialization*
