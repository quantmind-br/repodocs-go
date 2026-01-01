# PLAN: Fix Markdown Content Detection in LLMSStrategy

## Problem Statement

When crawling documentation from sources like `platform.claude.com`, the `LLMSStrategy` encounters URLs that serve **raw Markdown content** (Content-Type: `text/markdown`). The current implementation always passes fetched content to the HTML→Markdown converter pipeline, which:

1. Uses `goquery.NewDocumentFromReader()` internally
2. Which uses Go's `golang.org/x/net/html` parser
3. Which has a **hardcoded limit of 512 nested elements**

When Markdown with deeply nested lists (e.g., 18 levels of API parameter documentation) is parsed as HTML, the parser interprets each indented list item as a nested element, quickly exceeding the 512-node limit.

**Error observed:**
```
error="html: open stack of elements exceeds 512 nodes" url=https://platform.claude.com/docs/en/api/java/beta/messages/batches.md
```

**Evidence:**
- Content-Type: `text/markdown; charset=UTF-8`
- File size: 9,565 lines
- Max list nesting: 18 levels (36 spaces @ 2-space indent)

---

## Solution Overview

Detect when fetched content is already Markdown and handle it appropriately, bypassing the HTML parsing pipeline entirely.

**Approach:** Strategy-level detection (Option A from analysis)
- The `LLMSStrategy` already has access to `pageResp.ContentType`
- It should check content type before deciding how to process content
- If Markdown: use a new `MarkdownReader` that extracts metadata without HTML parsing
- If HTML: use existing `converter.Pipeline.Convert()`

---

## Current Code Flow

```
LLMSStrategy.Execute()
    │
    ├── fetcher.Get(ctx, link.URL)
    │       └── Returns: Response{Body, ContentType, ...}
    │
    ├── converter.Convert(ctx, body, url)  ← ALWAYS CALLED, assumes HTML
    │       │
    │       ├── ConvertToUTF8()
    │       ├── ExtractContent.Extract()     ← uses goquery (HTML parser)
    │       ├── Sanitizer.Sanitize()         ← uses goquery (HTML parser)
    │       ├── MarkdownConverter.Convert()  ← html-to-markdown library
    │       └── goquery.NewDocument...()     ← uses goquery (HTML parser) ← FAILS HERE
    │
    └── WriteDocument()
```

**Problem location:** `internal/strategies/llms.go:96`
```go
doc, err := s.converter.Convert(ctx, string(pageResp.Body), link.URL)
```

---

## Implementation Plan

### Phase 1: Content Type Detection Utility

**File:** `internal/converter/content_type.go` (NEW)

Create a utility function to detect Markdown content:

```go
package converter

import "strings"

// IsMarkdownContent checks if the content type or URL indicates markdown content.
// It checks both the Content-Type header and the URL extension.
func IsMarkdownContent(contentType, url string) bool {
    // Check Content-Type header (primary indicator)
    ct := strings.ToLower(contentType)
    if strings.Contains(ct, "text/markdown") ||
       strings.Contains(ct, "text/x-markdown") ||
       strings.Contains(ct, "application/markdown") {
        return true
    }
    
    // Check URL extension (fallback indicator)
    // Extract path from URL, handle query strings
    lowerURL := strings.ToLower(url)
    
    // Remove query string if present
    if idx := strings.Index(lowerURL, "?"); idx != -1 {
        lowerURL = lowerURL[:idx]
    }
    
    // Check common markdown extensions
    if strings.HasSuffix(lowerURL, ".md") ||
       strings.HasSuffix(lowerURL, ".markdown") ||
       strings.HasSuffix(lowerURL, ".mdown") {
        return true
    }
    
    return false
}

// IsHTMLContent checks if the content type indicates HTML content.
// Returns true for empty content type (assumes HTML for backward compatibility).
func IsHTMLContent(contentType string) bool {
    if contentType == "" {
        return true // Default assumption
    }
    ct := strings.ToLower(contentType)
    return strings.Contains(ct, "text/html") ||
           strings.Contains(ct, "application/xhtml")
}
```

**Test file:** `tests/unit/converter/content_type_test.go`

