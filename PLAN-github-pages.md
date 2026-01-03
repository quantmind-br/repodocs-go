# Implementation Plan: GitHub Pages Strategy (v3)

## Executive Summary

Create a new `GitHubPagesStrategy` to handle documentation extraction from GitHub Pages sites, including `*.github.io` and (optionally) custom domains via explicit override. These sites are often SPAs (Docusaurus, Jekyll, Hugo, VitePress, MkDocs), but many pages are still statically retrievable.

**Key Innovation**: Multi-tier HTTP discovery with sequential probes (parallel optional), plus HTTP-first extraction with browser fallback when content appears JS-rendered, reducing resource usage while preserving SPA correctness.

> **Note**: This is v3 of the plan, incorporating corrections from code review against the existing codebase.

---

## Problem Statement

### Current Behavior
When running `repodocs https://block.github.io/goose/`:
- Root page extracts correctly (470 words)
- All subpages return "301 Moved Permanently" with nginx placeholder content
- Generated files contain only "nginx" instead of actual documentation

### Root Cause
Many GitHub Pages sites are SPAs where all routes are handled by JavaScript. Direct HTTP fetches to subpages can return the SPA shell (often a 301 redirect to index.html) instead of rendered content, but some pages are still statically available and should be fetched without a browser when possible.

```
HTTP GET /docs/quickstart → 301 → nginx placeholder
Browser /docs/quickstart  → JS renders → Full documentation
```

---

## Research Findings

### Discovery Mechanism Availability (Tested)

| Site | sitemap.xml | Search Index | Needs Browser |
|------|-------------|--------------|---------------|
| `block.github.io/goose` | ✅ 229 URLs | ❌ | Discovery: No |
| `squidfunk.github.io/mkdocs-material` | ✅ | ✅ `/search/search_index.json` | Discovery: No |
| `facebook.github.io/docusaurus` | ❌ | ❌ | Discovery: Yes |
| `vitepress.dev` | ✅ | ✅ `/hashmap.json` | Discovery: No |
| `docs.github.com` | ❌ | ❌ (Next.js) | Discovery: Yes |
| `react.dev` | ❌ | ❌ (Algolia cloud) | Discovery: Yes |
| `vuejs.org` | ✅ | ❌ (Algolia cloud) | Discovery: No |
| `kubernetes.io` | ✅ | ✅ `/index.json` | Discovery: No |

**Conclusion**: ~70% of GitHub Pages sites have HTTP-discoverable URL lists. HTTP-only extraction can still succeed for a subset of pages even on SPA sites, so discovery and extraction should each have HTTP-first fallback paths.

### Search Index Formats by SSG

| SSG | Index Path | URL Field | Format |
|-----|------------|-----------|--------|
| **MkDocs** | `/search/search_index.json` | `location` | `{docs: [{location, title, text}]}` |
| **Docusaurus** | `/search-index.json` | `url` | `[{url, title, content}]` |
| **Hugo** | `/index.json` or `/search.json` | `permalink` or `url` | `[{permalink, title, content}]` |
| **VitePress** | `/hashmap.json` | keys are paths | `{path: hash}` |
| **Pagefind** | `/pagefind/pagefind-entry.json` | varies | Binary shards |
| **Jekyll** | `/search.json` | `url` | `[{url, title, content}]` |

---

## Solution Architecture

### Multi-Tier Discovery + Extraction Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GitHubPagesStrategy - Flow                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TIER 1: Fast HTTP Probes (Parallel, No Browser)                            │
│  ├── 1. /llms.txt (LLM-optimized, highest quality)                          │
│  ├── 2. /sitemap.xml, /sitemap-0.xml, /sitemap_index.xml (+ .xml.gz)        │
│  ├── 3. /search/search_index.json (MkDocs)                                  │
│  ├── 4. /search-index.json (Docusaurus)                                     │
│  ├── 5. /index.json, /search.json (Hugo, Fuse.js)                           │
│  ├── 6. /pagefind/pagefind-entry.json (Astro/Starlight)                     │
│  └── 7. /hashmap.json (VitePress)                                           │
│                                                                             │
│  ↓ If all probes fail                                                       │
│                                                                             │
│  TIER 2: Browser-Based Discovery (Fallback)                                 │
│  ├── 1. Render homepage with Rod                                            │
│  ├── 2. Extract links from nav/sidebar DOM elements                         │
│  └── 3. BFS crawl internal links (max depth 3, max pages)                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EXTRACTION: HTTP-first, Browser Fallback                                   │
│  ├── HTTP fetch and validate content (length + body text + SPA markers)     │
│  ├── If suspect SPA/empty -> render via browser                             │
│  ├── Convert HTML to Markdown via existing pipeline                         │
│  └── Deduplicate + normalize URLs across all sources                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Discovery | HTTP-first (parallel probes), browser fallback | 70%+ sites have HTTP-discoverable indexes |
| Extraction | HTTP-first with browser fallback | Many pages are static even on SPA sites; browser only when needed |
| Index parsing | Dedicated parsers per format | Each SSG has unique JSON structure |
| Link extraction | goquery (not regex) | Robust HTML parsing, handles edge cases |
| Concurrency | Max 5 browser tabs | Stability and resource management |
| Caching | Rendered HTML cached | Avoid re-rendering same pages |

---

## Implementation Details

### Phase 1: Strategy Detection

#### File: `internal/app/detector.go`

