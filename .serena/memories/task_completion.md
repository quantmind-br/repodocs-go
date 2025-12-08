# Task Completion Checklist

When completing a coding task, follow this checklist:

## Before Committing

### 1. Format Code
```bash
make fmt
```

### 2. Run Linter
```bash
make lint
```
Fix any issues reported by golangci-lint.

### 3. Run Static Analysis
```bash
make vet
```

### 4. Run Tests
```bash
# For quick verification
make test

# For comprehensive testing
make test-all
```

### 5. Check Test Coverage (if adding new code)
```bash
make coverage
```
Review the coverage report at `coverage/coverage.html`.

## Code Review Checklist

- [ ] Error handling: errors wrapped with context using `fmt.Errorf("context: %w", err)`
- [ ] Context: I/O operations accept and respect `context.Context`
- [ ] Interfaces: New code depends on domain interfaces, not concrete types
- [ ] Logging: Appropriate log levels used (Debug/Info/Error)
- [ ] Tests: Unit tests added for new functionality
- [ ] Documentation: Public APIs documented with Go doc comments

## Adding New Features

### Adding a New Strategy
1. Create implementation in `internal/strategies/`
2. Add strategy type constant in `internal/app/detector.go`
3. Update `DetectStrategy()` and `CreateStrategy()` in detector
4. Update `strategies.NewDependencies()` if new services needed

### Adding New Domain Types
1. Add interfaces to `internal/domain/interfaces.go`
2. Add models to `internal/domain/models.go`
3. Add domain errors to `internal/domain/errors.go`

### Adding New Infrastructure
1. Create package under `internal/`
2. Implement domain interfaces
3. Wire into `strategies.NewDependencies()`
