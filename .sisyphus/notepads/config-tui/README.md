# Config TUI - charmbracelet/huh Patterns Documentation

This directory contains comprehensive documentation of charmbracelet/huh usage patterns discovered in the repodocs-go TUI implementation.

## Files in This Directory

### 1. `huh-patterns-analysis.md` (302 lines)
**Comprehensive technical analysis of all huh patterns**

Contains:
- Project context and version information
- Detailed explanation of numeric value handling
- Form completion and state management patterns
- Validation function patterns with examples
- Form structure patterns
- Conversion flow (Config ↔ ConfigValues)
- Code style consistency analysis
- Detailed recommendations
- Summary table with status

**Use this when:** You need to understand WHY patterns work the way they do

### 2. `quick-reference.md` (251 lines)
**Copy-paste code patterns for common tasks**

Contains:
- Numeric input pattern (ready to copy)
- Form completion pattern (ready to copy)
- Validator patterns (ready to copy)
- Duration conversion pattern (ready to copy)
- Form structure pattern (ready to copy)
- Key rules and file references

**Use this when:** You're implementing a new form or validator

### 3. `README.md` (this file)
**Index and navigation guide**

## Quick Navigation

### I need to...

**Add a new numeric input field**
→ See `quick-reference.md` → "Numeric Input Pattern"

**Add a new validator**
→ See `quick-reference.md` → "Validator Patterns"

**Understand how form completion works**
→ See `huh-patterns-analysis.md` → "Form Completion & Callbacks"

**Understand numeric value handling**
→ See `huh-patterns-analysis.md` → "Numeric Value Handling Pattern"

**Add a new form category**
→ See `quick-reference.md` → "Form Structure Pattern"

**Understand the conversion flow**
→ See `huh-patterns-analysis.md` → "Conversion Flow"

## Key Patterns at a Glance

### Numeric Input (String-Based)
```go
// ConfigValues stores native types
type ConfigValues struct {
    Workers int
}

// Forms convert to strings
workersStr := strconv.Itoa(values.Workers)

// Validators parse strings
Validate(ValidateIntRange(1, 50))

// ToConfig() converts back
cfg.Workers = v.Workers
```

### Form Completion (State Polling)
```go
if m.currentForm.State == huh.StateCompleted {
    m.dirty = true
    m.state = stateMenu
    return m, nil
}
```

### Validator Pattern
```go
func ValidateIntRange(min, max int) func(string) error {
    return func(s string) error {
        if strings.TrimSpace(s) == "" {
            return nil  // Empty is valid
        }
        n, err := strconv.Atoi(s)
        if err != nil {
            return ErrInvalidNumber
        }
        if n < min || n > max {
            return fmt.Errorf("%w: must be between %d and %d", ErrInvalidRange, min, max)
        }
        return nil
    }
}
```

## Implementation Files

The patterns documented here are implemented in:

- `internal/tui/forms.go` - Form definitions (7 categories)
- `internal/tui/app.go` - State management and completion handling
- `internal/tui/validation.go` - Validator implementations
- `internal/tui/config_adapter.go` - Conversion logic (Config ↔ ConfigValues)
- `internal/tui/styles.go` - Theme configuration
- `internal/tui/categories.go` - Menu structure

## Key Findings

### ✅ Numeric Value Handling
**Status:** OPTIMAL - No changes needed

The string-based intermediate representation is the correct approach because:
- huh.Input only accepts *string pointers
- Clear separation between form layer (strings) and config layer (typed)
- No wrapper types or custom field types needed
- Minimal code, maximum clarity

### ✅ Form Completion Handling
**Status:** OPTIMAL - No changes needed

State polling is the correct approach because:
- No custom hooks/callbacks required
- Standard bubbletea pattern
- Values already bound via pointers
- Clear error handling in ToConfig()
- Explicit save flow (no implicit side effects)

### ✅ Code Style
**Status:** CONSISTENT - Follows project guidelines

Adherence to AGENTS.md:
- Naming conventions: ValidateXxx, CreateXxxForm, ToConfig/FromConfig
- Error handling: Wrapped with context, sentinel errors defined
- Testing: Table-driven tests, separate test files per module
- Architecture: Clear separation of concerns

## Recommendation

**NO CHANGES NEEDED**

The current implementation is:
- ✅ Minimal and focused
- ✅ Consistent with project style
- ✅ Following huh best practices
- ✅ Properly handling numeric values
- ✅ Properly handling form completion
- ✅ Production-ready

The pattern is optimal for this use case and should be maintained as-is.

## For Future Development

1. **Adding a new form field:**
   - Follow the pattern in `quick-reference.md`
   - Use appropriate validator from `validation.go`
   - Add conversion logic in `config_adapter.go`

2. **Adding a new validator:**
   - Follow the pattern in `quick-reference.md`
   - Use sentinel errors from `validation.go`
   - Wrap errors with context

3. **Modifying form completion:**
   - Refer to `huh-patterns-analysis.md` → "Form Completion Flow"
   - Maintain state polling pattern
   - Keep explicit save flow

4. **Debugging form issues:**
   - Check `huh.Form.State` in `app.go`
   - Verify validator functions in `validation.go`
   - Check conversion logic in `config_adapter.go`

## Related Documentation

- `PLAN-config-tui.md` - Original architecture plan
- `AGENTS.md` - Project code style guidelines
- `README.md` - Main project documentation

## Version Information

- **huh version:** v0.8.0
- **Analysis date:** 2025-01-13
- **Project:** repodocs-go
- **TUI package:** internal/tui/