```go
// Add new strategy type constant
const (
    // ... existing constants ...
    StrategyGitHubPages StrategyType = "github_pages"
)

// DetectStrategy - Add GitHub Pages detection AFTER Wiki, BEFORE Git
func DetectStrategy(rawURL string) StrategyType {
    // ... existing code through Wiki check ...

    // Check for Wiki (before generic Git)
    if strategies.IsWikiURL(rawURL) {
        return StrategyWiki
    }

    // NEW: Check for GitHub Pages
    if isGitHubPagesURL(rawURL) {
        return StrategyGitHubPages
    }

    // Check for Git repository (existing code)
    // Note: detector.go already excludes github.io from Git detection
    // ...
}

// isGitHubPagesURL checks if URL is a GitHub Pages site
func isGitHubPagesURL(rawURL string) bool {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return false
    }
    host := strings.ToLower(parsed.Host)
    if strings.HasSuffix(host, ".github.io") {
        return true
    }
    // Optional: allow override for custom domains (flag or option)
    return false
}
```

**Detection Priority Order:**
1. SSH Git URLs (`git@...`)
2. llms.txt
3. pkg.go.dev
4. docs.rs
5. sitemap.xml
6. Wiki URLs
7. **GitHub Pages (NEW)** ← Insert here
8. Git repositories
9. Crawler (default)

**Also update `CreateStrategy` and `GetAllStrategies`:**

```go
func CreateStrategy(strategyType StrategyType, deps *strategies.Dependencies) strategies.Strategy {
    switch strategyType {
    // ... existing cases ...
    case StrategyGitHubPages:
        return strategies.NewGitHubPagesStrategy(deps)
    // ... rest of cases ...
    }
}

func GetAllStrategies(deps *strategies.Dependencies) []strategies.Strategy {
    return []strategies.Strategy{
        strategies.NewLLMSStrategy(deps),
        strategies.NewPkgGoStrategy(deps),
        strategies.NewDocsRSStrategy(deps),
        strategies.NewSitemapStrategy(deps),
        strategies.NewWikiStrategy(deps),
        strategies.NewGitHubPagesStrategy(deps), // NEW
        strategies.NewGitStrategy(deps),
        strategies.NewCrawlerStrategy(deps),
    }
}
```

---

### Phase 2: Discovery Probes

#### File: `internal/strategies/github_pages_discovery.go` (NEW)

> **Note**: This file already exists with the parsers implemented. The code below shows the complete implementation with corrections.

**Key Implementation Notes**:
- Uses lowercase `parseLLMSLinks` from `llms.go` (same package, accessible)
- Uses lowercase `sitemapXML` and `sitemapIndexXML` from `sitemap.go` (same package, accessible)
- Parser functions are exported (capitalized) for testability
- Sequential probes (not parallel) for simplicity - parallel is optional enhancement

