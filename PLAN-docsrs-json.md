# PLAN: DocsRS JSON Strategy - Complete Pipeline Replacement

## Executive Summary

Replace the current HTML crawling-based DocsRS strategy with a JSON-based approach that downloads and parses the structured Rustdoc JSON output from docs.rs. This eliminates the need for BFS crawling, HTML parsing, and Markdown conversion, resulting in:

- **100x fewer HTTP requests** (1 JSON file vs 100+ HTML pages)
- **10x smaller data transfer** (762KB JSON vs ~131MB HTML)
- **No rate limiting concerns** (single request)
- **Structured type information** (function signatures, generics, bounds)
- **Pre-formatted Markdown** (docs are already in Markdown)
- **Complete documentation** (all items in one file)

---

## Part 1: Analysis of Current Implementation

### 1.1 Current Architecture (`docsrs.go`)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CURRENT PIPELINE (HTML Crawling)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  URL Input ──► parseURL() ──► discoverPages() ──► processPage()     │
│                                   (BFS)            (per page)        │
│                                    │                   │             │
│                                    ▼                   ▼             │
│                            HTTP GET x100+      HTML ─► Markdown      │
│                            (with delays)        (goquery + converter)│
│                                    │                   │             │
│                                    ▼                   ▼             │
│                            Rate Limited         ~131MB transferred   │
│                            500-1500ms/req                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Current Problems

| Problem | Impact | Severity |
|---------|--------|----------|
| BFS crawling requires 100+ HTTP requests | Slow extraction (5-10 minutes) | High |
| Random delays (500-1500ms) to avoid rate limits | Adds 50-150 seconds total | High |
| HTML parsing with goquery is fragile | Breaks when docs.rs changes layout | Medium |
| HTML→Markdown conversion loses information | Type signatures poorly formatted | Medium |
| No structured type information | Cannot generate type-aware docs | Medium |
| 131MB+ data transfer | Bandwidth waste | Low |

### 1.3 Current Dependencies (to be removed/replaced)

```go
// REMOVE these imports from new implementation:
"github.com/PuerkitoBio/goquery"  // No longer needed - JSON is structured
"math/rand"                        // No longer needed - no rate limiting delays

// KEEP these imports:
"github.com/quantmind-br/repodocs-go/internal/domain"
"github.com/quantmind-br/repodocs-go/internal/output"
"github.com/quantmind-br/repodocs-go/internal/utils"

// ADD these imports:
"github.com/klauspost/compress/zstd"  // For zstd decompression
"encoding/json"                        // For JSON parsing
```

---

## Part 2: Rustdoc JSON Format Analysis

### 2.1 Endpoint Structure

```
GET https://docs.rs/crate/{name}/{version}/json
    → Redirects to: https://static.docs.rs/{name}_{version}_{target}_latest.json.zst

Compression: zstd (can also request .gz with ?format=gz)
Content-Type: application/zstd
```

### 2.2 JSON Top-Level Structure

```json
{
  "root": 214,                    // ID of the crate root module
  "crate_version": "0.30.0",      // Crate version
  "format_version": 57,           // Rustdoc JSON format version (important!)
  "includes_private": false,      // Whether private items are included
  "index": { ... },               // Map of item ID → Item
  "paths": { ... },               // Map of item ID → path info for external items
  "external_crates": { ... },     // Map of crate ID → external crate info
  "target": { ... }               // Target platform info (optional)
}
```

### 2.3 Item Structure (in `index`)

```json
{
  "id": 214,                      // Unique item ID (integer or string)
  "crate_id": 0,                  // 0 = this crate, >0 = external crate
  "name": "ratatui",              // Item name (null for re-exports without rename)
  "span": {                       // Source location
    "filename": "src/lib.rs",
    "begin": [1, 1],              // [line, column]
    "end": [468, 14]
  },
  "visibility": "public",         // "public", "default", "crate", "restricted"
  "docs": "...",                  // Documentation in MARKDOWN (already formatted!)
  "links": {                      // Cross-references: name → item ID
    "`Terminal`": 94,
    "widgets": 103
  },
  "attrs": [...],                 // Attributes like #[cfg(...)]
  "deprecation": null,            // Deprecation info if deprecated
  "inner": { ... }                // Type-specific data (see 2.4)
}
```

### 2.4 Item Inner Types

| Type | `inner` Key | Description |
|------|-------------|-------------|
| Module | `module` | Contains `items: []int`, `is_crate: bool` |
| Struct | `struct` | Contains `kind`, `generics`, `impls` |
| Enum | `enum` | Contains `variants`, `generics`, `impls` |
| Trait | `trait` | Contains `items`, `generics`, `bounds`, `implementations` |
| Function | `function` | Contains `sig`, `generics`, `header`, `has_body` |
| Method | `function` | Same as function (context determines if method) |
| Type Alias | `type_alias` | Contains `type`, `generics` |
| Constant | `constant` | Contains `type`, `const_` |
| Static | `static` | Contains `type`, `mutable`, `expr` |
| Impl | `impl` | Contains `trait`, `for`, `items`, `generics` |
| Use/Re-export | `use` | Contains `source`, `name`, `id`, `is_glob` |
| Macro | `macro` | Contains `macro` (the macro definition) |
| AssocType | `assoc_type` | Associated type in trait |
| AssocConst | `assoc_const` | Associated constant in trait |

### 2.5 Type Representation

Types are represented as nested JSON objects:

```json
// Primitive type
{"primitive": "str"}

// Generic parameter
{"generic": "Self"}

// Resolved path (reference to another type)
{
  "resolved_path": {
    "path": "Rect",
    "id": 60,
    "args": null
  }
}

// Borrowed reference
{
  "borrowed_ref": {
    "lifetime": "'a",     // or null
    "is_mutable": true,
    "type": { ... }       // nested type
  }
}

// Slice
{"slice": { ... }}

// Array
{"array": {"type": {...}, "len": "10"}}

// Tuple
{"tuple": [type1, type2, ...]}

// Raw pointer
{"raw_pointer": {"is_mutable": true, "type": {...}}}

// Function pointer
{"function_pointer": {...}}

// Qualified path (e.g., <T as Trait>::Type)
{"qualified_path": {...}}

// DynTrait
{"dyn_trait": {...}}

// Impl trait
{"impl_trait": [...bounds...]}
```

### 2.6 Function Signature Structure

```json
{
  "function": {
    "sig": {
      "inputs": [
        ["self", {"borrowed_ref": {"is_mutable": false, "type": {"generic": "Self"}}}],
        ["area", {"resolved_path": {"path": "Rect", "id": 60}}],
        ["buf", {"borrowed_ref": {"is_mutable": true, "type": {"resolved_path": {"path": "Buffer", "id": 42}}}}]
      ],
      "output": {"resolved_path": {"path": "Result", "id": 100}},
      "is_c_variadic": false
    },
    "generics": {
      "params": [...],
      "where_predicates": [...]
    },
    "header": {
      "is_const": false,
      "is_unsafe": false,
      "is_async": false,
      "abi": "Rust"
    },
    "has_body": true
  }
}
```

---

## Part 3: New Architecture Design

### 3.1 New Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NEW PIPELINE (JSON-based)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  URL Input ──► parseURL() ──► fetchJSON() ──► parseIndex() ──►      │
│                                 (1 req)        (in-memory)           │
│                                    │                │                │
│                                    ▼                ▼                │
│                              zstd decompress   Build item tree       │
│                              (~762KB → ~5MB)        │                │
│                                                     ▼                │
│                                             generateDocs()           │
│                                             (parallel, no HTTP)      │
│                                                     │                │
│                                                     ▼                │
│                                             Write Markdown files     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         DocsRSStrategy (new)                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐                │
│  │ JSONFetcher │───►│ JSONParser   │───►│ DocGenerator  │                │
│  │             │    │              │    │               │                │
│  │ - Download  │    │ - Unmarshal  │    │ - Render MD   │                │
│  │ - Decompress│    │ - Build tree │    │ - Type sigs   │                │
│  │ - Cache     │    │ - Resolve    │    │ - Cross-refs  │                │
│  └─────────────┘    └──────────────┘    └───────────────┘                │
│         │                  │                    │                         │
│         ▼                  ▼                    ▼                         │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                        RustdocJSON Types                         │     │
│  │                                                                  │     │
│  │  RustdocIndex, Item, ItemInner, Type, Generics, FunctionSig...  │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 File Structure

