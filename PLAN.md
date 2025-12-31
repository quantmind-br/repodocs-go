# Implementation Plan: GitHub Wiki Strategy

## Executive Summary

This document provides a comprehensive implementation plan for adding GitHub Wiki extraction support to `repodocs-go`. The new `WikiStrategy` will enable users to extract, transform, and organize wiki documentation from any GitHub repository wiki into clean, hierarchically structured Markdown files.

**Target URL Example:** `https://github.com/Alexays/Waybar/wiki`

---

## Table of Contents

1. [Technical Analysis](#1-technical-analysis)
2. [Architecture Design](#2-architecture-design)
3. [Implementation Details](#3-implementation-details)
4. [File Changes](#4-file-changes)
5. [Testing Strategy](#5-testing-strategy)
6. [CLI Integration](#6-cli-integration)
7. [Implementation Phases](#7-implementation-phases)
8. [Risk Assessment](#8-risk-assessment)

---

## 1. Technical Analysis

### 1.1 GitHub Wiki Characteristics

| Property | Value | Notes |
|----------|-------|-------|
| **Storage** | Separate Git repository | `{repo}.wiki.git` |
| **API Access** | None | No REST/GraphQL API for wiki content |
| **Clone URL (HTTPS)** | `https://github.com/{owner}/{repo}.wiki.git` | Public wikis |
| **Clone URL (SSH)** | `git@github.com:{owner}/{repo}.wiki.git` | Authenticated |
| **File Structure** | Flat (root directory only) | No subdirectories supported |
| **File Format** | Markdown (`.md`), others supported | MediaWiki, AsciiDoc, etc. |
| **Archive Download** | Not available | Must use git clone |

### 1.2 Special Files

| File | Purpose | Required |
|------|---------|----------|
| `Home.md` | Wiki landing page (index) | Yes (created by default) |
| `_Sidebar.md` | Custom navigation sidebar | No (optional) |
| `_Footer.md` | Common footer for all pages | No (optional) |

### 1.3 Wiki Link Syntax

GitHub wikis support special link syntax:

```markdown
# Standard Markdown links
[Link Text](Page-Name)           # Without .md extension
[Link Text](Page-Name.md)        # With extension

# Wiki-style links (GitHub-specific)
[[Page Name]]                    # Auto-generates link text
[[Page Name|Custom Text]]        # Custom link text
[[Page Name#Section]]            # Link to section
```

### 1.4 URL Patterns to Detect

```
# Wiki root
https://github.com/{owner}/{repo}/wiki
https://github.com/{owner}/{repo}/wiki/

# Specific wiki page
https://github.com/{owner}/{repo}/wiki/{Page-Name}

# Direct clone URL
https://github.com/{owner}/{repo}.wiki.git
git@github.com:{owner}/{repo}.wiki.git
```

### 1.5 Sidebar Structure Example

```markdown
# _Sidebar.md example from a real wiki

# Table of Contents

## Getting Started
* [[Home]]
* [[Installation]]
* [[Quick Start]]

## Configuration
* [[Basic Configuration]]
* [[Advanced Settings]]
* [[Themes]]

## Development
* [[Contributing]]
* [[Building from Source]]
```

---

## 2. Architecture Design

### 2.1 Component Overview

```
+---------------------------------------------------------------------+
|                         WikiStrategy                                 |
+---------------------------------------------------------------------+
|  +------------------+  +------------------+  +--------------------+  |
|  |   URL Parser     |  |  Git Cloner      |  |   Sidebar Parser   |  |
|  |                  |  |                  |  |                    |  |
|  | - Detect wiki    |  | - Shallow clone  |  | - Parse _Sidebar   |  |
|  | - Extract owner  |  | - Handle auth    |  | - Build hierarchy  |  |
|  | - Extract repo   |  | - Temp dir mgmt  |  | - Extract order    |  |
|  +------------------+  +------------------+  +--------------------+  |
|                                                                      |
|  +------------------+  +------------------+  +--------------------+  |
|  |  Link Converter  |  |  Doc Processor   |  |   Output Manager   |  |
|  |                  |  |                  |  |                    |  |
|  | - [[...]] links  |  | - Read .md files |  | - Create hierarchy |  |
|  | - Relative URLs  |  | - Add metadata   |  | - Write documents  |  |
|  | - Section links  |  | - Frontmatter    |  | - Generate index   |  |
|  +------------------+  +------------------+  +--------------------+  |
+---------------------------------------------------------------------+
```

### 2.2 Data Flow

```
+--------------+     +--------------+     +--------------+
|  Wiki URL    |---->| Parse URL    |---->| Clone Wiki   |
|              |     |              |     | Repository   |
+--------------+     +--------------+     +------+-------+
                                                 |
                                                 v
+--------------+     +--------------+     +--------------+
| Write to     |<----| Process      |<----| Parse        |
| Output Dir   |     | Documents    |     | _Sidebar.md  |
+--------------+     +--------------+     +--------------+
       |
       v
+------------------------------------------------------+
|                    Output Structure                   |
|  docs/                                               |
|  +-- index.md              (Home.md)                 |
|  +-- getting-started/                                |
|  |   +-- installation.md                             |
|  |   +-- quick-start.md                              |
|  +-- configuration/                                  |
|  |   +-- basic.md                                    |
|  |   +-- advanced.md                                 |
|  +-- _wiki_metadata.json   (optional)                |
+------------------------------------------------------+
```

### 2.3 New Types

```go
// WikiPage represents a parsed wiki page
type WikiPage struct {
    Filename     string   // Original filename (e.g., "Page-Name.md")
    Title        string   // Extracted title (e.g., "Page Name")
    Content      string   // Markdown content
    Section      string   // Section from sidebar (e.g., "Getting Started")
    Order        int      // Order within section
    Links        []string // Extracted internal links
    IsHome       bool     // Is this Home.md?
    IsSpecial    bool     // Is _Sidebar.md or _Footer.md?
}

// WikiStructure represents the parsed sidebar hierarchy
type WikiStructure struct {
    Sections   []WikiSection
    Pages      map[string]*WikiPage // filename -> page
    HasSidebar bool
}

// WikiSection represents a section in the sidebar
type WikiSection struct {
    Name   string
    Order  int
    Pages  []string // Page filenames in order
}

// WikiInfo contains parsed wiki URL information
type WikiInfo struct {
    Owner       string
    Repo        string
    CloneURL    string
    Platform    string // "github", "gitlab" (future)
    TargetPage  string // Specific page if in URL
}
```

---

## 3. Implementation Details

### 3.1 File: `internal/strategies/wiki.go`

```go
package strategies

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    "strings"
    "time"

    "github.com/go-git/go-git/v5"
    githttp "github.com/go-git/go-git/v5/plumbing/transport/http"
    "github.com/quantmind-br/repodocs-go/internal/domain"
    "github.com/quantmind-br/repodocs-go/internal/output"
    "github.com/quantmind-br/repodocs-go/internal/utils"
    "github.com/schollz/progressbar/v3"
)

// WikiStrategy extracts documentation from GitHub wiki repositories
type WikiStrategy struct {
    writer *output.Writer
    logger *utils.Logger
}

// NewWikiStrategy creates a new wiki strategy
func NewWikiStrategy(deps *Dependencies) *WikiStrategy {
    return &WikiStrategy{
        writer: deps.Writer,
        logger: deps.Logger,
    }
}

// Name returns the strategy name
func (s *WikiStrategy) Name() string {
    return "wiki"
}

// CanHandle returns true if this strategy can handle the given URL
func (s *WikiStrategy) CanHandle(url string) bool {
    return IsWikiURL(url)
}

// IsWikiURL checks if a URL points to a GitHub wiki
func IsWikiURL(url string) bool {
    lower := strings.ToLower(url)

    // Pattern 1: github.com/{owner}/{repo}/wiki
    if strings.Contains(lower, "github.com") && strings.Contains(lower, "/wiki") {
        return true
    }

    // Pattern 2: {repo}.wiki.git
    if strings.HasSuffix(lower, ".wiki.git") {
        return true
    }

    return false
}

// Execute runs the wiki extraction strategy
func (s *WikiStrategy) Execute(ctx context.Context, url string, opts Options) error {
    s.logger.Info().Str("url", url).Msg("Starting wiki extraction")

    // Step 1: Parse wiki URL
    wikiInfo, err := ParseWikiURL(url)
    if err != nil {
        return fmt.Errorf("failed to parse wiki URL: %w", err)
    }

    s.logger.Debug().
        Str("owner", wikiInfo.Owner).
        Str("repo", wikiInfo.Repo).
        Str("clone_url", wikiInfo.CloneURL).
        Msg("Parsed wiki URL")

    // Step 2: Create temporary directory
    tmpDir, err := os.MkdirTemp("", "repodocs-wiki-*")
    if err != nil {
        return fmt.Errorf("failed to create temp dir: %w", err)
    }
    defer os.RemoveAll(tmpDir)

    // Step 3: Clone wiki repository
    if err := s.cloneWiki(ctx, wikiInfo.CloneURL, tmpDir); err != nil {
        return fmt.Errorf("failed to clone wiki: %w", err)
    }

    // Step 4: Parse wiki structure
    structure, err := s.parseWikiStructure(tmpDir)
    if err != nil {
        return fmt.Errorf("failed to parse wiki structure: %w", err)
    }

    s.logger.Info().
        Int("pages", len(structure.Pages)).
        Int("sections", len(structure.Sections)).
        Bool("has_sidebar", structure.HasSidebar).
        Msg("Parsed wiki structure")

    // Step 5: Process and write documents
    return s.processPages(ctx, structure, tmpDir, wikiInfo, opts)
}

// cloneWiki clones the wiki repository
func (s *WikiStrategy) cloneWiki(ctx context.Context, cloneURL, destDir string) error {
    s.logger.Info().Str("url", cloneURL).Msg("Cloning wiki repository")

    cloneOpts := &git.CloneOptions{
        URL:      cloneURL,
        Depth:    1, // Shallow clone for speed
        Progress: nil,
    }

    // Use HTTPS auth if available
    if token := os.Getenv("GITHUB_TOKEN"); token != "" {
        cloneOpts.Auth = &githttp.BasicAuth{
            Username: "token",
            Password: token,
        }
    }

    _, err := git.PlainCloneContext(ctx, destDir, false, cloneOpts)
    if err != nil {
        // Check if wiki doesn't exist
        if strings.Contains(err.Error(), "not found") ||
           strings.Contains(err.Error(), "404") {
            return fmt.Errorf("wiki not found or not enabled for this repository")
        }
        return err
    }

    return nil
}

// parseWikiStructure parses the wiki file structure and sidebar
func (s *WikiStrategy) parseWikiStructure(dir string) (*WikiStructure, error) {
    structure := &WikiStructure{
        Pages:    make(map[string]*WikiPage),
        Sections: []WikiSection{},
    }

    // Read all markdown files
    entries, err := os.ReadDir(dir)
    if err != nil {
        return nil, err
    }

    for _, entry := range entries {
        if entry.IsDir() {
            continue
        }

        name := entry.Name()
        ext := strings.ToLower(filepath.Ext(name))

        // Only process markdown files
        if ext != ".md" && ext != ".markdown" {
            continue
        }

        content, err := os.ReadFile(filepath.Join(dir, name))
        if err != nil {
            s.logger.Warn().Err(err).Str("file", name).Msg("Failed to read file")
            continue
        }

        page := &WikiPage{
            Filename:  name,
            Title:     filenameToTitle(name),
            Content:   string(content),
            IsHome:    strings.EqualFold(name, "Home.md"),
            IsSpecial: strings.HasPrefix(name, "_"),
        }

        structure.Pages[name] = page
    }

    // Parse sidebar if exists
    if sidebarPage, exists := structure.Pages["_Sidebar.md"]; exists {
        structure.HasSidebar = true
        structure.Sections = parseSidebarContent(sidebarPage.Content, structure.Pages)
    } else {
        // Create default structure (alphabetical, single section)
        structure.Sections = createDefaultStructure(structure.Pages)
    }

    return structure, nil
}

// processPages processes all wiki pages and writes them to output
func (s *WikiStrategy) processPages(
    ctx context.Context,
    structure *WikiStructure,
    tmpDir string,
    wikiInfo *WikiInfo,
    opts Options,
) error {
    // Count processable pages (exclude special files)
    var processablePages []*WikiPage
    for _, page := range structure.Pages {
        if !page.IsSpecial {
            processablePages = append(processablePages, page)
        }
    }

    // Apply limit
    if opts.Limit > 0 && len(processablePages) > opts.Limit {
        processablePages = processablePages[:opts.Limit]
    }

    // Create progress bar
    bar := progressbar.NewOptions(len(processablePages),
        progressbar.OptionSetDescription("Processing wiki pages"),
        progressbar.OptionShowCount(),
    )

    // Build base wiki URL for references
    baseWikiURL := fmt.Sprintf("https://github.com/%s/%s/wiki", wikiInfo.Owner, wikiInfo.Repo)

    // Process each page
    for _, page := range processablePages {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        if err := s.processPage(ctx, page, structure, baseWikiURL, opts); err != nil {
            s.logger.Warn().Err(err).Str("page", page.Filename).Msg("Failed to process page")
        }
        bar.Add(1)
    }

    s.logger.Info().
        Int("processed", len(processablePages)).
        Msg("Wiki extraction completed")

    return nil
}

// processPage processes a single wiki page
func (s *WikiStrategy) processPage(
    ctx context.Context,
    page *WikiPage,
    structure *WikiStructure,
    baseWikiURL string,
    opts Options,
) error {
    // Convert wiki-style links to standard markdown
    content := convertWikiLinks(page.Content, structure.Pages)

    // Generate page URL
    pageName := strings.TrimSuffix(page.Filename, filepath.Ext(page.Filename))
    pageURL := baseWikiURL
    if !page.IsHome {
        pageURL = fmt.Sprintf("%s/%s", baseWikiURL, pageName)
    }

    // Build relative path based on section
    relativePath := buildRelativePath(page, structure, opts.NoFolders)

    // Create document
    doc := &domain.Document{
        URL:            pageURL,
        Title:          page.Title,
        Content:        content,
        FetchedAt:      time.Now(),
        WordCount:      len(strings.Fields(content)),
        CharCount:      len(content),
        SourceStrategy: s.Name(),
        RelativePath:   relativePath,
    }

    // Write document
    if !opts.DryRun {
        return s.writer.Write(ctx, doc)
    }

    return nil
}
```

### 3.2 File: `internal/strategies/wiki_parser.go`

```go
package strategies

import (
    "fmt"
    "path/filepath"
    "regexp"
    "sort"
    "strings"
)

// WikiPage represents a parsed wiki page
type WikiPage struct {
    Filename  string
    Title     string
    Content   string
    Section   string
    Order     int
    Links     []string
    IsHome    bool
    IsSpecial bool
}

// WikiStructure represents the parsed wiki hierarchy
type WikiStructure struct {
    Sections   []WikiSection
    Pages      map[string]*WikiPage
    HasSidebar bool
}

// WikiSection represents a section in the sidebar
type WikiSection struct {
    Name  string
    Order int
    Pages []string
}

// WikiInfo contains parsed wiki URL information
type WikiInfo struct {
    Owner      string
    Repo       string
    CloneURL   string
    Platform   string
    TargetPage string
}

// ParseWikiURL parses a wiki URL and extracts repository information
func ParseWikiURL(rawURL string) (*WikiInfo, error) {
    // Normalize URL
    url := strings.TrimSuffix(rawURL, "/")

    // Pattern: https://github.com/{owner}/{repo}/wiki[/{page}]
    wikiPattern := regexp.MustCompile(
        `github\.com[:/]([^/]+)/([^/]+?)(?:\.wiki)?(?:/wiki)?(?:/([^/]+))?(?:\.git)?$`,
    )

    matches := wikiPattern.FindStringSubmatch(url)
    if len(matches) < 3 {
        return nil, fmt.Errorf("invalid wiki URL format: %s", rawURL)
    }

    owner := matches[1]
    repo := strings.TrimSuffix(matches[2], ".wiki")

    var targetPage string
    if len(matches) > 3 && matches[3] != "" {
        targetPage = matches[3]
    }

    // Build clone URL
    cloneURL := fmt.Sprintf("https://github.com/%s/%s.wiki.git", owner, repo)

    return &WikiInfo{
        Owner:      owner,
        Repo:       repo,
        CloneURL:   cloneURL,
        Platform:   "github",
        TargetPage: targetPage,
    }, nil
}

// filenameToTitle converts a wiki filename to a readable title
// Examples:
//   "Getting-Started.md" -> "Getting Started"
//   "API_Reference.md" -> "API Reference"
//   "Home.md" -> "Home"
func filenameToTitle(filename string) string {
    // Remove extension
    name := strings.TrimSuffix(filename, filepath.Ext(filename))

    // Replace hyphens and underscores with spaces
    name = strings.ReplaceAll(name, "-", " ")
    name = strings.ReplaceAll(name, "_", " ")

    // Capitalize first letter of each word
    words := strings.Fields(name)
    for i, word := range words {
        if len(word) > 0 {
            words[i] = strings.ToUpper(word[:1]) + word[1:]
        }
    }

    return strings.Join(words, " ")
}

// titleToFilename converts a title back to wiki filename format
// Examples:
//   "Getting Started" -> "Getting-Started"
//   "API Reference" -> "API-Reference"
func titleToFilename(title string) string {
    return strings.ReplaceAll(title, " ", "-")
}

// parseSidebarContent parses _Sidebar.md content to extract structure
func parseSidebarContent(content string, pages map[string]*WikiPage) []WikiSection {
    var sections []WikiSection
    var currentSection *WikiSection

    lines := strings.Split(content, "\n")
    sectionOrder := 0
    pageOrder := 0

    // Regex patterns
    headerPattern := regexp.MustCompile(`^#+\s*(.+)$`)                          // ## Section Name
    wikiLinkPattern := regexp.MustCompile(`\[\[([^\]|]+)(?:\|[^\]]+)?\]\]`)     // [[Page]] or [[Page|Text]]
    mdLinkPattern := regexp.MustCompile(`\[([^\]]+)\]\(([^)]+)\)`)              // [Text](Page)

    for _, line := range lines {
        trimmed := strings.TrimSpace(line)

        // Check for section header
        if matches := headerPattern.FindStringSubmatch(trimmed); len(matches) > 1 {
            // Save previous section
            if currentSection != nil && len(currentSection.Pages) > 0 {
                sections = append(sections, *currentSection)
            }

            // Start new section
            sectionOrder++
            pageOrder = 0
            currentSection = &WikiSection{
                Name:  strings.TrimSpace(matches[1]),
                Order: sectionOrder,
                Pages: []string{},
            }
            continue
        }

        // Check for wiki-style links [[Page Name]]
        if wikiMatches := wikiLinkPattern.FindAllStringSubmatch(trimmed, -1); len(wikiMatches) > 0 {
            for _, match := range wikiMatches {
                pageName := match[1]
                filename := findPageFilename(pageName, pages)
                if filename != "" {
                    pageOrder++
                    if page, exists := pages[filename]; exists {
                        page.Section = currentSection.Name
                        page.Order = pageOrder
                    }
                    if currentSection != nil {
                        currentSection.Pages = append(currentSection.Pages, filename)
                    }
                }
            }
            continue
        }

        // Check for markdown links [Text](Page)
        if mdMatches := mdLinkPattern.FindAllStringSubmatch(trimmed, -1); len(mdMatches) > 0 {
            for _, match := range mdMatches {
                pageName := match[2]
                // Remove .md extension if present
                pageName = strings.TrimSuffix(pageName, ".md")
                filename := findPageFilename(pageName, pages)
                if filename != "" {
                    pageOrder++
                    if page, exists := pages[filename]; exists {
                        page.Section = currentSection.Name
                        page.Order = pageOrder
                    }
                    if currentSection != nil {
                        currentSection.Pages = append(currentSection.Pages, filename)
                    }
                }
            }
        }
    }

    // Save last section
    if currentSection != nil && len(currentSection.Pages) > 0 {
        sections = append(sections, *currentSection)
    }

    return sections
}

// findPageFilename finds the actual filename for a page reference
func findPageFilename(pageName string, pages map[string]*WikiPage) string {
    // Try exact match with .md
    if _, exists := pages[pageName+".md"]; exists {
        return pageName + ".md"
    }

    // Try with hyphens instead of spaces
    hyphenated := strings.ReplaceAll(pageName, " ", "-") + ".md"
    if _, exists := pages[hyphenated]; exists {
        return hyphenated
    }

    // Try case-insensitive match
    for filename := range pages {
        if strings.EqualFold(strings.TrimSuffix(filename, ".md"), pageName) ||
           strings.EqualFold(strings.TrimSuffix(filename, ".md"), strings.ReplaceAll(pageName, " ", "-")) {
            return filename
        }
    }

    return ""
}

// createDefaultStructure creates a default structure when no sidebar exists
func createDefaultStructure(pages map[string]*WikiPage) []WikiSection {
    // Create single section with all pages alphabetically
    var pageNames []string
    for filename, page := range pages {
        if !page.IsSpecial {
            pageNames = append(pageNames, filename)
        }
    }

    sort.Strings(pageNames)

    // Move Home to front if exists
    for i, name := range pageNames {
        if strings.EqualFold(name, "Home.md") {
            pageNames = append([]string{name}, append(pageNames[:i], pageNames[i+1:]...)...)
            break
        }
    }

    // Assign order to pages
    for i, filename := range pageNames {
        if page, exists := pages[filename]; exists {
            page.Order = i + 1
            page.Section = "Documentation"
        }
    }

    return []WikiSection{
        {
            Name:  "Documentation",
            Order: 1,
            Pages: pageNames,
        },
    }
}

// convertWikiLinks converts wiki-style links to standard markdown
func convertWikiLinks(content string, pages map[string]*WikiPage) string {
    // Pattern 1: [[Page Name|Custom Text]] -> [Custom Text](./page-name.md)
    pattern1 := regexp.MustCompile(`\[\[([^\]|]+)\|([^\]]+)\]\]`)
    content = pattern1.ReplaceAllStringFunc(content, func(match string) string {
        matches := pattern1.FindStringSubmatch(match)
        if len(matches) == 3 {
            pageName := matches[1]
            linkText := matches[2]
            filename := titleToFilename(pageName) + ".md"
            return fmt.Sprintf("[%s](./%s)", linkText, strings.ToLower(filename))
        }
        return match
    })

    // Pattern 2: [[Page Name#Section]] -> [Page Name](./page-name.md#section)
    pattern2 := regexp.MustCompile(`\[\[([^\]#]+)#([^\]]+)\]\]`)
    content = pattern2.ReplaceAllStringFunc(content, func(match string) string {
        matches := pattern2.FindStringSubmatch(match)
        if len(matches) == 3 {
            pageName := matches[1]
            section := matches[2]
            filename := titleToFilename(pageName) + ".md"
            anchor := strings.ToLower(strings.ReplaceAll(section, " ", "-"))
            return fmt.Sprintf("[%s](./%s#%s)", pageName, strings.ToLower(filename), anchor)
        }
        return match
    })

    // Pattern 3: [[Page Name]] -> [Page Name](./page-name.md)
    pattern3 := regexp.MustCompile(`\[\[([^\]]+)\]\]`)
    content = pattern3.ReplaceAllStringFunc(content, func(match string) string {
        matches := pattern3.FindStringSubmatch(match)
        if len(matches) == 2 {
            pageName := matches[1]
            filename := titleToFilename(pageName) + ".md"
            return fmt.Sprintf("[%s](./%s)", pageName, strings.ToLower(filename))
        }
        return match
    })

    return content
}

// buildRelativePath builds the output path based on wiki structure
func buildRelativePath(page *WikiPage, structure *WikiStructure, flat bool) string {
    // For Home.md, always use index.md at root
    if page.IsHome {
        return "index.md"
    }

    // If flat mode or no sections, just use filename
    if flat || len(structure.Sections) == 0 || page.Section == "" {
        return strings.ToLower(page.Filename)
    }

    // Build hierarchical path: section/filename
    sectionDir := strings.ToLower(strings.ReplaceAll(page.Section, " ", "-"))
    filename := strings.ToLower(page.Filename)

    return filepath.Join(sectionDir, filename)
}
```

### 3.3 Updates to `internal/app/detector.go`

```go
// Add new strategy type constant
const (
    StrategyLLMS    StrategyType = "llms"
    StrategySitemap StrategyType = "sitemap"
    StrategyWiki    StrategyType = "wiki"     // NEW
    StrategyGit     StrategyType = "git"
    StrategyPkgGo   StrategyType = "pkggo"
    StrategyCrawler StrategyType = "crawler"
    StrategyUnknown StrategyType = "unknown"
)

// Update DetectStrategy function
func DetectStrategy(url string) StrategyType {
    lower := strings.ToLower(url)

    // Check for llms.txt first
    if strings.HasSuffix(lower, "/llms.txt") || strings.HasSuffix(lower, "llms.txt") {
        return StrategyLLMS
    }

    // Check for pkg.go.dev (before Git, since pkg.go.dev URLs contain github.com paths)
    if strings.Contains(lower, "pkg.go.dev") {
        return StrategyPkgGo
    }

    // Check for sitemap
    if strings.HasSuffix(lower, "sitemap.xml") ||
        strings.HasSuffix(lower, "sitemap.xml.gz") ||
        strings.Contains(lower, "sitemap") && strings.HasSuffix(lower, ".xml") {
        return StrategySitemap
    }

    // NEW: Check for GitHub Wiki (before generic Git)
    if strategies.IsWikiURL(url) {
        return StrategyWiki
    }

    // Check for Git repository
    // ... rest of existing code
}

// Update CreateStrategy function
func CreateStrategy(strategyType StrategyType, deps *strategies.Dependencies) strategies.Strategy {
    switch strategyType {
    case StrategyLLMS:
        return strategies.NewLLMSStrategy(deps)
    case StrategySitemap:
        return strategies.NewSitemapStrategy(deps)
    case StrategyWiki:                           // NEW
        return strategies.NewWikiStrategy(deps)  // NEW
    case StrategyGit:
        return strategies.NewGitStrategy(deps)
    case StrategyPkgGo:
        return strategies.NewPkgGoStrategy(deps)
    case StrategyCrawler:
        return strategies.NewCrawlerStrategy(deps)
    default:
        return nil
    }
}

// Update GetAllStrategies function
func GetAllStrategies(deps *strategies.Dependencies) []strategies.Strategy {
    return []strategies.Strategy{
        strategies.NewLLMSStrategy(deps),     // Most specific: /llms.txt
        strategies.NewPkgGoStrategy(deps),    // Specific: pkg.go.dev
        strategies.NewSitemapStrategy(deps),  // Specific: sitemap.xml
        strategies.NewWikiStrategy(deps),     // NEW: github.com/.../wiki
        strategies.NewGitStrategy(deps),      // General: github.com repositories
        strategies.NewCrawlerStrategy(deps),  // Catch-all: any HTTP URL
    }
}
```

### 3.4 Updates to `internal/utils/url.go`

```go
// Add new utility function

// IsWikiURL checks if a URL points to a GitHub wiki
func IsWikiURL(rawURL string) bool {
    lower := strings.ToLower(rawURL)

    // Pattern 1: github.com/{owner}/{repo}/wiki
    if strings.Contains(lower, "github.com") && strings.Contains(lower, "/wiki") {
        return true
    }

    // Pattern 2: {repo}.wiki.git
    if strings.HasSuffix(lower, ".wiki.git") {
        return true
    }

    return false
}

// Update GenerateOutputDirFromURL to handle wiki URLs
func GenerateOutputDirFromURL(rawURL string) string {
    // ... existing code ...

    // Handle Wiki URLs
    if IsWikiURL(rawURL) && strings.Contains(host, "github.com") {
        parts := strings.Split(pathStr, "/")
        if len(parts) >= 2 {
            // Use repo name + "-wiki" suffix
            name = parts[1] + "-wiki"
        }
    }

    // ... rest of existing code ...
}
```

---

## 4. File Changes

### 4.1 New Files

| File | Purpose | Lines (est.) |
|------|---------|--------------|
| `internal/strategies/wiki.go` | WikiStrategy implementation | ~250 |
| `internal/strategies/wiki_parser.go` | URL parser, sidebar parser, link converter | ~300 |
| `internal/strategies/wiki_test.go` | Unit tests for WikiStrategy | ~200 |
| `internal/strategies/wiki_parser_test.go` | Unit tests for parsers | ~250 |
| `tests/integration/strategies/wiki_test.go` | Integration tests | ~150 |

### 4.2 Modified Files

| File | Changes |
|------|---------|
| `internal/app/detector.go` | Add `StrategyWiki`, update detection logic |
| `internal/utils/url.go` | Add `IsWikiURL()` function |
| `internal/strategies/strategy.go` | No changes needed (uses existing interfaces) |

### 4.3 File Dependency Graph

```
internal/strategies/wiki.go
+-- internal/strategies/wiki_parser.go
+-- internal/domain/models.go (Document)
+-- internal/output/writer.go (Writer)
+-- internal/utils/logger.go (Logger)
+-- github.com/go-git/go-git/v5

internal/strategies/wiki_parser.go
+-- regexp
+-- strings
+-- path/filepath

internal/app/detector.go
+-- internal/strategies/wiki.go (IsWikiURL)
+-- internal/strategies (WikiStrategy)
```

---

## 5. Testing Strategy

### 5.1 Unit Tests

#### `wiki_parser_test.go`

```go
func TestParseWikiURL(t *testing.T) {
    tests := []struct {
        name     string
        url      string
        expected *WikiInfo
        wantErr  bool
    }{
        {
            name: "standard wiki URL",
            url:  "https://github.com/Alexays/Waybar/wiki",
            expected: &WikiInfo{
                Owner:    "Alexays",
                Repo:     "Waybar",
                CloneURL: "https://github.com/Alexays/Waybar.wiki.git",
                Platform: "github",
            },
        },
        {
            name: "wiki URL with page",
            url:  "https://github.com/owner/repo/wiki/Configuration",
            expected: &WikiInfo{
                Owner:      "owner",
                Repo:       "repo",
                CloneURL:   "https://github.com/owner/repo.wiki.git",
                Platform:   "github",
                TargetPage: "Configuration",
            },
        },
        {
            name: "direct clone URL",
            url:  "https://github.com/owner/repo.wiki.git",
            expected: &WikiInfo{
                Owner:    "owner",
                Repo:     "repo",
                CloneURL: "https://github.com/owner/repo.wiki.git",
                Platform: "github",
            },
        },
    }
    // ... test implementation
}

func TestFilenameToTitle(t *testing.T) {
    tests := []struct {
        input    string
        expected string
    }{
        {"Getting-Started.md", "Getting Started"},
        {"API_Reference.md", "API Reference"},
        {"Home.md", "Home"},
        {"advanced-configuration.md", "Advanced Configuration"},
    }
    // ... test implementation
}

func TestConvertWikiLinks(t *testing.T) {
    tests := []struct {
        input    string
        expected string
    }{
        {
            input:    "See [[Getting Started]] for more info",
            expected: "See [Getting Started](./getting-started.md) for more info",
        },
        {
            input:    "Check the [[Configuration|config page]]",
            expected: "Check the [config page](./configuration.md)",
        },
        {
            input:    "Go to [[Setup#Installation]]",
            expected: "Go to [Setup](./setup.md#installation)",
        },
    }
    // ... test implementation
}

func TestParseSidebarContent(t *testing.T) {
    sidebar := `
# Wiki

## Getting Started
* [[Home]]
* [[Installation]]

## Configuration
* [[Basic Config]]
* [[Advanced]]
`
    pages := map[string]*WikiPage{
        "Home.md":         {Filename: "Home.md"},
        "Installation.md": {Filename: "Installation.md"},
        "Basic-Config.md": {Filename: "Basic-Config.md"},
        "Advanced.md":     {Filename: "Advanced.md"},
    }

    sections := parseSidebarContent(sidebar, pages)

    assert.Len(t, sections, 2)
    assert.Equal(t, "Getting Started", sections[0].Name)
    assert.Len(t, sections[0].Pages, 2)
    // ... more assertions
}
```

### 5.2 Integration Tests

#### `tests/integration/strategies/wiki_test.go`

```go
func TestWikiStrategy_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    // Test with a real public wiki
    ctx := context.Background()
    tmpDir := t.TempDir()

    deps := createTestDependencies(tmpDir)
    defer deps.Close()

    strategy := strategies.NewWikiStrategy(deps)

    // Test CanHandle
    assert.True(t, strategy.CanHandle("https://github.com/Alexays/Waybar/wiki"))
    assert.False(t, strategy.CanHandle("https://github.com/owner/repo"))

    // Test Execute (uses real network)
    err := strategy.Execute(ctx, "https://github.com/Alexays/Waybar/wiki", strategies.Options{
        Output: tmpDir,
        Limit:  5, // Limit for faster tests
    })

    require.NoError(t, err)

    // Verify output
    files, _ := filepath.Glob(filepath.Join(tmpDir, "*.md"))
    assert.Greater(t, len(files), 0)
}
```

### 5.3 Test Data

Create test fixtures in `tests/testdata/wiki/`:

```
tests/testdata/wiki/
+-- _Sidebar.md
+-- Home.md
+-- Getting-Started.md
+-- Configuration.md
+-- Advanced-Topics.md
```

---

## 6. CLI Integration

### 6.1 Command Examples

```bash
# Extract entire wiki
repodocs https://github.com/Alexays/Waybar/wiki

# Extract with custom output directory
repodocs https://github.com/owner/repo/wiki -o ./waybar-docs

# Extract specific page only (future enhancement)
repodocs https://github.com/owner/repo/wiki/Configuration --single-page

# Dry run
repodocs https://github.com/owner/repo/wiki --dry-run

# Flat structure (no sections)
repodocs https://github.com/owner/repo/wiki --nofolders

# Verbose output
repodocs https://github.com/owner/repo/wiki -v

# With limit
repodocs https://github.com/owner/repo/wiki --limit 10
```

### 6.2 Output Examples

**Default (hierarchical based on sidebar):**
```
docs_waybar-wiki/
+-- index.md                    # Home.md
+-- getting-started/
|   +-- installation.md
|   +-- quick-start.md
+-- configuration/
|   +-- basic.md
|   +-- modules.md
+-- development/
    +-- contributing.md
```

**Flat mode (`--nofolders`):**
```
docs_waybar-wiki/
+-- index.md
+-- installation.md
+-- quick-start.md
+-- basic.md
+-- modules.md
+-- contributing.md
```

---

## 7. Implementation Phases

### Phase 1: Core Infrastructure (Day 1)

**Objective:** Basic wiki cloning and file reading

- [ ] Create `internal/strategies/wiki.go` skeleton
- [ ] Implement `ParseWikiURL()` function
- [ ] Implement `cloneWiki()` using go-git
- [ ] Add `IsWikiURL()` to utils
- [ ] Update `detector.go` with wiki detection
- [ ] Basic unit tests for URL parsing

**Deliverable:** Can clone a wiki repository to temp directory

### Phase 2: Structure Parsing (Day 2)

**Objective:** Parse wiki structure and sidebar

- [ ] Implement `filenameToTitle()` and `titleToFilename()`
- [ ] Implement `parseSidebarContent()` parser
- [ ] Implement `createDefaultStructure()` fallback
- [ ] Create `WikiPage` and `WikiStructure` types
- [ ] Unit tests for sidebar parsing

**Deliverable:** Can extract hierarchical structure from _Sidebar.md

### Phase 3: Link Conversion (Day 3)

**Objective:** Convert wiki-style links to standard markdown

- [ ] Implement `convertWikiLinks()` with all patterns:
  - `[[Page Name]]`
  - `[[Page Name|Text]]`
  - `[[Page#Section]]`
- [ ] Handle edge cases (case sensitivity, spaces vs hyphens)
- [ ] Unit tests for link conversion

**Deliverable:** Wiki content converted to portable markdown

### Phase 4: Output Generation (Day 4)

**Objective:** Generate organized output structure

- [ ] Implement `buildRelativePath()` for hierarchical output
- [ ] Implement `processPages()` main loop
- [ ] Implement `processPage()` document creation
- [ ] Integration with existing `Writer`
- [ ] Handle `Home.md` -> `index.md` conversion

**Deliverable:** Full wiki extraction to organized directories

### Phase 5: Testing & Polish (Day 5)

**Objective:** Comprehensive testing and edge cases

- [ ] Integration tests with real wikis
- [ ] Edge case handling (empty wiki, no sidebar, special characters)
- [ ] Error messages and logging
- [ ] Documentation updates
- [ ] Performance testing with large wikis

**Deliverable:** Production-ready wiki strategy

---

## 8. Risk Assessment

### 8.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Wiki doesn't exist | Medium | Low | Clear error message, validate before clone |
| Private wiki (auth required) | Medium | Medium | Support GITHUB_TOKEN, document requirements |
| Non-standard sidebar format | Medium | Low | Fallback to default structure |
| Very large wiki (>1000 pages) | Low | Medium | Implement pagination/limits |
| Network timeout during clone | Low | Low | Existing retry logic in go-git |
| Special characters in filenames | Medium | Low | Sanitize filenames |

### 8.2 Compatibility Risks

| Risk | Mitigation |
|------|------------|
| Different markdown flavors | Use existing converter pipeline |
| Non-markdown wiki pages | Skip or basic handling |
| Images/assets in wiki | Log warning, future enhancement |

### 8.3 Acceptance Criteria

- [ ] Successfully extracts Waybar wiki (https://github.com/Alexays/Waybar/wiki)
- [ ] Parses sidebar hierarchy correctly
- [ ] Converts all wiki link formats
- [ ] Generates clean, portable markdown
- [ ] Works with wikis without _Sidebar.md
- [ ] Handles authentication via GITHUB_TOKEN
- [ ] All tests pass
- [ ] No data races (mutex protection where needed)

---

## Appendix A: Reference Implementation Links

- GitHub Wiki Documentation: https://docs.github.com/en/communities/documenting-your-project-with-wikis
- go-git Library: https://github.com/go-git/go-git
- Example Wiki for Testing: https://github.com/Alexays/Waybar/wiki

## Appendix B: Related Files in Codebase

```
internal/strategies/git.go       # Reference for Git cloning patterns
internal/strategies/sitemap.go   # Reference for URL parsing
internal/strategies/llms.go      # Reference for content extraction
internal/output/writer.go        # Output writing interface
internal/domain/models.go        # Document model definition
internal/utils/url.go            # URL utility functions
```

---

*Plan created: 2025-12-31*
*Author: Claude Code Assistant*
*Status: Ready for Implementation*