```go
package strategies

import (
    "encoding/json"
    "encoding/xml"
    "fmt"
    "net/url"
    "strings"
)

// DiscoveryProbe defines a URL discovery mechanism
type DiscoveryProbe struct {
    Path   string
    Parser func(content []byte, baseURL string) ([]string, error)
    Name   string
}

// GetDiscoveryProbes returns all discovery probes in priority order
func GetDiscoveryProbes() []DiscoveryProbe {
    return []DiscoveryProbe{
        // Tier 1: LLM-optimized (highest quality)
        {"/llms.txt", ParseLLMsTxt, "llms.txt"},
        
        // Tier 2: Sitemaps (most common)
        {"/sitemap.xml", ParseSitemapXML, "sitemap.xml"},
        {"/sitemap-0.xml", ParseSitemapXML, "sitemap-0.xml"},
        {"/sitemap_index.xml", ParseSitemapIndexXML, "sitemap_index.xml"},
        
        // Tier 3: MkDocs (very reliable)
        {"/search/search_index.json", ParseMkDocsIndex, "mkdocs-search"},
        
        // Tier 4: Docusaurus
        {"/search-index.json", ParseDocusaurusIndex, "docusaurus-search"},
        
        // Tier 5: Hugo / Generic
        {"/index.json", ParseHugoIndex, "hugo-index"},
        {"/search.json", ParseGenericSearchIndex, "search.json"},
        
        // Tier 6: Modern SSGs
        {"/hashmap.json", ParseVitePressHashmap, "vitepress"},
    }
}

// ParseLLMsTxt parses llms.txt format (markdown links)
// Uses parseLLMSLinks from llms.go (same package)
func ParseLLMsTxt(content []byte, baseURL string) ([]string, error) {
    links := parseLLMSLinks(string(content))
    if len(links) == 0 {
        return nil, fmt.Errorf("no links found in llms.txt")
    }
    
    urls := make([]string, 0, len(links))
    for _, link := range links {
        urls = append(urls, resolveDiscoveryURL(link.URL, baseURL))
    }
    return urls, nil
}

// ParseSitemapXML parses standard sitemap.xml format
// Uses sitemapXML type from sitemap.go (same package)
func ParseSitemapXML(content []byte, baseURL string) ([]string, error) {
    var sitemap sitemapXML
    if err := xml.Unmarshal(content, &sitemap); err != nil {
        return nil, err
    }
    
    if len(sitemap.URLs) == 0 {
        return nil, fmt.Errorf("empty sitemap")
    }
    
    urls := make([]string, 0, len(sitemap.URLs))
    for _, u := range sitemap.URLs {
        if u.Loc != "" {
            urls = append(urls, u.Loc)
        }
    }
    return urls, nil
}

// ParseSitemapIndexXML parses sitemap index and returns nested sitemap URLs
// Uses sitemapIndexXML type from sitemap.go (same package)
func ParseSitemapIndexXML(content []byte, baseURL string) ([]string, error) {
    var index sitemapIndexXML
    if err := xml.Unmarshal(content, &index); err != nil {
        return nil, err
    }
    
    if len(index.Sitemaps) == 0 {
        return nil, fmt.Errorf("empty sitemap index")
    }
    
    // Return sitemap URLs (caller should fetch and parse each)
    urls := make([]string, 0, len(index.Sitemaps))
    for _, sm := range index.Sitemaps {
        urls = append(urls, sm.Loc)
    }
    return urls, nil
}

// MkDocsSearchIndex represents MkDocs search_index.json structure
type MkDocsSearchIndex struct {
    Docs []struct {
        Location string `json:"location"`
        Title    string `json:"title"`
        Text     string `json:"text"`
    } `json:"docs"`
}

// ParseMkDocsIndex parses MkDocs /search/search_index.json
func ParseMkDocsIndex(content []byte, baseURL string) ([]string, error) {
    var index MkDocsSearchIndex
    if err := json.Unmarshal(content, &index); err != nil {
        return nil, err
    }
    
    if len(index.Docs) == 0 {
        return nil, fmt.Errorf("empty MkDocs index")
    }
    
    seen := make(map[string]bool)
    var urls []string
    
    for _, doc := range index.Docs {
        loc := strings.Split(doc.Location, "#")[0]
        if loc == "" || loc == "." {
            loc = ""
        }
        
        fullURL := strings.TrimSuffix(baseURL, "/") + "/" + strings.TrimPrefix(loc, "/")
        
        if !seen[fullURL] {
            seen[fullURL] = true
            urls = append(urls, fullURL)
        }
    }
    
    return urls, nil
}

// DocusaurusSearchEntry represents a Docusaurus search index entry
type DocusaurusSearchEntry struct {
    URL     string `json:"url"`
    Title   string `json:"title"`
    Content string `json:"content"`
}

// ParseDocusaurusIndex parses Docusaurus /search-index.json
func ParseDocusaurusIndex(content []byte, baseURL string) ([]string, error) {
    var entries []DocusaurusSearchEntry
    if err := json.Unmarshal(content, &entries); err != nil {
        return nil, err
    }
    
    if len(entries) == 0 {
        return nil, fmt.Errorf("empty Docusaurus index")
    }
    
    urls := make([]string, 0, len(entries))
    for _, entry := range entries {
        if entry.URL != "" {
            urls = append(urls, resolveDiscoveryURL(entry.URL, baseURL))
        }
    }
    return urls, nil
}

// HugoSearchEntry represents a Hugo search index entry
type HugoSearchEntry struct {
    Permalink string `json:"permalink"`
    URL       string `json:"url"`
    Title     string `json:"title"`
}

// ParseHugoIndex parses Hugo /index.json
func ParseHugoIndex(content []byte, baseURL string) ([]string, error) {
    var entries []HugoSearchEntry
    if err := json.Unmarshal(content, &entries); err != nil {
        return nil, err
    }
    
    if len(entries) == 0 {
        return nil, fmt.Errorf("empty Hugo index")
    }
    
    urls := make([]string, 0, len(entries))
    for _, entry := range entries {
        urlStr := entry.Permalink
        if urlStr == "" {
            urlStr = entry.URL
        }
        if urlStr != "" {
            urls = append(urls, resolveDiscoveryURL(urlStr, baseURL))
        }
    }
    return urls, nil
}

// ParseGenericSearchIndex parses generic search.json format
func ParseGenericSearchIndex(content []byte, baseURL string) ([]string, error) {
    var entries []map[string]interface{}
    if err := json.Unmarshal(content, &entries); err != nil {
        return nil, err
    }
    
    if len(entries) == 0 {
        return nil, fmt.Errorf("empty search index")
    }
    
    urls := make([]string, 0, len(entries))
    for _, entry := range entries {
        for _, field := range []string{"url", "permalink", "href", "location", "path"} {
            if val, ok := entry[field].(string); ok && val != "" {
                urls = append(urls, resolveDiscoveryURL(val, baseURL))
                break
            }
        }
    }
    
    if len(urls) == 0 {
        return nil, fmt.Errorf("no URLs found in search index")
    }
    return urls, nil
}

// ParseVitePressHashmap parses VitePress hashmap.json
func ParseVitePressHashmap(content []byte, baseURL string) ([]string, error) {
    var hashmap map[string]string
    if err := json.Unmarshal(content, &hashmap); err != nil {
        return nil, err
    }
    
    if len(hashmap) == 0 {
        return nil, fmt.Errorf("empty VitePress hashmap")
    }
    
    urls := make([]string, 0, len(hashmap))
    for path := range hashmap {
        urlPath := strings.ReplaceAll(path, "_", "/")
        urlPath = strings.TrimSuffix(urlPath, ".md")
        
        fullURL := strings.TrimSuffix(baseURL, "/") + "/" + urlPath
        urls = append(urls, fullURL)
    }
    return urls, nil
}

// resolveDiscoveryURL resolves a potentially relative URL against a base URL
func resolveDiscoveryURL(href, baseURL string) string {
    if strings.HasPrefix(href, "http://") || strings.HasPrefix(href, "https://") {
        return href
    }

    parsed, err := url.Parse(baseURL)
    if err != nil {
        return baseURL + "/" + strings.TrimPrefix(href, "/")
    }

    ref, err := url.Parse(href)
    if err != nil {
        return baseURL + "/" + strings.TrimPrefix(href, "/")
    }

    return parsed.ResolveReference(ref).String()
}
```

---

### Phase 3: Main Strategy Implementation

#### File: `internal/strategies/github_pages.go` (NEW)

> **Note**: Corrections applied from code review.

**Key Implementation Notes**:
- Uses `doc.RenderedWithJS` (not `RenderedJS`) - matches `domain.Document` field
- `Writer.Exists()` method exists in `output.Writer`
- Includes missing `fetchOrRenderPage` method definition
- Exports `ShouldSkipGitHubPagesURL` for testability

```go
package strategies

import (
    "context"
    "fmt"
    "net/url"
    "regexp"
    "strings"
    "sync"
    "time"

    "github.com/PuerkitoBio/goquery"
    "github.com/quantmind-br/repodocs-go/internal/converter"
    "github.com/quantmind-br/repodocs-go/internal/domain"
    "github.com/quantmind-br/repodocs-go/internal/output"
    "github.com/quantmind-br/repodocs-go/internal/utils"
    "github.com/schollz/progressbar/v3"
)
```

