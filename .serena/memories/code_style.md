# Code Style and Conventions

## Go Version
- Go 1.25.5

## Formatting
- Use `gofmt` for formatting (`make fmt`)
- Standard Go formatting rules apply

## Linting
The project uses `golangci-lint` with the following linters enabled:
- `gofmt`, `govet`, `errcheck`, `staticcheck`
- `unused`, `gosimple`, `ineffassign`, `typecheck`
- `misspell`, `gocyclo` (max complexity: 15)
- `dupl` (threshold: 100), `gosec`, `unconvert`
- `goconst` (min-len: 3, min-occurrences: 3)
- `gocognit` (max complexity: 20)

Tests are excluded from `dupl` and `gosec` checks.

## Error Handling
- Wrap errors with context: `fmt.Errorf("context: %w", err)`
- Define domain errors in `internal/domain/errors.go`
- Don't swallow errors - always propagate or log them

## Logging
- Use zerolog via `internal/utils/Logger`
- Levels:
  - `Debug`: Internal flow details
  - `Info`: Milestones and important events
  - `Error`: Failures requiring attention

## Context
- All I/O operations must accept `context.Context` as the first parameter
- Respect context cancellation throughout the call chain

## Configuration
- All settings flow through `config.Config` struct
- Configuration loaded by Viper from files and environment

## Package Organization
- Use `internal/` for private packages
- Use `pkg/` for public/exportable packages
- Domain interfaces in `internal/domain/interfaces.go`
- Domain models in `internal/domain/models.go`

## Naming Conventions
- Standard Go naming conventions (CamelCase for exports, camelCase for unexported)
- Interface names don't use "I" prefix (e.g., `Strategy` not `IStrategy`)
- Structs implementing interfaces named descriptively (e.g., `CrawlerStrategy`)

## Dependency Injection
- Use constructor functions (e.g., `NewOrchestrator()`, `NewDependencies()`)
- Depend on interfaces, not concrete implementations
- All DI wiring happens in `strategies.NewDependencies()`
