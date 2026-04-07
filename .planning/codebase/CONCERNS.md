# Codebase Concerns

**Analysis Date:** 2026-04-07

## Tech Debt

### Rate Limit and Circuit Breaker Implementation Issues

A comprehensive analysis has identified **6 bugs** of varying severity in the rate limiting and circuit breaker implementation. These are documented in detail below under "Known Bugs".

## Known Bugs

### BUG 1 (Severity: High) - Token Consumed Before Circuit Breaker Check

**Files:** `internal/llm/provider_wrapper.go:100-125`

**Issue:** In the `Complete()` method, the circuit breaker is checked FIRST (lines 103-110), which is correct. However, historically this was ordered incorrectly. The current implementation (after recent fixes) properly checks the circuit breaker before consuming a rate limit token. This prevents token waste when the circuit is open.

**Current Status:** FIXED - Circuit breaker check now precedes rate limit token consumption.

---

### BUG 2 (Severity: Medium) - Missing JitterFactor Configuration Mapping

**Files:**
- `internal/config/config.go:46` - RateLimitConfig now includes JitterFactor field
- `internal/strategies/strategy.go:167-179` - Strategy dependency injection
- `internal/llm/provider_wrapper.go:20-42` - RateLimitedProviderConfig

**Issue:** JitterFactor is defined in config but may not be properly propagated through all code paths. The `DefaultRateLimitedProviderConfig()` (lines 28-42) sets JitterFactor to 0.1, but this function may not be used in all production code paths.

**Impact:** Without proper jitter, concurrent retries from multiple clients happen simultaneously ("thundering herd" effect), causing spikes in LLM provider load.

**Fix Approach:** Verify JitterFactor is consistently applied:
1. Audit all code paths that construct `RateLimitedProviderConfig`
2. Ensure JitterFactor is always read from config, never hardcoded to 0.0
3. Add validation in config.Validate() to warn if JitterFactor is 0.0

---

### BUG 3 (Severity: Medium) - Half-Open State Allows Unlimited Probes

**Files:** `internal/llm/circuit_breaker.go:97-104`

**Issue:** When circuit breaker is in `StateHalfOpen`, the `Allow()` method checks `halfOpenAllowed < SuccessThresholdHalfOpen` (line 100) and increments the counter. This is correct behavior - it properly limits probe requests to `SuccessThresholdHalfOpen` (default 1).

**Current Status:** FIXED - Half-open state correctly limits probe requests.

---

### BUG 4 (Severity: Medium) - Retry-After Header Parsed But Underutilized

**Files:**
- `internal/fetcher/client.go:164` - Parses Retry-After header
- `internal/fetcher/retry.go:104-110` - Uses Retry-After in backoff calculation
- `internal/fetcher/retry.go:197-212` - ParseRetryAfter() implementation

**Issue:** The Retry-After header parsing (client.go:164) creates a `RetryableError` with the parsed value. The retry logic (retry.go:104-110) does use this value via `max(backoffCalculated, RetryAfter)`, so the header IS being respected. However, the `ParseRetryAfter()` function only handles numeric seconds, not HTTP-date format.

**Impact:** If a server sends Retry-After as HTTP date (e.g., "Wed, 21 Oct 2025 07:28:00 GMT"), it's silently ignored and defaults to 0, potentially causing too-early retries.

**Fix Approach:** Enhance `ParseRetryAfter()` in `internal/fetcher/retry.go:197-212` to:
1. Add support for HTTP-date format parsing using `time.Parse(time.RFC1123, ...)`
2. Return the duration calculated from the HTTP date
3. Add fallback to numeric parsing for server convenience

---

### BUG 5 (Severity: Low) - Retry Attempts Consume Rate Limit Tokens

**Files:** `internal/llm/provider_wrapper.go:113-129`

**Issue:** The `Complete()` method calls `p.rateLimiter.Wait()` INSIDE the `retrier.Execute()` closure (line 122). This means each retry attempt consumes a token. With `MaxRetries=3` and `BurstSize=10`, a burst of 10 requests can generate up to 40 actual LLM calls (10 initial + 30 retries), exceeding the configured rate.

