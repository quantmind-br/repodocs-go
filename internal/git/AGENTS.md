<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/git

Thin wrapper around go-git for testability.

## Purpose

Provides a testable interface over the go-git library. Allows mocking Git operations in tests while delegating to the real implementation in production.

## Key Files

| File | Description |
|------|-------------|
| `interface.go` | Client interface with PlainCloneContext method |
| `client.go` | RealClient implementing the Client interface, delegates to go-git |
| `client_test.go` | Tests for the client |

## Interface

```go
type Client interface {
    PlainCloneContext(ctx context.Context, path string, isBare bool, o *git.CloneOptions) (*git.Repository, error)
}
```

## Dependencies

- **External**: github.com/go-git/go-git/v5
- **Internal**: None

## For AI Agents

- Use interface for dependency injection, not direct go-git imports
- NewClient() creates the real implementation
- Supports both bare and non-bare clones
- Context-based cancellation supported

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->