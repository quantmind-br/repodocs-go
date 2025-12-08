# Development Tasks: repodocs-go Refactoring

Based on [PLAN.md](./PLAN.md), executing 4 validated refactoring tasks.

---

## Task 1: Remove setDefaultsIfNotSet wrapper
**File**: `internal/config/loader.go`
**Risk**: Low

### Steps
- [ ] Read current `loader.go` to understand context
- [ ] Replace `setDefaultsIfNotSet(v)` with `setDefaults(v)` in `Load()`
- [ ] Remove `setDefaultsIfNotSet` function
- [ ] Run tests: `go test ./internal/config/...`

---

## Task 2: Remove dead SanitizeFilename
**File**: `internal/converter/sanitizer.go`
**Risk**: Low

### Steps
- [ ] Verify `SanitizeFilename` has no references
- [ ] Remove `SanitizeFilename` function from `sanitizer.go`
- [ ] Run tests: `go test ./internal/converter/...`

---

## Task 3: Remove crawler string helpers
**File**: `internal/strategies/crawler.go`
**Risk**: Low

### Steps
- [ ] Read current helper functions
- [ ] Refactor `isHTMLContentType` to use `strings.Contains`/`strings.ToLower`
- [ ] Remove: `contains`, `containsCaseSensitive`, `containsLower`, `lower`
- [ ] Run tests: `go test ./internal/strategies/...`

---

## Task 4: Consolidate Frontmatter struct
**Files**: `internal/converter/markdown.go`, `internal/domain/models.go`
**Risk**: Low

### Steps
- [ ] Verify `domain.Frontmatter` and `Document.ToFrontmatter()` exist
- [ ] Remove duplicate `Frontmatter` struct from `converter/markdown.go`
- [ ] Update `GenerateFrontmatter` to use `doc.ToFrontmatter()`
- [ ] Add import for `domain` package if needed
- [ ] Run tests: `go test ./internal/converter/... ./internal/domain/...`

---

## Final Validation

After all tasks:
```bash
make test      # All tests pass
make lint      # No linting errors
make build     # Successful build
```
