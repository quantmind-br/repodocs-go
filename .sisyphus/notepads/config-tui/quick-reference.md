# Quick Reference: huh Patterns in repodocs-go

## Numeric Input Pattern (COPY THIS)

```go
// 1. In ConfigValues struct (config_adapter.go)
type ConfigValues struct {
    Workers      int       // Store as native type
    Temperature  float64
    Timeout      string    // Durations stored as strings
}

// 2. In form builder (forms.go)
func CreateConcurrencyForm(values *ConfigValues) *huh.Form {
    workersStr := strconv.Itoa(values.Workers)  // Convert to string
    
    return huh.NewForm(
        huh.NewGroup(
            huh.NewInput().
                Key("workers").
                Value(&workersStr).                    // Bind string pointer
                Validate(ValidateIntRange(1, 50)),     // Validate as string
        ),
    ).WithTheme(GetTheme())
}

// 3. In validation.go
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

// 4. In config_adapter.go - conversion back
func (v *ConfigValues) ToConfig() (*config.Config, error) {
    // Values are already in ConfigValues as native types
    // Just use them directly
    cfg := &config.Config{
        Concurrency: config.ConcurrencyConfig{
            Workers: v.Workers,  // Already int
        },
    }
    return cfg, nil
}
```

## Form Completion Pattern (COPY THIS)

```go
// In app.go - Update() method
func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // ... other cases ...
    
    if m.state == stateForm && m.currentForm != nil {
        form, cmd := m.currentForm.Update(msg)
        if f, ok := form.(*huh.Form); ok {
            m.currentForm = f
        }
        
        // Check for completion
        if m.currentForm.State == huh.StateCompleted {
            m.dirty = true
            m.state = stateMenu
            return m, nil
        }
        
        return m, cmd
    }
    
    return m, nil
}

// On save
func (m Model) handleSave() (tea.Model, tea.Cmd) {
    cfg, err := m.values.ToConfig()  // Convert strings to types
    if err != nil {
        m.state = stateError
        m.err = err
        return m, nil
    }
    
    if m.saveFunc != nil {
        if err := m.saveFunc(cfg); err != nil {
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

## Validator Patterns (COPY THESE)

```go
// Integer range
func ValidateIntRange(min, max int) func(string) error {
    return func(s string) error {
        if strings.TrimSpace(s) == "" {
            return nil
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

// Float range
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

// Duration
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

// Positive integer
func ValidatePositiveInt(s string) error {
    if strings.TrimSpace(s) == "" {
        return nil
    }
    n, err := strconv.Atoi(s)
    if err != nil {
        return ErrInvalidNumber
    }
    if n < 1 {
        return ErrPositiveInt
    }
    return nil
}
```

## Duration Conversion Pattern (COPY THIS)

```go
// Config → Form (typed → string)
func formatDuration(d time.Duration) string {
    if d == 0 {
        return ""
    }
    return d.String()  // e.g., "30s", "5m"
}

// Form → Config (string → typed)
func parseDurationOrDefault(s string, defaultVal time.Duration) (time.Duration, error) {
    if s == "" {
        return defaultVal, nil
    }
    return time.ParseDuration(s)
}
```

## Form Structure Pattern (COPY THIS)

```go
func CreateXxxForm(values *ConfigValues) *huh.Form {
    // Convert numeric types to strings
    numStr := strconv.Itoa(values.NumericField)
    floatStr := strconv.FormatFloat(values.FloatField, 'f', 2, 64)
    
    return huh.NewForm(
        huh.NewGroup(
            // Text input
            huh.NewInput().
                Key("field_id").
                Title("Display Title").
                Description("Help text").
                Value(&stringField).
                Placeholder("example").
                CharLimit(256),
            
            // Numeric input (as string)
            huh.NewInput().
                Key("numeric_field").
                Title("Number").
                Value(&numStr).
                Validate(ValidateIntRange(1, 100)),
            
            // Boolean input
            huh.NewConfirm().
                Key("bool_field").
                Title("Enable Feature").
                Value(&boolField),
            
            // Select input
            huh.NewSelect[string]().
                Key("select_field").
                Title("Choose Option").
                Options(
                    huh.NewOption("Option 1", "opt1"),
                    huh.NewOption("Option 2", "opt2"),
                ).
                Value(&stringField),
        ),
    ).WithTheme(GetTheme())
}
```

## Key Rules

1. **Numeric inputs**: Always use string intermediate representation
2. **Form completion**: Check `huh.StateCompleted` in Update()
3. **Validation**: All validators accept `string`, return `error`
4. **Conversion**: Explicit ToConfig()/FromConfig() methods
5. **Error handling**: Wrap with context using `fmt.Errorf(..., %w, err)`
6. **Empty values**: Validators should allow empty strings (use defaults)

## Files to Reference

- `internal/tui/forms.go` - Form definitions
- `internal/tui/validation.go` - Validator implementations
- `internal/tui/config_adapter.go` - Conversion logic
- `internal/tui/app.go` - State management and completion handling