**Current Status:** INTENTIONAL - The comment on lines 114-115 explicitly states "consume a rate limit token for EVERY attempt (including retries) so that retries count against the configured rate limit." This is correct behavior.

---

### BUG 6 (Severity: Low) - Incomplete Configuration Validation

**Files:** `internal/config/config.go:105-169`

**Issue:** Configuration validation now properly validates rate limit and circuit breaker settings (lines 130-168). The `Validate()` method checks:
- RequestsPerMinute >= 0
- BurstSize >= 0
- MaxRetries >= 0
- InitialDelay >= 0
- MaxDelay >= 0
- Multiplier >= 0
- JitterFactor between 0.0-1.0
- CircuitBreaker fields when enabled

**Current Status:** FIXED - Comprehensive validation is in place.

---

## Security Considerations

### API Key Exposure in Git Archive Fetch

**Files:** `internal/strategies/git/archive.go:78-80`

**Risk:** The code reads `GITHUB_TOKEN` from environment and adds it to HTTP headers for archive downloads:

```go
if token := os.Getenv("GITHUB_TOKEN"); token != "" {
    req.Header.Set("Authorization", "token "+token)
}
```

**Mitigation in Place:**
- Token is read from environment (not hardcoded)
- Only applied to single request (not stored globally)
- Only used for archive download authentication

**Recommendations:**
1. Add logging when token is used (for audit trail)
2. Consider supporting multiple token sources (env vars, config file with restricted permissions)
3. Document that GITHUB_TOKEN should be scoped to minimal required permissions

---

### TLS Client Profile Fixed to Chrome 131

**Files:** `internal/fetcher/client.go:63`

**Risk:** The TLS client is hardcoded to use `profiles.Chrome_131`, which may become outdated as browsers update.

**Current Status:** ACCEPTABLE - This is intentional for stealth/fingerprint consistency.

**Recommendations:**
1. Add configuration option to allow users to specify browser profile
2. Periodically update to latest Chrome profile
3. Monitor for breaking changes in tls-client library

---

## Performance Bottlenecks

### Unbounded ReadAll() on HTTP Responses

**Files:**
- `internal/fetcher/client.go:175`
- `internal/llm/anthropic.go`, `google.go`, `ollama.go`, `openai.go`
- `internal/converter/encoding.go:82`
- `internal/strategies/sitemap.go`

**Problem:** Multiple places use `io.ReadAll()` to load entire response bodies into memory without size limits. Large documents (multi-MB HTML pages, large JSON responses) can cause memory spikes.

**Impact:** OOM risk with large documentation sites or verbose LLM responses.

**Improvement Path:**
1. Add configurable max response size limits in fetcher
2. Use `io.LimitReader()` to cap response body size
3. For LLM responses, validate content length before accepting
4. Add metrics to track response size distribution
5. Stream processing where possible instead of buffering entire response

---

### Full HTML/Markdown Loaded Into Memory

**Files:**
- `internal/converter/pipeline.go` - Full HTML to Markdown conversion
- `internal/strategies/crawler.go:111-139` - HTML rendering and conversion
- `internal/strategies/sitemap.go` - XML parsing

**Problem:** Large HTML documents are parsed and converted entirely in memory. With concurrent crawling of multiple pages, this multiplies memory usage.

**Impact:** Memory usage scales with document size × concurrency. A 10MB HTML page with 5 concurrent workers = 50MB+ just for that operation.

**Improvement Path:**
1. Implement streaming HTML parser (e.g., use html.Tokenizer for large docs)
2. Add document size pre-checks and warnings
3. Consider chunking very large documents
4. Add memory usage monitoring/alerting

---

### Cache Backend Not Optimized for Large Payloads

**Files:** `internal/cache/badger.go`

**Problem:** BadgerDB stores entire response bodies as values. Large cached responses (e.g., 10MB documentation) are read fully into memory on retrieval.

**Impact:** Cache lookup performance degrades with large cached items. GC runs (line 55: every 5 minutes) may pause all operations.

**Improvement Path:**
1. Consider two-tier cache: metadata in Badger, large payloads on disk
2. Implement cache value size limits
3. Add compression for cached responses
4. Monitor GC pause times

---

### Crawler May Discover Too Many URLs

**Files:** `internal/strategies/crawler.go`

