<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/config

YAML configuration management using Viper with environment variable support.

## Purpose

Handles application configuration loading from YAML files, environment variables (REPODOCS_* prefix), and CLI flag bindings. Provides validation and sensible defaults for all configuration options.

## Key Files

| File | Description |
|------|-------------|
| `config.go` | Config struct with nested configs (Output, Concurrency, Cache, Rendering, Stealth, Exclude, Logging, LLM, Git). Includes Validate() method and ParseSize/FormatSize helpers. |
| `loader.go` | Load() and LoadWithViper() functions. Config search: current directory first, then ~/.repodocs/. Env vars with REPODOCS_ prefix. |
| `defaults.go` | All default constants (timeouts, TTLs, concurrency, rate limits). DefaultExcludePatterns and DefaultDocPaths. |
| `config_test.go` | Tests for config validation and loading. |

## Config Types

- **LLMConfig**: Provider, APIKey, BaseURL, Model, MaxTokens, Temperature, Timeout, MaxRetries, RateLimit, CircuitBreaker
- **RateLimitConfig**: Enabled, RequestsPerMinute, BurstSize, MaxRetries, InitialDelay, MaxDelay, Multiplier, CircuitBreaker
- **CircuitBreakerConfig**: Enabled, FailureThreshold, SuccessThresholdHalfOpen, ResetTimeout
- **OutputConfig**: Directory, Flat, JSONMetadata, Overwrite
- **ConcurrencyConfig**: Workers, Timeout, MaxDepth
- **CacheConfig**: Enabled, TTL, Directory
- **RenderingConfig**: ForceJS, JSTimeout, ScrollToEnd
- **StealthConfig**: UserAgent, RandomDelayMin, RandomDelayMax
- **LoggingConfig**: Level, Format
- **GitConfig**: MaxFileSize

## Dependencies

- **External**: github.com/spf13/viper, gopkg.in/yaml.v3
- **Internal**: None

## For AI Agents

- Config search order: current directory → ~/.repodocs/
- Environment variables: REPODOCS_OUTPUT_DIRECTORY, REPODOCS_CACHE_TTL, etc. (nested via ".")
- Use ParseSize() for human-readable sizes (10MB, 1GB, 512KB)
- Validate() applies defaults for invalid values (workers < 1 → DefaultWorkers)
- Cache directory: ~/.repodocs/cache
- Config file: ~/.repodocs/config.yaml

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->