```go
package converter_test

import (
    "testing"
    
    "github.com/quantmind-br/repodocs-go/internal/converter"
    "github.com/stretchr/testify/assert"
)

func TestIsMarkdownContent(t *testing.T) {
    tests := []struct {
        name        string
        contentType string
        url         string
        want        bool
    }{
        // Content-Type based detection
        {
            name:        "text/markdown content type",
            contentType: "text/markdown; charset=UTF-8",
            url:         "https://example.com/docs/page",
            want:        true,
        },
        {
            name:        "text/x-markdown content type",
            contentType: "text/x-markdown",
            url:         "https://example.com/docs/page",
            want:        true,
        },
        {
            name:        "application/markdown content type",
            contentType: "application/markdown",
            url:         "https://example.com/docs/page",
            want:        true,
        },
        {
            name:        "text/html content type",
            contentType: "text/html; charset=utf-8",
            url:         "https://example.com/docs/page.md",
            want:        false, // Content-Type takes precedence
        },
        
        // URL based detection (when content-type doesn't indicate markdown)
        {
            name:        "URL with .md extension",
            contentType: "",
            url:         "https://example.com/docs/readme.md",
            want:        true,
        },
        {
            name:        "URL with .markdown extension",
            contentType: "",
            url:         "https://example.com/docs/readme.markdown",
            want:        true,
        },
        {
            name:        "URL with .md and query string",
            contentType: "",
            url:         "https://example.com/docs/readme.md?v=1",
            want:        true,
        },
        {
            name:        "URL with .html extension",
            contentType: "",
            url:         "https://example.com/docs/page.html",
            want:        false,
        },
        
        // Edge cases
        {
            name:        "empty content type and no extension",
            contentType: "",
            url:         "https://example.com/docs/page",
            want:        false,
        },
        {
            name:        "case insensitive content type",
            contentType: "TEXT/MARKDOWN",
            url:         "https://example.com/docs/page",
            want:        true,
        },
        {
            name:        "case insensitive URL",
            contentType: "",
            url:         "https://example.com/docs/README.MD",
            want:        true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := converter.IsMarkdownContent(tt.contentType, tt.url)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

---

### Phase 2: Markdown Reader

**File:** `internal/converter/markdown_reader.go` (NEW)

Create a reader that processes Markdown content directly without HTML parsing:

```go
package converter

import (
    "crypto/sha256"
    "encoding/hex"
    "net/url"
    "regexp"
    "strings"
    "time"
    
    "github.com/quantmind-br/repodocs-go/internal/domain"
    "gopkg.in/yaml.v3"
)

// MarkdownReader reads and extracts metadata from markdown content
// without using HTML parsing (avoids the 512 node limit issue).
type MarkdownReader struct{}

// NewMarkdownReader creates a new markdown reader
func NewMarkdownReader() *MarkdownReader {
    return &MarkdownReader{}
}

// Frontmatter represents YAML frontmatter commonly found in markdown files
type Frontmatter struct {
    Title       string   `yaml:"title"`
    Description string   `yaml:"description"`
    Summary     string   `yaml:"summary"`
    Author      string   `yaml:"author"`
    Date        string   `yaml:"date"`
    Tags        []string `yaml:"tags"`
    Category    string   `yaml:"category"`
}

// Read processes markdown content and returns a Document.
// It extracts metadata from frontmatter and markdown syntax without HTML parsing.
func (r *MarkdownReader) Read(content, sourceURL string) (*domain.Document, error) {
    // 1. Parse frontmatter if present
    frontmatter, body := r.parseFrontmatter(content)
    
    // 2. Extract title (frontmatter > first # heading)
    title := r.extractTitle(frontmatter, body)
    
    // 3. Extract description (frontmatter > first paragraph)
    description := r.extractDescription(frontmatter, body)
    
    // 4. Extract headers from markdown syntax
    headers := r.extractHeaders(body)
    
    // 5. Extract links from markdown syntax
    links := r.extractLinks(body, sourceURL)
    
    // 6. Calculate statistics
    plainText := StripMarkdown(body)
    wordCount := CountWords(plainText)
    charCount := CountChars(plainText)
    contentHash := r.calculateHash(body)
    
    // 7. Build document
    return &domain.Document{
        URL:            sourceURL,
        Title:          title,
        Description:    description,
        Content:        body,           // Already markdown, use as-is
        HTMLContent:    "",             // No HTML source for markdown files
        FetchedAt:      time.Now(),
        ContentHash:    contentHash,
        WordCount:      wordCount,
        CharCount:      charCount,
        Links:          links,
        Headers:        headers,
        RenderedWithJS: false,
        SourceStrategy: "",             // Set by caller
        CacheHit:       false,          // Set by caller
    }, nil
}

