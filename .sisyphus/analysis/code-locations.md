# Code Locations - Detailed Reference

## ConfigValues Struct (INCOMPLETE)

**File**: `internal/tui/config_adapter.go`
**Lines**: 12-47

```go
type ConfigValues struct {
    OutputDirectory string
    OutputFlat      bool
    OutputOverwrite bool
    JSONMetadata    bool

    Workers  int
    Timeout  string
    MaxDepth int

    CacheEnabled   bool
    CacheTTL       string
    CacheDirectory string

    ForceJS     bool
    JSTimeout   string
    ScrollToEnd bool

    UserAgent      string
    RandomDelayMin string
    RandomDelayMax string

    LogLevel  string
    LogFormat string

    LLMProvider        string
    LLMAPIKey          string
    LLMBaseURL         string
    LLMModel           string
    LLMMaxTokens       int
    LLMTemperature     float64
    LLMTimeout         string
    LLMEnhanceMetadata bool

    Exclude []string
    
    // ❌ MISSING:
    // - LLMMaxRetries
    // - LLMRateLimit* (all fields)
    // - LLMCircuitBreaker* (all fields)
}
```

**Missing Fields**:
- `LLMMaxRetries` (int)
- `LLMRateLimitEnabled` (bool)
- `LLMRateLimitRequestsPerMinute` (int)
- `LLMRateLimitBurstSize` (int)
- `LLMRateLimitMaxRetries` (int)
- `LLMRateLimitInitialDelay` (string)
- `LLMRateLimitMaxDelay` (string)
- `LLMRateLimitMultiplier` (float64)
- `LLMCircuitBreakerEnabled` (bool)
- `LLMCircuitBreakerFailureThreshold` (int)
- `LLMCircuitBreakerSuccessThresholdHalfOpen` (int)
- `LLMCircuitBreakerResetTimeout` (string)

---

## FromConfig() Function (INCOMPLETE)

**File**: `internal/tui/config_adapter.go`
**Lines**: 50-87

```go
func FromConfig(cfg *config.Config) *ConfigValues {
    return &ConfigValues{
        OutputDirectory: cfg.Output.Directory,
        OutputFlat:      cfg.Output.Flat,
        OutputOverwrite: cfg.Output.Overwrite,
        JSONMetadata:    cfg.Output.JSONMetadata,

        Workers:  cfg.Concurrency.Workers,
        Timeout:  formatDuration(cfg.Concurrency.Timeout),
        MaxDepth: cfg.Concurrency.MaxDepth,

        CacheEnabled:   cfg.Cache.Enabled,
        CacheTTL:       formatDuration(cfg.Cache.TTL),
        CacheDirectory: cfg.Cache.Directory,

        ForceJS:     cfg.Rendering.ForceJS,
        JSTimeout:   formatDuration(cfg.Rendering.JSTimeout),
        ScrollToEnd: cfg.Rendering.ScrollToEnd,

        UserAgent:      cfg.Stealth.UserAgent,
        RandomDelayMin: formatDuration(cfg.Stealth.RandomDelayMin),
        RandomDelayMax: formatDuration(cfg.Stealth.RandomDelayMax),

        LogLevel:  cfg.Logging.Level,
        LogFormat: cfg.Logging.Format,

        LLMProvider:        cfg.LLM.Provider,
        LLMAPIKey:          cfg.LLM.APIKey,
        LLMBaseURL:         cfg.LLM.BaseURL,
        LLMModel:           cfg.LLM.Model,
        LLMMaxTokens:       cfg.LLM.MaxTokens,
        LLMTemperature:     cfg.LLM.Temperature,
        LLMTimeout:         formatDuration(cfg.LLM.Timeout),
        LLMEnhanceMetadata: cfg.LLM.EnhanceMetadata,

        Exclude: cfg.Exclude,
        
        // ❌ MISSING:
        // cfg.LLM.MaxRetries
        // cfg.LLM.RateLimit.Enabled
        // cfg.LLM.RateLimit.RequestsPerMinute
        // cfg.LLM.RateLimit.BurstSize
        // cfg.LLM.RateLimit.MaxRetries
        // cfg.LLM.RateLimit.InitialDelay
        // cfg.LLM.RateLimit.MaxDelay
        // cfg.LLM.RateLimit.Multiplier
        // cfg.LLM.RateLimit.CircuitBreaker.Enabled
        // cfg.LLM.RateLimit.CircuitBreaker.FailureThreshold
        // cfg.LLM.RateLimit.CircuitBreaker.SuccessThresholdHalfOpen
        // cfg.LLM.RateLimit.CircuitBreaker.ResetTimeout
    }
}
```

