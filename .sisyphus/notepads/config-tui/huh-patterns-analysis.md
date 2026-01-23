# charmbracelet/huh Usage Patterns - Comprehensive Analysis

## Project Context
- **Project**: repodocs-go (Go CLI tool for documentation extraction)
- **huh Version**: v0.8.0
- **TUI Package**: `internal/tui/`
- **Status**: Active implementation with 7 form categories

## Key Findings

### 1. NUMERIC VALUE HANDLING PATTERN

#### Current Implementation (String-Based Conversion)
The project uses a **string-based intermediate representation** for numeric values:

```go
// In forms.go - CreateConcurrencyForm()
func CreateConcurrencyForm(values *ConfigValues) *huh.Form {
    workersStr := strconv.Itoa(values.Workers)      // int -> string
    maxDepthStr := strconv.Itoa(values.MaxDepth)    // int -> string
    
    return huh.NewForm(
        huh.NewGroup(
            huh.NewInput().
                Key("workers").
                Value(&workersStr).                  // Bind to string pointer
                Validate(ValidateIntRange(1, 50)),   // Validate as string
            // ...
        ),
    )
}
```

#### Why This Pattern?
1. **huh.Input only accepts `*string`** - Cannot directly bind to `*int` or `*float64`
2. **Validation happens on strings** - Custom validators parse and validate
3. **Conversion happens post-form** - In `ToConfig()` method

#### Numeric Types Handled
- **Integers**: Workers, MaxDepth, LLMMaxTokens
- **Floats**: LLMTemperature (formatted with `strconv.FormatFloat(..., 'f', 2, 64)`)
- **Durations**: Timeout, CacheTTL, JSTimeout, RandomDelayMin/Max (stored as strings, parsed as time.Duration)

### 2. FORM COMPLETION & CALLBACKS

#### State Management Pattern
```go
// In app.go - Update() method
if m.currentForm.State == huh.StateCompleted {
    m.dirty = true
    m.state = stateMenu
    return m, nil
}
```

**Key Points:**
- Forms use `huh.StateCompleted` to signal completion
- No explicit callbacks/hooks - completion is detected via state polling
- After completion, form values are already bound to ConfigValues struct
- Conversion to Config happens later in `handleSave()` method

#### Form Completion Flow
1. User completes form (presses Enter on last field)
2. `huh.Form.State` becomes `huh.StateCompleted`
3. Model detects state change in `Update()` method
4. Sets `m.dirty = true` and transitions back to menu
5. On save, calls `m.values.ToConfig()` to convert strings to typed values

### 3. VALIDATION PATTERN

#### Validator Functions
All validators follow this signature:
```go
func ValidateXxx(s string) error
```

#### Examples from validation.go
```go
// Integer range validation
func ValidateIntRange(min, max int) func(string) error {
    return func(s string) error {
        if strings.TrimSpace(s) == "" {
            return nil  // Empty is valid (uses default)
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

// Float range validation
func ValidateFloatRange(min, max float64) func(string) error {
    return func(s string) error {
        if strings.TrimSpace(s) == "" {
            return nil
        }
        n, err := strconv.ParseFloat(s, 64)
        if err != nil {
            return fmt.Errorf("must be a valid decimal number")
        }
        if n < min || n > max {
            return fmt.Errorf("%w: must be between %.2f and %.2f", ErrInvalidRange, min, max)
        }
        return nil
    }
}

// Duration validation
func ValidateDuration(s string) error {
    if strings.TrimSpace(s) == "" {
        return nil
    }
    _, err := time.ParseDuration(s)
    if err != nil {
        return fmt.Errorf("invalid duration format (use: 30s, 5m, 1h): %w", err)
    }
    return nil
}
```

### 4. FORM STRUCTURE PATTERNS

#### ConfigValues Struct (config_adapter.go)
```go
type ConfigValues struct {
    // Strings (direct binding)
    OutputDirectory string
    UserAgent       string
    
    // Integers (converted to/from strings)
    Workers      int
    MaxDepth     int
    LLMMaxTokens int
    
    // Floats (converted to/from strings)
    LLMTemperature float64
    
    // Durations (stored as strings, parsed as time.Duration)
    Timeout        string
    CacheTTL       string
    JSTimeout      string
    
    // Booleans (direct binding)
    OutputFlat      bool
    CacheEnabled    bool
}
```

