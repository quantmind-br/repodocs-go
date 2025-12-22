# PLAN.md: Inclusion Selector Enhancement

## 1. Executive Summary

This plan enhances the `--content-selector` feature in RepoDocs to provide precise HTML content targeting via CSS selectors. After code analysis, the **current implementation is mostly correct** but has specific issues that need addressing.

**Key Findings:**
- The selector flow (`CLI → Orchestrator → Dependencies → Pipeline → ExtractContent`) is properly wired
- The `extractWithSelector` already falls back to Readability when selector doesn't match
- The Sanitizer receives only the scoped HTML fragment (not the full document)

**Real Issues Identified:**
1. Metadata extraction uses original HTML instead of scoped content for headers/links
2. No support for exclusion selectors (complement to inclusion)
3. Multiple matching elements: only `.First()` is taken without user control
4. No logging/warning when selector doesn't match and fallback is used

---

## 2. Current Architecture Analysis

### 2.1 Data Flow (Working Correctly)
```
cmd/main.go
  └─ --content-selector flag → OrchestratorOptions.ContentSelector
      └─ orchestrator.go → DependencyOptions.ContentSelector
          └─ strategy.go → Options.ContentSelector
              └─ Dependencies.NewPipeline() → PipelineOptions.ContentSelector
                  └─ pipeline.go → ExtractContent.selector
                      └─ readability.go → extractWithSelector()
```

### 2.2 Pipeline.Convert Flow (Current)
```go
// Step 1: UTF-8 conversion
// Step 2: Extract content (selector applied HERE - correct)
// Step 3: Sanitize (receives scoped HTML - correct)
// Step 4: Convert to Markdown
// Step 5: Extract metadata (BUG: uses original 'html' for headers/links)
// Step 6: Calculate statistics
// Step 7: Build document
```

### 2.3 extractWithSelector Behavior (Current)
```go
func (e *ExtractContent) extractWithSelector(html, sourceURL string) (string, string, error) {
    doc, _ := goquery.NewDocumentFromReader(...)
    content := doc.Find(e.selector).First()  // Issue: only first match
    if content.Length() == 0 {
        return e.extractWithReadability(html, sourceURL)  // Fallback works
    }
    // ...
}
```

---

## 3. Issues & Solutions

### Issue 1: Metadata Extraction Uses Wrong Source

**Problem:** In `pipeline.go`, headers and links are extracted from the original `html` instead of the `sanitized` content:
```go
headers := ExtractHeaders(sanitized)  // OK
links := ExtractLinks(sanitized, sourceURL)  // OK - but see below
```

Actually reviewing the code more carefully, this is **already correct**. Both use `sanitized`. No change needed here.

### Issue 2: No Warning on Selector Fallback

**Problem:** When the selector doesn't match, the system silently falls back to Readability. Users may not realize their selector is invalid.

**Solution:** Add logging in `extractWithSelector`:
```go
func (e *ExtractContent) extractWithSelector(html, sourceURL string) (string, string, error) {
    // ...
    if content.Length() == 0 {
        log.Debug().
            Str("selector", e.selector).
            Str("url", sourceURL).
            Msg("Content selector not found, falling back to Readability")
        return e.extractWithReadability(html, sourceURL)
    }
    // ...
}
```

**File:** `internal/converter/readability.go`

### Issue 3: Multiple Elements Behavior

**Problem:** When selector matches multiple elements, only `.First()` is taken. This may not be the desired behavior for selectors like `article` or `.section`.

**Solution:** Add option to combine all matches:
```go
type ExtractContent struct {
    selector    string
    combineAll  bool  // New: if true, combines all matching elements
}
```

**Decision:** For simplicity, use goquery's native comma-separated selector support. Example: `--content-selector "article, .content"` already works with goquery and returns combined content if we adjust to not use `.First()`.

**Proposed Change:**
```go
func (e *ExtractContent) extractWithSelector(html, sourceURL string) (string, string, error) {
    // ...
    content := doc.Find(e.selector)
    if content.Length() == 0 {
        return e.extractWithReadability(html, sourceURL)
    }
    
    // Combine all matches instead of just first
    var combined strings.Builder
    content.Each(func(i int, sel *goquery.Selection) {
        if h, err := sel.Html(); err == nil {
            combined.WriteString(h)
        }
    })
    
    return combined.String(), title, nil
}
```

