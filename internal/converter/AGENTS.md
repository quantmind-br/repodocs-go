# AGENTS.md - converter

**Generated:** 2026-02-20 | **Package:** internal/converter

HTML → Markdown conversion pipeline with encoding detection, content extraction, sanitization, and markdown generation.

## Pipeline Stages

```
Raw HTML → Encoding → Readability → Sanitizer → Markdown
```

| Stage | File | Purpose |
|-------|------|---------|
| 1. Encoding | `encoding.go` | Detect charset, convert to UTF-8 |
| 2. Extraction | `readability.go` | Extract main content (go-readability) |
| 3. Sanitization | `sanitizer.go` | Remove nav/ads, normalize URLs |
| 4. Conversion | `markdown.go` | HTML nodes → Markdown (html-to-markdown) |
| Orchestration | `pipeline.go` | Coordinates all stages |

## Additional Readers

| File | Purpose |
|------|---------|
| `markdown_reader.go` | Parse existing .md files, extract frontmatter |
| `plaintext_reader.go` | Handle plain text content |
| `content_type.go` | Detect content type (HTML/Markdown/plaintext) |

## Where to Look

| Task | File | Key Functions |
|------|------|---------------|
| Fix encoding issues | `encoding.go` | `DetectEncoding`, `ConvertToUTF8` |
| Content not extracting | `readability.go` | `ExtractContent.Extract`, `extractWithSelector` |
| Unwanted elements in output | `sanitizer.go` | `TagsToRemove`, `ClassesToRemove`, `IDsToRemove` |
| Markdown formatting | `markdown.go` | `MarkdownConverter.Convert`, `cleanMarkdown` |
| Add CSS selector support | `pipeline.go` | `ConvertHTMLWithSelector` |
| Frontmatter parsing | `markdown_reader.go` | `MarkdownReader.Read`, `parseFrontmatter` |
| Content type detection | `content_type.go` | `IsHTMLContent`, `IsMarkdownContent` |

## Key Types

```go
Pipeline           // Main orchestrator: NewPipeline(opts) → Convert(html)
ExtractContent     // Readability wrapper with CSS selector support
Sanitizer          // URL normalization, element removal
MarkdownConverter  // html-to-markdown with cleanup
MarkdownReader     // Existing markdown file parser
PlainTextReader    // Plain text file parser
```

## Notes

- **encoding.go**: Only file with `nolint` directives (charset detection edge cases)
- Sanitizer uses configurable blocklists: `TagsToRemove`, `ClassesToRemove`, `IDsToRemove`
- Pipeline accepts `ContentSelector` and `ExcludeSelector` for targeted extraction
- `GenerateFrontmatter()` and `AddFrontmatter()` in markdown.go for YAML metadata
