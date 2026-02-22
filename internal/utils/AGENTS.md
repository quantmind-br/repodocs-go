# AGENTS.md - internal/utils

Shared utilities: URL normalization, filesystem helpers, logging, worker pool, progress tracking.

## Files

| File | Lines | Purpose |
|------|-------|---------|
| `url.go` | 423 | **HOTSPOT** - URL normalization (critical for caching) |
| `fs.go` | - | Filesystem operations, archive extraction |
| `logger.go` | - | Structured logging wrapper |
| `workerpool.go` | - | Concurrent worker pool |
| `progress.go` | - | Progress tracking |

## Where to Look

| Task | File | Key Functions |
|------|------|---------------|
| URL normalization issues | `url.go` | `NormalizeURL`, `IsInternalLink`, `ExtractBaseURL` |
| Cache key problems | `url.go` | All URL ops use normalized keys |
| File I/O issues | `fs.go` | `CopyFile`, `ExtractArchive`, `EnsureDir` |
| Worker concurrency | `workerpool.go` | `NewWorkerPool`, `Submit`, `Shutdown` |
| Logging configuration | `logger.go` | `NewLogger`, log levels |

## Key Patterns

```go
// URL normalization is CRITICAL for cache hit rate
normalized, err := utils.NormalizeURL(rawURL)

// Worker pool for concurrent operations
pool := utils.NewWorkerPool(ctx, maxWorkers)
pool.Submit(task)
```

## Complexity Notes

- **url.go (423 lines)**: Complex URL normalization - handles schemes, ports, fragments, query params, path cleaning. All caching relies on normalized URLs - bugs here cause cache misses.
