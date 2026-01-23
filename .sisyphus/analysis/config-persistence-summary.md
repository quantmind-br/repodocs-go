# Config Persistence - Quick Summary

## The Problem

When users run `repodocs config` (TUI), they can **lose configuration data** for advanced LLM features.

### What Gets Lost

- `llm.rate_limit.*` (entire nested struct)
  - `enabled`, `requests_per_minute`, `burst_size`, `max_retries`, `initial_delay`, `max_delay`, `multiplier`
- `llm.rate_limit.circuit_breaker.*` (nested within rate_limit)
  - `enabled`, `failure_threshold`, `success_threshold_half_open`, `reset_timeout`
- `llm.max_retries` (deprecated field)

### Why It Happens

```
Load Config → FromConfig() → ConfigValues (MISSING FIELDS) → ToConfig() → Save
                                    ↑
                        RateLimit fields dropped here
                        Never copied to ConfigValues
                        Never reconstructed in ToConfig()
```

### Example Scenario

```yaml
# Original config.yaml
llm:
  rate_limit:
    enabled: true
    requests_per_minute: 100
    burst_size: 20

# User runs: repodocs config
# Edits something, saves

# Result in config.yaml
llm:
  rate_limit:
    enabled: false        ← LOST!
    requests_per_minute: 0
    burst_size: 0
```

## Root Causes

1. **ConfigValues struct is incomplete**
   - Missing RateLimit fields
   - Missing CircuitBreaker fields
   - Missing MaxRetries field

2. **No merge logic in Save()**
   - Entire config is replaced
   - Original config not read before saving
   - No preservation of unmapped fields

3. **FromConfig() doesn't copy RateLimit**
   - Silently drops the fields
   - No error or warning

4. **ToConfig() doesn't reconstruct RateLimit**
   - Creates zero values instead
   - Disables rate limiting

## Files Involved

| File | Issue |
|------|-------|
| `internal/tui/config_adapter.go` | ConfigValues missing fields; FromConfig/ToConfig incomplete |
| `internal/config/loader.go` | Save() does full replacement, no merge |
| `internal/tui/forms.go` | No form for RateLimit/CircuitBreaker |
| `internal/tui/app.go` | Calls ToConfig() without merge logic |

## Impact

- **Severity**: HIGH
- **Scope**: Anyone using TUI with rate limiting configured
- **Detection**: Silent - no error message
- **Recovery**: Manual config file editing

## Quick Fix Checklist

- [ ] Add RateLimit fields to ConfigValues struct
- [ ] Add CircuitBreaker fields to ConfigValues struct
- [ ] Update FromConfig() to copy RateLimit/CircuitBreaker
- [ ] Update ToConfig() to reconstruct RateLimit/CircuitBreaker
- [ ] Create TUI form for LLM advanced settings
- [ ] Add tests for TUI round-trip with RateLimit
- [ ] Update documentation

## Testing

Current tests don't catch this because:
- Integration tests use direct Save/Load (not TUI)
- TUI adapter tests don't include RateLimit in test data
- No round-trip test: Load → TUI → Save → Load

## Related Code Locations

**ConfigValues struct**: `internal/tui/config_adapter.go:12-47`
**FromConfig()**: `internal/tui/config_adapter.go:50-87`
**ToConfig()**: `internal/tui/config_adapter.go:90-166`
**Save()**: `internal/config/loader.go:156-172`
**Default config**: `internal/config/defaults.go:87-142`
**TUI forms**: `internal/tui/forms.go`
**TUI app**: `internal/tui/app.go:183-202` (handleSave)