**Problem**: Lines 76-85 copy LLM fields but skip RateLimit and MaxRetries.

---

## ToConfig() Function (INCOMPLETE)

**File**: `internal/tui/config_adapter.go`
**Lines**: 90-166

```go
func (v *ConfigValues) ToConfig() (*config.Config, error) {
    timeout, err := parseDurationOrDefault(v.Timeout, config.DefaultTimeout)
    if err != nil {
        return nil, fmt.Errorf("invalid timeout: %w", err)
    }

    cacheTTL, err := parseDurationOrDefault(v.CacheTTL, config.DefaultCacheTTL)
    if err != nil {
        return nil, fmt.Errorf("invalid cache_ttl: %w", err)
    }

    jsTimeout, err := parseDurationOrDefault(v.JSTimeout, config.DefaultJSTimeout)
    if err != nil {
        return nil, fmt.Errorf("invalid js_timeout: %w", err)
    }

    delayMin, err := parseDurationOrDefault(v.RandomDelayMin, config.DefaultRandomDelayMin)
    if err != nil {
        return nil, fmt.Errorf("invalid random_delay_min: %w", err)
    }

    delayMax, err := parseDurationOrDefault(v.RandomDelayMax, config.DefaultRandomDelayMax)
    if err != nil {
        return nil, fmt.Errorf("invalid random_delay_max: %w", err)
    }

    llmTimeout, err := parseDurationOrDefault(v.LLMTimeout, 30*time.Second)
    if err != nil {
        return nil, fmt.Errorf("invalid llm_timeout: %w", err)
    }

    cfg := &config.Config{
        Output: config.OutputConfig{
            Directory:    v.OutputDirectory,
            Flat:         v.OutputFlat,
            Overwrite:    v.OutputOverwrite,
            JSONMetadata: v.JSONMetadata,
        },
        Concurrency: config.ConcurrencyConfig{
            Workers:  v.Workers,
            Timeout:  timeout,
            MaxDepth: v.MaxDepth,
        },
        Cache: config.CacheConfig{
            Enabled:   v.CacheEnabled,
            TTL:       cacheTTL,
            Directory: v.CacheDirectory,
        },
        Rendering: config.RenderingConfig{
            ForceJS:     v.ForceJS,
            JSTimeout:   jsTimeout,
            ScrollToEnd: v.ScrollToEnd,
        },
        Stealth: config.StealthConfig{
            UserAgent:      v.UserAgent,
            RandomDelayMin: delayMin,
            RandomDelayMax: delayMax,
        },
        Logging: config.LoggingConfig{
            Level:  v.LogLevel,
            Format: v.LogFormat,
        },
        LLM: config.LLMConfig{
            Provider:        v.LLMProvider,
            APIKey:          v.LLMAPIKey,
            BaseURL:         v.LLMBaseURL,
            Model:           v.LLMModel,
            MaxTokens:       v.LLMMaxTokens,
            Temperature:     v.LLMTemperature,
            Timeout:         llmTimeout,
            EnhanceMetadata: v.LLMEnhanceMetadata,
            // ❌ MISSING:
            // MaxRetries: v.LLMMaxRetries,
            // RateLimit: config.RateLimitConfig{
            //     Enabled:           v.LLMRateLimitEnabled,
            //     RequestsPerMinute: v.LLMRateLimitRequestsPerMinute,
            //     BurstSize:         v.LLMRateLimitBurstSize,
            //     MaxRetries:        v.LLMRateLimitMaxRetries,
            //     InitialDelay:      delayMin,
            //     MaxDelay:          delayMax,
            //     Multiplier:        v.LLMRateLimitMultiplier,
            //     CircuitBreaker: config.CircuitBreakerConfig{
            //         Enabled:                  v.LLMCircuitBreakerEnabled,
            //         FailureThreshold:         v.LLMCircuitBreakerFailureThreshold,
            //         SuccessThresholdHalfOpen: v.LLMCircuitBreakerSuccessThresholdHalfOpen,
            //         ResetTimeout:             cbResetTimeout,
            //     },
            // },
        },
        Exclude: v.Exclude,
    }

    return cfg, nil
}
```

