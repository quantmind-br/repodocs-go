# PLAN: Incremental Synchronization

> **Phase**: 1.2 (Foundation)  
> **Priority**: High  
> **Status**: Ready for Implementation (after 1.1)  
> **Complexity**: Medium | **Value**: Very High

## Overview

Track processed page hashes to enable incremental updates. Subsequent runs only fetch and process changed content.

## Problem Solved

Re-running extraction on large sites is slow and expensive (compute, network, LLM API costs for metadata enhancement). Users need efficient sync without full re-processing.

## User Benefit

10-100x faster maintenance runs. Enables scheduled sync workflows (cron).

## Implementation Notes

### New Components

- New package: `internal/state` (manager.go, models.go)
- Leverage existing `ContentHash` field in `domain.Document`

### State Storage

JSON file in output dir (`.repodocs-state.json`) or BadgerDB

### State Schema

```json
{
  "version": 1,
  "source_url": "https://docs.example.com",
  "last_sync": "2026-01-09T12:00:00Z",
  "pages": {
    "https://docs.example.com/intro": {
      "content_hash": "abc123...",
      "fetched_at": "2026-01-09T12:00:00Z",
      "file_path": "docs/intro.md"
    }
  }
}
```

### CLI Flags

- `--sync`: enable incremental mode (explicit opt-in)
- `--full-sync`: force complete re-processing (ignore state)

### Deletion Tracking

Pages in state but not discovered during crawl should be:
- Marked as deleted in index, OR
- Have their files removed (with `--prune` flag)

## Affected Areas

- `internal/strategies/crawler.go`, `git.go` (state check before processing)
- `internal/output/writer.go` (state update after write)

## Tasks

- [ ] Define state models in `internal/state/models.go`
- [ ] Implement state manager in `internal/state/manager.go`
- [ ] Add `--sync` and `--full-sync` flags to CLI
- [ ] Integrate state check in crawler strategy (skip unchanged pages)
- [ ] Integrate state check in git strategy
- [ ] Update state after successful write
- [ ] Implement deletion detection
- [ ] Add `--prune` flag to remove deleted files
- [ ] Write unit tests for state manager
- [ ] Write integration test for incremental sync
- [ ] Update documentation/README

## Dependencies

- Phase 1.1 (Manifest) should be complete first (for multi-source state management)

## Acceptance Criteria

- [ ] First run creates state file in output directory
- [ ] Subsequent `--sync` runs skip unchanged pages (based on content hash)
- [ ] Changed pages are re-fetched and re-processed
- [ ] New pages are processed normally
- [ ] `--full-sync` ignores state and re-processes everything
- [ ] Deleted pages are detected and optionally pruned
- [ ] State file is human-readable (JSON)
- [ ] Measurable performance improvement on large sites (10x+ faster)
