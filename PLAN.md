# Git Path Filtering Implementation Plan

## Overview

Add support for filtering Git repository processing to a specific subdirectory, enabling users to extract documentation from only a portion of a repository.

### Target Usage

```bash
# Option 1: Direct URL with path
repodocs https://github.com/steveyegge/beads/tree/main/docs

# Option 2: Using --filter flag
repodocs https://github.com/steveyegge/beads --filter docs

# Option 3: Nested paths
repodocs https://github.com/owner/repo/tree/main/docs/api/v2
```

---

## Current State Analysis

### Files Affected

| File | Current Behavior | Required Change |
|------|------------------|-----------------|
| `internal/app/detector.go:56` | Rejects URLs with `/tree/` for Git | Accept `/tree/` URLs |
| `internal/strategies/git.go:80-98` | `CanHandle()` rejects `/tree/` | Accept `/tree/` URLs |
| `internal/strategies/git.go:102-144` | `Execute()` processes all files | Parse path, apply filter |
| `internal/strategies/git.go:388-414` | `findDocumentationFiles()` walks all dirs | Filter by subpath |

### Current URL Rejection Logic

```go
// detector.go:56 - Currently REJECTS /tree/ URLs
(strings.Contains(lower, "github.com") && 
 !strings.Contains(lower, "/blob/") && 
 !strings.Contains(lower, "/tree/"))  // ← Blocks path URLs

// git.go:95 - Same rejection
!strings.Contains(lower, "/tree/")
```

---

## Detailed Implementation

### Phase 1: URL Parsing Infrastructure

#### 1.1 New Type: `gitURLInfo`

**File:** `internal/strategies/git.go`

**Location:** After line 151 (after `repoInfo` struct)

```go
// gitURLInfo contains parsed Git URL information including optional path
type gitURLInfo struct {
    repoURL  string // Clean repository URL (without /tree/... suffix)
    platform string // github, gitlab, bitbucket
    owner    string
    repo     string
    branch   string // Branch from URL (empty if not specified)
    subPath  string // Subdirectory path (empty if root)
}
```

#### 1.2 New Function: `parseGitURLWithPath()`

**File:** `internal/strategies/git.go`

**Location:** After `parseGitURL()` function (after line 216)

**Algorithm:**

```
Input: https://github.com/steveyegge/beads/tree/main/docs/api
                         ↓
Step 1: Detect platform patterns
        Match: github.com/steveyegge/beads/tree/main/docs/api
                         ↓
Step 2: Extract owner/repo
        owner: steveyegge
        repo: beads
                         ↓
Step 3: Find /tree/ segment
        Found at position after "beads"
                         ↓
Step 4: Parse branch and path
        Remaining: main/docs/api
        Branch: main (first segment)
        SubPath: docs/api (rest)
                         ↓
Step 5: Reconstruct clean repo URL
        repoURL: https://github.com/steveyegge/beads
```

**Implementation:**

```go
// parseGitURLWithPath extracts repository URL and optional subpath from Git URLs
// Handles URLs like:
//   - https://github.com/owner/repo
//   - https://github.com/owner/repo/tree/main
//   - https://github.com/owner/repo/tree/main/docs
//   - https://gitlab.com/owner/repo/-/tree/main/docs
func (s *GitStrategy) parseGitURLWithPath(url string) (*gitURLInfo, error) {
    info := &gitURLInfo{}
    lower := strings.ToLower(url)

    // Platform-specific patterns
    patterns := []struct {
        platform    string
        repoPattern *regexp.Regexp
        treePattern *regexp.Regexp
    }{
        {
            platform:    "github",
            repoPattern: regexp.MustCompile(`^(https?://github\.com/([^/]+)/([^/]+))`),
            treePattern: regexp.MustCompile(`/tree/([^/]+)(?:/(.+))?$`),
        },
        {
            platform:    "gitlab",
            repoPattern: regexp.MustCompile(`^(https?://gitlab\.com/([^/]+)/([^/]+))`),
            treePattern: regexp.MustCompile(`/-/tree/([^/]+)(?:/(.+))?$`),
        },
        {
            platform:    "bitbucket",
            repoPattern: regexp.MustCompile(`^(https?://bitbucket\.org/([^/]+)/([^/]+))`),
            treePattern: regexp.MustCompile(`/src/([^/]+)(?:/(.+))?$`),
        },
    }

    for _, p := range patterns {
        if !strings.Contains(lower, p.platform) {
            continue
        }

        // Extract base repo URL
        repoMatches := p.repoPattern.FindStringSubmatch(url)
        if len(repoMatches) < 4 {
            continue
        }

        info.platform = p.platform
        info.repoURL = repoMatches[1]
        info.owner = repoMatches[2]
        info.repo = strings.TrimSuffix(repoMatches[3], ".git")

        // Extract tree path if present
        treeMatches := p.treePattern.FindStringSubmatch(url)
        if len(treeMatches) >= 2 {
            info.branch = treeMatches[1]
            if len(treeMatches) >= 3 && treeMatches[2] != "" {
                info.subPath = strings.TrimSuffix(treeMatches[2], "/")
            }
        }

        return info, nil
    }

    return nil, fmt.Errorf("unsupported git URL format: %s", url)
}
```

#### 1.3 Unit Tests for `parseGitURLWithPath()`

**File:** `internal/strategies/git_strategy_test.go`

**Test Cases:**

| Input URL | Expected repoURL | Expected branch | Expected subPath |
|-----------|------------------|-----------------|------------------|
| `https://github.com/owner/repo` | `https://github.com/owner/repo` | `""` | `""` |
| `https://github.com/owner/repo.git` | `https://github.com/owner/repo` | `""` | `""` |
| `https://github.com/owner/repo/tree/main` | `https://github.com/owner/repo` | `main` | `""` |
| `https://github.com/owner/repo/tree/main/docs` | `https://github.com/owner/repo` | `main` | `docs` |
| `https://github.com/owner/repo/tree/main/docs/api` | `https://github.com/owner/repo` | `main` | `docs/api` |
| `https://github.com/owner/repo/tree/develop/src` | `https://github.com/owner/repo` | `develop` | `src` |
| `https://gitlab.com/owner/repo/-/tree/main/docs` | `https://gitlab.com/owner/repo` | `main` | `docs` |
| `https://bitbucket.org/owner/repo/src/main/docs` | `https://bitbucket.org/owner/repo` | `main` | `docs` |