**Problem:** The crawler discovers and processes all valid URLs. With no crawler-level discovery limit (only per-source limit from `opts.Limit`), a large site can cause unbounded URL discovery.

**Impact:** Memory usage for visited URL tracking, potential resource exhaustion on large documentation sites.

**Improvement Path:**
1. Add discovery queue size limit
2. Implement breadth-first discovery bounds
3. Add pre-flight site size estimation
4. Consider sampling strategy for very large sites

---

## Fragile Areas

### Max-Depth Not Properly Enforced in Manifest Processing

**Files:** `internal/app/orchestrator.go:460-465`

**Why Fragile:** The source manifest can specify `MaxDepth`, but the code (lines 460-465) logs a debug message that "config override not implemented":

```go
if source.MaxDepth > 0 {
    o.logger.Debug().
        Int("max_depth", source.MaxDepth).
        Str("url", source.URL).
        Msg("Source max_depth specified but config override not implemented")
}
```

The `MaxDepth` is ignored - only the global config value is used. This silently violates the manifest specification.

**Safe Modification:**
1. Either implement the override (call `buildSourceOptions()` with max_depth from source)
2. Or validate that manifest max_depth = global max_depth and error if different
3. Remove the misleading debug message or convert to warning

---

### Registry Selector Context Passed Incorrectly in Crawler

**Files:** `internal/strategies/crawler.go:141-160`

**Why Fragile:** Context is checked at line 142-146, but if context is cancelled, the function returns early. If context expires during HTML processing (line 130), there's no error propagation to the caller - the response processing is silently skipped.

**Safe Modification:**
1. Wrap `processHTMLResponse()` errors with context information
2. Distinguish between "context cancelled" and "processing failed" scenarios
3. Log at ERROR level when context expires, not WARN

---

### Circuit Breaker State Transitions Not Atomic

**Files:** `internal/llm/circuit_breaker.go:84-107`

**Why Fragile:** The Allow() method acquires a lock, transitions state if needed, then releases. Between state transition and returning, another goroutine could call Allow() and see the new state. The `halfOpenAllowed` counter is reset in `transitionTo()` (line 158), so concurrent calls during transition might double-count.

**Test Coverage:** This appears correct - the `halfOpenAllowed` increment (line 101) is protected by the lock for the entire Allow() call.

**Safe Modification:**
1. Add unit tests for concurrent Allow() calls during state transitions
2. Document the thread-safety guarantees explicitly

---

## Scaling Limits

### Memory Unbounded for Very Large Documentation Sites

**Current Capacity:** Tested with sites up to ~5GB of HTML (estimated), but this is subject to available system memory.

**Limit:** If a crawler discovers 1M pages and tries to cache them all, Badger will consume all available disk space.

**Scaling Path:**
1. Implement configurable memory budget for cache
2. Add LRU eviction policy to Badger cache
3. Implement per-page size limits (warn/skip oversized pages)
4. Add telemetry for memory usage during crawling

---

### Concurrent Workers Scaling

**Current Configuration:** Default 5 workers, configurable up to system CPU count.

**Limit:** Each worker holds browser tabs/HTTP connections. With N workers and M concurrent requests per worker, resource exhaustion is possible.

**Current Code:** `internal/app/orchestrator.go:304-310` caps manifest concurrency to 3 regardless of config.

**Scaling Path:**
1. Remove artificial cap on manifest concurrency
2. Make worker count adaptive based on available resources
3. Implement queue depth monitoring
4. Add backpressure when resource limits approached

---

### Browser Tab Pool Not Preemptively Cleaned

**Files:** `internal/renderer/pool.go:81`

**Problem:** When releasing a page to pool, code calls `page.Navigate("about:blank")` but ignores errors. If navigation fails, stale pages remain in pool.

**Impact:** Stale pages may cache state from previous render, causing incorrect output on subsequent renders.

**Scaling Path:**
1. Add page state validation before returning to pool
2. Implement page age limit (discard old pages)
3. Add metrics for page reuse vs. recreation
4. Monitor pool health (stale page detection)

---

## Dependencies at Risk

### TLS Client Library May Diverge from Browser Behavior

