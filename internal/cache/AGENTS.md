# AGENTS.md - internal/cache

BadgerDB-based persistent caching layer.

## Files

| File | Purpose |
|------|---------|
| `badger.go` | Main BadgerDB implementation |
| `interface.go` | `domain.Cache` implementation |
| `keys.go` | Cache key generation |

## Where to Look

| Task | File |
|------|------|
| Cache TTL issues | `badger.go` - `Options` struct |
| Key generation | `keys.go` |
| Implementation details | `interface.go` |

## Key Types

```go
type BadgerCache struct { /* wraps badger.DB */ }
type Options struct {
    Directory string
    InMemory  bool
    Logger    bool
    TTL       time.Duration
}
```

## Conventions

- In-memory for tests: `NewBadgerCache(Options{InMemory: true})`
- Default directory: `~/.repodocs/cache`
- Background GC for compaction
- Logger disabled by default

## Anti-Patterns

- **NO direct Badger calls** - Use `Cache` interface
- **NO global cache instances** - Inject via Dependencies
