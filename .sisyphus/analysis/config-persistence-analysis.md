# Config Persistence Analysis: repodocs config TUI

## Executive Summary

The `repodocs config` command uses a TUI (Terminal User Interface) to edit configuration. The persistence mechanism has **critical gaps** where nested complex fields are **not exposed in the TUI** and therefore **cannot be edited or persisted** through the interactive interface.

**Risk Level: HIGH** - Users can lose configuration for advanced features when using the TUI.

---

## Architecture Overview

### Load → Edit → Save Flow

```
config.Load()
    ↓
FromConfig(cfg) → ConfigValues (form-friendly struct)
    ↓
TUI Forms (user edits ConfigValues)
    ↓
ToConfig() → config.Config (reconstructed)
    ↓
config.Save(cfg) → YAML marshal → file write
```

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **Config struct** | `internal/config/config.go` | Full config model with all fields |
| **ConfigValues struct** | `internal/tui/config_adapter.go` | Form-friendly subset of Config |
| **FromConfig()** | `internal/tui/config_adapter.go` | Config → ConfigValues (load for editing) |
| **ToConfig()** | `internal/tui/config_adapter.go` | ConfigValues → Config (save) |
| **Save()** | `internal/config/loader.go` | YAML marshal + file write |
| **TUI Forms** | `internal/tui/forms.go` | Form definitions for each category |

---

## Critical Finding: Missing Fields in TUI

### Fields NOT Exposed in ConfigValues

The `ConfigValues` struct in `internal/tui/config_adapter.go` is missing:

#### 1. **LLM.RateLimit** (entire nested struct)
```go
// In config.Config:
type LLMConfig struct {
    RateLimit RateLimitConfig `mapstructure:"rate_limit"`  // ← NOT IN ConfigValues
}

// In ConfigValues:
type ConfigValues struct {
    // ... no RateLimit fields at all
}
```

**Impact**: Users cannot edit:
- `llm.rate_limit.enabled`
- `llm.rate_limit.requests_per_minute`
- `llm.rate_limit.burst_size`
- `llm.rate_limit.max_retries`
- `llm.rate_limit.initial_delay`
- `llm.rate_limit.max_delay`
- `llm.rate_limit.multiplier`

#### 2. **LLM.RateLimit.CircuitBreaker** (nested within RateLimit)
```go
// In RateLimitConfig:
type RateLimitConfig struct {
    CircuitBreaker CircuitBreakerConfig `mapstructure:"circuit_breaker"`  // ← NOT IN ConfigValues
}

// In ConfigValues:
// ... no CircuitBreaker fields at all
```

**Impact**: Users cannot edit:
- `llm.rate_limit.circuit_breaker.enabled`
- `llm.rate_limit.circuit_breaker.failure_threshold`
- `llm.rate_limit.circuit_breaker.success_threshold_half_open`
- `llm.rate_limit.circuit_breaker.reset_timeout`

#### 3. **LLM.MaxRetries** (deprecated but still in struct)
```go
// In config.Config:
type LLMConfig struct {
    MaxRetries int `mapstructure:"max_retries"` // Deprecated: use RateLimit.MaxRetries
}

// In ConfigValues:
// ... no MaxRetries field
```