---

### Phase 2: Detection Layer Updates

#### 2.1 Modify `detector.go`

**File:** `internal/app/detector.go`

**Change:** Line 56 - Remove `/tree/` exclusion for GitHub

**Before:**
```go
(strings.Contains(lower, "github.com") && 
 !strings.Contains(lower, "/blob/") && 
 !strings.Contains(lower, "/tree/"))
```

**After:**
```go
(strings.Contains(lower, "github.com") && 
 !strings.Contains(lower, "/blob/"))
```

**Rationale:** 
- `/blob/` = single file view → not supported (would need different handling)
- `/tree/` = directory view → now supported with path filtering

#### 2.2 Modify `git.go CanHandle()`

**File:** `internal/strategies/git.go`

**Change:** Lines 93-96 - Same removal of `/tree/` exclusion

**Before:**
```go
(strings.Contains(lower, "github.com") && 
 !strings.Contains(lower, "/blob/") && 
 !strings.Contains(lower, "/tree/"))
```

**After:**
```go
(strings.Contains(lower, "github.com") && 
 !strings.Contains(lower, "/blob/"))
```

---

### Phase 3: Execute() Flow Modification

#### 3.1 Modify `Execute()` Function

**File:** `internal/strategies/git.go`

**Changes to `Execute()` (lines 102-144):**

```go
func (s *GitStrategy) Execute(ctx context.Context, url string, opts Options) error {
    s.logger.Info().Str("url", url).Msg("Starting git extraction")

    // NEW: Parse URL to extract repository URL and optional subpath
    urlInfo, err := s.parseGitURLWithPath(url)
    if err != nil {
        return fmt.Errorf("failed to parse git URL: %w", err)
    }

    // NEW: Determine filter path (URL subpath takes precedence over opts.FilterURL)
    filterPath := urlInfo.subPath
    if filterPath == "" && opts.FilterURL != "" {
        filterPath = opts.FilterURL
    }

    // NEW: Log filter if active
    if filterPath != "" {
        s.logger.Info().
            Str("filter_path", filterPath).
            Msg("Path filter active - only processing files under this directory")
    }

    // Create temporary directory
    tmpDir, err := os.MkdirTemp("", "repodocs-git-*")
    if err != nil {
        return fmt.Errorf("failed to create temp dir: %w", err)
    }
    defer os.RemoveAll(tmpDir)

    // MODIFIED: Use clean repoURL for download
    repoURL := urlInfo.repoURL
    
    // Try archive download first (faster)
    branch, method, err := s.tryArchiveDownload(ctx, repoURL, tmpDir)
    if err != nil {
        s.logger.Info().Err(err).Msg("Archive download failed, using git clone")
        branch, err = s.cloneRepository(ctx, repoURL, tmpDir)
        if err != nil {
            return fmt.Errorf("failed to acquire repository: %w", err)
        }
        method = "clone"
    }

    // NEW: Override branch if specified in URL
    if urlInfo.branch != "" {
        branch = urlInfo.branch
    }

    s.logger.Info().
        Str("method", method).
        Str("branch", branch).
        Msg("Repository acquired successfully")

    // MODIFIED: Pass filterPath to findDocumentationFiles
    files, err := s.findDocumentationFiles(tmpDir, filterPath)
    if err != nil {
        return err
    }

    // NEW: Check if filter resulted in no files
    if len(files) == 0 && filterPath != "" {
        return fmt.Errorf("no documentation files found under path: %s", filterPath)
    }

    s.logger.Info().Int("count", len(files)).Msg("Found documentation files")

    // Apply limit
    if opts.Limit > 0 && len(files) > opts.Limit {
        files = files[:opts.Limit]
    }

    // MODIFIED: Use clean repoURL for file URLs
    return s.processFiles(ctx, files, tmpDir, repoURL, branch, opts)
}
```