// parseFrontmatter extracts YAML frontmatter from markdown content.
// Returns the frontmatter (if present) and the remaining body content.
func (r *MarkdownReader) parseFrontmatter(content string) (*Frontmatter, string) {
    content = strings.TrimSpace(content)
    
    // Frontmatter must start with ---
    if !strings.HasPrefix(content, "---") {
        return nil, content
    }
    
    // Find the closing ---
    rest := content[3:]
    
    // Skip initial newline if present
    if strings.HasPrefix(rest, "\n") {
        rest = rest[1:]
    } else if strings.HasPrefix(rest, "\r\n") {
        rest = rest[2:]
    }
    
    // Find closing delimiter (--- at start of line)
    var closingIdx int
    lines := strings.Split(rest, "\n")
    yamlLines := []string{}
    foundClosing := false
    
    for i, line := range lines {
        trimmed := strings.TrimRight(line, "\r")
        if trimmed == "---" {
            closingIdx = i
            foundClosing = true
            break
        }
        yamlLines = append(yamlLines, line)
    }
    
    if !foundClosing {
        // No closing ---, treat entire content as body
        return nil, content
    }
    
    // Parse YAML frontmatter
    yamlContent := strings.Join(yamlLines, "\n")
    var fm Frontmatter
    if err := yaml.Unmarshal([]byte(yamlContent), &fm); err != nil {
        // Malformed YAML, return content as-is
        return nil, content
    }
    
    // Extract body (everything after closing ---)
    bodyLines := lines[closingIdx+1:]
    body := strings.TrimSpace(strings.Join(bodyLines, "\n"))
    
    return &fm, body
}

// extractTitle extracts the document title.
// Priority: frontmatter.Title > first # heading
func (r *MarkdownReader) extractTitle(fm *Frontmatter, body string) string {
    // Try frontmatter first
    if fm != nil && fm.Title != "" {
        return fm.Title
    }
    
    // Try first # heading (not inside code block)
    inCodeBlock := false
    lines := strings.Split(body, "\n")
    
    for _, line := range lines {
        trimmed := strings.TrimSpace(line)
        
        // Track code blocks
        if strings.HasPrefix(trimmed, "```") || strings.HasPrefix(trimmed, "~~~") {
            inCodeBlock = !inCodeBlock
            continue
        }
        
        if inCodeBlock {
            continue
        }
        
        // Check for # heading (ATX style)
        if strings.HasPrefix(trimmed, "# ") {
            return strings.TrimSpace(strings.TrimPrefix(trimmed, "# "))
        }
    }
    
    return ""
}

// extractDescription extracts the document description.
// Priority: frontmatter.Description > frontmatter.Summary > first paragraph
func (r *MarkdownReader) extractDescription(fm *Frontmatter, body string) string {
    // Try frontmatter first
    if fm != nil {
        if fm.Description != "" {
            return fm.Description
        }
        if fm.Summary != "" {
            return fm.Summary
        }
    }
    
    // Try first non-empty paragraph (not a heading, not a list, not code)
    inCodeBlock := false
    lines := strings.Split(body, "\n")
    var paragraphLines []string
    
    for _, line := range lines {
        trimmed := strings.TrimSpace(line)
        
        // Track code blocks
        if strings.HasPrefix(trimmed, "```") || strings.HasPrefix(trimmed, "~~~") {
            inCodeBlock = !inCodeBlock
            continue
        }
        
        if inCodeBlock {
            continue
        }
        
        // Skip headings
        if strings.HasPrefix(trimmed, "#") {
            continue
        }
        
        // Skip lists
        if strings.HasPrefix(trimmed, "- ") ||
           strings.HasPrefix(trimmed, "* ") ||
           strings.HasPrefix(trimmed, "+ ") ||
           regexp.MustCompile(`^\d+\.\s`).MatchString(trimmed) {
            continue
        }
        
        // Skip empty lines (end of paragraph)
        if trimmed == "" {
            if len(paragraphLines) > 0 {
                break // Found complete paragraph
            }
            continue
        }
        
        // Skip horizontal rules
        if regexp.MustCompile(`^[-*_]{3,}$`).MatchString(trimmed) {
            continue
        }
        
        paragraphLines = append(paragraphLines, trimmed)
    }
    
    if len(paragraphLines) > 0 {
        desc := strings.Join(paragraphLines, " ")
        // Limit description length
        if len(desc) > 300 {
            desc = desc[:297] + "..."
        }
        return desc
    }
    
    return ""
}

