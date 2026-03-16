<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# tests/mocks

Generated mock implementations using go.uber.org/mock (gomock). Mocks are regenerated from domain interfaces when interfaces change.

## Purpose

Auto-generated mock implementations for all domain interfaces, enabling isolated unit testing without real network calls or external dependencies.

## Key Files

| File | Description |
|------|-------------|
| `domain.go` | Mocks for domain interfaces: MockStrategy, MockFetcher, MockRenderer, MockCache, MockConverter, MockWriter, MockLLMProvider. Also includes SimpleMockCache, SimpleMockFetcher, MultiResponseMockFetcher helpers |
| `git.go` | MockClient - gomock implementation for git.Client interface |
| `git_mock.go` | Legacy testify/mock MockGitClient for backward compatibility |

## Subdirectories

None - flat package structure.

## For AI Agents

### Regeneration Command

```bash
# Regenerate domain mocks
mockgen -source=internal/domain/interfaces.go -destination=tests/mocks/domain.go -package mocks

# Regenerate git mock
mockgen -source=internal/git/interface.go -destination=tests/mocks/git.go -package mocks
```

### Usage with gomock

```go
ctrl := gomock.NewController(t)
defer ctrl.Finish()

mockFetcher := mocks.NewMockFetcher(ctrl)
mockFetcher.EXPECT().Get(gomock.Any(), "https://example.com").
    Return(&domain.Response{StatusCode: 200, Body: []byte("test")}, nil)
```

### Usage with Simple Helpers

```go
// No controller needed
mockCache := mocks.NewSimpleMockCache()
defer mockCache.Close()
```

### Dependencies

- **Internal:** `internal/domain`, `internal/git`
- **External:** `go.uber.org/mock/gomock`, `github.com/go-git/go-git/v5`

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->