```
internal/strategies/
├── docsrs.go                    # Main strategy (REPLACE completely)
├── docsrs_json.go               # NEW: JSON types and parsing
├── docsrs_renderer.go           # NEW: Markdown rendering from JSON
├── docsrs_types.go              # NEW: Go types for Rustdoc JSON
├── docsrs_test.go               # Tests (REPLACE completely)
└── docsrs_testdata/             # NEW: Test fixtures
    ├── ratatui_sample.json      # Sample JSON for testing
    └── expected_output/         # Golden files for output comparison
```

---

## Part 4: Implementation Details

### 4.1 Go Types for Rustdoc JSON (`docsrs_types.go`)

```go
package strategies

// RustdocIndex represents the top-level rustdoc JSON structure
type RustdocIndex struct {
    Root           int                       `json:"root"`
    CrateVersion   string                    `json:"crate_version"`
    FormatVersion  int                       `json:"format_version"`
    IncludesPrivate bool                     `json:"includes_private"`
    Index          map[string]*RustdocItem   `json:"index"`
    Paths          map[string]*RustdocPath   `json:"paths"`
    ExternalCrates map[string]*ExternalCrate `json:"external_crates"`
}

// RustdocItem represents a single item in the index
type RustdocItem struct {
    ID          int                    `json:"id"`
    CrateID     int                    `json:"crate_id"`
    Name        *string                `json:"name"`          // nullable
    Span        *RustdocSpan           `json:"span"`
    Visibility  string                 `json:"visibility"`
    Docs        *string                `json:"docs"`          // nullable, MARKDOWN!
    Links       map[string]int         `json:"links"`
    Attrs       []RustdocAttr          `json:"attrs"`
    Deprecation *RustdocDeprecation    `json:"deprecation"`
    Inner       RustdocItemInner       `json:"inner"`
}

// RustdocItemInner wraps the type-specific inner data
// Only one field will be non-nil at a time
type RustdocItemInner struct {
    Module     *RustdocModule     `json:"module,omitempty"`
    Struct     *RustdocStruct     `json:"struct,omitempty"`
    Enum       *RustdocEnum       `json:"enum,omitempty"`
    Trait      *RustdocTrait      `json:"trait,omitempty"`
    Function   *RustdocFunction   `json:"function,omitempty"`
    TypeAlias  *RustdocTypeAlias  `json:"type_alias,omitempty"`
    Constant   *RustdocConstant   `json:"constant,omitempty"`
    Static     *RustdocStatic     `json:"static,omitempty"`
    Impl       *RustdocImpl       `json:"impl,omitempty"`
    Use        *RustdocUse        `json:"use,omitempty"`
    Macro      *RustdocMacro      `json:"macro,omitempty"`
    AssocType  *RustdocAssocType  `json:"assoc_type,omitempty"`
    AssocConst *RustdocAssocConst `json:"assoc_const,omitempty"`
}

// RustdocSpan represents source code location
type RustdocSpan struct {
    Filename string `json:"filename"`
    Begin    [2]int `json:"begin"` // [line, column]
    End      [2]int `json:"end"`
}

// RustdocModule represents a module item
type RustdocModule struct {
    IsCrate    bool  `json:"is_crate"`
    Items      []int `json:"items"`
    IsStripped bool  `json:"is_stripped"`
}

// RustdocFunction represents a function/method
type RustdocFunction struct {
    Sig      RustdocFunctionSig `json:"sig"`
    Generics RustdocGenerics    `json:"generics"`
    Header   RustdocHeader      `json:"header"`
    HasBody  bool               `json:"has_body"`
}

// RustdocFunctionSig represents a function signature
type RustdocFunctionSig struct {
    Inputs      [][2]interface{} `json:"inputs"`  // [[name, type], ...]
    Output      *RustdocType     `json:"output"`  // nullable (void)
    IsVariadic  bool             `json:"is_c_variadic"`
}

// RustdocType represents a type (complex nested structure)
// This uses interface{} because types have many variants
type RustdocType map[string]interface{}

// RustdocGenerics represents generic parameters
type RustdocGenerics struct {
    Params          []RustdocGenericParam    `json:"params"`
    WherePredicates []RustdocWherePredicate  `json:"where_predicates"`
}

// RustdocHeader represents function header attributes
type RustdocHeader struct {
    IsConst  bool   `json:"is_const"`
    IsUnsafe bool   `json:"is_unsafe"`
    IsAsync  bool   `json:"is_async"`
    ABI      string `json:"abi"`
}

// RustdocTrait represents a trait
type RustdocTrait struct {
    IsAuto          bool             `json:"is_auto"`
    IsUnsafe        bool             `json:"is_unsafe"`
    IsDynCompatible bool             `json:"is_dyn_compatible"`
    Items           []int            `json:"items"`
    Generics        RustdocGenerics  `json:"generics"`
    Bounds          []interface{}    `json:"bounds"`
    Implementations []int            `json:"implementations"`
}

// RustdocStruct represents a struct
type RustdocStruct struct {
    Kind     interface{}     `json:"kind"`      // "unit", "tuple", or struct fields
    Generics RustdocGenerics `json:"generics"`
    Impls    []int           `json:"impls"`
}

// RustdocEnum represents an enum
type RustdocEnum struct {
    Variants     []int           `json:"variants"`
    Generics     RustdocGenerics `json:"generics"`
    Impls        []int           `json:"impls"`
    VariantsStripped bool        `json:"variants_stripped"`
}

// RustdocImpl represents an impl block
type RustdocImpl struct {
    IsUnsafe          bool            `json:"is_unsafe"`
    Generics          RustdocGenerics `json:"generics"`
    ProvidedMethods   []string        `json:"provided_trait_methods"`
    Trait             *RustdocPath    `json:"trait"`  // nullable (inherent impl)
    For               RustdocType     `json:"for"`
    Items             []int           `json:"items"`
    IsNegative        bool            `json:"is_negative"`
    IsSynthetic       bool            `json:"is_synthetic"`
    BlanketImpl       *RustdocType    `json:"blanket_impl"`
}

// RustdocUse represents a re-export (use statement)
type RustdocUse struct {
    Source string `json:"source"`
    Name   string `json:"name"`
    ID     *int   `json:"id"`     // nullable if external
    IsGlob bool   `json:"is_glob"`
}

// RustdocPath represents a path reference
type RustdocPath struct {
    Path string      `json:"path"`
    ID   int         `json:"id"`
    Args interface{} `json:"args"`
}

// RustdocAttr represents an attribute
type RustdocAttr struct {
    Other string `json:"other,omitempty"`
}

// RustdocDeprecation represents deprecation info
type RustdocDeprecation struct {
    Since string `json:"since"`
    Note  string `json:"note"`
}

// ExternalCrate represents an external crate reference
type ExternalCrate struct {
    Name        string `json:"name"`
    HTMLRootURL string `json:"html_root_url"`
}

// RustdocGenericParam represents a generic parameter
type RustdocGenericParam struct {
    Name  string      `json:"name"`
    Kind  interface{} `json:"kind"`
}

// RustdocWherePredicate represents a where clause predicate
type RustdocWherePredicate interface{}
```

### 4.2 JSON Fetcher (`docsrs_json.go`)

