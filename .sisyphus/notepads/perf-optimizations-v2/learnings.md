# Learnings - perf-optimizations-v2

## Session: ses_40e8c97b9ffeEKG7d0Px0kjHiG (2026-01-24)

### Context
Implementing 4 performance optimizations:
1. perf-001: Parallelize Manifest Source Processing (orchestrator.go)
2. perf-002: Batch Sitemap Index Processing (sitemap.go)
3. perf-003: Reduce Scroll Sleep Duration (rod.go)
4. perf-005: Add Configurable Max File Size (git/processor.go + config.go)

### Conventions Discovered
- [Pending - will be filled as work progresses]

### Patterns Found
- `utils.ParallelForEach` is the standard pattern for concurrent processing
- Used in: sitemap.go, docsrs.go, llms.go, github_pages.go, git/processor.go
- Pattern: `errors := utils.ParallelForEach(ctx, items, concurrency, func(ctx, item) error)`

### perf-001: Parallelize Manifest Source Processing (Completed)

#### Implementation Pattern
- Replaced sequential for loop with `utils.ParallelForEach`
- Concurrency capped at min(workers, 3) to avoid overwhelming resources
- Used `sync.Mutex` to protect shared results slice
- Created cancellation context for ContinueOnError=false mode

#### Key Learnings
1. **ContinueOnError behavior changes with parallelization**:
   - Sequential: stops before reaching later sources
   - Parallel: all sources within concurrency limit start, cancellation stops pending
   - Existing tests may need adjustment for new parallel behavior

2. **Index preservation pattern**: Use `sourceWithIndex` struct to track original position:
   ```go
   type sourceWithIndex struct {
       source manifest.Source
       index  int
   }
   ```

3. **Two cancellation modes**:
   - ContinueOnError=true: use parent ctx, process all
   - ContinueOnError=false: create child ctx with cancel, cancel on first error

4. **Log field naming convention**: `source_idx` and `source_url` for manifest logging

### perf-002: Batch Sitemap Index Processing (Completed)

#### Implementation Pattern
- Process each nested sitemap immediately after fetching (instead of collecting all first)
- Track cumulative processed count for limit enforcement
- Maintain progress bar across batches with updateProgressTotal

#### Key Learnings
1. **Batched processing improves UX**: Users see progress immediately instead of waiting
2. **Limit tracking across batches**: Track `totalProcessed` and calculate remaining for each batch

### perf-003: Reduce Scroll Sleep Duration (Completed)

#### Implementation Pattern
- Reduced sleep from 500ms to 300ms (~40% faster per iteration)
- Added stable height detection: exit early after 2 consecutive stable readings
- Maintained max 10 iterations safety limit

#### Key Learnings
1. **Early exit optimization**: Most static pages stabilize quickly, no need for full 10 iterations
2. **Stable counter reset**: Must reset when height changes to avoid false positives

### perf-005: Add Configurable Max File Size (Completed)

#### Implementation Pattern
- Added `GitConfig` struct with `MaxFileSize` string field
- Implemented `ParseSize()` helper supporting KB, MB, GB suffixes (case-insensitive)
- Added validation in `Config.Validate()` - invalid sizes reset to default

#### Key Learnings
1. **Human-readable config**: "10MB" is more user-friendly than bytes
2. **Graceful defaults**: Invalid/empty values fall back to 10MB default
3. **Config validation pattern**: Validate and log warning, but don't fail

---

## Final Summary

All 4 optimizations completed and pushed:
- **perf-001**: `5ebf4d8` - perf(orchestrator): parallelize manifest source processing
- **perf-002**: `2f139d0` - perf(sitemap): batch sitemap index processing for faster start
- **perf-003**: `a8e4505` - perf(renderer): reduce scroll sleep for faster page processing
- **perf-005**: `23bc030` - feat(config): add configurable max file size for git strategy

Test files created:
- `tests/unit/app/orchestrator_parallel_manifest_test.go`
- `tests/unit/strategies/sitemap_batch_test.go`
- `tests/unit/renderer/scroll_test.go`
- `tests/unit/config/git_config_test.go`