---

### Phase 4: File Discovery Modification

#### 4.1 Modify `findDocumentationFiles()`

**File:** `internal/strategies/git.go`

**Change:** Add `filterPath` parameter

**Before (lines 388-414):**
```go
func (s *GitStrategy) findDocumentationFiles(dir string) ([]string, error) {
    var files []string
    err := filepath.WalkDir(dir, func(path string, d fs.DirEntry, err error) error {
        // ... existing logic
    })
    return files, err
}
```

**After:**
```go
// findDocumentationFiles walks the directory and finds all documentation files
// If filterPath is set, only files under that subdirectory are included
func (s *GitStrategy) findDocumentationFiles(dir string, filterPath string) ([]string, error) {
    var files []string
    
    // Determine the directory to walk
    walkDir := dir
    if filterPath != "" {
        walkDir = filepath.Join(dir, filterPath)
        
        // Verify the filter directory exists
        info, err := os.Stat(walkDir)
        if err != nil {
            if os.IsNotExist(err) {
                return nil, fmt.Errorf("filter path does not exist in repository: %s", filterPath)
            }
            return nil, fmt.Errorf("failed to access filter path: %w", err)
        }
        if !info.IsDir() {
            return nil, fmt.Errorf("filter path is not a directory: %s", filterPath)
        }
        
        s.logger.Debug().
            Str("filter_path", filterPath).
            Str("walk_dir", walkDir).
            Msg("Walking filtered directory")
    }
    
    err := filepath.WalkDir(walkDir, func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }

        // Skip ignored directories
        if d.IsDir() {
            if IgnoreDirs[d.Name()] {
                return fs.SkipDir
            }
            return nil
        }

        // Check file extension
        ext := strings.ToLower(filepath.Ext(path))
        if DocumentExtensions[ext] {
            files = append(files, path)
        }

        return nil
    })

    return files, err
}
```

---

### Phase 5: processFile() URL Generation Fix

#### 5.1 Update `processFile()` for Correct URLs

**File:** `internal/strategies/git.go`

**Issue:** When using a filter path, the generated file URLs should reflect the actual path in the repository.

**Current (line 460):**
```go
fileURL := repoURL + "/blob/" + branch + "/" + relPathURL
```

**Analysis:** This should work correctly because:
- `relPath` is calculated relative to `tmpDir` (the repo root)
- Even when walking a subdirectory, `filepath.Rel(tmpDir, path)` gives the full path from repo root

**Verification needed:** Ensure `relPath` calculation works correctly with filtered walks.

---

### Phase 6: Edge Cases & Error Handling

#### 6.1 Edge Cases to Handle

| Case | Handling |
|------|----------|
| Non-existent filter path | Return descriptive error: "filter path does not exist in repository: X" |
| Filter path is a file, not directory | Return error: "filter path is not a directory: X" |
| No .md files under filter path | Return error: "no documentation files found under path: X" |
| Trailing slashes in path | Normalize: `strings.TrimSuffix(path, "/")` |
| URL-encoded paths | Decode: `url.PathUnescape()` |
| Windows backslashes | Convert: `strings.ReplaceAll(path, "\\", "/")` |
| Empty filter path from URL | Treat as no filter (process all) |
| Both URL path and --filter flag | URL path takes precedence |

#### 6.2 Path Normalization Function

**New helper function:**

```go
// normalizeFilterPath cleans and validates a filter path
func normalizeFilterPath(path string) string {
    if path == "" {
        return ""
    }
    
    // URL decode if needed
    decoded, err := url.PathUnescape(path)
    if err == nil {
        path = decoded
    }
    
    // Normalize separators
    path = strings.ReplaceAll(path, "\\", "/")
    
    // Remove leading/trailing slashes
    path = strings.Trim(path, "/")
    
    // Clean the path (resolve .., remove double slashes)
    path = filepath.Clean(path)
    
    return path
}
```