```go
package strategies

import (
    "context"
    "fmt"
    "io"
    "net/http"
    
    "github.com/klauspost/compress/zstd"
)

// DocsRSJSONEndpoint returns the JSON endpoint URL for a crate
func DocsRSJSONEndpoint(crateName, version string) string {
    return fmt.Sprintf("https://docs.rs/crate/%s/%s/json", crateName, version)
}

// fetchRustdocJSON downloads and decompresses the rustdoc JSON
func (s *DocsRSStrategy) fetchRustdocJSON(ctx context.Context, crateName, version string) (*RustdocIndex, error) {
    endpoint := DocsRSJSONEndpoint(crateName, version)
    
    s.logger.Info().
        Str("crate", crateName).
        Str("version", version).
        Str("endpoint", endpoint).
        Msg("Fetching rustdoc JSON")
    
    // Use the fetcher's underlying HTTP client for the request
    // The JSON endpoint returns zstd-compressed data
    resp, err := s.fetcher.Get(ctx, endpoint)
    if err != nil {
        return nil, fmt.Errorf("failed to fetch rustdoc JSON: %w", err)
    }
    
    // Check if response is compressed
    contentType := resp.Headers.Get("Content-Type")
    var jsonData []byte
    
    if contentType == "application/zstd" || contentType == "application/x-zstd" {
        // Decompress zstd
        decoder, err := zstd.NewReader(nil)
        if err != nil {
            return nil, fmt.Errorf("failed to create zstd decoder: %w", err)
        }
        defer decoder.Close()
        
        jsonData, err = decoder.DecodeAll(resp.Body, nil)
        if err != nil {
            return nil, fmt.Errorf("failed to decompress zstd: %w", err)
        }
    } else {
        // Assume uncompressed JSON
        jsonData = resp.Body
    }
    
    s.logger.Debug().
        Int("compressed_size", len(resp.Body)).
        Int("decompressed_size", len(jsonData)).
        Msg("Decompressed rustdoc JSON")
    
    // Parse JSON
    var index RustdocIndex
    if err := json.Unmarshal(jsonData, &index); err != nil {
        return nil, fmt.Errorf("failed to parse rustdoc JSON: %w", err)
    }
    
    s.logger.Info().
        Int("items", len(index.Index)).
        Int("format_version", index.FormatVersion).
        Msg("Parsed rustdoc JSON")
    
    return &index, nil
}
```

### 4.3 Markdown Renderer (`docsrs_renderer.go`)

```go
package strategies

import (
    "fmt"
    "strings"
)

// RustdocRenderer converts rustdoc JSON items to Markdown
type RustdocRenderer struct {
    index    *RustdocIndex
    crateName string
    version   string
}

// NewRustdocRenderer creates a new renderer
func NewRustdocRenderer(index *RustdocIndex, crateName, version string) *RustdocRenderer {
    return &RustdocRenderer{
        index:     index,
        crateName: crateName,
        version:   version,
    }
}

// RenderItem renders a single item to Markdown
func (r *RustdocRenderer) RenderItem(item *RustdocItem) string {
    var sb strings.Builder
    
    // Get item type and name
    itemType := r.getItemType(item)
    name := ""
    if item.Name != nil {
        name = *item.Name
    }
    
    // Title
    if name != "" {
        sb.WriteString(fmt.Sprintf("# %s `%s`\n\n", itemType, name))
    }
    
    // Deprecation warning
    if item.Deprecation != nil {
        sb.WriteString("> **Deprecated**")
        if item.Deprecation.Since != "" {
            sb.WriteString(fmt.Sprintf(" since %s", item.Deprecation.Since))
        }
        if item.Deprecation.Note != "" {
            sb.WriteString(fmt.Sprintf(": %s", item.Deprecation.Note))
        }
        sb.WriteString("\n\n")
    }
    
    // Render type signature based on item type
    sig := r.renderSignature(item)
    if sig != "" {
        sb.WriteString("```rust\n")
        sb.WriteString(sig)
        sb.WriteString("\n```\n\n")
    }
    
    // Documentation (already in Markdown!)
    if item.Docs != nil && *item.Docs != "" {
        // Resolve cross-references in docs
        docs := r.resolveCrossRefs(*item.Docs, item.Links)
        sb.WriteString(docs)
        sb.WriteString("\n\n")
    }
    
    // Render children for modules
    if item.Inner.Module != nil {
        sb.WriteString(r.renderModuleContents(item))
    }
    
    // Render trait items
    if item.Inner.Trait != nil {
        sb.WriteString(r.renderTraitContents(item))
    }
    
    // Render struct/enum impls
    if item.Inner.Struct != nil || item.Inner.Enum != nil {
        sb.WriteString(r.renderImplContents(item))
    }
    
    return sb.String()
}

// getItemType returns the human-readable item type
func (r *RustdocRenderer) getItemType(item *RustdocItem) string {
    switch {
    case item.Inner.Module != nil:
        if item.Inner.Module.IsCrate {
            return "Crate"
        }
        return "Module"
    case item.Inner.Struct != nil:
        return "Struct"
    case item.Inner.Enum != nil:
        return "Enum"
    case item.Inner.Trait != nil:
        return "Trait"
    case item.Inner.Function != nil:
        return "Function"
    case item.Inner.TypeAlias != nil:
        return "Type Alias"
    case item.Inner.Constant != nil:
        return "Constant"
    case item.Inner.Static != nil:
        return "Static"
    case item.Inner.Macro != nil:
        return "Macro"
    case item.Inner.Use != nil:
        return "Re-export"
    default:
        return "Item"
    }
}

// renderSignature renders the type signature for an item
func (r *RustdocRenderer) renderSignature(item *RustdocItem) string {
    switch {
    case item.Inner.Function != nil:
        return r.renderFunctionSignature(item)
    case item.Inner.Trait != nil:
        return r.renderTraitSignature(item)
    case item.Inner.Struct != nil:
        return r.renderStructSignature(item)
    case item.Inner.Enum != nil:
        return r.renderEnumSignature(item)
    case item.Inner.TypeAlias != nil:
        return r.renderTypeAliasSignature(item)
    case item.Inner.Constant != nil:
        return r.renderConstantSignature(item)
    default:
        return ""
    }
}