**Updates**:
- Extraction is HTTP-first; browser render only when content is missing or looks like SPA shell.
- Add per-page normalization/deduplication (strip fragments, unify trailing slash, collapse index.html).
- Add optional `--github-pages` override for custom domains.
- Respect `robots.txt` disallow and optional crawl-delay (best effort).
- Discovery URLs are host-filtered and de-duplicated before extraction.

// GitHubPagesStrategy extracts documentation from GitHub Pages sites (*.github.io)
type GitHubPagesStrategy struct {
    deps           *Dependencies
    fetcher        domain.Fetcher
    renderer       domain.Renderer
    converter      *converter.Pipeline
    markdownReader *converter.MarkdownReader
    writer         *output.Writer
    logger         *utils.Logger
}

// NewGitHubPagesStrategy creates a new GitHub Pages strategy
func NewGitHubPagesStrategy(deps *Dependencies) *GitHubPagesStrategy {
    if deps == nil {
        return &GitHubPagesStrategy{
            markdownReader: converter.NewMarkdownReader(),
        }
    }
    return &GitHubPagesStrategy{
        deps:           deps,
        fetcher:        deps.Fetcher,
        renderer:       deps.Renderer,
        converter:      deps.Converter,
        markdownReader: converter.NewMarkdownReader(),
        writer:         deps.Writer,
        logger:         deps.Logger,
    }
}

// Name returns the strategy name
func (s *GitHubPagesStrategy) Name() string {
    return "github_pages"
}

// CanHandle returns true if URL is a GitHub Pages site
func (s *GitHubPagesStrategy) CanHandle(rawURL string) bool {
    return IsGitHubPagesURL(rawURL)
}

// IsGitHubPagesURL checks if a URL is a GitHub Pages site
func IsGitHubPagesURL(rawURL string) bool {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return false
    }
    host := strings.ToLower(parsed.Host)
    return strings.HasSuffix(host, ".github.io")
}

// Execute runs the GitHub Pages extraction strategy
func (s *GitHubPagesStrategy) Execute(ctx context.Context, inputURL string, opts Options) error {
    s.logger.Info().
        Str("url", inputURL).
        Msg("Starting GitHub Pages extraction")

    // Validate renderer is available
    if s.renderer == nil {
        return fmt.Errorf("browser renderer required for GitHub Pages sites; use --render-js or ensure Chrome is available")
    }

    // Normalize base URL
    baseURL, err := s.normalizeBaseURL(inputURL)
    if err != nil {
        return fmt.Errorf("invalid URL: %w", err)
    }

    // Phase 1: Discover URLs (HTTP-first, browser fallback)
    urls, discoveryMethod, err := s.discoverURLs(ctx, baseURL, opts)
    if err != nil {
        return fmt.Errorf("URL discovery failed: %w", err)
    }

    if len(urls) == 0 {
        s.logger.Warn().Msg("No URLs discovered")
        return nil
    }

    s.logger.Info().
        Int("count", len(urls)).
        Str("method", discoveryMethod).
        Msg("URLs discovered")

    // Apply filters and limits
    urls = s.filterURLs(urls, baseURL, opts)
    if opts.Limit > 0 && len(urls) > opts.Limit {
        urls = urls[:opts.Limit]
    }

    s.logger.Info().
        Int("count", len(urls)).
        Msg("Processing URLs via browser rendering")

    // Phase 2: Extract content (HTTP-first, browser fallback)
    return s.processURLs(ctx, urls, opts)
}

// discoverURLs finds all URLs using multi-tier discovery
func (s *GitHubPagesStrategy) discoverURLs(ctx context.Context, baseURL string, opts Options) ([]string, string, error) {
    // Tier 1: Try HTTP probes in parallel
    urls, method, err := s.discoverViaHTTPProbes(ctx, baseURL)
    if err == nil && len(urls) > 0 {
        return urls, method, nil
    }

    s.logger.Debug().Err(err).Msg("HTTP discovery failed, falling back to browser crawl")

    // Tier 2: Browser-based discovery
    urls, err = s.discoverViaBrowser(ctx, baseURL, opts)
    if err != nil {
        return nil, "", fmt.Errorf("browser discovery failed: %w", err)
    }

    return urls, "browser-crawl", nil
}

// discoverViaHTTPProbes tries all HTTP-based discovery methods
func (s *GitHubPagesStrategy) discoverViaHTTPProbes(ctx context.Context, baseURL string) ([]string, string, error) {
    probes := GetDiscoveryProbes()

    // Run probes in parallel with short timeouts; take first success
    // NOTE: implement context with per-probe timeout, then select on results
    // Filter to same host and normalize/deduplicate results before returning.

    // Try probes in priority order
    for _, probe := range probes {
        select {
        case <-ctx.Done():
            return nil, "", ctx.Err()
        default:
        }

        probeURL := strings.TrimSuffix(baseURL, "/") + probe.Path
        
        resp, err := s.fetcher.Get(ctx, probeURL)
        if err != nil {
            s.logger.Debug().Str("probe", probe.Name).Str("url", probeURL).Err(err).Msg("Probe failed")
            continue
        }
        
        if resp.StatusCode != 200 {
            s.logger.Debug().Str("probe", probe.Name).Int("status", resp.StatusCode).Msg("Probe returned non-200")
            continue
        }

        urls, err := probe.Parser(resp.Body, baseURL)
        if err != nil {
            s.logger.Debug().Str("probe", probe.Name).Err(err).Msg("Failed to parse probe response")
            continue
        }

        if len(urls) > 0 {
            s.logger.Info().
                Str("probe", probe.Name).
                Int("urls", len(urls)).
                Msg("Discovery probe succeeded")
            return urls, probe.Name, nil
        }
    }

    return nil, "", fmt.Errorf("all HTTP probes failed")
}

