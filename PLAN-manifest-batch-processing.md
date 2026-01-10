# PLAN: Project Manifest / Batch Processing

> **Phase**: 1.1 (Foundation)  
> **Priority**: High  
> **Status**: Ready for Implementation  
> **Complexity**: Medium | **Value**: Very High

## Overview

Support a YAML/JSON manifest file that defines multiple documentation sources with per-source configurations. Enables reproducible, one-command data ingestion for complex RAG pipelines.

## Problem Solved

Users building RAG pipelines need to ingest from multiple sources (GitHub repo + marketing site + API reference). Running CLI manually for each URL is error-prone and tedious.

## User Benefit

Enables reproducible, version-controllable, one-command data ingestion for multi-source knowledge bases.

## Implementation Notes

### New Components

- New package: `internal/manifest` (loader.go, types.go)
- Add `--manifest <path>` flag to CLI

### Manifest Schema

```yaml
sources:
  - url: https://docs.example.com
    strategy: crawler  # optional, auto-detect if omitted
    content_selector: "article.main"
    exclude: ["*/changelog/*"]
    max_depth: 3
  - url: https://github.com/org/repo
    strategy: git
    include: ["docs/**/*.md"]
options:
  continue_on_error: true  # per-run default
  output: ./knowledge-base
```

### Behavior

- Process sources sequentially (simplifies error handling)
- Per-source progress reporting
- `continue_on_error` option (default: false) to handle partial failures

## Affected Areas

- `cmd/repodocs/main.go` (new flag)
- `internal/app/orchestrator.go` (manifest execution loop)

## Tasks

- [ ] Define manifest types in `internal/manifest/types.go`
- [ ] Implement YAML/JSON loader in `internal/manifest/loader.go`
- [ ] Add `--manifest` flag to CLI
- [ ] Implement manifest execution loop in orchestrator
- [ ] Add per-source progress reporting
- [ ] Handle `continue_on_error` behavior
- [ ] Write unit tests for manifest parsing
- [ ] Write integration test for multi-source extraction
- [ ] Update documentation/README

## Dependencies

- None (foundational feature)

## Acceptance Criteria

- [ ] Can define multiple sources in a single manifest file
- [ ] Each source can have independent configuration (selectors, depth, exclude patterns)
- [ ] CLI accepts `--manifest` flag and processes all sources
- [ ] Progress is reported per-source
- [ ] `continue_on_error: true` allows partial success
- [ ] `continue_on_error: false` fails fast on first error