// renderFunctionSignature renders a function signature
func (r *RustdocRenderer) renderFunctionSignature(item *RustdocItem) string {
    fn := item.Inner.Function
    if fn == nil {
        return ""
    }
    
    var sb strings.Builder
    
    // Header (pub, async, unsafe, const)
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    if fn.Header.IsConst {
        sb.WriteString("const ")
    }
    if fn.Header.IsAsync {
        sb.WriteString("async ")
    }
    if fn.Header.IsUnsafe {
        sb.WriteString("unsafe ")
    }
    
    sb.WriteString("fn ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    
    // Generics
    sb.WriteString(r.renderGenerics(&fn.Generics))
    
    // Parameters
    sb.WriteString("(")
    for i, input := range fn.Sig.Inputs {
        if i > 0 {
            sb.WriteString(", ")
        }
        // input is [name, type]
        if len(input) >= 2 {
            name := fmt.Sprintf("%v", input[0])
            typeStr := r.renderType(input[1])
            if name == "self" {
                sb.WriteString(typeStr) // self types are rendered fully
            } else {
                sb.WriteString(fmt.Sprintf("%s: %s", name, typeStr))
            }
        }
    }
    sb.WriteString(")")
    
    // Return type
    if fn.Sig.Output != nil {
        sb.WriteString(" -> ")
        sb.WriteString(r.renderTypeValue(fn.Sig.Output))
    }
    
    // Where clauses
    sb.WriteString(r.renderWhereClauses(&fn.Generics))
    
    return sb.String()
}

// renderType renders a type from the JSON representation
func (r *RustdocRenderer) renderType(t interface{}) string {
    if t == nil {
        return "()"
    }
    
    switch v := t.(type) {
    case map[string]interface{}:
        return r.renderTypeMap(v)
    case string:
        return v
    default:
        return fmt.Sprintf("%v", v)
    }
}

// renderTypeMap renders a type from a map representation
func (r *RustdocRenderer) renderTypeMap(t map[string]interface{}) string {
    // Check each type variant
    if prim, ok := t["primitive"]; ok {
        return fmt.Sprintf("%v", prim)
    }
    
    if gen, ok := t["generic"]; ok {
        return fmt.Sprintf("%v", gen)
    }
    
    if resolved, ok := t["resolved_path"].(map[string]interface{}); ok {
        path := fmt.Sprintf("%v", resolved["path"])
        // TODO: render generic args
        return path
    }
    
    if borrowed, ok := t["borrowed_ref"].(map[string]interface{}); ok {
        mut := ""
        if borrowed["is_mutable"] == true {
            mut = "mut "
        }
        lifetime := ""
        if l, ok := borrowed["lifetime"].(string); ok && l != "" {
            lifetime = l + " "
        }
        inner := r.renderType(borrowed["type"])
        return fmt.Sprintf("&%s%s%s", lifetime, mut, inner)
    }
    
    if slice, ok := t["slice"]; ok {
        return fmt.Sprintf("[%s]", r.renderType(slice))
    }
    
    if tuple, ok := t["tuple"].([]interface{}); ok {
        parts := make([]string, len(tuple))
        for i, elem := range tuple {
            parts[i] = r.renderType(elem)
        }
        return fmt.Sprintf("(%s)", strings.Join(parts, ", "))
    }
    
    // Add more type variants as needed...
    return "..."
}

// renderTypeValue renders a RustdocType value
func (r *RustdocRenderer) renderTypeValue(t *RustdocType) string {
    if t == nil {
        return "()"
    }
    return r.renderTypeMap(map[string]interface{}(*t))
}

// renderGenerics renders generic parameters
func (r *RustdocRenderer) renderGenerics(g *RustdocGenerics) string {
    if len(g.Params) == 0 {
        return ""
    }
    
    parts := make([]string, len(g.Params))
    for i, p := range g.Params {
        parts[i] = p.Name
        // TODO: render bounds
    }
    
    return fmt.Sprintf("<%s>", strings.Join(parts, ", "))
}

// renderWhereClauses renders where clauses
func (r *RustdocRenderer) renderWhereClauses(g *RustdocGenerics) string {
    if len(g.WherePredicates) == 0 {
        return ""
    }
    
    // TODO: Implement where clause rendering
    return ""
}

// resolveCrossRefs resolves cross-references in documentation
func (r *RustdocRenderer) resolveCrossRefs(docs string, links map[string]int) string {
    // Replace `[name]` references with proper links
    result := docs
    for name, id := range links {
        // Look up the target item to get its path
        if targetItem, ok := r.index.Index[fmt.Sprintf("%d", id)]; ok {
            targetName := ""
            if targetItem.Name != nil {
                targetName = *targetItem.Name
            }
            // Convert to docs.rs URL
            targetURL := fmt.Sprintf("https://docs.rs/%s/%s/%s/%s",
                r.crateName, r.version, r.crateName, targetName)
            
            // Replace the reference
            result = strings.ReplaceAll(result, 
                fmt.Sprintf("[%s]", name),
                fmt.Sprintf("[%s](%s)", name, targetURL))
        }
    }
    
    return result
}

// renderModuleContents renders the contents of a module
func (r *RustdocRenderer) renderModuleContents(item *RustdocItem) string {
    mod := item.Inner.Module
    if mod == nil || len(mod.Items) == 0 {
        return ""
    }
    
    var sb strings.Builder
    sb.WriteString("## Contents\n\n")
    
    // Group items by type
    groups := make(map[string][]*RustdocItem)
    for _, childID := range mod.Items {
        if child, ok := r.index.Index[fmt.Sprintf("%d", childID)]; ok {
            itemType := r.getItemType(child)
            groups[itemType] = append(groups[itemType], child)
        }
    }
    
    // Render each group
    order := []string{"Module", "Struct", "Enum", "Trait", "Function", "Type Alias", "Constant", "Macro"}
    for _, itemType := range order {
        if items, ok := groups[itemType]; ok && len(items) > 0 {
            sb.WriteString(fmt.Sprintf("### %ss\n\n", itemType))
            for _, child := range items {
                if child.Name != nil {
                    sb.WriteString(fmt.Sprintf("- [`%s`](#%s)\n", *child.Name, strings.ToLower(*child.Name)))
                }
            }
            sb.WriteString("\n")
        }
    }
    
    return sb.String()
}

// renderTraitContents renders trait items (methods, associated types)
func (r *RustdocRenderer) renderTraitContents(item *RustdocItem) string {
    trait := item.Inner.Trait
    if trait == nil || len(trait.Items) == 0 {
        return ""
    }
    
    var sb strings.Builder
    sb.WriteString("## Required Methods\n\n")
    
    for _, childID := range trait.Items {
        if child, ok := r.index.Index[fmt.Sprintf("%d", childID)]; ok {
            if child.Inner.Function != nil {
                sb.WriteString(fmt.Sprintf("### `%s`\n\n", *child.Name))
                sb.WriteString("```rust\n")
                sb.WriteString(r.renderFunctionSignature(child))
                sb.WriteString("\n```\n\n")
                if child.Docs != nil {
                    sb.WriteString(*child.Docs)
                    sb.WriteString("\n\n")
                }
            }
        }
    }
    
    return sb.String()
}

// renderImplContents renders impl block items
func (r *RustdocRenderer) renderImplContents(item *RustdocItem) string {
    var impls []int
    if item.Inner.Struct != nil {
        impls = item.Inner.Struct.Impls
    } else if item.Inner.Enum != nil {
        impls = item.Inner.Enum.Impls
    }
    
    if len(impls) == 0 {
        return ""
    }
    
    var sb strings.Builder
    sb.WriteString("## Implementations\n\n")
    
    for _, implID := range impls {
        if impl, ok := r.index.Index[fmt.Sprintf("%d", implID)]; ok {
            if impl.Inner.Impl != nil {
                // Render impl header
                if impl.Inner.Impl.Trait != nil {
                    sb.WriteString(fmt.Sprintf("### impl %s\n\n", impl.Inner.Impl.Trait.Path))
                } else {
                    sb.WriteString("### impl\n\n")
                }
                
                // Render impl methods
                for _, methodID := range impl.Inner.Impl.Items {
                    if method, ok := r.index.Index[fmt.Sprintf("%d", methodID)]; ok {
                        if method.Name != nil {
                            sb.WriteString(fmt.Sprintf("#### `%s`\n\n", *method.Name))
                            if method.Inner.Function != nil {
                                sb.WriteString("```rust\n")
                                sb.WriteString(r.renderFunctionSignature(method))
                                sb.WriteString("\n```\n\n")
                            }
                            if method.Docs != nil {
                                sb.WriteString(*method.Docs)
                                sb.WriteString("\n\n")
                            }
                        }
                    }
                }
            }
        }
    }
    
    return sb.String()
}

// Additional signature renderers...

func (r *RustdocRenderer) renderTraitSignature(item *RustdocItem) string {
    trait := item.Inner.Trait
    if trait == nil {
        return ""
    }
    
    var sb strings.Builder
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    if trait.IsUnsafe {
        sb.WriteString("unsafe ")
    }
    if trait.IsAuto {
        sb.WriteString("auto ")
    }
    sb.WriteString("trait ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    sb.WriteString(r.renderGenerics(&trait.Generics))
    
    return sb.String()
}

func (r *RustdocRenderer) renderStructSignature(item *RustdocItem) string {
    st := item.Inner.Struct
    if st == nil {
        return ""
    }
    
    var sb strings.Builder
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    sb.WriteString("struct ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    sb.WriteString(r.renderGenerics(&st.Generics))
    
    return sb.String()
}

func (r *RustdocRenderer) renderEnumSignature(item *RustdocItem) string {
    en := item.Inner.Enum
    if en == nil {
        return ""
    }
    
    var sb strings.Builder
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    sb.WriteString("enum ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    sb.WriteString(r.renderGenerics(&en.Generics))
    
    return sb.String()
}

func (r *RustdocRenderer) renderTypeAliasSignature(item *RustdocItem) string {
    ta := item.Inner.TypeAlias
    if ta == nil {
        return ""
    }
    
    var sb strings.Builder
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    sb.WriteString("type ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    // TODO: render generics and actual type
    
    return sb.String()
}

func (r *RustdocRenderer) renderConstantSignature(item *RustdocItem) string {
    c := item.Inner.Constant
    if c == nil {
        return ""
    }
    
    var sb strings.Builder
    if item.Visibility == "public" {
        sb.WriteString("pub ")
    }
    sb.WriteString("const ")
    if item.Name != nil {
        sb.WriteString(*item.Name)
    }
    // TODO: render type and value
    
    return sb.String()
}
```

### 4.4 Main Strategy Implementation (`docsrs.go` - Complete Replacement)

```go
package strategies

import (
    "context"
    "encoding/json"
    "fmt"
    "net/url"
    "strings"
    "time"

    "github.com/klauspost/compress/zstd"
    "github.com/quantmind-br/repodocs-go/internal/domain"
    "github.com/quantmind-br/repodocs-go/internal/output"
    "github.com/quantmind-br/repodocs-go/internal/utils"
    "github.com/schollz/progressbar/v3"
)

// DocsRSURL holds parsed docs.rs URL information
type DocsRSURL struct {
    CrateName    string
    Version      string
    ModulePath   string
    IsCratePage  bool
    IsSourceView bool
}

// DocsRSStrategy extracts documentation from docs.rs using the rustdoc JSON API
type DocsRSStrategy struct {
    deps     *Dependencies
    fetcher  domain.Fetcher
    writer   *output.Writer
    logger   *utils.Logger
    baseHost string
}

// NewDocsRSStrategy creates a new docs.rs strategy
func NewDocsRSStrategy(deps *Dependencies) *DocsRSStrategy {
    if deps == nil {
        return &DocsRSStrategy{baseHost: "docs.rs"}
    }
    return &DocsRSStrategy{
        deps:     deps,
        fetcher:  deps.Fetcher,
        writer:   deps.Writer,
        logger:   deps.Logger,
        baseHost: "docs.rs",
    }
}

// Name returns the strategy name
func (s *DocsRSStrategy) Name() string {
    return "docsrs"
}

// CanHandle returns true if this strategy can handle the given URL
func (s *DocsRSStrategy) CanHandle(rawURL string) bool {
    parsed, err := s.parseURL(rawURL)
    if err != nil {
        return false
    }
    if parsed.IsSourceView {
        return false
    }
    return parsed.CrateName != ""
}

// SetFetcher sets the fetcher (for testing)
func (s *DocsRSStrategy) SetFetcher(f domain.Fetcher) {
    s.fetcher = f
}

// SetBaseHost sets the base host (for testing)
func (s *DocsRSStrategy) SetBaseHost(host string) {
    s.baseHost = host
}

// parseURL parses a docs.rs URL
func (s *DocsRSStrategy) parseURL(rawURL string) (*DocsRSURL, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return nil, err
    }

    if !strings.Contains(u.Host, s.baseHost) {
        return nil, fmt.Errorf("not a docs.rs URL")
    }

    u.Fragment = ""
    u.RawQuery = ""

    segments := strings.Split(strings.Trim(u.Path, "/"), "/")
    if len(segments) == 0 || segments[0] == "" {
        return nil, fmt.Errorf("empty path")
    }

    result := &DocsRSURL{}

    // Handle /crate/{name}/{version} format
    if segments[0] == "crate" {
        result.IsCratePage = true
        if len(segments) >= 2 {
            result.CrateName = segments[1]
        }
        if len(segments) >= 3 {
            result.Version = segments[2]
        } else {
            result.Version = "latest"
        }
        if len(segments) >= 4 && (segments[3] == "source" || segments[3] == "src") {
            result.IsSourceView = true
        }
        return result, nil
    }

    // Handle /{crate}/{version}/{crate}/... format
    for _, seg := range segments {
        if seg == "src" || seg == "source" {
            result.IsSourceView = true
        }
    }

    result.CrateName = segments[0]
    if len(segments) >= 2 {
        result.Version = segments[1]
    } else {
        result.Version = "latest"
    }
    if len(segments) >= 4 {
        result.ModulePath = strings.Join(segments[3:], "/")
    }

    return result, nil
}

// Execute runs the docs.rs extraction using JSON API
func (s *DocsRSStrategy) Execute(ctx context.Context, rawURL string, opts Options) error {
    s.logger.Info().Str("url", rawURL).Msg("Starting docs.rs JSON extraction")

    if s.fetcher == nil {
        return fmt.Errorf("docsrs strategy fetcher is nil")
    }
    if s.writer == nil {
        return fmt.Errorf("docsrs strategy writer is nil")
    }

    // Parse URL to extract crate name and version
    baseInfo, err := s.parseURL(rawURL)
    if err != nil {
        return fmt.Errorf("invalid docs.rs URL: %w", err)
    }

    s.logger.Info().
        Str("crate", baseInfo.CrateName).
        Str("version", baseInfo.Version).
        Msg("Parsed docs.rs URL")

    // Fetch the rustdoc JSON
    index, err := s.fetchRustdocJSON(ctx, baseInfo.CrateName, baseInfo.Version)
    if err != nil {
        return fmt.Errorf("failed to fetch rustdoc JSON: %w", err)
    }

    // Create renderer
    renderer := NewRustdocRenderer(index, baseInfo.CrateName, baseInfo.Version)

    // Collect items to process
    items := s.collectItems(index, opts)
    s.logger.Info().Int("count", len(items)).Msg("Collected items to process")

    if opts.Limit > 0 && len(items) > opts.Limit {
        items = items[:opts.Limit]
        s.logger.Info().Int("limit", opts.Limit).Msg("Applied item limit")
    }

    // Progress bar
    bar := progressbar.NewOptions(len(items),
        progressbar.OptionSetDescription("Extracting docs.rs (JSON)"),
        progressbar.OptionShowCount(),
        progressbar.OptionShowIts(),
    )

    // Process items (can be parallel since no HTTP requests)
    errors := utils.ParallelForEach(ctx, items, opts.Concurrency, func(ctx context.Context, item *RustdocItem) error {
        defer bar.Add(1)
        return s.processItem(ctx, item, renderer, baseInfo, opts)
    })

    if err := utils.FirstError(errors); err != nil {
        return err
    }

    s.logger.Info().Int("items", len(items)).Msg("docs.rs JSON extraction completed")
    return nil
}

// fetchRustdocJSON downloads and parses the rustdoc JSON
func (s *DocsRSStrategy) fetchRustdocJSON(ctx context.Context, crateName, version string) (*RustdocIndex, error) {
    endpoint := fmt.Sprintf("https://docs.rs/crate/%s/%s/json", crateName, version)

    s.logger.Debug().Str("endpoint", endpoint).Msg("Fetching rustdoc JSON")

    resp, err := s.fetcher.Get(ctx, endpoint)
    if err != nil {
        return nil, fmt.Errorf("failed to fetch: %w", err)
    }

    // Decompress if zstd
    var jsonData []byte
    contentType := resp.ContentType
    if strings.Contains(contentType, "zstd") || strings.HasSuffix(endpoint, ".zst") {
        decoder, err := zstd.NewReader(nil)
        if err != nil {
            return nil, fmt.Errorf("failed to create zstd decoder: %w", err)
        }
        defer decoder.Close()

        jsonData, err = decoder.DecodeAll(resp.Body, nil)
        if err != nil {
            return nil, fmt.Errorf("failed to decompress: %w", err)
        }
    } else {
        jsonData = resp.Body
    }

    s.logger.Debug().
        Int("compressed", len(resp.Body)).
        Int("decompressed", len(jsonData)).
        Msg("Processed rustdoc JSON")

    var index RustdocIndex
    if err := json.Unmarshal(jsonData, &index); err != nil {
        return nil, fmt.Errorf("failed to parse JSON: %w", err)
    }

    s.logger.Info().
        Int("items", len(index.Index)).
        Int("format_version", index.FormatVersion).
        Str("crate_version", index.CrateVersion).
        Msg("Parsed rustdoc index")

    return &index, nil
}

// collectItems collects all items to process based on options
func (s *DocsRSStrategy) collectItems(index *RustdocIndex, opts Options) []*RustdocItem {
    var items []*RustdocItem

    for _, item := range index.Index {
        // Skip external crate items
        if item.CrateID != 0 {
            continue
        }

        // Skip items without names (anonymous re-exports)
        if item.Name == nil {
            continue
        }

        // Skip items without documentation (unless they have children)
        if item.Docs == nil && !s.hasDocumentableChildren(item) {
            continue
        }

        // Skip private items
        if item.Visibility != "public" && item.Visibility != "default" {
            continue
        }

        // Skip re-exports (they point to other items)
        if item.Inner.Use != nil {
            continue
        }

        items = append(items, item)
    }

    return items
}

// hasDocumentableChildren checks if an item has children worth documenting
func (s *DocsRSStrategy) hasDocumentableChildren(item *RustdocItem) bool {
    switch {
    case item.Inner.Module != nil:
        return len(item.Inner.Module.Items) > 0
    case item.Inner.Trait != nil:
        return len(item.Inner.Trait.Items) > 0
    case item.Inner.Struct != nil:
        return len(item.Inner.Struct.Impls) > 0
    case item.Inner.Enum != nil:
        return len(item.Inner.Enum.Variants) > 0 || len(item.Inner.Enum.Impls) > 0
    default:
        return false
    }
}

// processItem processes a single item and writes the document
func (s *DocsRSStrategy) processItem(ctx context.Context, item *RustdocItem, renderer *RustdocRenderer, baseInfo *DocsRSURL, opts Options) error {
    // Generate URL for this item
    itemURL := s.buildItemURL(item, baseInfo)

    // Check if already exists (unless force)
    if !opts.Force && s.writer.Exists(itemURL) {
        return nil
    }

    // Render Markdown
    markdown := renderer.RenderItem(item)
    if markdown == "" {
        return nil
    }

    // Create document
    doc := &domain.Document{
        URL:            itemURL,
        Title:          s.buildItemTitle(item),
        Content:        markdown,
        Description:    s.buildItemDescription(item, baseInfo),
        SourceStrategy: s.Name(),
        FetchedAt:      time.Now(),
        Tags:           s.buildItemTags(item, baseInfo),
    }

    // Write document
    if !opts.DryRun {
        if err := s.deps.WriteDocument(ctx, doc); err != nil {
            s.logger.Warn().Err(err).Str("url", itemURL).Msg("Failed to write document")
            return nil
        }
    }

    return nil
}

// buildItemURL constructs the docs.rs URL for an item
func (s *DocsRSStrategy) buildItemURL(item *RustdocItem, baseInfo *DocsRSURL) string {
    name := ""
    if item.Name != nil {
        name = *item.Name
    }

    itemType := ""
    switch {
    case item.Inner.Module != nil:
        if item.Inner.Module.IsCrate {
            return fmt.Sprintf("https://docs.rs/%s/%s/%s/",
                baseInfo.CrateName, baseInfo.Version, baseInfo.CrateName)
        }
        itemType = "mod"
    case item.Inner.Struct != nil:
        itemType = "struct"
    case item.Inner.Enum != nil:
        itemType = "enum"
    case item.Inner.Trait != nil:
        itemType = "trait"
    case item.Inner.Function != nil:
        itemType = "fn"
    case item.Inner.TypeAlias != nil:
        itemType = "type"
    case item.Inner.Constant != nil:
        itemType = "constant"
    case item.Inner.Macro != nil:
        itemType = "macro"
    default:
        itemType = "item"
    }

    // Build path from span if available
    path := baseInfo.CrateName
    if item.Span != nil && item.Span.Filename != "" {
        // Convert src/widgets/mod.rs to widgets
        spanPath := strings.TrimPrefix(item.Span.Filename, "src/")
        spanPath = strings.TrimSuffix(spanPath, ".rs")
        spanPath = strings.TrimSuffix(spanPath, "/mod")
        if spanPath != "lib" && spanPath != "" {
            path = baseInfo.CrateName + "/" + strings.ReplaceAll(spanPath, "/", "::")
        }
    }

    return fmt.Sprintf("https://docs.rs/%s/%s/%s/%s.%s.html",
        baseInfo.CrateName, baseInfo.Version, path, itemType, name)
}

// buildItemTitle creates a title for the document
func (s *DocsRSStrategy) buildItemTitle(item *RustdocItem) string {
    name := ""
    if item.Name != nil {
        name = *item.Name
    }

    switch {
    case item.Inner.Module != nil:
        if item.Inner.Module.IsCrate {
            return fmt.Sprintf("Crate %s", name)
        }
        return fmt.Sprintf("Module %s", name)
    case item.Inner.Struct != nil:
        return fmt.Sprintf("Struct %s", name)
    case item.Inner.Enum != nil:
        return fmt.Sprintf("Enum %s", name)
    case item.Inner.Trait != nil:
        return fmt.Sprintf("Trait %s", name)
    case item.Inner.Function != nil:
        return fmt.Sprintf("Function %s", name)
    case item.Inner.TypeAlias != nil:
        return fmt.Sprintf("Type %s", name)
    case item.Inner.Macro != nil:
        return fmt.Sprintf("Macro %s", name)
    default:
        return name
    }
}

// buildItemDescription creates a description for the document
func (s *DocsRSStrategy) buildItemDescription(item *RustdocItem, baseInfo *DocsRSURL) string {
    itemType := "item"
    switch {
    case item.Inner.Module != nil:
        itemType = "module"
    case item.Inner.Struct != nil:
        itemType = "struct"
    case item.Inner.Enum != nil:
        itemType = "enum"
    case item.Inner.Trait != nil:
        itemType = "trait"
    case item.Inner.Function != nil:
        itemType = "function"
    case item.Inner.TypeAlias != nil:
        itemType = "type"
    case item.Inner.Macro != nil:
        itemType = "macro"
    }

    stability := "stable"
    if item.Deprecation != nil {
        stability = "deprecated"
    }

    return fmt.Sprintf("crate:%s version:%s type:%s stability:%s",
        baseInfo.CrateName, baseInfo.Version, itemType, stability)
}

// buildItemTags creates tags for the document
func (s *DocsRSStrategy) buildItemTags(item *RustdocItem, baseInfo *DocsRSURL) []string {
    itemType := "item"
    switch {
    case item.Inner.Module != nil:
        itemType = "module"
    case item.Inner.Struct != nil:
        itemType = "struct"
    case item.Inner.Enum != nil:
        itemType = "enum"
    case item.Inner.Trait != nil:
        itemType = "trait"
    case item.Inner.Function != nil:
        itemType = "function"
    case item.Inner.TypeAlias != nil:
        itemType = "type"
    case item.Inner.Macro != nil:
        itemType = "macro"
    }

    tags := []string{
        "docs.rs",
        "rust",
        baseInfo.CrateName,
        itemType,
    }

    if item.Deprecation != nil {
        tags = append(tags, "deprecated")
    }

    return tags
}
```

---

## Part 5: Testing Strategy

### 5.1 Test File Structure

```
tests/
├── unit/
│   └── strategies/
│       ├── docsrs_json_test.go      # JSON parsing tests
│       ├── docsrs_renderer_test.go  # Markdown rendering tests
│       └── docsrs_strategy_test.go  # Strategy integration tests
├── integration/
│   └── strategies/
│       └── docsrs_integration_test.go  # Full pipeline tests
└── testdata/
    └── docsrs/
        ├── ratatui_sample.json      # Sample rustdoc JSON (subset)
        ├── serde_sample.json        # Another sample for variety
        └── expected/
            ├── ratatui_crate.md     # Expected output for crate root
            ├── widget_trait.md      # Expected output for a trait
            └── function.md          # Expected output for a function
```

### 5.2 Unit Tests

#### 5.2.1 JSON Parsing Tests (`docsrs_json_test.go`)

```go
package strategies_test

import (
    "testing"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "github.com/quantmind-br/repodocs-go/internal/strategies"
)

func TestRustdocIndexParsing(t *testing.T) {
    tests := []struct {
        name     string
        json     string
        wantErr  bool
        validate func(t *testing.T, idx *strategies.RustdocIndex)
    }{
        {
            name: "minimal valid index",
            json: `{
                "root": 1,
                "crate_version": "1.0.0",
                "format_version": 57,
                "includes_private": false,
                "index": {},
                "paths": {},
                "external_crates": {}
            }`,
            wantErr: false,
            validate: func(t *testing.T, idx *strategies.RustdocIndex) {
                assert.Equal(t, 1, idx.Root)
                assert.Equal(t, "1.0.0", idx.CrateVersion)
                assert.Equal(t, 57, idx.FormatVersion)
            },
        },
        {
            name: "with module item",
            json: `{
                "root": 1,
                "crate_version": "1.0.0",
                "format_version": 57,
                "includes_private": false,
                "index": {
                    "1": {
                        "id": 1,
                        "crate_id": 0,
                        "name": "mycrate",
                        "visibility": "public",
                        "docs": "Crate documentation",
                        "inner": {
                            "module": {
                                "is_crate": true,
                                "items": [2, 3]
                            }
                        }
                    }
                },
                "paths": {},
                "external_crates": {}
            }`,
            wantErr: false,
            validate: func(t *testing.T, idx *strategies.RustdocIndex) {
                require.Contains(t, idx.Index, "1")
                item := idx.Index["1"]
                assert.Equal(t, "mycrate", *item.Name)
                require.NotNil(t, item.Inner.Module)
                assert.True(t, item.Inner.Module.IsCrate)
                assert.Equal(t, []int{2, 3}, item.Inner.Module.Items)
            },
        },
        {
            name: "with function item",
            json: `{
                "root": 1,
                "crate_version": "1.0.0",
                "format_version": 57,
                "includes_private": false,
                "index": {
                    "2": {
                        "id": 2,
                        "crate_id": 0,
                        "name": "my_function",
                        "visibility": "public",
                        "docs": "Function docs",
                        "inner": {
                            "function": {
                                "sig": {
                                    "inputs": [["x", {"primitive": "i32"}]],
                                    "output": {"primitive": "bool"},
                                    "is_c_variadic": false
                                },
                                "generics": {"params": [], "where_predicates": []},
                                "header": {
                                    "is_const": false,
                                    "is_unsafe": false,
                                    "is_async": false,
                                    "abi": "Rust"
                                },
                                "has_body": true
                            }
                        }
                    }
                },
                "paths": {},
                "external_crates": {}
            }`,
            wantErr: false,
            validate: func(t *testing.T, idx *strategies.RustdocIndex) {
                require.Contains(t, idx.Index, "2")
                item := idx.Index["2"]
                assert.Equal(t, "my_function", *item.Name)
                require.NotNil(t, item.Inner.Function)
                assert.False(t, item.Inner.Function.Header.IsAsync)
            },
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            idx, err := strategies.ParseRustdocJSON([]byte(tt.json))
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            tt.validate(t, idx)
        })
    }
}

func TestTypeRendering(t *testing.T) {
    tests := []struct {
        name     string
        typeJSON map[string]interface{}
        expected string
    }{
        {
            name:     "primitive",
            typeJSON: map[string]interface{}{"primitive": "i32"},
            expected: "i32",
        },
        {
            name:     "generic",
            typeJSON: map[string]interface{}{"generic": "T"},
            expected: "T",
        },
        {
            name: "borrowed ref",
            typeJSON: map[string]interface{}{
                "borrowed_ref": map[string]interface{}{
                    "is_mutable": false,
                    "type":       map[string]interface{}{"primitive": "str"},
                },
            },
            expected: "&str",
        },
        {
            name: "mutable borrowed ref",
            typeJSON: map[string]interface{}{
                "borrowed_ref": map[string]interface{}{
                    "is_mutable": true,
                    "type":       map[string]interface{}{"generic": "T"},
                },
            },
            expected: "&mut T",
        },
        {
            name: "resolved path",
            typeJSON: map[string]interface{}{
                "resolved_path": map[string]interface{}{
                    "path": "Vec",
                    "id":   10,
                },
            },
            expected: "Vec",
        },
        {
            name: "slice",
            typeJSON: map[string]interface{}{
                "slice": map[string]interface{}{"primitive": "u8"},
            },
            expected: "[u8]",
        },
        {
            name: "tuple",
            typeJSON: map[string]interface{}{
                "tuple": []interface{}{
                    map[string]interface{}{"primitive": "i32"},
                    map[string]interface{}{"primitive": "bool"},
                },
            },
            expected: "(i32, bool)",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            renderer := strategies.NewRustdocRenderer(nil, "test", "1.0.0")
            result := renderer.RenderTypeMap(tt.typeJSON)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

#### 5.2.2 Renderer Tests (`docsrs_renderer_test.go`)

```go
package strategies_test

func TestFunctionSignatureRendering(t *testing.T) {
    // Test rendering of function signatures
}

func TestTraitRendering(t *testing.T) {
    // Test rendering of trait documentation
}

func TestCrossReferenceResolution(t *testing.T) {
    // Test that [`Type`] links are resolved correctly
}

func TestModuleContentsRendering(t *testing.T) {
    // Test that module children are listed correctly
}
```

### 5.3 Integration Tests

```go
package integration_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "github.com/stretchr/testify/require"
    
    "github.com/quantmind-br/repodocs-go/internal/strategies"
    "github.com/quantmind-br/repodocs-go/tests/testutil"
)

func TestDocsRSJSONExtraction(t *testing.T) {
    // Load sample JSON
    sampleJSON := testutil.LoadFixture(t, "docsrs/ratatui_sample.json")
    
    // Create mock server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/crate/ratatui/0.30.0/json" {
            w.Header().Set("Content-Type", "application/json")
            w.Write(sampleJSON)
        }
    }))
    defer server.Close()
    
    // Create strategy with test dependencies
    deps := testutil.NewTestDependencies(t)
    strategy := strategies.NewDocsRSStrategy(deps)
    strategy.SetBaseHost("localhost")
    
    // Execute
    ctx := context.Background()
    err := strategy.Execute(ctx, server.URL+"/crate/ratatui/0.30.0", strategies.DefaultOptions())
    require.NoError(t, err)
    
    // Verify output
    docs := deps.Writer.GetWrittenDocuments()
    require.NotEmpty(t, docs)
    
    // Check crate root document
    crateDoc := findDocByTitle(docs, "Crate ratatui")
    require.NotNil(t, crateDoc)
    require.Contains(t, crateDoc.Content, "terminal user interfaces")
}
```

---

## Part 6: Migration Plan

### 6.1 Phase 1: Preparation (Day 1)

1. **Add zstd dependency**
   ```bash
   go get github.com/klauspost/compress/zstd
   ```

2. **Create new files**
   - `internal/strategies/docsrs_types.go`
   - `internal/strategies/docsrs_json.go`
   - `internal/strategies/docsrs_renderer.go`

3. **Create test fixtures**
   - Download sample JSON: `curl -sL "https://docs.rs/crate/ratatui/0.30.0/json" | zstd -d > tests/testdata/docsrs/ratatui_sample.json`
   - Create smaller subset for unit tests

### 6.2 Phase 2: Implementation (Days 2-3)

1. **Implement types** (`docsrs_types.go`)
   - All Go structs for JSON parsing
   - Unit tests for parsing

2. **Implement JSON fetcher** (`docsrs_json.go`)
   - Download and decompress logic
   - Caching integration
   - Unit tests

3. **Implement renderer** (`docsrs_renderer.go`)
   - Type signature rendering
   - Cross-reference resolution
   - Module contents
   - Unit tests with golden files

### 6.3 Phase 3: Integration (Day 4)

1. **Replace main strategy** (`docsrs.go`)
   - Remove HTML crawling code
   - Remove goquery dependency
   - Implement new Execute flow
   - Keep CanHandle and parseURL logic

2. **Update tests**
   - Replace existing tests with new ones
   - Add integration tests
   - Verify golden file outputs

### 6.4 Phase 4: Verification (Day 5)

1. **Manual testing**
   ```bash
   # Test with ratatui
   go run ./cmd/repodocs https://docs.rs/ratatui/0.30.0
   
   # Test with serde
   go run ./cmd/repodocs https://docs.rs/serde/1.0.0
   
   # Test with tokio
   go run ./cmd/repodocs https://docs.rs/tokio/1.0.0
   ```

2. **Compare outputs**
   - Compare new JSON-based output with old HTML-based output
   - Verify all major sections are present
   - Check type signatures are correct

3. **Performance benchmarks**
   ```bash
   # Old implementation
   time go run ./cmd/repodocs https://docs.rs/ratatui/0.30.0 --output /tmp/old
   
   # New implementation  
   time go run ./cmd/repodocs https://docs.rs/ratatui/0.30.0 --output /tmp/new
   ```

### 6.5 Rollback Plan

If issues are discovered:

1. **Git revert** to previous commit
2. **Feature flag** (optional):
   ```go
   if opts.UseJSONAPI {
       return s.executeJSON(ctx, rawURL, opts)
   }
   return s.executeHTML(ctx, rawURL, opts)
   ```

---

## Part 7: Dependencies and Go Module Updates

### 7.1 New Dependencies

```go
// go.mod additions
require (
    github.com/klauspost/compress v1.17.0  // For zstd decompression
)
```

### 7.2 Removed Dependencies (from docsrs.go)

```go
// These are no longer needed in docsrs.go:
// "github.com/PuerkitoBio/goquery" - still used by other strategies
// "math/rand" - no more random delays
```

### 7.3 Internal Dependencies

```go
// Still required:
"github.com/quantmind-br/repodocs-go/internal/domain"
"github.com/quantmind-br/repodocs-go/internal/output"
"github.com/quantmind-br/repodocs-go/internal/utils"

// No longer required for docsrs:
"github.com/quantmind-br/repodocs-go/internal/converter"  // Markdown is pre-formatted
```

---

## Part 8: Error Handling and Edge Cases

### 8.1 Error Cases

| Error | Handling |
|-------|----------|
| JSON endpoint 404 | Return error with message about crate/version not found |
| zstd decompression fails | Try gzip fallback, then return error |
| JSON parse error | Return error with format_version mismatch warning |
| Empty index | Return error indicating no documentation available |
| Network timeout | Respect context cancellation, return error |

### 8.2 Edge Cases

| Case | Handling |
|------|----------|
| Pre-May 2025 releases (no JSON) | Fall back to HTML crawling or return error |
| Very large crates (100MB+ JSON) | Stream decompression, process in batches |
| Private items in index | Skip items with `crate_id != 0` |
| Missing documentation | Skip items without `docs` field |
| Deprecated items | Add deprecation notice to output |
| Feature-gated items | Include feature gate info in output |

### 8.3 Format Version Compatibility

```go
// Supported format versions
const (
    MinFormatVersion = 30  // Oldest supported
    MaxFormatVersion = 60  // Newest tested
)

func (s *DocsRSStrategy) checkFormatVersion(version int) error {
    if version < MinFormatVersion {
        return fmt.Errorf("rustdoc JSON format version %d is too old (min: %d)", version, MinFormatVersion)
    }
    if version > MaxFormatVersion {
        s.logger.Warn().Int("version", version).Msg("Untested format version, proceeding anyway")
    }
    return nil
}
```

---

## Part 9: Performance Expectations

### 9.1 Benchmarks (Expected)

| Metric | Old (HTML) | New (JSON) | Improvement |
|--------|-----------|------------|-------------|
| HTTP Requests | 100-500 | 1 | 100-500x |
| Data Transfer | 50-150 MB | 0.5-5 MB | 10-100x |
| Time (ratatui) | 3-5 min | 5-15 sec | 12-60x |
| Time (serde) | 5-10 min | 10-30 sec | 10-30x |
| Memory Peak | 200-500 MB | 50-150 MB | 2-4x |

### 9.2 Bottlenecks

1. **JSON Parsing**: Large crates (tokio) may have 50MB+ JSON
   - Mitigation: Stream parsing if needed
   
2. **Markdown Generation**: Generating 1000+ documents
   - Mitigation: Already parallel, uses worker pool

3. **Disk I/O**: Writing many small files
   - Mitigation: Batch writes if needed

---

## Part 10: Future Enhancements

### 10.1 Potential Improvements

1. **Incremental Updates**
   - Cache JSON file with ETag
   - Only regenerate changed items

2. **Cross-Crate References**
   - Use `external_crates` to link to dependency docs
   - Generate inter-crate navigation

3. **Source Code Links**
   - Use `span` information to link to source files
   - Integrate with GitHub/GitLab source view

4. **Search Index Generation**
   - Build search index from JSON
   - Enable full-text search in output

5. **Custom Rendering Templates**
   - Allow users to customize Markdown output format
   - Support different output formats (HTML, RST)

### 10.2 Maintenance Considerations

1. **Format Version Updates**
   - Monitor rustdoc JSON format changes
   - Add tests for new format versions

2. **docs.rs API Changes**
   - Monitor docs.rs changelog
   - Update endpoint URLs if needed

---

## Appendix A: Sample Rustdoc JSON (Minimal)

```json
{
  "root": 1,
  "crate_version": "1.0.0",
  "format_version": 57,
  "includes_private": false,
  "index": {
    "1": {
      "id": 1,
      "crate_id": 0,
      "name": "example",
      "span": {"filename": "src/lib.rs", "begin": [1, 1], "end": [100, 1]},
      "visibility": "public",
      "docs": "# Example Crate\n\nThis is an example crate.",
      "links": {},
      "attrs": [],
      "deprecation": null,
      "inner": {
        "module": {
          "is_crate": true,
          "items": [2, 3],
          "is_stripped": false
        }
      }
    },
    "2": {
      "id": 2,
      "crate_id": 0,
      "name": "hello",
      "span": {"filename": "src/lib.rs", "begin": [10, 1], "end": [15, 1]},
      "visibility": "public",
      "docs": "Says hello.\n\n# Examples\n\n```rust\nexample::hello();\n```",
      "links": {},
      "attrs": [],
      "deprecation": null,
      "inner": {
        "function": {
          "sig": {
            "inputs": [],
            "output": null,
            "is_c_variadic": false
          },
          "generics": {"params": [], "where_predicates": []},
          "header": {"is_const": false, "is_unsafe": false, "is_async": false, "abi": "Rust"},
          "has_body": true
        }
      }
    }
  },
  "paths": {},
  "external_crates": {}
}
```

---

## Appendix B: Expected Markdown Output

For the sample JSON above, expected output for `hello` function:

```markdown
# Function `hello`

```rust
pub fn hello()
```

Says hello.

# Examples

```rust
example::hello();
```
```

---

## Checklist

- [ ] Add `github.com/klauspost/compress/zstd` dependency
- [ ] Create `docsrs_types.go` with all Go types
- [ ] Create `docsrs_json.go` with fetcher logic
- [ ] Create `docsrs_renderer.go` with Markdown rendering
- [ ] Replace `docsrs.go` with new implementation
- [ ] Remove unused imports (goquery from docsrs)
- [ ] Create test fixtures in `tests/testdata/docsrs/`
- [ ] Write unit tests for JSON parsing
- [ ] Write unit tests for type rendering
- [ ] Write unit tests for signature rendering
- [ ] Write integration tests
- [ ] Run manual tests with real crates
- [ ] Compare output quality with old implementation
- [ ] Measure performance improvement
- [ ] Update AGENTS.md if needed
- [ ] Create PR with detailed description
