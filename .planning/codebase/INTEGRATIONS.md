# External Integrations

**Analysis Date:** 2026-04-07

## APIs & External Services

**Large Language Models (LLM):**
- OpenAI - Code generation and metadata enhancement
  - SDK/Client: HTTP client in `internal/llm/openai.go`
  - Auth: `REPODOCS_LLM_API_KEY` environment variable
  - Endpoint: Configurable via `base_url` in config (default: `https://api.openai.com/v1`)
  - Configuration: `config.llm.provider=openai`

- Anthropic Claude - Metadata and content analysis
  - SDK/Client: HTTP client in `internal/llm/anthropic.go`
  - Auth: `REPODOCS_LLM_API_KEY` environment variable
  - Endpoint: Configurable (default: `https://api.anthropic.com/v1`)
  - Configuration: `config.llm.provider=anthropic`
  - Version: Uses Anthropic API version 2023-06-01

- Google Gemini - Alternative LLM provider
  - SDK/Client: HTTP client in `internal/llm/google.go`
  - Auth: `REPODOCS_LLM_API_KEY` environment variable
  - Endpoint: `https://generativelanguage.googleapis.com`
  - Configuration: `config.llm.provider=google`

- OpenAI-Compatible (Ollama, vLLM, Groq, Together AI)
  - SDK/Client: HTTP client in `internal/llm/openai.go` (reused for compatibility)
  - Auth: `REPODOCS_LLM_API_KEY` environment variable (may not be required for local Ollama)
  - Endpoint: Customizable via `base_url`
  - Examples:
    - Ollama: `http://localhost:11434/v1`
    - vLLM: `http://localhost:8000/v1`
    - Groq: `https://api.groq.com/openai/v1`
    - Together AI: `https://api.together.xyz/v1`
  - Configuration: `config.llm.provider=openai` with custom `base_url`

- Ollama (Local LLM)
  - SDK/Client: HTTP client in `internal/llm/ollama.go`
  - Auth: Not required (local service)
  - Endpoint: Default `http://localhost:11434`
  - Configuration: `config.llm.provider=ollama`

## Data Storage

**Databases:**
- None (application uses only local cache, no persistent database)

**Caching:**
- BadgerDB (embedded key-value store)
  - Type: Embedded database
  - Storage: `~/.repodocs/cache/` directory (configurable)
  - Client: `github.com/dgraph-io/badger/v4`
  - Implementation: `internal/cache/badger.go`
  - Features:
    - Persistent key-value cache with TTL support
    - Automatic garbage collection (runs every 5 minutes)
    - Can run in-memory for testing
    - HTTP response caching enabled by default (24-hour TTL)

**File Storage:**
- Local filesystem only
  - Output: `./docs` or user-specified directory (via `-o` flag)
  - Configuration: `config.output.directory`
  - Cache: `~/.repodocs/cache/`
  - Config: `~/.repodocs/config.yaml`

## Authentication & Identity

**Auth Provider:**
- No central authentication system
- Per-integration API keys stored in configuration or environment variables:
  - OpenAI: `REPODOCS_LLM_API_KEY`
  - Anthropic: `REPODOCS_LLM_API_KEY`
  - Google: `REPODOCS_LLM_API_KEY`
  - Ollama: None required

**Implementation:**
- Environment variable injection: `REPODOCS_*` prefix binds to config values
- Config file storage: `~/.repodocs/config.yaml` (user-maintained)
- No OAuth/OIDC, API key-based authentication only

## Monitoring & Observability

**Error Tracking:**
- None (no external error tracking service integrated)
- Error handling via domain errors in `internal/domain/errors.go`

**Logs:**
- Structured logging via zerolog
  - Output: stdout/stderr
  - Levels: debug, info, warn, error
  - Formats: text (pretty) or JSON
  - Configuration: `config.logging.level` and `config.logging.format`

**Metrics:**
- Progress bar display via schollz/progressbar
  - Used during document processing
  - No external metrics collection

## CI/CD & Deployment

**Hosting:**
- Self-hosted (no cloud hosting dependency)
- Binary distribution via GitHub releases

**CI Pipeline:**
- GitHub Actions (`.github/workflows/ci.yml` and `release.yml`)
- Automated linting, testing, and releases

**Build & Distribution:**
- goreleaser v2 for multi-platform builds
  - Creates binaries for: Linux, macOS, Windows
  - Both x86_64 and ARM architectures
  - Run with `make release-dry` for dry-run

