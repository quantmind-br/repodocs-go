<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-07 -->

# internal/llm

Provider factory + resilience wrappers + metadata enrichment. This package owns all LLM-specific runtime behavior.

## Providers

| Provider | Files | Notes |
|----------|-------|-------|
| OpenAI | `openai.go` | Uses `DefaultOpenAIBaseURL` |
| Anthropic | `anthropic.go` | Uses `DefaultAnthropicBaseURL` |
| Google | `google.go` | Uses `DefaultGoogleBaseURL` |
| Ollama | `ollama.go` | Local provider; API key not required |
| LMStudio | `lmstudio.go` | Local provider; OpenAI-compatible API; zero-config defaults (localhost) |

Factory entrypoints: `NewProviderFromConfig()` and `NewProvider()` in `provider.go`.

## Contract

All providers implement `domain.LLMProvider`:

```go
Name() string
Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error)
Close() error
```

## Runtime Layers

```text
base provider
  → optional RateLimitedProvider
  → retry / rate limit / circuit breaker behavior
  → MetadataEnhancer
```

| Component | File | Purpose |
|-----------|------|---------|
| Provider factory | `provider.go` | Base URLs, provider selection, config validation |
| Retry | `retry.go` | Backoff with jitter |
| Rate limit | `ratelimit.go` | Request pacing / token-bucket behavior |
| Circuit breaker | `circuit_breaker.go` | Open/half-open/closed guardrail |
| Wrapper | `provider_wrapper.go` | Compose resilience around any provider |
| Metadata | `metadata.go` | Strict JSON extraction prompt + enhancer |

## Where to Look

| Task | File |
|------|------|
| Add provider | `<provider>.go` + `provider.go` switch + `DefaultBaseURL()` |
| Change base URL defaults | `provider.go` constants |
| Tune retry/rate-limit/circuit breaker | `retry.go`, `ratelimit.go`, `circuit_breaker.go` |
| Change summary/tag/category extraction | `metadata.go` |

## Project-Specific Notes

- `ollama` and `lmstudio` are the only providers allowed without an API key.
- `lmstudio` uses OpenAI-compatible chat completions format; zero-config defaults to localhost.
- `internal/config.LLMConfig.MaxRetries` is deprecated; prefer `RateLimit.MaxRetries`.
- Metadata prompt is intentionally strict: valid JSON only, exactly `summary`, `tags`, `category`.
- `NewDependencies()` in `internal/strategies/strategy.go` decides whether providers/wrappers are enabled.

## Anti-Patterns

- Do not hardcode provider selection outside `provider.go`.
- Do not bypass wrappers if the caller expects configured rate limiting / circuit breaking.
- Do not change metadata JSON shape without updating downstream consumers/tests.


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