// discoverViaBrowser uses browser rendering to crawl and discover URLs
func (s *GitHubPagesStrategy) discoverViaBrowser(ctx context.Context, baseURL string, opts Options) ([]string, error) {
    visited := make(map[string]bool)
    toVisit := []string{baseURL}
    var discovered []string

    maxDepth := opts.MaxDepth
    if maxDepth <= 0 {
        maxDepth = 3
    }

    for depth := 0; depth < maxDepth && len(toVisit) > 0; depth++ {
        var nextLevel []string

        for _, pageURL := range toVisit {
            if visited[pageURL] {
                continue
            }
            visited[pageURL] = true
            discovered = append(discovered, pageURL)

            // Check limit during discovery
            if opts.Limit > 0 && len(discovered) >= opts.Limit*2 {
                // Discover 2x limit to allow for filtering
                return discovered, nil
            }

            // Render page and extract links
            links, err := s.extractLinksFromRenderedPage(ctx, pageURL, baseURL)
            if err != nil {
                s.logger.Debug().Err(err).Str("url", pageURL).Msg("Failed to extract links")
                continue
            }

            for _, link := range links {
                if !visited[link] {
                    nextLevel = append(nextLevel, link)
                }
            }
        }

        toVisit = nextLevel
        s.logger.Debug().Int("depth", depth+1).Int("queued", len(toVisit)).Msg("Browser crawl depth completed")
    }

    return discovered, nil
}

// extractLinksFromRenderedPage renders a page and extracts internal links
func (s *GitHubPagesStrategy) extractLinksFromRenderedPage(ctx context.Context, pageURL, baseURL string) ([]string, error) {
    html, err := s.renderPage(ctx, pageURL)
    if err != nil {
        return nil, err
    }

    return s.extractLinksWithGoquery(html, baseURL)
}

// extractLinksWithGoquery uses goquery for robust HTML parsing
func (s *GitHubPagesStrategy) extractLinksWithGoquery(html, baseURL string) ([]string, error) {
    doc, err := goquery.NewDocumentFromReader(strings.NewReader(html))
    if err != nil {
        return nil, err
    }

    parsedBase, _ := url.Parse(baseURL)
    baseHost := parsedBase.Host

    seen := make(map[string]bool)
    var links []string

    // Extract from navigation elements first (most relevant)
    selectors := []string{
        "nav a[href]",
        "[role='navigation'] a[href]",
        ".sidebar a[href]",
        ".menu a[href]",
        ".nav a[href]",
        ".toc a[href]",
        "a[href]", // Fallback to all links
    }

    for _, selector := range selectors {
        doc.Find(selector).Each(func(_ int, sel *goquery.Selection) {
            href, exists := sel.Attr("href")
            if !exists || href == "" {
                return
            }

            // Skip non-navigable links
            if strings.HasPrefix(href, "#") ||
                strings.HasPrefix(href, "javascript:") ||
                strings.HasPrefix(href, "mailto:") ||
                strings.HasPrefix(href, "tel:") {
                return
            }

            // Resolve relative URLs
            resolved, err := url.Parse(href)
            if err != nil {
                return
            }

            if !resolved.IsAbs() {
                resolved = parsedBase.ResolveReference(resolved)
            }

            // Filter to same host
            if resolved.Host != baseHost {
                return
            }

            // Normalize: remove fragment, trailing slash
            resolved.Fragment = ""
            normalized := resolved.String()
            normalized = strings.TrimSuffix(normalized, "/")

            if !seen[normalized] && !ShouldSkipGitHubPagesURL(normalized) {
                seen[normalized] = true
                links = append(links, normalized)
            }
        })

        // If we found links in nav elements, prefer those
        if len(links) > 10 && selector != "a[href]" {
            break
        }
    }

    return links, nil
}

// renderPage renders a page using the browser
func (s *GitHubPagesStrategy) renderPage(ctx context.Context, pageURL string) (string, error) {
    return s.renderer.Render(ctx, pageURL, domain.RenderOptions{
        Timeout:     90 * time.Second,
        WaitStable:  3 * time.Second,
        ScrollToEnd: true,
    })
}

// filterURLs applies filter and exclude patterns
func (s *GitHubPagesStrategy) filterURLs(urls []string, baseURL string, opts Options) []string {
    var excludeRegexps []*regexp.Regexp
    for _, pattern := range opts.Exclude {
        if re, err := regexp.Compile(pattern); err == nil {
            excludeRegexps = append(excludeRegexps, re)
        }
    }

    var filtered []string
    for _, u := range urls {
        // Apply base URL filter
        if opts.FilterURL != "" && !strings.HasPrefix(u, opts.FilterURL) {
            continue
        }

        // Apply exclude patterns
        excluded := false
        for _, re := range excludeRegexps {
            if re.MatchString(u) {
                excluded = true
                break
            }
        }
        if excluded {
            continue
        }

        // Skip non-content URLs
        if ShouldSkipGitHubPagesURL(u) {
            continue
        }

        filtered = append(filtered, u)
    }

    return filtered
}

// shouldSkipGitHubPagesURL returns true for URLs that typically don't contain documentation
// Exported as ShouldSkipGitHubPagesURL for testing
func ShouldSkipGitHubPagesURL(u string) bool {
    lower := strings.ToLower(u)
    skipPatterns := []string{
        "/assets/", "/static/", "/_next/", "/_nuxt/",
        "/img/", "/images/", "/media/",
        "/css/", "/js/", "/fonts/",
        ".png", ".jpg", ".jpeg", ".gif", ".svg", ".ico", ".webp",
        ".css", ".js", ".json", ".xml",
        ".woff", ".woff2", ".ttf", ".eot",
        ".pdf", ".zip", ".tar", ".gz",
        "/feed.xml", "/rss.xml", "/atom.xml",
        "/tags/", "/authors/", "/category/",
        "/404", "/sitemap",
    }

    for _, pattern := range skipPatterns {
        if strings.Contains(lower, pattern) {
            return true
        }
    }
    return false
}

