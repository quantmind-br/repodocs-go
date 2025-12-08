# Refactoring Plan: repodocs-go Optimization

## Analysis Summary

After analyzing the suggestions in `otimização.md` and validating against the actual codebase, I found that **4 out of 6 suggestions are valid and worth implementing**. Two suggestions were rejected after deeper analysis.

---

## Validated Refactoring Tasks

### Task 1: Remove Redundant String Helpers (crawler.go)

**Status**: VALID
**Priority**: High (simple, low-risk)
**Files**: `internal/strategies/crawler.go`

**Problem**: Four functions reimplementing standard library functionality:
- `contains(s, substr string) bool` → reimplements `strings.Contains`
- `containsCaseSensitive(s, substr string) bool` → reimplements `strings.Contains`
- `containsLower(s, substr string) bool` → same as above with pre-lowercased input
- `lower(s string) string` → reimplements `strings.ToLower`

**Solution**:
1. Replace `isHTMLContentType` implementation with:
   ```go
   func isHTMLContentType(contentType string) bool {
       if contentType == "" {
           return true
       }
       lower := strings.ToLower(contentType)
       return strings.Contains(lower, "text/html") ||
           strings.Contains(lower, "application/xhtml")
   }
   ```
2. Remove functions: `contains`, `containsCaseSensitive`, `containsLower`, `lower`

---

### Task 2: Remove Duplicate Frontmatter Struct (converter/markdown.go)

**Status**: VALID
**Priority**: High (DRY principle)
**Files**: `internal/converter/markdown.go`, `internal/domain/models.go`

**Problem**: Identical `Frontmatter` struct defined in two places:
- `internal/domain/models.go:101-108` (canonical location)
- `internal/converter/markdown.go:70-77` (duplicate)

Additionally, `Document.ToFrontmatter()` method exists in domain but is **unused**.

**Solution**:
1. Remove `Frontmatter` struct from `internal/converter/markdown.go`
2. Update `GenerateFrontmatter` to use `domain.Frontmatter` and `doc.ToFrontmatter()`:
   ```go
   func GenerateFrontmatter(doc *domain.Document) (string, error) {
       fm := doc.ToFrontmatter()
       data, err := yaml.Marshal(fm)
       if err != nil {
           return "", err
       }
       return fmt.Sprintf("---\n%s---\n\n", string(data)), nil
   }
   ```

---

### Task 3: Remove Redundant setDefaultsIfNotSet (config/loader.go)

**Status**: VALID
**Priority**: Medium (dead code)
**Files**: `internal/config/loader.go`

**Problem**: Function `setDefaultsIfNotSet` is a completely redundant wrapper:
```go
func setDefaultsIfNotSet(v *viper.Viper) {
    setDefaults(v)  // Just calls setDefaults!
}
```

**Solution**:
1. In `Load()` function (line 16), replace `setDefaultsIfNotSet(v)` with `setDefaults(v)`
2. Remove `setDefaultsIfNotSet` function entirely

---

### Task 4: Remove Dead SanitizeFilename (converter/sanitizer.go)

**Status**: VALID
**Priority**: High (dead code)
**Files**: `internal/converter/sanitizer.go`

**Problem**:
- `converter.SanitizeFilename` has **zero references** (dead code)
- `utils.SanitizeFilename` is the active, more robust version (handles Windows reserved names, preserves extensions on truncation)

**Solution**:
1. Remove `SanitizeFilename` function from `internal/converter/sanitizer.go`

---

## Rejected Suggestions

### Rejected 1: Flag/Configuration Consolidation (cmd/repodocs/main.go)

**Status**: REJECTED
**Reason**: After analysis, the current design is intentional and correct.

The suggestion proposed merging CLI flags into `config.Config`. However:
- Flags like `--dry-run`, `--limit`, `--force` are **transient runtime options**
- They should NOT be persisted in configuration files
- The separation between `config.Config` (persistent settings) and `OrchestratorOptions` (runtime flags) is architecturally sound

The current pattern of reading CLI flags separately and passing them via `OrchestratorOptions` is the correct approach for distinguishing between:
1. **Persistent config** (viper-managed): output directory, concurrency, cache TTL
2. **Runtime flags** (CLI-only): dry-run, limit, force, filter

---

### Rejected 2: Remove utils.Is*URL Functions

**Status**: PARTIALLY REJECTED
**Reason**: Functions serve testing purposes and don't hurt production code.

The `utils.Is*URL` functions (IsGitURL, IsSitemapURL, IsLLMSURL, IsPkgGoDevURL) are only used in tests. While they duplicate logic from `DetectStrategy`, they:
- Provide cleaner test assertions
- Don't impact production code size
- Serve as utility functions for potential external use

**Recommendation**: Keep them but consider adding a comment noting they're primarily for testing. If the package is refactored later, these could be moved to a `_test.go` file.

---

## Implementation Order

Execute tasks in this order to minimize conflicts:

1. **Task 3**: Remove `setDefaultsIfNotSet` (smallest, isolated change)
2. **Task 4**: Remove dead `converter.SanitizeFilename`
3. **Task 1**: Remove crawler string helpers
4. **Task 2**: Consolidate Frontmatter (requires import changes)

## Testing Strategy

After each task:
```bash
make test          # Ensure no regressions
make lint          # Check for unused imports, etc.
make build         # Verify compilation
```

## Estimated Impact

- **Lines removed**: ~60-70 lines of dead/redundant code
- **Risk level**: Low (removing unused code, consolidating duplicates)
- **Test coverage**: No negative impact (removing dead code)
