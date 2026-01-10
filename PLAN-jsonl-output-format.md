# PLAN: JSONL Output Format

> **Phase**: 2.1 (Output Flexibility)  
> **Priority**: Medium  
> **Status**: Ready for Implementation  
> **Complexity**: Low | **Value**: Medium

## Overview

Add structured JSONL output format for seamless downstream tool integration.

## Problem Solved

Current output is Markdown files. Users building RAG pipelines need structured data for ingestion into LangChain, LlamaIndex, or custom pipelines.

## User Benefit

Clean handoff to RAG ingestion tools without intermediate parsing.

## Implementation Notes

### CLI Flag

Add `--output-format` flag with options:
- `markdown` (default) - current behavior
- `jsonl` - one JSON document per line

### JSONL Format

One document per line:

```json
{"url": "...", "title": "...", "content": "...", "source": "crawler", "fetched_at": "...", "tags": [...], "summary": "..."}
```

### Output File

When using `--output-format jsonl`:
- Write to `<output-dir>/documents.jsonl`
- Each line is a complete JSON object
- Include all frontmatter metadata in JSON object

### Compatibility

- Compatible with LangChain document loaders
- Compatible with LlamaIndex SimpleDirectoryReader (with custom parser)
- Easy to parse with standard tools (`jq`, Python `json` module)

## Affected Areas

- `internal/output/` (new jsonl.go)
- `cmd/repodocs/main.go` (new flag)
- `internal/config/config.go`

## Tasks

- [ ] Add `--output-format` flag to CLI
- [ ] Create `internal/output/jsonl.go` with JSONL writer
- [ ] Define JSONL document schema (based on Document model)
- [ ] Integrate JSONL writer with orchestrator
- [ ] Handle concurrent writes (buffered/mutex)
- [ ] Write unit tests for JSONL writer
- [ ] Write integration test for JSONL output
- [ ] Add documentation with LangChain/LlamaIndex integration examples

## Dependencies

- None (can be implemented independently)

## Acceptance Criteria

- [ ] `--output-format jsonl` produces valid JSONL file
- [ ] Each line is valid JSON
- [ ] All document metadata is included (url, title, content, source, fetched_at, tags, summary, etc.)
- [ ] File can be parsed by `jq` and Python's `json` module
- [ ] Works with both single-URL and manifest modes
- [ ] Documentation includes integration examples for LangChain and LlamaIndex
