# Performance Optimizations - Validated & Prioritized

> **Analysis Date**: 2025-01-24
> **Status**: Reviewed and validated against actual codebase
> **Verdict**: 1 high-priority, 1 medium-priority, 2 low-priority, 1 discarded

---

## P1 - High Priority

### perf-001: Parallelize Manifest Source Processing

**Impact**: High | **Effort**: Medium | **Confidence**: High

**Problem**: The orchestrator processes manifest sources sequentially in `RunManifest`. A slow source blocks all subsequent sources.

**Location**: `internal/app/orchestrator.go` lines 263-311

**Current Behavior**:
```go
for i, source := range manifestCfg.Sources {
    // ... blocks until each source completes
    err := o.Run(ctx, source.URL, opts)
}
```

**Implementation**:
1. Replace sequential loop with `utils.ParallelForEach` (already used in sitemap.go, processor.go)
2. Add configurable limit for concurrent sources (default: 3)
3. Ensure log messages include source identifier for traceability
4. Call `FlushMetadata` and `SaveState` after all parallel sources complete

**Expected Improvement**: Execution time drops from `Sum(source_times)` to approximately `Max(source_times)` for multi-source manifests. For 10 sources averaging 2 minutes each: 20 min → ~5 min.

**Tradeoffs**:
- Interleaved logs (mitigated by structured logging with source context)
- Shared rate limiting across sources requires coordination

---

## P2 - Medium Priority

### perf-002: Batch Sitemap Index Processing (Simplified)

**Impact**: Medium | **Effort**: Low | **Confidence**: Medium

**Problem**: `processSitemapIndex` collects ALL URLs from all nested sitemaps before processing any.

**Location**: `internal/strategies/sitemap.go` lines 112-149

**Current Behavior**:
```go
var allURLs []domain.SitemapURL
for _, sitemapURL := range sitemap.Sitemaps {
    urls, err := s.collectURLsFromSitemap(ctx, sitemapURL)
    allURLs = append(allURLs, urls...)  // Accumulates everything
}
// Only then starts processing
return s.processURLs(ctx, allURLs, opts)
```

**Implementation** (Simplified from original proposal):
1. Process each nested sitemap's URLs immediately after collecting them
2. Maintain a shared progress counter across batches
3. Apply limit across all batches, not per-batch

**Why NOT full streaming**: 
- Recursive sitemaps make channel-based streaming complex
- Sorting by `lastmod` requires all URLs anyway (current feature)
- The memory concern is modest (~10MB for 50k URLs)

**Expected Improvement**: Reduced peak memory, immediate processing start, simpler than full streaming.

**Note**: Consider adding `--stream-sitemap` flag for power users who want pure streaming (at cost of losing `lastmod` sorting).

---

## P3 - Low Priority (Minor Tweaks)

### perf-003: Reduce Scroll Sleep Duration

**Impact**: Low | **Effort**: Small | **Confidence**: High

**Problem**: Fixed 500ms sleep per scroll iteration adds latency.

**Location**: `internal/renderer/rod.go` line 239

**Implementation**:
1. Reduce `time.Sleep` from 500ms to 300ms
2. Add early exit if scroll height is stable for 2 consecutive checks

**Why NOT complex idle detection**: The current approach is battle-tested. Using `WaitRequestIdle` in scroll loops is fragile due to persistent connections (analytics, websockets).

**Expected Improvement**: ~30% reduction in scroll time per page (marginal overall impact).

---

### perf-005: Add Configurable Max File Size

**Impact**: Low | **Effort**: Small | **Confidence**: High

**Problem**: Files up to 10MB are read into memory. With high concurrency, this could cause spikes.

**Location**: `internal/strategies/git/processor.go` lines 117-119

**Current Behavior**:
```go
if info.Size() > 10*1024*1024 {
    return nil  // Skip files > 10MB
}
```

**Implementation**:
1. Add `max_file_size` to config (default: 2MB instead of 10MB)
2. Most documentation files are <100KB; 2MB is generous
3. Skip `sync.Pool` complexity - not worth it for this case

**Why NOT sync.Pool**: 
- Adds complexity and subtle bugs (buffer reuse issues)
- Streaming hash then re-reading doubles I/O
- The `Document.Content` field requires the full string anyway

---

## Discarded

### ~~perf-004: Incremental Metadata Flushing~~ 

**Status**: DISCARDED - Premature optimization

**Reason**:
- `SimpleDocumentMetadata` is ~500 bytes per document
- 100k documents = ~50MB - trivial for any machine doing a 100k page crawl
- Real memory hogs are HTTP response buffers and DOM trees (10-100x larger)
- Changing to NDJSON breaks backward compatibility for downstream consumers
- No user would notice the difference

**Alternative**: If memory becomes a real issue, focus on HTTP buffer reuse and DOM cleanup first.

---

## Implementation Order

| Order | ID | Task | Effort | Dependencies |
|-------|-----|------|--------|--------------|
| 1 | perf-001 | Parallelize manifest sources | Medium | None |
| 2 | perf-002 | Batch sitemap processing | Low | None |
| 3 | perf-003 | Reduce scroll sleep | Small | None |
| 4 | perf-005 | Add max_file_size config | Small | None |

---

## Summary

| Category | Count | Details |
|----------|-------|---------|
| Runtime | 2 | perf-001 (high value), perf-003 (low value) |
| Memory | 1 | perf-002 (simplified approach) |
| Config | 1 | perf-005 (user-tunable) |
| Discarded | 1 | perf-004 (non-problem) |

**Estimated Total Savings**:
- perf-001: 2-5x faster for multi-source manifests
- perf-002: Reduced peak memory, faster first-page processing
- perf-003: Marginal (~30% per-page scroll time)
- perf-005: Configurable memory/compatibility tradeoff
