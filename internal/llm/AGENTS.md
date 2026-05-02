<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-05-01 | Updated: 2026-05-01 -->

# internal/llm

Factory for 5 LLM providers with resilience: token bucket rate limiting, exponential backoff retry, circuit breaker. Optional metadata enhancement generates summaries/tags/categories.

## Structure

```
├── provider.go              # Factory: NewProvider(), NewProviderFromConfig()
├── provider_wrapper.go      # RateLimitedProvider: rate limit + retry + CB
├── openai.go                # OpenAI client
├── anthropic.go             # Anthropic client
├── google.go                # Google Gemini client
├── ollama.go                # Ollama local client
├── lmstudio.go              # LMStudio local client
├── retry.go                 # Exponential backoff with jitter
├── ratelimit.go             # Token bucket rate limiter
├── circuit_breaker.go       # Closed/open/half-open states
└── metadata.go              # MetadataEnhancer + JSON extraction
```

## Where to Look

| Task | File | Notes |
|------|------|-------|
| Add provider | New file + `provider.go` switch | Implement `domain.LLMProvider` |
| Fix retry | `retry.go` | `CalculateBackoff()` uses `cfg.JitterFactor` |
| Rate limit tuning | `provider_wrapper.go` | RPM, burst, max retries, circuit breaker config |
| Metadata prompt | `metadata.go` | `buildPrompt()` generates structured prompt |
| JSON extraction | `metadata.go` | `extractJSON()` handles code blocks, brace matching |
| Base URLs | `provider.go` | `DefaultBaseURL()` per provider |

## Conventions

- Provider constructor: `NewXProvider(cfg ProviderConfig, httpClient *http.Client)`
- Timeout defaults: 60s (300s for LMStudio)
- `NoOpRateLimiter` when RPM=0; `NoOpCircuitBreaker` when disabled
- Retry on: 429, 500, 502, 503, 504; NOT on context cancellation
- Metadata prompt requires JSON with `summary`, `tags`, `category`

## Anti-Patterns

- Avoid `LLMConfig.MaxRetries` — deprecated, use rate limit config
- Don't create providers directly; use `NewProviderFromConfig()` for validation
- Don't change metadata JSON shape without updating consumers/tests

## Gotchas

- 6 historical rate limit/circuit breaker bugs (all fixed, regression tests added)
- `CalculateBackoff` was hardcoded to 0.1 jitter, now uses config
- Ollama and LMStudio don't require API keys
- Metadata enhancement truncates content > 8000 chars


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
