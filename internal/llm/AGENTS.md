# AGENTS.md - internal/llm

**Generated:** 2026-02-20 | **Package:** internal/llm

Multi-provider LLM abstraction with circuit breaker resilience and rate limiting.

## Providers

| Provider | File | Model Default | Notes |
|----------|------|---------------|-------|
| OpenAI | `openai.go` | gpt-4o-mini | Chat completions API |
| Anthropic | `anthropic.go` | claude-3-haiku | Messages API |
| Google | `google.go` | gemini-1.5-flash | Generative Language API |

All implement `domain.LLMProvider`: `Name()`, `Complete(ctx, prompt)`, `Close()`.

## Resilience Layer

```
LLMProvider → RateLimitedProvider → CircuitBreaker → RateLimiter → Retrier
```

| Component | File | Purpose |
|-----------|------|---------|
| Circuit Breaker | `circuit_breaker.go` | Fail-fast on repeated errors (Closed→Open→HalfOpen) |
| Rate Limiter | `ratelimit.go` | Token bucket, requests/minute + burst |
| Retrier | `retry.go` | Exponential backoff with jitter |
| Wrapper | `provider_wrapper.go` | Composes all into `RateLimitedProvider` |

## Where to Look

| Task | File |
|------|------|
| Add new LLM provider | Create `<provider>.go`, update `provider.go` factory |
| Change retry/backoff | `retry.go` |
| Tune circuit breaker | `circuit_breaker.go` (`DefaultCircuitBreakerConfig`) |
| Modify metadata prompts | `metadata.go` (`metadataSystemPrompt`) |
| Wrap provider with resilience | `provider_wrapper.go` (`NewRateLimitedProvider`) |

## Adding a Provider

1. Create `<provider>.go` implementing `domain.LLMProvider`
2. Implement `Name()`, `Complete(ctx, prompt)`, `Close()`
3. Add constructor `New<Provider>Provider(cfg ProviderConfig)`
4. Register in `provider.go` switch (`NewProviderFromConfig`)
5. Add default base URL in `DefaultBaseURL()`

## Key Types

```go
type ProviderConfig struct {
    Provider, APIKey, BaseURL, Model string
    MaxTokens int; Temperature float64
}

type RateLimitedProvider struct { /* wraps any LLMProvider with resilience */ }
type MetadataEnhancer struct { /* uses LLMProvider for summaries/tags */ }
```