**Impact**: Deprecated field cannot be edited through TUI (though it's marked deprecated).

---

## Persistence Risk Analysis

### Scenario 1: Load → Edit → Save (LOSS OF DATA)

```
1. User has config.yaml with rate_limit settings:
   llm:
     rate_limit:
       enabled: true
       requests_per_minute: 100
       burst_size: 20

2. User runs: repodocs config
   - config.Load() reads full config including rate_limit
   - FromConfig() creates ConfigValues (rate_limit fields DROPPED)
   - TUI shows only LLMProvider, LLMAPIKey, etc.

3. User edits LLMProvider and saves
   - ToConfig() creates new Config with rate_limit = ZERO VALUES
   - config.Save() writes to YAML:
     llm:
       rate_limit:
         enabled: false        ← LOST original value!
         requests_per_minute: 0
         burst_size: 0
         ...

4. Result: Rate limiting disabled, burst size reset to 0
```

### Scenario 2: Merge Behavior (NONE - FULL REPLACEMENT)

The current implementation does **NOT merge** with the original config:

```go
// In app.go handleSave():
cfg, err := m.values.ToConfig()  // ← Creates NEW Config from scratch
if m.saveFunc != nil {
    if err := m.saveFunc(cfg); err != nil {  // ← Saves ENTIRE config
        ...
    }
}

// In loader.go Save():
data, err := yaml.Marshal(cfg)  // ← Marshals the NEW config
if err := os.WriteFile(path, data, 0644); err != nil {  // ← Overwrites file
    ...
}
```

**No merge logic exists.** The entire config is replaced with what the TUI provides.

---

## Field-by-Field Persistence Status

### ✅ Properly Persisted (in ConfigValues)

| Field | Type | Persisted | Notes |
|-------|------|-----------|-------|
| `output.directory` | string | ✅ | Direct mapping |
| `output.flat` | bool | ✅ | Direct mapping |
| `output.overwrite` | bool | ✅ | Direct mapping |
| `output.json_metadata` | bool | ✅ | Direct mapping |
| `concurrency.workers` | int | ✅ | Direct mapping |
| `concurrency.timeout` | duration | ✅ | String ↔ Duration conversion |
| `concurrency.max_depth` | int | ✅ | Direct mapping |
| `cache.enabled` | bool | ✅ | Direct mapping |
| `cache.ttl` | duration | ✅ | String ↔ Duration conversion |
| `cache.directory` | string | ✅ | Direct mapping |
| `rendering.force_js` | bool | ✅ | Direct mapping |
| `rendering.js_timeout` | duration | ✅ | String ↔ Duration conversion |
| `rendering.scroll_to_end` | bool | ✅ | Direct mapping |
| `stealth.user_agent` | string | ✅ | Direct mapping |
| `stealth.random_delay_min` | duration | ✅ | String ↔ Duration conversion |
| `stealth.random_delay_max` | duration | ✅ | String ↔ Duration conversion |
| `logging.level` | string | ✅ | Direct mapping |
| `logging.format` | string | ✅ | Direct mapping |
| `llm.provider` | string | ✅ | Direct mapping |
| `llm.api_key` | string | ✅ | Direct mapping |
| `llm.base_url` | string | ✅ | Direct mapping |
| `llm.model` | string | ✅ | Direct mapping |
| `llm.max_tokens` | int | ✅ | Direct mapping |
| `llm.temperature` | float64 | ✅ | Direct mapping |
| `llm.timeout` | duration | ✅ | String ↔ Duration conversion |
| `llm.enhance_metadata` | bool | ✅ | Direct mapping |
| `exclude` | []string | ✅ | Direct mapping (slice) |

### ❌ NOT Persisted (missing from ConfigValues)

| Field | Type | Persisted | Impact |
|-------|------|-----------|--------|
| `llm.max_retries` | int | ❌ | Deprecated field, but still in struct |
| `llm.rate_limit.*` | RateLimitConfig | ❌ | **CRITICAL** - entire nested struct lost |
| `llm.rate_limit.enabled` | bool | ❌ | Rate limiting disabled on save |
| `llm.rate_limit.requests_per_minute` | int | ❌ | Reset to 0 |
| `llm.rate_limit.burst_size` | int | ❌ | Reset to 0 |
| `llm.rate_limit.max_retries` | int | ❌ | Reset to 0 |
| `llm.rate_limit.initial_delay` | duration | ❌ | Reset to 0 |
| `llm.rate_limit.max_delay` | duration | ❌ | Reset to 0 |
| `llm.rate_limit.multiplier` | float64 | ❌ | Reset to 0 |
| `llm.rate_limit.circuit_breaker.*` | CircuitBreakerConfig | ❌ | **CRITICAL** - entire nested struct lost |
| `llm.rate_limit.circuit_breaker.enabled` | bool | ❌ | Circuit breaker disabled on save |
| `llm.rate_limit.circuit_breaker.failure_threshold` | int | ❌ | Reset to 0 |
| `llm.rate_limit.circuit_breaker.success_threshold_half_open` | int | ❌ | Reset to 0 |
| `llm.rate_limit.circuit_breaker.reset_timeout` | duration | ❌ | Reset to 0 |

---

## Code Evidence

### FromConfig() - Drops RateLimit

```go
// internal/tui/config_adapter.go:50-87
func FromConfig(cfg *config.Config) *ConfigValues {
    return &ConfigValues{
        // ... all other fields ...
        LLMProvider:        cfg.LLM.Provider,
        LLMAPIKey:          cfg.LLM.APIKey,
        LLMBaseURL:         cfg.LLM.BaseURL,
        LLMModel:           cfg.LLM.Model,
        LLMMaxTokens:       cfg.LLM.MaxTokens,
        LLMTemperature:     cfg.LLM.Temperature,
        LLMTimeout:         formatDuration(cfg.LLM.Timeout),
        LLMEnhanceMetadata: cfg.LLM.EnhanceMetadata,
        // ← RateLimit and MaxRetries NOT copied
        Exclude: cfg.Exclude,
    }
}
```

### ToConfig() - Creates Zero Values for Missing Fields

```go
// internal/tui/config_adapter.go:90-166
func (v *ConfigValues) ToConfig() (*config.Config, error) {
    // ... parse durations ...
    cfg := &config.Config{
        // ... all other fields ...
        LLM: config.LLMConfig{
            Provider:        v.LLMProvider,
            APIKey:          v.LLMAPIKey,
            BaseURL:         v.LLMBaseURL,
            Model:           v.LLMModel,
            MaxTokens:       v.LLMMaxTokens,
            Temperature:     v.LLMTemperature,
            Timeout:         llmTimeout,
            EnhanceMetadata: v.LLMEnhanceMetadata,
            // ← RateLimit field NOT set, defaults to zero value
            // ← MaxRetries field NOT set, defaults to 0
        },
        Exclude: v.Exclude,
    }
    return cfg, nil
}
```

### Save() - Full Replacement (No Merge)

```go
// internal/config/loader.go:156-172
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

---

## Slice Field Handling

### Exclude Field (Properly Handled)

The `Exclude` field is a slice and **IS properly persisted**:

```go
// ConfigValues includes:
Exclude []string

// FromConfig copies it:
Exclude: cfg.Exclude,

// ToConfig preserves it:
Exclude: v.Exclude,

// YAML output:
exclude:
  - '.*\.pdf$'
  - '.*/login.*'
  - '.*/logout.*'
```

**Why it works**: The slice is directly mapped in ConfigValues, and YAML marshaling handles slices correctly.

---

## Default Values Behavior

### When Loading Missing Config

If config file doesn't exist:
```go
// main.go:494-507
func runConfigEdit(cmd *cobra.Command, args []string) error {
    cfg, err := config.Load()
    if err != nil {
        cfg = config.Default()  // ← Falls back to defaults
    }
    return tui.Run(tui.Options{
        Config:     cfg,
        SaveFunc:   config.Save,
        Accessible: accessible,
    })
}
```

### Defaults Include RateLimit

```go
// internal/config/defaults.go:87-142
func Default() *Config {
    return &Config{
        // ...
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

**Problem**: Defaults are lost when TUI saves because ToConfig() doesn't preserve them.

---

## Test Coverage Gaps

### Integration Tests (config_integration_test.go)

Tests verify save/load works, but **don't test TUI round-trip**:

```go
// Tests direct Save/Load, not TUI flow
func TestConfigSaveAndLoad(t *testing.T) {
    original := &config.Config{ /* ... */ }
    err := config.SaveTo(original, configPath)
    // ← Doesn't test FromConfig → ToConfig → Save
}
```

### TUI Adapter Tests (config_adapter_test.go)

Tests exist for FromConfig/ToConfig, but **don't test RateLimit**:

```go
// TestFromConfig and TestToConfig exist
// But they don't include RateLimit fields in test data
func TestFromConfig(t *testing.T) {
    cfg := &config.Config{
        // ... no RateLimit in test ...
    }
}
```

---

## Risk Summary

### 🔴 Critical Risks

1. **Data Loss on TUI Save**
   - Users with rate_limit config lose it when using TUI
   - No warning or error message
   - Silent data corruption

2. **No Merge Logic**
   - TUI completely replaces config file
   - No preservation of fields not in ConfigValues
   - Any future nested fields will have same problem

3. **Deprecated Field Handling**
   - `llm.max_retries` is deprecated but still in struct
   - TUI doesn't expose it, so users can't migrate to rate_limit
   - Creates confusion about which field to use

### 🟡 Medium Risks

1. **Incomplete Test Coverage**
   - No tests for TUI round-trip with RateLimit
   - No tests for data loss scenario
   - Integration tests don't cover TUI flow

2. **Documentation Gap**
   - README doesn't mention RateLimit/CircuitBreaker
   - TUI help text doesn't explain what's not editable
   - Users unaware of missing fields

### 🟢 Low Risks

1. **Slice Handling**
   - Exclude field works correctly
   - YAML marshaling handles slices properly
   - No risk here

---

## Recommendations

### Immediate (Fix Data Loss)

1. **Add RateLimit fields to ConfigValues**
   ```go
   type ConfigValues struct {
       // ... existing fields ...
       
       // LLM Rate Limiting
       LLMRateLimitEnabled           bool
       LLMRateLimitRequestsPerMinute int
       LLMRateLimitBurstSize         int
       LLMRateLimitMaxRetries        int
       LLMRateLimitInitialDelay      string
       LLMRateLimitMaxDelay          string
       LLMRateLimitMultiplier        float64
       
       // Circuit Breaker
       LLMCircuitBreakerEnabled                  bool
       LLMCircuitBreakerFailureThreshold         int
       LLMCircuitBreakerSuccessThresholdHalfOpen int
       LLMCircuitBreakerResetTimeout             string
   }
   ```

2. **Update FromConfig() to copy RateLimit**
   ```go
   func FromConfig(cfg *config.Config) *ConfigValues {
       // ... existing code ...
       LLMRateLimitEnabled:           cfg.LLM.RateLimit.Enabled,
       LLMRateLimitRequestsPerMinute: cfg.LLM.RateLimit.RequestsPerMinute,
       // ... etc ...
   }
   ```

3. **Update ToConfig() to reconstruct RateLimit**
   ```go
   func (v *ConfigValues) ToConfig() (*config.Config, error) {
       // ... existing code ...
       LLM: config.LLMConfig{
           // ... existing fields ...
           RateLimit: config.RateLimitConfig{
               Enabled:           v.LLMRateLimitEnabled,
               RequestsPerMinute: v.LLMRateLimitRequestsPerMinute,
               // ... etc ...
           },
       },
   }
   ```

4. **Add TUI form for LLM Advanced Settings**
   ```go
   func CreateLLMAdvancedForm(values *ConfigValues) *huh.Form {
       // Rate limiting and circuit breaker fields
   }
   ```

### Short-term (Prevent Future Issues)

1. **Implement Merge Logic**
   - Load original config before TUI
   - Merge TUI changes with original
   - Preserve fields not in ConfigValues

2. **Add Validation Tests**
   - Test TUI round-trip with all fields
   - Test data loss scenarios
   - Test merge behavior

3. **Update Documentation**
   - Document which fields are editable in TUI
   - Explain RateLimit/CircuitBreaker purpose
   - Add warning about data loss risk

### Long-term (Architecture)

1. **Consider Partial Config Updates**
   - Allow updating only specific sections
   - Preserve other sections from original file

2. **Add Config Versioning**
   - Track config schema version
   - Migrate old configs safely

3. **Improve Field Mapping**
   - Use reflection to auto-map fields
   - Reduce manual mapping errors
   - Easier to add new fields

---

## Conclusion

The config persistence mechanism has a **critical gap** where nested complex fields (RateLimit, CircuitBreaker) are silently lost when using the TUI. This is a **data loss bug** that needs immediate fixing.

The root cause is that `ConfigValues` doesn't include all fields from `Config`, and the save logic does a full replacement instead of merging with the original config.

**Severity: HIGH** - Users can lose important configuration without warning.