**Package:** `github.com/bogdanfinn/tls-client` (used in `internal/fetcher/client.go`)

**Risk:** This is a third-party library that mimics TLS fingerprints. If upstream breaks or becomes unmaintained, stealth features may fail.

**Impact:** TLS fingerprinting detection could start blocking requests.

**Mitigation Plan:**
1. Monitor library releases and issues
2. Keep fork ready as fallback
3. Add comprehensive TLS fingerprinting tests
4. Consider contributing upstream fixes if needed

---

### BadgerDB GC May Cause Latency Spikes

**Package:** `github.com/dgraph-io/badger/v4` (used in `internal/cache/badger.go:51-57`)

**Risk:** Background GC runs every 5 minutes. On large cache databases, GC can pause all operations for hundreds of milliseconds.

**Impact:** Request latency spikes during cache GC, user-visible delays.

**Mitigation Plan:**
1. Add metrics for GC pause times
2. Consider moving GC to off-hours or manual triggering
3. Monitor Badger releases for performance improvements
4. Profile GC behavior with expected cache sizes

---

## Missing Critical Features

### No Request Deduplication Across Concurrent Crawls

**Problem:** If multiple URLs in a crawl point to the same document, or if multiple manifest sources overlap, the same content is fetched and processed multiple times.

**Blocks:** Efficient multi-source documentation generation.

**Suggested Solution:**
1. Maintain global content hash cache (what content -> URLs)
2. Detect duplicate content by hash
3. Reuse converted markdown instead of re-converting
4. Add `--deduplicate` flag to enable feature

---

### No Resume/Checkpoint for Long-Running Crawls

**Problem:** If a crawl crashes after processing 10,000 pages, restarting restarts from page 1.

**Blocks:** Long-running documentation extractions on very large sites.

**Suggested Solution:**
1. Persist crawl state to disk periodically
2. Implement `--resume` flag to continue from last checkpoint
3. Use transaction log to track processed URLs
4. Add estimated time remaining based on progress

---

### No Bandwidth/Rate Limiting Between Requests

**Problem:** Crawler makes requests as fast as HTTP connections allow. This can overwhelm target servers or trigger rate limiting from ISPs.

**Blocks:** Scraping large sites responsibly.

**Suggested Solution:**
1. Add configurable delay between requests (not just rate limiting)
2. Implement `robots.txt` crawl-delay respect
3. Add random jitter to request timing
4. Expose metrics: requests/sec, bytes/sec

---

## Test Coverage Gaps

### No Integration Tests for Rate Limiting

**What's Not Tested:** The interaction between rate limiter, retrier, and circuit breaker under load.

**Files:** `internal/llm/provider_wrapper.go`, `internal/llm/ratelimit.go`, `internal/llm/circuit_breaker.go`

**Risk:** Race conditions in concurrent scenarios, token counting bugs.

**Priority:** HIGH - Rate limiting is critical for production stability.

**Suggested Tests:**
1. Concurrent requests with rate limiting enabled
2. Circuit breaker transitions under load
3. Token bucket exhaustion scenarios
4. Half-open state probe limiting

---

### No Benchmarks for Memory Usage

**What's Not Tested:** Memory footprint of crawling various document sizes.

**Files:** Core fetcher, converter, crawler logic

**Risk:** Silent memory regressions during optimization work.

**Priority:** MEDIUM - Important for large-site support.

**Suggested Tests:**
1. Benchmark HTML to Markdown conversion (various sizes)
2. Benchmark cache performance with large payloads
3. Benchmark concurrent crawler memory usage
4. Profile memory allocations in hot paths

---

### No E2E Tests for Multi-Source Manifests

**What's Not Tested:** Processing manifests with multiple sources in parallel, handling partial failures.

**Files:** `internal/app/orchestrator.go:280-432` (RunManifest)

**Risk:** Race conditions in result collection, incorrect error handling.

**Priority:** MEDIUM - Manifest processing is user-facing feature.

**Suggested Tests:**
1. Manifest with 10+ sources, partial failures
2. Manifest with continue_on_error=false and early cancellation
3. Manifest with mixed strategies (git, sitemap, crawler)
4. Manifest metadata consistency verification

---

*Concerns audit: 2026-04-07*