#### Form Creation Pattern
```go
func CreateXxxForm(values *ConfigValues) *huh.Form {
    // 1. Convert numeric types to strings
    numStr := strconv.Itoa(values.NumericField)
    
    // 2. Create form with groups
    return huh.NewForm(
        huh.NewGroup(
            huh.NewInput().
                Key("field_id").
                Title("Display Title").
                Description("Help text").
                Value(&stringField).           // Bind to string pointer
                Placeholder("example").
                CharLimit(256).
                Validate(ValidatorFunc),       // Validate as string
            
            huh.NewConfirm().
                Key("bool_field").
                Value(&boolField),             // Direct bool binding
            
            huh.NewSelect[string]().
                Key("select_field").
                Options(
                    huh.NewOption("Label", "value"),
                ).
                Value(&stringField),
        ),
    ).WithTheme(GetTheme())
}
```

### 5. CONVERSION FLOW

#### String → Typed Value (ToConfig)
```go
func (v *ConfigValues) ToConfig() (*config.Config, error) {
    // Parse integers
    workers := v.Workers  // Already int in ConfigValues
    
    // Parse floats
    temp := v.LLMTemperature  // Already float64
    
    // Parse durations
    timeout, err := parseDurationOrDefault(v.Timeout, config.DefaultTimeout)
    if err != nil {
        return nil, fmt.Errorf("invalid timeout: %w", err)
    }
    
    // Build config
    cfg := &config.Config{
        Concurrency: config.ConcurrencyConfig{
            Workers:  workers,
            Timeout:  timeout,
            MaxDepth: v.MaxDepth,
        },
        // ...
    }
    return cfg, nil
}
```

#### Typed Value → String (FromConfig)
```go
func FromConfig(cfg *config.Config) *ConfigValues {
    return &ConfigValues{
        Workers:  cfg.Concurrency.Workers,           // int -> int
        Timeout:  formatDuration(cfg.Concurrency.Timeout),  // time.Duration -> string
        MaxDepth: cfg.Concurrency.MaxDepth,          // int -> int
        // ...
    }
}

func formatDuration(d time.Duration) string {
    if d == 0 {
        return ""
    }
    return d.String()  // e.g., "30s", "5m"
}
```

## Recommended Minimal Change

### For Numeric Input Handling
**Current approach is CORRECT and MINIMAL:**
1. ✅ Store numeric values as `int`/`float64` in ConfigValues
2. ✅ Convert to strings only when creating forms
3. ✅ Use custom validators that parse strings
4. ✅ Convert back to typed values in ToConfig()

**Why this is minimal:**
- No wrapper types needed
- No custom field types
- Leverages huh's string-based Input
- Clear separation: form layer (strings) vs config layer (typed)

### For Form Completion Handling
**Current approach is CORRECT and MINIMAL:**
1. ✅ Detect completion via `huh.Form.State == huh.StateCompleted`
2. ✅ No callbacks needed - state polling is sufficient
3. ✅ Values already bound to ConfigValues via pointers
4. ✅ Conversion happens in explicit `handleSave()` method

**Why this is minimal:**
- No custom hooks/callbacks required
- Standard bubbletea pattern
- Clear error handling in ToConfig()
- Explicit save flow (no implicit side effects)

## Code Style Consistency

### Naming Conventions (Followed)
- ✅ Validator functions: `ValidateXxx(s string) error`
- ✅ Validator factories: `ValidateXxxRange(min, max) func(string) error`
- ✅ Form builders: `CreateXxxForm(values *ConfigValues) *huh.Form`
- ✅ Conversion methods: `FromConfig()`, `ToConfig()`

### Error Handling (Followed)
- ✅ Sentinel errors in validation.go (ErrRequired, ErrInvalidNumber, etc.)
- ✅ Wrapped errors with context: `fmt.Errorf("invalid timeout: %w", err)`
- ✅ Validation errors returned from validators
- ✅ Conversion errors returned from ToConfig()

### Testing Patterns (Observed)
- ✅ Table-driven tests in validation_test.go
- ✅ Mock config in config_adapter_test.go
- ✅ Separate test files for each module

## Summary Table

| Aspect | Pattern | Status |
|--------|---------|--------|
| Numeric Input | String intermediate + validators | ✅ Minimal & Correct |
| Form Completion | State polling (StateCompleted) | ✅ Minimal & Correct |
| Validation | Custom validators on strings | ✅ Minimal & Correct |
| Conversion | Explicit ToConfig/FromConfig | ✅ Minimal & Correct |
| Error Handling | Wrapped errors with context | ✅ Consistent |
| Code Style | Naming conventions followed | ✅ Consistent |

## No Changes Needed
The current implementation is already:
- ✅ Minimal and focused
- ✅ Consistent with project style
- ✅ Following huh best practices
- ✅ Properly handling numeric values
- ✅ Properly handling form completion

The pattern is production-ready and should be maintained as-is.