**Problem**: Lines 152-161 create LLM config but don't include RateLimit or MaxRetries. These fields default to zero values.

---

## Save() Function (FULL REPLACEMENT)

**File**: `internal/config/loader.go`
**Lines**: 156-172

```go
func Save(cfg *Config) error {
    if err := EnsureConfigDir(); err != nil {
        return fmt.Errorf("failed to create config directory: %w", err)
    }

    data, err := yaml.Marshal(cfg)  // ← Marshals entire config
    if err != nil {
        return fmt.Errorf("failed to marshal config: %w", err)
    }

    path := ConfigFilePath()
    if err := os.WriteFile(path, data, 0644); err != nil {  // ← Overwrites file
        return fmt.Errorf("failed to write config file: %w", err)
    }

    return nil
}
```

**Problem**: 
- Line 161: `yaml.Marshal(cfg)` marshals the entire config
- Line 167: `os.WriteFile()` overwrites the entire file
- No merge logic - original config is completely replaced
- Any fields not in ConfigValues are lost

---

## handleSave() in TUI App

**File**: `internal/tui/app.go`
**Lines**: 183-202

```go
func (m Model) handleSave() (tea.Model, tea.Cmd) {
    cfg, err := m.values.ToConfig()  // ← Creates NEW Config from ConfigValues
    if err != nil {
        m.state = stateError
        m.err = err
        return m, nil
    }

    if m.saveFunc != nil {
        if err := m.saveFunc(cfg); err != nil {  // ← Calls config.Save()
            m.state = stateError
            m.err = err
            return m, nil
        }
    }

    m.state = stateSaved
    m.dirty = false
    return m, nil
}
```

**Problem**: 
- Line 184: `ToConfig()` creates a new Config without RateLimit
- Line 191: `saveFunc(cfg)` saves the incomplete Config
- No merge with original config

---

## Config Struct Definition

**File**: `internal/config/config.go`
**Lines**: 6-15

```go
type Config struct {
    Output      OutputConfig      `mapstructure:"output"`
    Concurrency ConcurrencyConfig `mapstructure:"concurrency"`
    Cache       CacheConfig       `mapstructure:"cache"`
    Rendering   RenderingConfig   `mapstructure:"rendering"`
    Stealth     StealthConfig     `mapstructure:"stealth"`
    Exclude     []string          `mapstructure:"exclude"`
    Logging     LoggingConfig     `mapstructure:"logging"`
    LLM         LLMConfig         `mapstructure:"llm"`
}
```

---

## LLMConfig Struct Definition

**File**: `internal/config/config.go`
**Lines**: 18-29

```go
type LLMConfig struct {
    Provider        string          `mapstructure:"provider"`
    APIKey          string          `mapstructure:"api_key"`
    BaseURL         string          `mapstructure:"base_url"`
    Model           string          `mapstructure:"model"`
    MaxTokens       int             `mapstructure:"max_tokens"`
    Temperature     float64         `mapstructure:"temperature"`
    Timeout         time.Duration   `mapstructure:"timeout"`
    MaxRetries      int             `mapstructure:"max_retries"` // Deprecated
    EnhanceMetadata bool            `mapstructure:"enhance_metadata"`
    RateLimit       RateLimitConfig `mapstructure:"rate_limit"`  // ← NOT IN ConfigValues
}
```

---

## RateLimitConfig Struct Definition

**File**: `internal/config/config.go`
**Lines**: 32-41

```go
type RateLimitConfig struct {
    Enabled           bool                 `mapstructure:"enabled"`
    RequestsPerMinute int                  `mapstructure:"requests_per_minute"`
    BurstSize         int                  `mapstructure:"burst_size"`
    MaxRetries        int                  `mapstructure:"max_retries"`
    InitialDelay      time.Duration        `mapstructure:"initial_delay"`
    MaxDelay          time.Duration        `mapstructure:"max_delay"`
    Multiplier        float64              `mapstructure:"multiplier"`
    CircuitBreaker    CircuitBreakerConfig `mapstructure:"circuit_breaker"`
}
```

---

## CircuitBreakerConfig Struct Definition

