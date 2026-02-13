# AGENTS.md - internal/fetcher

**Package:** `internal/fetcher`
**Purpose:** Stealth HTTP client with caching, bot avoidance, and retry logic.

## OVERVIEW
High-level HTTP client wrapping `tls-client` to bypass bot detection with integrated caching and exponential backoff.

## STRUCTURE
- `client.go`: Main `Client` implementation; manages caching, retries, and `tls-client` lifecycle.
- `stealth.go`: Bot avoidance logic; User-Agent rotation, TLS fingerprinting, and randomized header generation.
- `transport.go`: `StealthTransport` (implements `http.RoundTripper`) for integration with standard libraries or third-party tools like Colly.
- `retry.go`: Exponential backoff implementation using `cenkalti/backoff/v4`.

## WHERE TO LOOK
| Task | File |
|------|------|
| Modify User-Agent pool / Stealth headers | `stealth.go` |
| Adjust retry intervals / Multipliers | `retry.go` |
| Change caching persistence / TTL logic | `client.go` |
| Integrate with `http.Client` based tools | `transport.go` |

## KEY TYPES
- `Client`: Core orchestrator for all fetch operations.
- `ClientOptions`: Configuration struct (Timeout, Retries, Cache settings).
- `StealthTransport`: Adapter to use `fetcher.Client` as a standard `http.RoundTripper`.
- `Retrier`: Encapsulates backoff state and logic.

## CONVENTIONS
- **Decoupled Responses**: Methods return `domain.Response` to avoid leaking `fhttp` or `http` internals.
- **TLS Fingerprinting**: Defaults to `profiles.Chrome_131` via `tls-client` to mimic modern browsers.
- **Context Awareness**: All fetch operations MUST accept and respect `context.Context` for cancellation/timeouts.
- **Retry Logic**: Only errors marked as `domain.IsRetryable(err)` are retried.

## ANTI-PATTERNS
- **NO `net/http.DefaultClient`**: Bypasses all stealth and fingerprinting features.
- **NO Manual Decompression**: `tls-client` handles this; `StealthTransport` strips `Content-Encoding` to prevent double-decompression errors in callers.
- **Avoid Static Headers**: Use `StealthHeaders()` to ensure randomized, consistent header sets.
- **No Hardcoded Delays**: Use `RandomDelay()` from `stealth.go` for human-like pacing.
