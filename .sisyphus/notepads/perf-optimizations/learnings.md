# Learnings

## 2026-01-24 Session Start

### Research Findings
- html-to-markdown v2 supports `ConvertNode(*html.Node)` - can use `sel.Get(0)` to extract node from goquery
- Current pipeline parses HTML 2-3 times per document (confirmed in pipeline.go)
- TabPool eagerly creates 5 tabs at startup (pool.go:31-39)
- Discovery probes are sequential for loop (github_pages.go:139-178)

### Key Conventions
- Test location: `tests/unit/<package>/<file>_test.go`
- Use testify: `require.NoError(t, err)` for fatal, `assert.Equal()` for non-fatal
- Error pattern: wrap with context using `fmt.Errorf("failed: %w", err)`
- File size constant: `10*1024*1024` (10MB)

### perf-006: Pre-check File Size Before Reading (COMPLETED)
- **Problem**: `ProcessFile` called `os.ReadFile()` before checking file size, causing OOM with large files
- **Solution**: Use `os.Stat()` to check `info.Size()` BEFORE reading file into memory
- **Pattern**: Always stat large files before loading them into memory
- **Test file**: `tests/unit/git/processor_test.go` with 3 test cases for large/normal/error handling

### perf-004: Eliminate Redundant HTML Parsing in Converter (COMPLETED)
- **Problem**: Pipeline parsed HTML 2-3 times per document: initial goquery parse, then serialization to string, then reparse in `md.ConvertString()`
- **Solution**: Use `md.ConvertNode(*html.Node)` directly instead of serializing DOM to string
- **Implementation**:
  - Added `MarkdownConverter.ConvertNode()` method using `md.ConvertNode()`
  - Added `MarkdownConverter.ConvertNodes()` method for multiple matched elements
  - Updated pipeline.go to pass DOM nodes directly: `sel.Get(0)` returns `*html.Node`
  - Used `htmlpkg` alias for import since `html` is used as parameter name in `Convert()`
- **Pattern**: When you have a parsed DOM tree, pass nodes directly to avoid serialize→reparse overhead
- **Gotcha**: When selector matches multiple elements, must iterate over all matches (not just `Get(0)`)
- **Files modified**: `internal/converter/markdown.go`, `internal/converter/pipeline.go`
- **Test file**: `tests/unit/converter/convert_node_test.go` with golden tests for equivalence

### perf-001: Parallelize GitHub Pages Discovery Probes (COMPLETED)
- **Problem**: `discoverViaHTTPProbes` iterates sequentially through 9 probe URLs. Network requests are the bottleneck - waiting for 404s sequentially adds up (e.g., 9 probes × 200ms = 1.8s latency)
- **Solution**: Fire all probes concurrently using goroutines, collect results via channel, return highest-priority (lowest index) success
- **Implementation**:
  - Use `sync.WaitGroup` to track goroutine completion
  - Results channel to collect successful probe results with priority
  - Close channel when all probes complete, then select best result
  - Goroutines respect context cancellation via fetcher.Get()
- **Critical semantic preserved**: Returns highest-priority probe (first in list), NOT the fastest
- **Pattern**: When multiple network requests are independent, fire all concurrently but prioritize results by original order
- **Files modified**: `internal/strategies/github_pages.go` (discoverViaHTTPProbes function)
- **Test file**: `tests/unit/strategies/github_pages_parallel_test.go` with 4 tests for priority, cancellation, error isolation, and parallelism