**File**: `internal/config/config.go`
**Lines**: 44-49

```go
type CircuitBreakerConfig struct {
    Enabled                  bool          `mapstructure:"enabled"`
    FailureThreshold         int           `mapstructure:"failure_threshold"`
    SuccessThresholdHalfOpen int           `mapstructure:"success_threshold_half_open"`
    ResetTimeout             time.Duration `mapstructure:"reset_timeout"`
}
```

---

## Default Config with RateLimit

**File**: `internal/config/defaults.go`
**Lines**: 87-142

```go
func Default() *Config {
    return &Config{
        // ... other fields ...
        LLM: LLMConfig{
            MaxTokens:   DefaultLLMMaxTokens,
            Temperature: DefaultLLMTemperature,
            Timeout:     DefaultLLMTimeout,
            MaxRetries:  DefaultLLMMaxRetries,
            RateLimit: RateLimitConfig{
                Enabled:           DefaultRateLimitEnabled,           // true
                RequestsPerMinute: DefaultRateLimitRequestsPerMinute, // 60
                BurstSize:         DefaultRateLimitBurstSize,         // 10
                MaxRetries:        DefaultRateLimitMaxRetries,        // 3
                InitialDelay:      DefaultRateLimitInitialDelay,      // 1s
                MaxDelay:          DefaultRateLimitMaxDelay,          // 60s
                Multiplier:        DefaultRateLimitMultiplier,        // 2.0
                CircuitBreaker: CircuitBreakerConfig{
                    Enabled:                  DefaultCircuitBreakerEnabled,                  // true
                    FailureThreshold:         DefaultCircuitBreakerFailureThreshold,         // 5
                    SuccessThresholdHalfOpen: DefaultCircuitBreakerSuccessThresholdHalfOpen, // 1
                    ResetTimeout:             DefaultCircuitBreakerResetTimeout,             // 30s
                },
            },
        },
    }
}
```

---

## TUI Forms

**File**: `internal/tui/forms.go`

- `CreateOutputForm()` - Lines 9-40
- `CreateConcurrencyForm()` - Lines 41-76
- `CreateCacheForm()` - Lines 77-105
- `CreateRenderingForm()` - Lines 106-132
- `CreateStealthForm()` - Lines 133-164
- `CreateLoggingForm()` - Lines 165-194
- `CreateLLMForm()` - Lines 195-272

**Problem**: No form for RateLimit or CircuitBreaker settings.

---

## Test Files

### Integration Tests (config_integration_test.go)

**File**: `tests/integration/config/config_integration_test.go`

- `TestConfigSaveAndLoad()` - Lines 15-75
  - Tests direct Save/Load, not TUI flow
  - Doesn't test RateLimit preservation

- `TestRoundTripWithAllFields()` - Lines 113-178
  - Tests Save/Load round-trip
  - Includes RateLimit in test data
  - But doesn't test TUI flow (FromConfig → ToConfig)

### TUI Adapter Tests (config_adapter_test.go)

**File**: `internal/tui/config_adapter_test.go`

- `TestFromConfig()` - Lines 13-94
  - Tests FromConfig conversion
  - Doesn't include RateLimit in test data
  - Doesn't verify RateLimit is preserved

- `TestToConfig()` - Lines 96-171
  - Tests ToConfig conversion
  - Doesn't include RateLimit in test data
  - Doesn't verify RateLimit is reconstructed

**Problem**: No tests for TUI round-trip with RateLimit.

---

## Summary of Changes Needed

| File | Lines | Change |
|------|-------|--------|
| `internal/tui/config_adapter.go` | 12-47 | Add RateLimit/CircuitBreaker fields to ConfigValues |
| `internal/tui/config_adapter.go` | 50-87 | Copy RateLimit/CircuitBreaker in FromConfig() |
| `internal/tui/config_adapter.go` | 90-166 | Reconstruct RateLimit/CircuitBreaker in ToConfig() |
| `internal/tui/forms.go` | - | Add CreateLLMAdvancedForm() for RateLimit/CircuitBreaker |
| `internal/tui/categories.go` | - | Add "LLM Advanced" category |
| `internal/tui/config_adapter_test.go` | - | Add tests for RateLimit round-trip |
| `tests/integration/config/config_integration_test.go` | - | Add TUI round-trip test |
