<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/output

Markdown file writing with metadata collection.

## Purpose

Handles saving documents to the filesystem with YAML frontmatter. Supports flat directory structure, dry-run mode, force overwrites, and JSON metadata collection.

## Key Files

| File | Description |
|------|-------------|
| `writer.go` | Writer struct with Write(ctx, doc) for saving documents. WriterOptions (BaseDir, Flat, JSONMetadata, Force, DryRun, Collector). Handles path generation, frontmatter, dry-run mode. |
| `collector.go` | MetadataCollector for aggregating document metadata (thread-safe via sync.RWMutex). CollectorOptions. Writes metadata.json summary. |
| `writer_test.go` | Tests for writing |
| `collector_test.go` | Tests for metadata collection |

## Writer Options

- **BaseDir**: Output directory (default: "./docs")
- **Flat**: Use flat directory structure (no subdirectories)
- **JSONMetadata**: Enable metadata collection
- **Force**: Overwrite existing files
- **DryRun**: Simulate writes without saving

## Metadata Collector

- Thread-safe via sync.RWMutex
- Builds SimpleMetadataIndex with source URL, strategy, document count
- Flush() writes metadata.json to base directory
- Useful for tracking extracted documents

## Dependencies

- **External**: None
- **Internal**: github.com/quantmind-br/repodocs/internal/converter, github.com/quantmind-br/repodocs/internal/domain, github.com/quantmind-br/repodocs/internal/utils

## For AI Agents

- Write() checks for existing files unless Force is true
- Raw files (IsRawFile) use GenerateRawPathFromRelative
- Regular documents use GeneratePathFromRelative or GeneratePath
- Frontmatter added via converter.AddFrontmatter
- FlushMetadata() must be called after writes if JSONMetadata enabled

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->