// processURLs processes all URLs using HTTP-first extraction with browser fallback
func (s *GitHubPagesStrategy) processURLs(ctx context.Context, urls []string, opts Options) error {
    bar := progressbar.NewOptions(len(urls),
        progressbar.OptionSetDescription("Rendering"),
        progressbar.OptionShowCount(),
        progressbar.OptionShowIts(),
    )

    // Limit browser concurrency for stability
    concurrency := opts.Concurrency
    if concurrency > 5 {
        concurrency = 5
    }
    if concurrency <= 0 {
        concurrency = 2
    }

    var mu sync.Mutex
    var processedCount, successCount int

    errors := utils.ParallelForEach(ctx, urls, concurrency, func(ctx context.Context, pageURL string) error {
        defer func() {
            mu.Lock()
            bar.Add(1)
            processedCount++
            mu.Unlock()
        }()

        // Check if already exists
        if !opts.Force && s.writer.Exists(pageURL) {
            return nil
        }

        // HTTP-first fetch
        html, usedBrowser, err := s.fetchOrRenderPage(ctx, pageURL)
        if err != nil {
            s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to fetch/render page")
            return nil
        }

        // Validate content
        if s.isEmptyOrErrorContent(html) {
            s.logger.Debug().Str("url", pageURL).Msg("Empty or error content, skipping")
            return nil
        }

        // Convert HTML to document
        doc, err := s.converter.Convert(ctx, html, pageURL)
        if err != nil {
            s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to convert page")
            return nil
        }

        // Validate converted content
        if len(strings.TrimSpace(doc.Content)) < 50 {
            s.logger.Debug().Str("url", pageURL).Msg("Converted content too short, skipping")
            return nil
        }

        // Set metadata
        doc.SourceStrategy = s.Name()
        doc.FetchedAt = time.Now()
        doc.RenderedWithJS = usedBrowser

        // Write document
        if !opts.DryRun {
            if err := s.deps.WriteDocument(ctx, doc); err != nil {
                s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to write document")
                return nil
            }
            mu.Lock()
            successCount++
            mu.Unlock()
        }

        return nil
    })

    if err := utils.FirstError(errors); err != nil {
        return err
    }

    s.logger.Info().
        Int("processed", processedCount).
        Int("success", successCount).
        Int("total", len(urls)).
        Msg("GitHub Pages extraction completed")

    return nil
}

// isEmptyOrErrorContent checks if rendered content is empty or an error page
func (s *GitHubPagesStrategy) isEmptyOrErrorContent(html string) bool {
    if len(html) < 300 {
        return true
    }

    lower := strings.ToLower(html)
    errorIndicators := []string{
        "301 moved permanently",
        "302 found",
        "404 not found",
        "page not found",
        "<!doctype html><html><head></head><body></body></html>",
        "<body></body>",
        "<body> </body>",
        "access denied",
        "403 forbidden",
    }

    for _, indicator := range errorIndicators {
        if strings.Contains(lower, indicator) {
            return true
        }
    }

    // Check for minimal content (just boilerplate)
    doc, err := goquery.NewDocumentFromReader(strings.NewReader(html))
    if err != nil {
        return true
    }

    // Check if body has meaningful text content
    bodyText := strings.TrimSpace(doc.Find("body").Text())
    if len(bodyText) < 60 {
        return true
    }

    return false
}

// fetchOrRenderPage tries HTTP fetch first, falls back to browser rendering if content looks like SPA shell
func (s *GitHubPagesStrategy) fetchOrRenderPage(ctx context.Context, pageURL string) (string, bool, error) {
    // Try HTTP fetch first
    resp, err := s.fetcher.Get(ctx, pageURL)
    if err == nil && resp.StatusCode == 200 {
        html := string(resp.Body)
        
        // Check if content looks like actual content (not SPA shell)
        if !s.looksLikeSPAShell(html) {
            return html, false, nil
        }
        
        s.logger.Debug().Str("url", pageURL).Msg("Content looks like SPA shell, falling back to browser")
    }
    
    // Fall back to browser rendering
    html, err := s.renderPage(ctx, pageURL)
    if err != nil {
        return "", false, err
    }
    
    return html, true, nil
}

// looksLikeSPAShell checks if HTML content appears to be an SPA shell without rendered content
func (s *GitHubPagesStrategy) looksLikeSPAShell(html string) bool {
    // Very short content is likely a shell
    if len(html) < 500 {
        return true
    }
    
    lower := strings.ToLower(html)
    
    // Check for common SPA indicators
    spaIndicators := []string{
        `<div id="app"></div>`,
        `<div id="root"></div>`,
        `<div id="__next"></div>`,
        `<div id="__nuxt"></div>`,
        "loading...",
        "please enable javascript",
    }
    
    for _, indicator := range spaIndicators {
        if strings.Contains(lower, indicator) {
            // Check if there's substantial content after the indicator
            doc, err := goquery.NewDocumentFromReader(strings.NewReader(html))
            if err != nil {
                return true
            }
            
            bodyText := strings.TrimSpace(doc.Find("body").Text())
            // If body has less than 100 chars of text, it's likely a shell
            return len(bodyText) < 100
        }
    }
    
    return false
}

// normalizeBaseURL normalizes the input URL to a base URL
func (s *GitHubPagesStrategy) normalizeBaseURL(inputURL string) (string, error) {
    parsed, err := url.Parse(inputURL)
    if err != nil {
        return "", err
    }

    if parsed.Scheme == "" {
        parsed.Scheme = "https"
    }

    // Keep the path for project subpaths (e.g., /goose/)
    base := fmt.Sprintf("%s://%s", parsed.Scheme, parsed.Host)

    if parsed.Path != "" && parsed.Path != "/" {
        // Clean path: remove trailing slash, keep structure
        path := strings.TrimSuffix(parsed.Path, "/")
        base += path
    }

    return base, nil
}
```

---

### Phase 4: Tests

#### File: `tests/unit/strategies/github_pages_test.go` (NEW)

**Updates**:
- Export or wrap parser functions for testing (avoid referencing unexported symbols).
- Add tests for URL normalization and deduplication.
- Add tests for HTTP-first extraction fallback logic.

```go
package strategies_test

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/quantmind-br/repodocs-go/internal/strategies"
)