// headingRegex matches markdown headings (ATX style: # Heading)
var headingRegex = regexp.MustCompile(`^(#{1,6})\s+(.+)$`)

// extractHeaders extracts all headers from markdown content.
// Returns a map of heading level (h1-h6) to list of heading texts.
func (r *MarkdownReader) extractHeaders(body string) map[string][]string {
    headers := make(map[string][]string)
    
    inCodeBlock := false
    lines := strings.Split(body, "\n")
    
    for _, line := range lines {
        trimmed := strings.TrimSpace(line)
        
        // Track code blocks
        if strings.HasPrefix(trimmed, "```") || strings.HasPrefix(trimmed, "~~~") {
            inCodeBlock = !inCodeBlock
            continue
        }
        
        if inCodeBlock {
            continue
        }
        
        // Match heading
        matches := headingRegex.FindStringSubmatch(trimmed)
        if len(matches) == 3 {
            level := len(matches[1]) // Number of # characters
            text := strings.TrimSpace(matches[2])
            
            // Remove trailing # if present (closing ATX style)
            text = strings.TrimRight(text, "#")
            text = strings.TrimSpace(text)
            
            if text != "" {
                key := "h" + string('0'+byte(level))
                headers[key] = append(headers[key], text)
            }
        }
    }
    
    return headers
}

// markdownLinkRegex matches markdown links: [text](url) or [text](url "title")
var markdownLinkRegex = regexp.MustCompile(`\[([^\]]*)\]\(([^)\s]+)(?:\s+"[^"]*")?\)`)

// extractLinks extracts all links from markdown content.
// Resolves relative URLs against the base URL.
func (r *MarkdownReader) extractLinks(body, baseURL string) []string {
    var links []string
    seen := make(map[string]bool)
    
    base, _ := url.Parse(baseURL)
    
    inCodeBlock := false
    lines := strings.Split(body, "\n")
    
    for _, line := range lines {
        trimmed := strings.TrimSpace(line)
        
        // Track code blocks (don't extract links from code)
        if strings.HasPrefix(trimmed, "```") || strings.HasPrefix(trimmed, "~~~") {
            inCodeBlock = !inCodeBlock
            continue
        }
        
        if inCodeBlock {
            continue
        }
        
        // Find all links in line
        matches := markdownLinkRegex.FindAllStringSubmatch(line, -1)
        for _, match := range matches {
            if len(match) >= 3 {
                href := strings.TrimSpace(match[2])
                
                // Skip empty, anchor-only, javascript, mailto, tel links
                if href == "" ||
                   strings.HasPrefix(href, "#") ||
                   strings.HasPrefix(href, "javascript:") ||
                   strings.HasPrefix(href, "mailto:") ||
                   strings.HasPrefix(href, "tel:") {
                    continue
                }
                
                // Resolve relative URLs
                if base != nil && !strings.HasPrefix(href, "http://") && !strings.HasPrefix(href, "https://") {
                    if refURL, err := url.Parse(href); err == nil {
                        href = base.ResolveReference(refURL).String()
                    }
                }
                
                // Deduplicate
                if !seen[href] {
                    seen[href] = true
                    links = append(links, href)
                }
            }
        }
    }
    
    return links
}

// calculateHash computes SHA256 hash of content
func (r *MarkdownReader) calculateHash(content string) string {
    hash := sha256.Sum256([]byte(content))
    return hex.EncodeToString(hash[:])
}
```

**Test file:** `tests/unit/converter/markdown_reader_test.go`

```go
package converter_test