## Environment Configuration

**Required env vars (for LLM usage):**
- `REPODOCS_LLM_PROVIDER` - LLM provider type (openai, anthropic, google, ollama)
- `REPODOCS_LLM_API_KEY` - API key (except for local Ollama)
- `REPODOCS_LLM_MODEL` - Model ID/name

**Optional env vars:**
- `REPODOCS_LLM_BASE_URL` - Custom API endpoint
- `REPODOCS_LLM_MAX_TOKENS` - Max response tokens
- `REPODOCS_LLM_TEMPERATURE` - Response randomness
- `REPODOCS_CACHE_ENABLED` - Enable/disable caching
- `REPODOCS_CACHE_TTL` - Cache time-to-live
- `REPODOCS_CONCURRENCY_WORKERS` - Parallel fetch workers
- All `config.yaml` values accessible via `REPODOCS_*` prefix

**Secrets location:**
- Environment variables (recommended)
- Configuration file `~/.repodocs/config.yaml` (user-managed, not committed)
- No built-in secret management; users responsible for securing API keys

## HTTP Clients & Network

**Primary HTTP Client:**
- tls-client (bogdanfinn/tls-client v1.11.2)
  - TLS fingerprint spoofing to mimic Chrome browser
  - Chrome profile 131 TLS configuration
  - Random TLS extension ordering (anti-detection)
  - Custom User-Agent support
  - Location: `internal/fetcher/client.go`

**Secondary HTTP Features:**
- Stealth mode with:
  - Custom User-Agent headers
  - Random delays (configurable: 500ms - 2s default)
  - Retry with exponential backoff (cenkalti/backoff v4.3.0)
  - Automatic redirect following (configurable)

**Browser Automation:**
- Rod v0.116.2 for headless Chrome/Chromium
  - DevTools Protocol communication
  - JavaScript rendering and DOM manipulation
  - Anti-automation detection bypass (rod/stealth)
  - Configurable timeout (default: 60s)
  - Tab pool for concurrent page rendering
  - Location: `internal/renderer/rod.go`

## Web Technologies Parsed

**Source Types Supported:**
- HTML websites (via crawler strategy)
- Git repositories (via go-git)
- Sitemaps (XML parsing)
- pkg.go.dev (Go package documentation)
- llms.txt files (custom format)
- Markdown files within repositories
- JavaScript-rendered content (via Rod)

**Content Extraction:**
- CSS selector-based extraction (configurable)
- Readability algorithm (go-readability)
- HTML sanitization (remove scripts, ads, navigation)
- HTML to Markdown conversion (html-to-markdown v2)

## Webhooks & Callbacks

**Incoming:**
- None

**Outgoing:**
- None

## Rate Limiting & Circuit Breaker

**LLM Request Management:**
- Rate limiting (optional, enabled by default)
  - `config.llm.rate_limit.enabled` - Enable/disable
  - `config.llm.rate_limit.requests_per_minute` - Throttle per minute (default: 60)
  - `config.llm.rate_limit.burst_size` - Max concurrent requests (default: 10)
  - `config.llm.rate_limit.max_retries` - Retry attempts (default: 3)
  - Implementation: `internal/llm/ratelimit.go`

- Circuit breaker (optional, enabled by default)
  - `config.llm.rate_limit.circuit_breaker.enabled` - Enable/disable
  - `config.llm.rate_limit.circuit_breaker.failure_threshold` - Failures before open (default: 5)
  - `config.llm.rate_limit.circuit_breaker.success_threshold_half_open` - Successes to close (default: 1)
  - `config.llm.rate_limit.circuit_breaker.reset_timeout` - Time to retry (default: 30s)
  - Implementation: `internal/llm/circuit_breaker.go`

- Retry strategy
  - Initial delay: 1s (configurable)
  - Max delay: 60s (configurable)
  - Multiplier: 2.0x backoff (configurable)
  - Jitter: 0.1 factor (configurable)

## Web Scraping Features

**Site Detection:**
- URL-to-strategy mapping in `internal/app/detector.go`
- Detection order: LLMS → PkgGo → Sitemap → Git → Crawler

**Anti-Bot Measures:**
- TLS fingerprint spoofing (bogdanfinn/tls-client)
- Chrome-like headers via fhttp
- Custom User-Agent headers
- Random delays between requests
- Rate limiting and backoff
- Stealth browser mode (disable automation detection)

---

*Integration audit: 2026-04-07*