func TestIsGitHubPagesURL(t *testing.T) {
    tests := []struct {
        name     string
        url      string
        expected bool
    }{
        // Positive cases
        {"standard github pages", "https://block.github.io/goose/", true},
        {"user github pages", "https://username.github.io/", true},
        {"with deep path", "https://org.github.io/project/docs/guide/", true},
        {"http scheme", "http://foo.github.io/bar", true},
        
        // Negative cases
        {"github.com", "https://github.com/owner/repo", false},
        {"other domain", "https://example.com/docs", false},
        {"docs.github.com", "https://docs.github.com/en", false},
        {"partial match", "https://notgithub.io/something", false},
        {"subdomain of github.io lookalike", "https://api.github.io.example.com/", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := strategies.IsGitHubPagesURL(tt.url)
            assert.Equal(t, tt.expected, result)
        })
    }
}

func TestParseMkDocsIndex(t *testing.T) {
    content := []byte(`{
        "docs": [
            {"location": "index.html", "title": "Home", "text": "Welcome"},
            {"location": "guide/getting-started.html", "title": "Getting Started", "text": "..."},
            {"location": "api/reference.html#section", "title": "API", "text": "..."}
        ]
    }`)
    
    urls, err := strategies.ParseMkDocsIndex(content, "https://example.github.io/docs")
    require.NoError(t, err)
    
    assert.Len(t, urls, 3)
    assert.Contains(t, urls, "https://example.github.io/docs/index.html")
    assert.Contains(t, urls, "https://example.github.io/docs/guide/getting-started.html")
    assert.Contains(t, urls, "https://example.github.io/docs/api/reference.html")
}

func TestParseVitePressHashmap(t *testing.T) {
    content := []byte(`{
        "guide_getting-started.md": "abc123",
        "guide_installation.md": "def456",
        "index.md": "xyz789"
    }`)
    
    urls, err := strategies.ParseVitePressHashmap(content, "https://example.github.io")
    require.NoError(t, err)
    
    assert.Len(t, urls, 3)
    assert.Contains(t, urls, "https://example.github.io/guide/getting-started")
    assert.Contains(t, urls, "https://example.github.io/guide/installation")
    assert.Contains(t, urls, "https://example.github.io/index")
}

func TestShouldSkipGitHubPagesURL(t *testing.T) {
    tests := []struct {
        url      string
        expected bool
    }{
        // Should skip
        {"https://example.github.io/assets/logo.png", true},
        {"https://example.github.io/static/main.js", true},
        {"https://example.github.io/_next/data/build.json", true},
        {"https://example.github.io/img/banner.jpg", true},
        {"https://example.github.io/feed.xml", true},
        {"https://example.github.io/blog/tags/go", true},
        
        // Should NOT skip
        {"https://example.github.io/docs/getting-started", false},
        {"https://example.github.io/guides/tutorial", false},
        {"https://example.github.io/api/reference", false},
        {"https://example.github.io/", false},
    }

    for _, tt := range tests {
        t.Run(tt.url, func(t *testing.T) {
            result := strategies.ShouldSkipGitHubPagesURL(tt.url)
            assert.Equal(t, tt.expected, result)
        })
    }
}

func TestGitHubPagesStrategy_Name(t *testing.T) {
    strategy := strategies.NewGitHubPagesStrategy(nil)
    assert.Equal(t, "github_pages", strategy.Name())
}

func TestGitHubPagesStrategy_CanHandle(t *testing.T) {
    strategy := strategies.NewGitHubPagesStrategy(nil)
    
    assert.True(t, strategy.CanHandle("https://block.github.io/goose/"))
    assert.True(t, strategy.CanHandle("https://user.github.io/"))
    assert.False(t, strategy.CanHandle("https://github.com/block/goose"))
    assert.False(t, strategy.CanHandle("https://example.com/"))
}
```

#### File: `tests/unit/strategies/github_pages_discovery_test.go` (NEW)

```go
package strategies_test

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/quantmind-br/repodocs-go/internal/strategies"
)

func TestGetDiscoveryProbes(t *testing.T) {
    probes := strategies.GetDiscoveryProbes()
    
    // Verify order (llms.txt first, sitemaps second)
    assert.Equal(t, "/llms.txt", probes[0].Path)
    assert.Equal(t, "/sitemap.xml", probes[1].Path)
    
    // Verify we have all expected probes
    paths := make([]string, len(probes))
    for i, p := range probes {
        paths[i] = p.Path
    }
    
    assert.Contains(t, paths, "/search/search_index.json") // MkDocs
    assert.Contains(t, paths, "/search-index.json")        // Docusaurus
    assert.Contains(t, paths, "/index.json")               // Hugo
    assert.Contains(t, paths, "/hashmap.json")             // VitePress
}

func TestParseSitemapXML(t *testing.T) {
    content := []byte(`<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
        <url><loc>https://example.github.io/</loc></url>
        <url><loc>https://example.github.io/docs/</loc></url>
        <url><loc>https://example.github.io/api/</loc></url>
    </urlset>`)
    
    urls, err := strategies.ParseSitemapXML(content, "https://example.github.io")
    require.NoError(t, err)
    
    assert.Len(t, urls, 3)
    assert.Contains(t, urls, "https://example.github.io/")
    assert.Contains(t, urls, "https://example.github.io/docs/")
}