### Issue 4: Exclusion Selectors

**Problem:** Users may want to exclude specific elements within their selected content (e.g., exclude `.warning` boxes).

**Solution:** Add `--exclude-selector` flag that runs after inclusion:
```go
type PipelineOptions struct {
    BaseURL          string
    ContentSelector  string
    ExcludeSelector  string  // New
}
```

**Implementation:** Apply exclusion after extraction but before sanitization:
```go
// In Pipeline.Convert
content, title, err := p.extractor.Extract(html, sourceURL)
if p.excludeSelector != "" {
    content = p.removeExcluded(content, p.excludeSelector)
}
sanitized, err := p.sanitizer.Sanitize(content)
```

---

## 4. Implementation Plan

### Phase 1: Logging & Diagnostics (Low Risk)
**Priority:** High | **Effort:** 1h

| Task | File | Description |
|------|------|-------------|
| 1.1 | `internal/converter/readability.go` | Add debug logging when selector fallback occurs |
| 1.2 | `internal/converter/readability.go` | Add debug logging showing matched element count |

### Phase 2: Multi-Element Support (Medium Risk)
**Priority:** Medium | **Effort:** 2h

| Task | File | Description |
|------|------|-------------|
| 2.1 | `internal/converter/readability.go` | Change `extractWithSelector` to combine all matches |
| 2.2 | `tests/unit/pipeline_test.go` | Add test for multiple element matching |
| 2.3 | `tests/unit/pipeline_test.go` | Add test for comma-separated selectors |

### Phase 3: Exclusion Selector (Medium Risk)
**Priority:** Low | **Effort:** 3h

| Task | File | Description |
|------|------|-------------|
| 3.1 | `internal/converter/pipeline.go` | Add `ExcludeSelector` to `PipelineOptions` |
| 3.2 | `internal/converter/pipeline.go` | Implement `removeExcluded()` method |
| 3.3 | `cmd/repodocs/main.go` | Add `--exclude-selector` flag |
| 3.4 | `internal/app/orchestrator.go` | Wire `ExcludeSelector` through options |
| 3.5 | `internal/strategies/strategy.go` | Add to `Options` struct |
| 3.6 | `tests/unit/pipeline_test.go` | Add exclusion selector tests |

---

## 5. Detailed Code Changes

### 5.1 Phase 1: readability.go Logging

```go
package converter

import (
    "github.com/rs/zerolog/log"
    // ... existing imports
)

func (e *ExtractContent) extractWithSelector(html, sourceURL string) (string, string, error) {
    doc, err := goquery.NewDocumentFromReader(strings.NewReader(html))
    if err != nil {
        return "", "", err
    }

    content := doc.Find(e.selector)
    matchCount := content.Length()
    
    log.Debug().
        Str("selector", e.selector).
        Int("matches", matchCount).
        Str("url", sourceURL).
        Msg("Content selector applied")

    if matchCount == 0 {
        log.Debug().
            Str("selector", e.selector).
            Str("url", sourceURL).
            Msg("Selector not found, falling back to Readability algorithm")
        return e.extractWithReadability(html, sourceURL)
    }

    title := extractTitle(doc)
    
    // Combine all matches
    var combined strings.Builder
    content.Each(func(i int, sel *goquery.Selection) {
        if h, err := sel.Html(); err == nil {
            combined.WriteString(h)
        }
    })

    return combined.String(), title, nil
}
```

### 5.2 Phase 3: Exclusion Selector

**pipeline.go changes:**
```go
type PipelineOptions struct {
    BaseURL         string
    ContentSelector string
    ExcludeSelector string
}

type Pipeline struct {
    sanitizer       *Sanitizer
    extractor       *ExtractContent
    mdConverter     *MarkdownConverter
    excludeSelector string
}

func NewPipeline(opts PipelineOptions) *Pipeline {
    // ... existing code ...
    return &Pipeline{
        sanitizer:       sanitizer,
        extractor:       extractor,
        mdConverter:     mdConverter,
        excludeSelector: opts.ExcludeSelector,
    }
}

func (p *Pipeline) removeExcluded(html, selector string) string {
    if selector == "" {
        return html
    }
    doc, err := goquery.NewDocumentFromReader(strings.NewReader(html))
    if err != nil {
        return html
    }
    doc.Find(selector).Remove()
    result, _ := doc.Html()
    return result
}
```