import (
    "testing"
    
    "github.com/quantmind-br/repodocs-go/internal/converter"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestMarkdownReader_Read(t *testing.T) {
    reader := converter.NewMarkdownReader()
    
    t.Run("full document with frontmatter", func(t *testing.T) {
        content := `---
title: Test Document
description: A test description
tags:
  - test
  - example
---

# Introduction

This is the first paragraph.

## Section One

Some content with a [link](https://example.com).

### Subsection

- List item 1
- List item 2
`
        
        doc, err := reader.Read(content, "https://example.com/docs/test.md")
        require.NoError(t, err)
        
        assert.Equal(t, "Test Document", doc.Title)
        assert.Equal(t, "A test description", doc.Description)
        assert.Equal(t, "https://example.com/docs/test.md", doc.URL)
        assert.NotEmpty(t, doc.ContentHash)
        assert.Greater(t, doc.WordCount, 0)
        
        // Check headers
        assert.Contains(t, doc.Headers, "h1")
        assert.Contains(t, doc.Headers, "h2")
        assert.Contains(t, doc.Headers, "h3")
        assert.Equal(t, []string{"Introduction"}, doc.Headers["h1"])
        
        // Check links
        assert.Contains(t, doc.Links, "https://example.com")
    })
    
    t.Run("document without frontmatter", func(t *testing.T) {
        content := `# My Title

This is the description paragraph.

## Section

Content here.
`
        
        doc, err := reader.Read(content, "https://example.com/doc.md")
        require.NoError(t, err)
        
        assert.Equal(t, "My Title", doc.Title)
        assert.Equal(t, "This is the description paragraph.", doc.Description)
    })
    
    t.Run("relative links are resolved", func(t *testing.T) {
        content := `Check out [the guide](./guide.md) and [API docs](/api/reference.md).`
        
        doc, err := reader.Read(content, "https://example.com/docs/intro.md")
        require.NoError(t, err)
        
        assert.Contains(t, doc.Links, "https://example.com/docs/guide.md")
        assert.Contains(t, doc.Links, "https://example.com/api/reference.md")
    })
    
    t.Run("code blocks are ignored for headers and links", func(t *testing.T) {
        content := "# Real Title\n\n```markdown\n# Not a title\n[not a link](http://ignored.com)\n```\n"
        
        doc, err := reader.Read(content, "https://example.com/test.md")
        require.NoError(t, err)
        
        assert.Equal(t, "Real Title", doc.Title)
        assert.Len(t, doc.Headers["h1"], 1)
        assert.NotContains(t, doc.Links, "http://ignored.com")
    })
    
    t.Run("empty content", func(t *testing.T) {
        doc, err := reader.Read("", "https://example.com/empty.md")
        require.NoError(t, err)
        
        assert.Equal(t, "", doc.Title)
        assert.Equal(t, 0, doc.WordCount)
    })
    
    t.Run("malformed frontmatter is treated as content", func(t *testing.T) {
        content := `---
invalid: yaml: content: [
---

# Title
`
        
        doc, err := reader.Read(content, "https://example.com/test.md")
        require.NoError(t, err)
        
        // Should not extract from malformed frontmatter
        // Title might come from # heading or be empty
        assert.NotEmpty(t, doc.Content)
    })
}

func TestMarkdownReader_ParseFrontmatter(t *testing.T) {
    reader := converter.NewMarkdownReader()
    
    tests := []struct {
        name         string
        content      string
        wantTitle    string
        wantHasBody  bool
    }{
        {
            name: "standard frontmatter",
            content: `---
title: Hello World
---

Body content`,
            wantTitle:   "Hello World",
            wantHasBody: true,
        },
        {
            name: "no frontmatter",
            content: `# Title

Body content`,
            wantTitle:   "",
            wantHasBody: true,
        },
        {
            name: "frontmatter only",
            content: `---
title: Only Frontmatter
---`,
            wantTitle:   "Only Frontmatter",
            wantHasBody: false,
        },
        {
            name: "unclosed frontmatter",
            content: `---
title: Unclosed

Body content`,
            wantTitle:   "",
            wantHasBody: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            doc, err := reader.Read(tt.content, "https://example.com/test.md")
            require.NoError(t, err)
            
            if tt.wantTitle != "" {
                assert.Equal(t, tt.wantTitle, doc.Title)
            }
            if tt.wantHasBody {
                assert.NotEmpty(t, doc.Content)
            }
        })
    }
}

