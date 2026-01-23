# Config Persistence Analysis - Complete Documentation

This directory contains a comprehensive analysis of the `repodocs config` command's configuration loading, editing, and persistence mechanism.

## 📋 Documents

### 1. **config-persistence-summary.md** (START HERE)
**Quick reference** - 3.2 KB
- The problem in plain English
- Why it happens
- Example scenario
- Root causes
- Impact assessment
- Quick fix checklist

**Read this first** if you want a quick understanding of the issue.

---

### 2. **config-persistence-analysis.md** (DETAILED)
**Complete technical analysis** - 17 KB
- Executive summary
- Architecture overview
- Critical findings with code evidence
- Persistence risk analysis
- Field-by-field status table
- Test coverage gaps
- Detailed recommendations

**Read this** for comprehensive understanding of the problem and solutions.

---

### 3. **config-flow-diagram.md** (VISUAL)
**Flow diagrams and visualizations** - 17 KB
- Current flow with data loss highlighted
- Field mapping visualization
- Data loss scenarios
- Proposed fix flow
- Code location map

**Read this** if you prefer visual explanations.

---

### 4. **code-locations.md** (REFERENCE)
**Detailed code locations and line numbers** - 15 KB
- ConfigValues struct (incomplete)
- FromConfig() function (incomplete)
- ToConfig() function (incomplete)
- Save() function (full replacement)
- handleSave() in TUI app
- All struct definitions
- Test file locations
- Summary of changes needed

**Use this** as a reference when implementing fixes.

---

## 🎯 The Problem (TL;DR)

When users run `repodocs config` (TUI), they can **lose configuration data** for advanced LLM features:

```
Load Config → FromConfig() → ConfigValues (MISSING FIELDS) → ToConfig() → Save
                                    ↑
                        RateLimit fields dropped here
```

**What gets lost:**
- `llm.rate_limit.*` (entire nested struct)
- `llm.rate_limit.circuit_breaker.*` (nested within rate_limit)
- `llm.max_retries` (deprecated field)

**Why:** ConfigValues struct doesn't include these fields, so they're silently dropped when saving.

---

## 🔴 Severity: HIGH

- **Data Loss**: Silent, no warning
- **Scope**: Anyone using TUI with rate limiting configured
- **Detection**: Hard to notice - config file is overwritten
- **Recovery**: Manual config file editing required

---

## 📊 Analysis Highlights

### Missing Fields in ConfigValues

| Field | Type | Impact |
|-------|------|--------|
| `llm.rate_limit.enabled` | bool | Rate limiting disabled on save |
| `llm.rate_limit.requests_per_minute` | int | Reset to 0 |
| `llm.rate_limit.burst_size` | int | Reset to 0 |
| `llm.rate_limit.max_retries` | int | Reset to 0 |
| `llm.rate_limit.initial_delay` | duration | Reset to 0 |
| `llm.rate_limit.max_delay` | duration | Reset to 0 |
| `llm.rate_limit.multiplier` | float64 | Reset to 0 |
| `llm.rate_limit.circuit_breaker.*` | struct | Entire struct lost |

### Root Causes

1. **ConfigValues struct is incomplete** - Missing RateLimit/CircuitBreaker fields
2. **No merge logic in Save()** - Entire config is replaced, not merged
3. **FromConfig() doesn't copy RateLimit** - Silently drops the fields
4. **ToConfig() doesn't reconstruct RateLimit** - Creates zero values instead

---

## 🔧 Quick Fix Checklist

- [ ] Add RateLimit fields to ConfigValues struct
- [ ] Add CircuitBreaker fields to ConfigValues struct
- [ ] Update FromConfig() to copy RateLimit/CircuitBreaker
- [ ] Update ToConfig() to reconstruct RateLimit/CircuitBreaker
- [ ] Create TUI form for LLM advanced settings
- [ ] Add tests for TUI round-trip with RateLimit
- [ ] Update documentation

---

## 📁 Files to Modify

| File | Change |
|------|--------|
| `internal/tui/config_adapter.go` | Add missing fields, update FromConfig/ToConfig |
| `internal/tui/forms.go` | Add CreateLLMAdvancedForm() |
| `internal/tui/categories.go` | Add "LLM Advanced" category |
| `internal/tui/config_adapter_test.go` | Add RateLimit round-trip tests |
| `tests/integration/config/config_integration_test.go` | Add TUI round-trip test |

---

## 🧪 Test Coverage Gaps

Current tests don't catch this because:
- Integration tests use direct Save/Load (not TUI)
- TUI adapter tests don't include RateLimit in test data
- No round-trip test: Load → TUI → Save → Load

---

## 📚 How to Use This Analysis

### For Quick Understanding
1. Read **config-persistence-summary.md**
2. Look at **config-flow-diagram.md** for visual explanation

### For Implementation
1. Read **config-persistence-analysis.md** for detailed recommendations
2. Use **code-locations.md** as reference for exact line numbers
3. Follow the "Quick Fix Checklist"

### For Code Review
1. Check **code-locations.md** for all affected files
2. Verify all missing fields are added to ConfigValues
3. Verify FromConfig() copies all fields
4. Verify ToConfig() reconstructs all fields
5. Verify tests cover the round-trip

---

## 🔗 Related Code

**Config Loading**: `internal/config/loader.go`
**Config Struct**: `internal/config/config.go`
**TUI Adapter**: `internal/tui/config_adapter.go`
**TUI App**: `internal/tui/app.go`
**TUI Forms**: `internal/tui/forms.go`
**Tests**: `tests/integration/config/` and `internal/tui/config_adapter_test.go`

---

## 📝 Notes

- The `Exclude` field (slice) is properly persisted - no issues there
- Duration fields are converted to/from strings for form editing - this works correctly
- The issue is specific to nested complex structs (RateLimit, CircuitBreaker)
- This is a **data loss bug**, not a feature request

---

## ✅ Verification Checklist

After implementing fixes:

- [ ] All RateLimit fields added to ConfigValues
- [ ] All CircuitBreaker fields added to ConfigValues
- [ ] FromConfig() copies all new fields
- [ ] ToConfig() reconstructs all new fields
- [ ] TUI form shows all new fields
- [ ] Tests verify round-trip with RateLimit
- [ ] Tests verify round-trip with CircuitBreaker
- [ ] No data loss on save
- [ ] Documentation updated
- [ ] Build passes
- [ ] Tests pass

---

Generated: 2025-01-13
Analysis Type: Config Persistence & TUI Data Loss
Severity: HIGH