---

### Phase 7: Testing Strategy

#### 7.1 Unit Tests

**File:** `internal/strategies/git_strategy_test.go`

| Test Function | Description |
|---------------|-------------|
| `TestParseGitURLWithPath_GitHub` | Various GitHub URL formats |
| `TestParseGitURLWithPath_GitLab` | GitLab URL formats |
| `TestParseGitURLWithPath_Bitbucket` | Bitbucket URL formats |
| `TestParseGitURLWithPath_NoPath` | URLs without /tree/ segment |
| `TestParseGitURLWithPath_InvalidURL` | Error handling for invalid URLs |
| `TestFindDocumentationFiles_WithFilter` | Filtering works correctly |
| `TestFindDocumentationFiles_NonExistentPath` | Error on invalid filter path |
| `TestFindDocumentationFiles_FileNotDir` | Error when path points to file |
| `TestNormalizeFilterPath` | Path normalization cases |
| `TestCanHandle_WithTreeURL` | Accepts /tree/ URLs now |
| `TestCanHandle_RejectsBlobURL` | Still rejects /blob/ URLs |

#### 7.2 Integration Tests

**File:** `tests/integration/strategies/git_filter_test.go`

| Test | Description |
|------|-------------|
| `TestGitStrategy_FilterByURLPath` | End-to-end with /tree/ URL |
| `TestGitStrategy_FilterByFlag` | End-to-end with --filter flag |
| `TestGitStrategy_FilterPrecedence` | URL path overrides --filter |
| `TestGitStrategy_NoFilesInPath` | Proper error when no docs found |

#### 7.3 E2E Tests

**File:** `tests/e2e/git_filter_test.go`

```go
func TestE2E_GitPathFilter(t *testing.T) {
    // Test with a real public repository
    // Use a small repo with known structure
}
```

---

### Phase 8: Documentation Updates

#### 8.1 Update README.md

Add section under "Git Repository Extraction":

```markdown
### Filtering to Specific Directories

Extract documentation from only a specific directory within a repository:

```bash
# Using URL path (recommended)
repodocs https://github.com/owner/repo/tree/main/docs

# Using --filter flag
repodocs https://github.com/owner/repo --filter docs

# Nested paths work too
repodocs https://github.com/owner/repo/tree/main/docs/api/v2
```
```

#### 8.2 Update CLI Help

**File:** `cmd/repodocs/main.go`

Update `--filter` flag description:

```go
rootCmd.PersistentFlags().String("filter", "", 
    "Path filter - for web: base URL filter; for git: subdirectory to process")
```

---

## Implementation Order

```
Phase 1: URL Parsing (foundation)
    │
    ├── 1.1 Add gitURLInfo struct
    ├── 1.2 Implement parseGitURLWithPath()
    └── 1.3 Add unit tests
           │
Phase 2: Detection Updates
    │
    ├── 2.1 Modify detector.go
    └── 2.2 Modify CanHandle()
           │
Phase 3: Execute() Flow
    │
    └── 3.1 Modify Execute() to use filter
           │
Phase 4: File Discovery
    │
    └── 4.1 Modify findDocumentationFiles()
           │
Phase 5: Edge Cases
    │
    ├── 5.1 Add normalizeFilterPath()
    └── 5.2 Error handling
           │
Phase 6: Testing
    │
    ├── 6.1 Unit tests
    ├── 6.2 Integration tests
    └── 6.3 E2E tests
           │
Phase 7: Documentation
    │
    ├── 7.1 Update README
    └── 7.2 Update CLI help
```

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Branch names with slashes confuse parser | Medium | Document limitation; users can use --filter |
| Breaking existing Git URL handling | High | Comprehensive tests; only remove /tree/ exclusion |
| Performance impact on large repos | Low | Filter is applied during walk, not after |
| Platform-specific URL format changes | Medium | Regex patterns are configurable |

---

## Success Criteria

- [ ] URLs with `/tree/` are routed to Git strategy
- [ ] `--filter docs` processes only `docs/**/*.md` files
- [ ] `/tree/main/docs` extracts the same as `--filter docs`
- [ ] Non-existent paths return clear error message
- [ ] All existing Git tests still pass
- [ ] New unit tests cover all edge cases
- [ ] Integration test with real repository passes

---

## Estimated Effort

| Phase | Effort |
|-------|--------|
| Phase 1: URL Parsing | 2 hours |
| Phase 2: Detection | 30 min |
| Phase 3: Execute Flow | 1 hour |
| Phase 4: File Discovery | 1 hour |
| Phase 5: Edge Cases | 1 hour |
| Phase 6: Testing | 2 hours |
| Phase 7: Documentation | 30 min |
| **Total** | **~8 hours** |