---

## 6. Test Cases

### 6.1 New Test: Multiple Element Matching
```go
func TestPipeline_ContentSelector_MultipleMatches(t *testing.T) {
    html := `<!DOCTYPE html>
    <html><body>
        <article class="post">First article</article>
        <article class="post">Second article</article>
        <aside>Sidebar</aside>
    </body></html>`

    pipeline := converter.NewPipeline(converter.PipelineOptions{
        ContentSelector: "article.post",
    })

    doc, err := pipeline.Convert(context.Background(), html, "https://example.com")
    require.NoError(t, err)

    assert.Contains(t, doc.Content, "First article")
    assert.Contains(t, doc.Content, "Second article")
    assert.NotContains(t, doc.Content, "Sidebar")
}
```

### 6.2 New Test: Comma-Separated Selectors
```go
func TestPipeline_ContentSelector_CommaSeparated(t *testing.T) {
    html := `<!DOCTYPE html>
    <html><body>
        <div id="title">Page Title</div>
        <nav>Navigation</nav>
        <div id="body">Main content</div>
    </body></html>`

    pipeline := converter.NewPipeline(converter.PipelineOptions{
        ContentSelector: "#title, #body",
    })

    doc, err := pipeline.Convert(context.Background(), html, "https://example.com")
    require.NoError(t, err)

    assert.Contains(t, doc.Content, "Page Title")
    assert.Contains(t, doc.Content, "Main content")
    assert.NotContains(t, doc.Content, "Navigation")
}
```

### 6.3 New Test: Exclusion Selector
```go
func TestPipeline_ExcludeSelector(t *testing.T) {
    html := `<!DOCTYPE html>
    <html><body>
        <article>
            <h1>Title</h1>
            <div class="warning">Warning box to exclude</div>
            <p>Main content</p>
        </article>
    </body></html>`

    pipeline := converter.NewPipeline(converter.PipelineOptions{
        ContentSelector: "article",
        ExcludeSelector: ".warning",
    })

    doc, err := pipeline.Convert(context.Background(), html, "https://example.com")
    require.NoError(t, err)

    assert.Contains(t, doc.Content, "Title")
    assert.Contains(t, doc.Content, "Main content")
    assert.NotContains(t, doc.Content, "Warning box")
}
```

---

## 7. Resolved Questions

| Question | Decision | Rationale |
|----------|----------|-----------|
| Multiple selectors support? | **Yes, native** | goquery already supports comma-separated selectors |
| Exclusion selectors? | **Yes, new flag** | Adds `--exclude-selector` flag for explicit exclusions |
| Multiple matches behavior? | **Combine all** | Change `.First()` to combine all matching elements |
| Invalid selector handling? | **Silent fallback + log** | Keep fallback to Readability, add debug logging |

---

## 8. Success Criteria

1. **Unit Tests:** All new tests in `pipeline_test.go` pass
2. **Existing Tests:** No regression in existing `TestPipeline_ContentSelector` tests
3. **Functional:** `repodocs URL --content-selector "article"` extracts all `<article>` elements
4. **Functional:** `repodocs URL --content-selector "#main" --exclude-selector ".ads"` works correctly
5. **Logging:** Debug mode shows selector match count and fallback warnings

---

## 9. Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Breaking existing selector behavior | Low | High | Keep fallback logic, add tests first |
| Performance impact from combining elements | Low | Low | goquery is efficient; no extra parsing |
| Complex selectors failing | Low | Medium | goquery handles most CSS3 selectors |

---

## 10. Implementation Order

1. **Write tests first** (TDD approach)
2. Add logging (Phase 1) - safest change
3. Implement multi-element support (Phase 2)
4. Add exclusion selector (Phase 3) - if needed

**Estimated Total Effort:** 6 hours

---

## 11. Dependencies

- **goquery** v1.8+ (already installed) - CSS selector support
- **zerolog** (already installed) - structured logging
- No new dependencies required
