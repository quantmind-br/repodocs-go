# AGENTS.md - internal/domain

Core interfaces, models, and sentinel errors. All other packages depend on this.

## Interfaces

| Interface | Purpose |
|-----------|---------|
| `Strategy` | Extraction strategy (Name, CanHandle, Execute) |
| `Fetcher` | HTTP client with caching |
| `Renderer` | Headless browser for JS sites |
| `Cache` | Persistent cache operations |
| `Converter` | HTML→Markdown conversion |
| `Writer` | Markdown output with frontmatter |
| `LLMProvider` | AI completion interface |

## Models

| Type | File | Purpose |
|------|------|---------|
| `Document` | models.go | Processed page with content, metadata |
| `Page` | models.go | Raw fetched page before conversion |
| `CacheEntry` | models.go | Cached HTTP response |
| `Sitemap` | models.go | Parsed sitemap XML |
| `Frontmatter` | models.go | YAML metadata for markdown |

## Deprecated Types (avoid)

| Old | Replacement |
|-----|-------------|
| `Metadata` | `SimpleMetadata` |
| `MetadataIndex` | `SimpleMetadataIndex` |
| `DocumentMetadata` | `SimpleDocumentMetadata` |

## Errors

Sentinel errors in `errors.go`:
- `ErrCacheMiss` - Cache key not found
- `ErrFetchFailed` - HTTP request failed
- `ErrInvalidURL` - URL validation failed
- `ErrStrategyNotFound` - No strategy handles URL

## Where to Look

| Task | Notes |
|------|-------|
| Add new interface | Define here, implement in package |
| Model changes | Update converters/writers accordingly |
| Error handling | Add sentinel errors here |

## Anti-Patterns

- **NO domain logic** - This is pure interfaces + data types
- **NO implementation** - Implementations go in packages
