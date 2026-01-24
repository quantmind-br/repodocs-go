# Performance Optimizations - Analyzed & Prioritized

> **Analysis Date**: 2025-01-24
> **Status**: Reviewed and reordered by implementation priority

## Implementation Order

Ordered by: Dependencies → Value/Effort ratio → Risk mitigation

---

## Phase 1: Critical Fixes (Correctness)

### perf-006: Pre-check File Size Before Reading
**Priority**: HIGH | **Effort**: TRIVIAL | **Category**: memory

**Problem**: `ProcessFile` in `internal/strategies/git/processor.go` reads entire file with `os.ReadFile()` before checking the 10MB limit. With 50 concurrent workers, large files cause memory spikes or OOM.

**Current (Bug)**:
```go
content, err := os.ReadFile(path) // Entire file in memory
if len(content) > 10*1024*1024 {  // Check too late
    return nil
}
```

**Fix**:
```go
info, err := os.Stat(path)
if err != nil { return err }
if info.Size() > 10*1024*1024 {
    return nil // Skip before reading
}
content, err := os.ReadFile(path)
```

**Affected Files**: `internal/strategies/git/processor.go`

---

## Phase 2: High-Impact Optimizations

### perf-004: Eliminate Triple HTML Parsing in Converter
**Priority**: HIGH | **Effort**: MEDIUM | **Category**: runtime

**Problem**: HTML is parsed up to 3 times per document:
1. `goquery.NewDocumentFromReader` for metadata extraction
2. Readability output → `goquery.NewDocumentFromReader` again
3. `md.ConvertString()` parses HTML internally

**Expected Improvement**: 20-40% CPU reduction in conversion phase (impacts every page)

**Implementation Strategy**:
1. Keep DOM through readability phase instead of serializing to string
2. Investigate `html-to-markdown/v2` API for `ConvertNode` or `ConvertSelection`
3. If no DOM API available, consider streaming adapter or alternative library

**Affected Files**: 
- `internal/converter/pipeline.go`
- `internal/converter/markdown.go`
- `internal/converter/readability.go`

**Tradeoffs**: May require library research or adapter code

---

### perf-002: Lazy Browser Tab Initialization
**Priority**: MEDIUM | **Effort**: MEDIUM | **Category**: runtime

**Problem**: `NewTabPool` eagerly creates all `maxTabs` (default 5) browser pages at startup. Expensive for runs that don't need full concurrency or any browser rendering.

**Current**: Pre-create all tabs in initialization loop
**Proposed**: Create tabs on-demand in `Acquire()` up to `maxTabs`

**Implementation**:
```go
type TabPool struct {
    browser   *rod.Browser
    tabs      chan *rod.Page
    maxTabs   int
    created   int        // NEW: track created count
    createMu  sync.Mutex // NEW: protect creation
}

func (p *TabPool) Acquire(ctx context.Context) (*rod.Page, error) {
    select {
    case tab := <-p.tabs:
        return tab, nil
    default:
        // Try to create new tab if under limit
        p.createMu.Lock()
        defer p.createMu.Unlock()
        if p.created < p.maxTabs {
            tab, err := StealthPage(p.browser)
            if err != nil { return nil, err }
            p.created++
            return tab, nil
        }
    }
    // Block waiting for release
    select {
    case tab := <-p.tabs:
        return tab, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

**Affected Files**:
- `internal/renderer/pool.go`
- `internal/renderer/rod.go`

**Tradeoffs**: First acquisition incurs tab creation latency (acceptable)

---

## Phase 3: Nice-to-Have Optimizations

### perf-001: Parallelize GitHub Pages Discovery Probes
**Priority**: LOW | **Effort**: SMALL | **Category**: network

**Problem**: 9 discovery probes (llms.txt, sitemap.xml, etc.) are fetched sequentially. Adds ~1-2s latency on cold starts.

**Important Constraint**: Current code returns the highest-PRIORITY successful probe, not the fastest. Parallelization must preserve this semantic.

**Implementation**:
```go
func (s *Strategy) discoverViaHTTPProbes(ctx context.Context, baseURL string) ([]string, string, error) {
    probes := GetDiscoveryProbes()
    type result struct {
        priority int
        name     string
        urls     []string
    }
    
    var wg sync.WaitGroup
    results := make(chan result, len(probes))
    
    for i, probe := range probes {
        wg.Add(1)
        go func(idx int, p DiscoveryProbe) {
            defer wg.Done()
            // ... fetch and parse ...
            if len(urls) > 0 {
                results <- result{priority: idx, name: p.Name, urls: urls}
            }
        }(i, probe)
    }
    
    go func() { wg.Wait(); close(results) }()
    
    // Collect all successful, return highest priority (lowest index)
    var best *result
    for r := range results {
        if best == nil || r.priority < best.priority {
            best = &r
        }
    }
    // ...
}
```

**Affected Files**: `internal/strategies/github_pages.go`

**Tradeoffs**: Slight increase in concurrent network requests

---

## Deferred Features

### perf-005: Pipeline Git File Discovery and Processing
**Status**: DEFERRED | **Reason**: Low ROI for typical documentation repos

**Problem**: `FindDocumentationFiles` uses blocking `filepath.WalkDir`, collecting all paths before processing starts.

**Why Deferred**:
- Documentation repos are typically small (<1000 files)
- `WalkDir` is fast for small trees
- Channel-based refactor adds complexity (error handling, backpressure)
- Profile first to confirm this is actually a bottleneck

**If Needed Later**:
- Refactor to write discovered paths to a buffered channel
- `ProcessFiles` reads from channel concurrently
- Handle context cancellation and early termination

---

## Removed Features

### ~~perf-003: Enable Compression for BadgerDB Cache~~
**Status**: DISCARDED | **Reason**: Likely already enabled by default

**Original Claim**: BadgerDB stores HTML uncompressed

**Finding**: BadgerDB v4 enables ZSTD compression by default on base levels. No explicit configuration needed unless trying to DISABLE compression.

**Action**: No implementation required. If disk usage becomes a concern, verify actual compression status first.

---

## Summary

| Phase | ID | Feature | Effort | Impact |
|-------|-----|---------|--------|--------|
| 1 | perf-006 | Pre-check file size | Trivial | Prevents OOM |
| 2 | perf-004 | Eliminate triple parsing | Medium | -30% CPU |
| 2 | perf-002 | Lazy tab init | Medium | Faster startup |
| 3 | perf-001 | Parallel probes | Small | -1-2s discovery |
| - | perf-005 | Pipeline git walk | - | Deferred |
| - | perf-003 | BadgerDB compression | - | Discarded |

**Estimated Total Impact**: 
- Faster cold starts (lazy tabs + parallel probes)
- ~30% CPU reduction in conversion phase
- Memory safety for large file handling
- No unnecessary complexity from deferred/discarded items
