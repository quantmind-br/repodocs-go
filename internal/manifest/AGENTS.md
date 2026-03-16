<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/manifest

Multi-source manifest file loading and validation. Supports YAML and JSON formats.

## Purpose

Loads and validates RepoDocs manifest files that define multiple documentation sources with per-source configurations. Enables batch processing of diverse documentation sources.

## Key Files

| File | Description |
|------|-------------|
| `doc.go` | Package documentation |
| `types.go` | Config (Sources + Options), Source (URL, Strategy, selectors, filters), Options (ContinueOnError, Output, Concurrency, CacheTTL). Validate() and DefaultOptions(). |
| `loader.go` | Loader struct with Load(path) and LoadFromBytes(data, ext). Applies defaults after parsing. |
| `errors.go` | Sentinel errors (ErrNoSources, ErrEmptyURL, ErrInvalidFormat, ErrFileNotFound, ErrUnsupportedExt) |
| `loader_test.go` | Tests for loading |
| `types_test.go` | Tests for validation |

## Types

- **Config**: Sources []Source, Options Options
- **Source**: URL, Strategy, ContentSelector, ExcludeSelector, Exclude, Include, MaxDepth, RenderJS, Limit
- **Options**: ContinueOnError, Output, Concurrency, CacheTTL

## Sentinel Errors

- ErrNoSources: manifest has no sources defined
- ErrEmptyURL: source is missing required URL field
- ErrInvalidFormat: file is not valid YAML or JSON
- ErrFileNotFound: manifest file does not exist
- ErrUnsupportedExt: unsupported file extension

## Dependencies

- **External**: gopkg.in/yaml.v3
- **Internal**: None

## For AI Agents

- Supported formats: .yaml, .yml, .json
- Defaults applied: Output: "./docs", Concurrency: 5, CacheTTL: 24h
- Validate() checks for at least one source with non-empty URL

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->