func TestParseDocusaurusIndex(t *testing.T) {
    content := []byte(`[
        {"url": "/docs/intro", "title": "Introduction", "content": "..."},
        {"url": "/docs/getting-started", "title": "Getting Started", "content": "..."}
    ]`)
    
    urls, err := strategies.ParseDocusaurusIndex(content, "https://example.github.io")
    require.NoError(t, err)
    
    assert.Len(t, urls, 2)
    assert.Contains(t, urls, "https://example.github.io/docs/intro")
    assert.Contains(t, urls, "https://example.github.io/docs/getting-started")
}

func TestParseHugoIndex(t *testing.T) {
    content := []byte(`[
        {"permalink": "https://example.github.io/posts/first/", "title": "First Post"},
        {"url": "/posts/second/", "title": "Second Post"}
    ]`)
    
    urls, err := strategies.ParseHugoIndex(content, "https://example.github.io")
    require.NoError(t, err)
    
    assert.Len(t, urls, 2)
}

func TestParseLLMsTxt(t *testing.T) {
    content := []byte(`# Documentation

- [Getting Started](/docs/getting-started)
- [API Reference](https://example.github.io/api)
- [Contributing](/contributing)
`)
    
    urls, err := strategies.ParseLLMsTxt(content, "https://example.github.io")
    require.NoError(t, err)
    
    assert.Len(t, urls, 3)
}
```

---

## File Changes Summary

| File | Action | Description |
|------|--------|-------------|
| `internal/app/detector.go` | Modify | Add `StrategyGitHubPages` type, detection, and cases in `CreateStrategy`/`GetAllStrategies` |
| `internal/strategies/github_pages.go` | Create | Main strategy implementation |
| `internal/strategies/github_pages_discovery.go` | Create | Multi-tier discovery probes |
| `tests/unit/strategies/github_pages_test.go` | Create | Unit tests for strategy |
| `tests/unit/strategies/github_pages_discovery_test.go` | Create | Unit tests for discovery |
| `tests/unit/app/detector_github_pages_test.go` | Create | Detection tests |
| `tests/integration/strategies/github_pages_integration_test.go` | Create | Browser-based integration tests |

---

## Dependencies

**No new dependencies required.** The `github.com/PuerkitoBio/goquery` package is already present in `go.mod` (v1.11.0) as a transitive dependency through colly.

---

## Testing Plan

### Unit Tests
1. URL detection (`IsGitHubPagesURL`) and override flag for custom domains
2. Discovery probe ordering and parsing (parallel probe logic)
3. Search index parsers (MkDocs, Docusaurus, Hugo, VitePress, sitemap index + .xml.gz)
4. URL normalization/deduplication and host filtering
5. URL filtering (`shouldSkipGitHubPagesURL`) with reduced defaults
6. Content validation (`isEmptyOrErrorContent`) with updated thresholds
7. HTTP-first extraction fallback behavior (fetch -> render)
8. Link extraction with goquery

### Integration Tests (require browser)
1. HTTP probe discovery with mock server
2. Browser-based discovery with real renderer
3. Full extraction pipeline with test site
4. robots.txt handling (disallow + crawl-delay)

### E2E Tests
1. `repodocs https://block.github.io/goose/ --limit 10`
2. Verify files have real content (not "nginx")
3. Test with MkDocs Material site (has search index)

---

## Rollout Plan

### Phase 1: Discovery Infrastructure (Day 1)
- [ ] Create `github_pages_discovery.go` with all parsers
- [ ] Add parallel probe execution + per-probe timeout
- [ ] Add sitemap index recursion + .xml.gz support
- [ ] Unit tests for all parsers
- [ ] Add goquery dependency (if not present)

### Phase 2: Main Strategy (Day 2)
- [ ] Create `github_pages.go` with HTTP-first extraction + browser fallback
- [ ] Add detection to `detector.go` and optional `--github-pages` override
- [ ] Add URL normalization/deduplication + host filtering
- [ ] Add robots.txt handling (best effort)
- [ ] Unit tests for strategy

### Phase 3: Integration (Day 3)
- [ ] Integration tests with browser
- [ ] E2E test with real GitHub Pages sites
- [ ] Performance optimization (timeouts, render selectors)

### Phase 4: Polish (Day 4)
- [ ] Edge case handling
- [ ] CI pipeline verification

---

## Success Criteria

1. **Functional**: `repodocs https://block.github.io/goose/` extracts all pages with real content
2. **No nginx**: Zero files containing only "nginx" or "301 Moved Permanently"
3. **HTTP-first**: Sites with sitemaps/search indexes don't require browser for discovery
4. **Tests Pass**: All unit, integration, and E2E tests pass
5. **Performance**: Discovery via HTTP probes completes in < 5 seconds
6. **Backward Compatible**: No changes to existing strategy behavior

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Browser unavailable | Medium | High | Clear error message, use HTTP-only extraction when possible |
| Rate limiting | Low | Medium | Add delays, respect robots.txt |
| Search index format changes | Low | Low | Graceful fallback to browser discovery |
| Memory issues with many tabs | Low | High | Hard limit of 5 concurrent tabs |
| goquery parsing edge cases | Low | Low | Fallback regex extraction |
| Custom domain GitHub Pages undetected | Medium | Medium | Optional `--github-pages` override or explicit strategy selection |
| Over-aggressive skip heuristics | Medium | Medium | Reduce defaults + allow user excludes/config |
| HTTP-first false positives (SPA shell) | Medium | Medium | Stronger SPA detection + browser fallback |

---

## Future Enhancements (Out of Scope)

1. **Pagefind binary shard parsing** - Complex format, low priority
2. **Algolia index extraction** - Requires API key, unlikely to be public
3. **Incremental updates** - Use sitemap lastmod for delta extraction
4. **Custom SSG detection** - Fingerprint SSG from HTML and use specific selectors
5. **robots.txt crawl-delay** - Respect rate limiting hints