func TestMarkdownReader_ExtractHeaders(t *testing.T) {
    reader := converter.NewMarkdownReader()
    
    content := `# H1 Title

## H2 Section

### H3 Subsection

#### H4 Deep

##### H5 Deeper

###### H6 Deepest

## Another H2
`
    
    doc, err := reader.Read(content, "https://example.com/test.md")
    require.NoError(t, err)
    
    assert.Len(t, doc.Headers["h1"], 1)
    assert.Len(t, doc.Headers["h2"], 2)
    assert.Len(t, doc.Headers["h3"], 1)
    assert.Len(t, doc.Headers["h4"], 1)
    assert.Len(t, doc.Headers["h5"], 1)
    assert.Len(t, doc.Headers["h6"], 1)
    
    assert.Equal(t, "H1 Title", doc.Headers["h1"][0])
    assert.Contains(t, doc.Headers["h2"], "H2 Section")
    assert.Contains(t, doc.Headers["h2"], "Another H2")
}

func TestMarkdownReader_ExtractLinks(t *testing.T) {
    reader := converter.NewMarkdownReader()
    
    content := `
[External](https://external.com)
[Internal](/docs/page.md)
[Relative](./sibling.md)
[With Title](https://example.com "Example Site")
[Anchor Only](#section)
[Email](mailto:test@example.com)
[Phone](tel:+1234567890)
[JavaScript](javascript:void(0))
`
    
    doc, err := reader.Read(content, "https://base.com/docs/current.md")
    require.NoError(t, err)
    
    // Should include
    assert.Contains(t, doc.Links, "https://external.com")
    assert.Contains(t, doc.Links, "https://base.com/docs/page.md")
    assert.Contains(t, doc.Links, "https://base.com/docs/sibling.md")
    assert.Contains(t, doc.Links, "https://example.com")
    
    // Should NOT include
    for _, link := range doc.Links {
        assert.NotContains(t, link, "#section")
        assert.NotContains(t, link, "mailto:")
        assert.NotContains(t, link, "tel:")
        assert.NotContains(t, link, "javascript:")
    }
}
```

---

### Phase 3: Integrate in LLMSStrategy

**File:** `internal/strategies/llms.go` (MODIFY)

Add import and modify the Execute method:

```go
// Add to imports:
// "github.com/quantmind-br/repodocs-go/internal/converter"

// Modify Execute method, replace lines 88-100:

// Fetch page
pageResp, err := s.fetcher.Get(ctx, link.URL)
if err != nil {
    s.logger.Warn().Err(err).Str("url", link.URL).Msg("Failed to fetch page")
    return nil // Continue with other pages
}

var doc *domain.Document

// Check if content is already markdown (skip HTML conversion)
if converter.IsMarkdownContent(pageResp.ContentType, link.URL) {
    s.logger.Debug().
        Str("url", link.URL).
        Str("content_type", pageResp.ContentType).
        Msg("Detected markdown content, using direct reader")
    
    mdReader := converter.NewMarkdownReader()
    doc, err = mdReader.Read(string(pageResp.Body), link.URL)
    if err != nil {
        s.logger.Warn().Err(err).Str("url", link.URL).Msg("Failed to read markdown")
        return nil
    }
} else {
    // Standard HTML to Markdown conversion
    doc, err = s.converter.Convert(ctx, string(pageResp.Body), link.URL)
    if err != nil {
        s.logger.Warn().Err(err).Str("url", link.URL).Msg("Failed to convert page")
        return nil
    }
}

// Set metadata (unchanged from here)
doc.SourceStrategy = s.Name()
doc.CacheHit = pageResp.FromCache
doc.FetchedAt = time.Now()
```

**Test file:** `tests/unit/strategies/llms_test.go` (ADD TESTS)

Add test cases for markdown content handling:

```go
func TestLLMSStrategy_MarkdownContent(t *testing.T) {
    // Test that markdown content is handled without HTML parsing
    // Mock server returning text/markdown content-type
    // Verify document is created successfully
}

