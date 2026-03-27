<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/state

Sync state management for incremental updates. Tracks which pages have been processed.

## Purpose

Maintains synchronization state to enable incremental updates. Uses content hashing to detect changed pages. Supports pruning of deleted pages between sync runs.

## Key Files

| File | Description |
|------|-------------|
| `manager.go` | Manager struct with Load/Save/MarkSeen/ShouldProcess/GetDeletedPages/RemoveDeletedFromState/Stats. Uses content hashing for change detection. Thread-safe via sync.RWMutex + sync.Map for seenURLs. |
| `models.go` | SyncState (versioned, with Pages map), PageState (ContentHash, FetchedAt, FilePath). StateVersion = 1. |
| `errors.go` | ErrStateNotFound, ErrStateCorrupted, ErrVersionMismatch |
| `state_test.go` | Tests |

## State File

- Location: .repodocs-state.json in output directory
- Format: JSON with version, source URL, strategy, last sync time, pages map

## Models

- **SyncState**: Version, SourceURL, Strategy, LastSync, Pages map
- **PageState**: ContentHash, FetchedAt, FilePath
- **StateVersion**: 1

## Dependencies

- **External**: None
- **Internal**: github.com/quantmind-br/repodocs/internal/utils

## For AI Agents

- Load() reads state from disk, returns ErrStateNotFound if missing
- ShouldProcess(url, contentHash) returns true if page missing or hash changed
- Update(url, page) marks a page as processed
- MarkSeen(url) tracks URLs seen in current sync run
- GetDeletedPages() returns pages not seen in current run (for pruning)
- RemoveDeletedFromState() removes unseen pages from state
- Disabled mode skips all operations (for full-sync)

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->