func TestLLMSStrategy_MixedContent(t *testing.T) {
    // Test handling both HTML and markdown URLs in same batch
}
```

---

### Phase 4: Consider Other Strategies

Check if other strategies might encounter markdown content:

| Strategy | Could encounter markdown? | Action needed? |
|----------|---------------------------|----------------|
| `CrawlerStrategy` | Unlikely (filters by `text/html`) | No - already filters |
| `SitemapStrategy` | Possible (sitemaps can list .md files) | Maybe - review |
| `PkgGoStrategy` | No (pkg.go.dev serves HTML) | No |
| `GitStrategy` | Yes (repos contain .md files) | Already handles (reads files directly) |

**Recommendation:** Focus on LLMSStrategy first. If sitemaps are found to list markdown URLs, apply same pattern there.

---

## File Summary

### New Files to Create

| File | Purpose |
|------|---------|
| `internal/converter/content_type.go` | Content type detection utilities |
| `internal/converter/markdown_reader.go` | Markdown content reader (no HTML parsing) |
| `tests/unit/converter/content_type_test.go` | Unit tests for content type detection |
| `tests/unit/converter/markdown_reader_test.go` | Unit tests for markdown reader |

### Files to Modify

| File | Changes |
|------|---------|
| `internal/strategies/llms.go` | Add markdown detection and handling |
| `tests/unit/strategies/llms_test.go` | Add tests for markdown handling |

---

## Testing Plan

### Unit Tests

1. **content_type_test.go**
   - Content-Type header detection (text/markdown, text/x-markdown, application/markdown)
   - URL extension detection (.md, .markdown, .mdown)
   - Case insensitivity
   - Query string handling
   - Priority (Content-Type over URL)

2. **markdown_reader_test.go**
   - Frontmatter parsing (valid, invalid, missing)
   - Title extraction (frontmatter, heading, none)
   - Description extraction (frontmatter, paragraph, none)
   - Header extraction (all levels, code block exclusion)
   - Link extraction (absolute, relative, filtered types)
   - Edge cases (empty, code blocks, deeply nested)

3. **llms_test.go additions**
   - Markdown content detection and handling
   - HTML content still works
   - Mixed content batch processing

### Integration Tests

1. **llms_integration_test.go**
   - Real markdown file processing
   - End-to-end with mock server

### Manual Testing

```bash
# Test with the problematic URL
./build/repodocs llms https://platform.claude.com/llms.txt \
    --filter-url "https://platform.claude.com/docs/en/api/java/beta/messages/batches" \
    --limit 5 \
    -v

# Verify no "512 nodes" errors
# Verify markdown files are processed correctly
```

---

## Rollout Steps

1. **Create content_type.go** with tests (Phase 1)
2. **Create markdown_reader.go** with tests (Phase 2)
3. **Run all existing tests** - ensure no regression
4. **Modify llms.go** with new detection logic (Phase 3)
5. **Add llms.go tests** for new behavior
6. **Manual test** with platform.claude.com
7. **Review SitemapStrategy** for similar issues (Phase 4)

---

## Success Criteria

1. No `html: open stack of elements exceeds 512 nodes` errors
2. All previously failing URLs now succeed
3. Markdown documents have correct:
   - Title (from frontmatter or first heading)
   - Description (from frontmatter or first paragraph)
   - Headers (all levels extracted)
   - Links (resolved to absolute URLs)
   - Word count and content hash
4. HTML documents continue to work as before
5. All unit tests pass
6. All integration tests pass

---

## Estimated Effort

| Phase | Estimated Time |
|-------|----------------|
| Phase 1: Content type detection | 30 min |
| Phase 2: Markdown reader | 1.5 hours |
| Phase 3: LLMSStrategy integration | 30 min |
| Phase 4: Other strategies review | 15 min |
| Testing & validation | 1 hour |
| **Total** | **~4 hours** |

---

## Dependencies

### External Packages (already in go.mod)

- `gopkg.in/yaml.v3` - For frontmatter parsing (already used elsewhere)

### No New Dependencies Required

The implementation uses only:
- Standard library (`regexp`, `strings`, `net/url`, `crypto/sha256`)
- Existing internal packages
- Existing yaml.v3 dependency

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Markdown parsing misses edge cases | Medium | Low | Comprehensive test suite |
| Content-Type detection false positives | Low | Medium | Check both header AND extension |
| Performance regression | Low | Low | Markdown parsing is simpler than HTML |
| Breaking existing HTML handling | Low | High | Extensive test coverage, careful integration |

---

## Future Enhancements (Out of Scope)

1. **Reference-style link support**: `[text][ref]` ... `[ref]: url`
2. **Setext-style headers**: Underlined headers (`===`, `---`)
3. **Table extraction**: Parse markdown tables
4. **Image extraction**: Parse `![alt](src)` syntax
5. **HTML embedded in markdown**: Handle mixed content

These can be added later if needed.
