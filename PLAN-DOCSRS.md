# Plano de Implementação: DocsRS Strategy

Este documento descreve o plano para implementar um pipeline de extração de documentação especializado para o site [docs.rs](https://docs.rs/), que hospeda documentação de pacotes Rust gerada pelo `rustdoc`.

---

## 1. Análise do Alvo (docs.rs)

### 1.1 Estrutura de URLs

O docs.rs utiliza padrões de URL previsíveis baseados em crate, versão e caminho do módulo:

| Padrão | Exemplo | Descrição |
|--------|---------|-----------|
| `/{crate}/` | `/serde/` | Landing page (redireciona para latest) |
| `/{crate}/{version}/` | `/serde/1.0.0/` | Página raiz da versão |
| `/{crate}/{version}/{crate}/` | `/serde/1.0.0/serde/` | Documentação raiz do crate |
| `/{crate}/{version}/{crate}/{path}` | `/serde/1.0.0/serde/de/` | Módulo/struct/trait específico |
| `/crate/{crate}/{version}` | `/crate/serde/1.0.0` | Página de informações do crate |
| `/crate/{crate}/{version}/source/` | `/crate/serde/1.0.0/source/` | Visualização do código fonte |

### 1.2 Estrutura HTML

O HTML é gerado pelo `rustdoc` com wrapper customizado do docs.rs:

| Componente | Seletor Primário | Propósito |
|------------|------------------|-----------|
| **Conteúdo Principal** | `#main-content` | Área de documentação |
| **Fallback** | `section.content, main` | Caso `#main-content` não exista |
| **Blocos de Doc** | `.docblock` | Seções de texto de documentação |
| **Sidebar** | `.sidebar` | Navegação e hierarquia de módulos |
| **Navegação** | `.nav-container` | Barra superior com busca e versão |
| **Título** | `.main-heading h1` | Título principal da página |
| **Metadados** | `script#crate-metadata` | JSON com nome e versão do crate |
| **Tabelas de Itens** | `.item-table` | Listas de módulos, traits, funções |
| **Link de Fonte** | `.main-heading .src` | Link para código fonte |

### 1.3 Elementos a Excluir

**Seletores CSS para exclusão:**
```css
.sidebar, .nav-container, .sidebar-elems, .search-form, 
.search-results-title, #search, .mobile-topbar, .out-of-band,
.since, .srclink, script, style, link[rel=stylesheet]
```

**Paths a excluir:**
- `/src/` - Visualização de código fonte
- `/source/` - Alternativa de código fonte
- `/all.html` - Página "todos os itens" (geralmente enorme)
- `/-/rustdoc.static/` - Assets estáticos do rustdoc
- `/-/static/` - Assets estáticos do docs.rs

**Extensões a excluir:**
- `.js`, `.css`, `.svg`, `.png`, `.ico`, `.woff`, `.woff2`, `.ttf`

**Arquivos específicos a excluir:**
- `search-index.js`, `sidebar-items.js`, `crates.js`, `aliases.js`
- `source-script.js`, `storage.js`, `settings.js`

### 1.4 Considerações de Anti-bot

- **Proteção:** docs.rs usa CDN moderno (Cloudflare/Fastly) com fingerprinting TLS
- **Mitigação:** Usar `fetcher.Client` existente com `tls-client` para simular browser
- **Rate Limiting:** Delay de 500-1500ms entre requisições para evitar bloqueios
- **Headers:** User-Agent e Accept realistas já configurados no fetcher

---

## 2. Arquitetura da Solução

### 2.1 Arquivos a Criar/Modificar

| Arquivo | Ação | Descrição |
|---------|------|-----------|
| `internal/strategies/docsrs.go` | Criar | Implementação da estratégia |
| `internal/app/detector.go` | Modificar | Adicionar detecção e factory |
| `tests/unit/strategies/docsrs_strategy_test.go` | Criar | Testes unitários |
| `tests/fixtures/docsrs/*.html` | Criar | Fixtures HTML para testes |

### 2.2 Estruturas de Dados

```go
// DocsRSURL representa uma URL parseada do docs.rs
type DocsRSURL struct {
    CrateName    string // Nome do crate (ex: "serde")
    Version      string // Versão ou "latest" (ex: "1.0.0")
    ModulePath   string // Caminho após /{crate}/{version}/{crate}/ (ex: "de/struct.Deserializer")
    IsCratePage  bool   // true se /crate/{name}/{version}
    IsSourceView bool   // true se contém /source/ ou /src/
}

// DocsRSMetadata representa metadados extraídos de uma página
type DocsRSMetadata struct {
    CrateName   string // Nome do crate
    Version     string // Versão resolvida
    ModulePath  string // Caminho do módulo
    ItemType    string // "mod", "struct", "fn", "trait", "enum", "page"
    Title       string // Título extraído do h1
    SourceURL   string // URL original da página
    Stability   string // "stable", "nightly", "deprecated"
}

// DocsRSStrategy implementa a extração de docs.rs
type DocsRSStrategy struct {
    deps      *Dependencies
    fetcher   domain.Fetcher
    converter *converter.Pipeline
    writer    *output.Writer
    logger    *utils.Logger
}
```

### 2.3 Interface Strategy

```go
func NewDocsRSStrategy(deps *Dependencies) *DocsRSStrategy
func (s *DocsRSStrategy) Name() string                                           // "docsrs"
func (s *DocsRSStrategy) CanHandle(url string) bool                              // Valida host docs.rs
func (s *DocsRSStrategy) Execute(ctx context.Context, url string, opts Options) error
```

---

## 3. Implementação Detalhada

### 3.1 Etapa 1: Parsing de URLs

#### parseDocsRSPath

```go
func parseDocsRSPath(rawURL string) (*DocsRSURL, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return nil, err
    }
    
    if !strings.EqualFold(u.Host, "docs.rs") {
        return nil, fmt.Errorf("not a docs.rs URL")
    }
    
    // Remover fragment e query
    u.Fragment = ""
    u.RawQuery = ""
    
    segments := strings.Split(strings.Trim(u.Path, "/"), "/")
    if len(segments) == 0 || segments[0] == "" {
        return nil, fmt.Errorf("empty path")
    }
    
    result := &DocsRSURL{}
    
    // Handle /crate/{name}/{version} pages
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
    
    // Check for /src/ in path (source view)
    for _, seg := range segments {
        if seg == "src" || seg == "source" {
            result.IsSourceView = true
        }
    }
    
    // Standard doc URL: /{crate}/{version}/{crate}/{path}
    result.CrateName = segments[0]
    
    if len(segments) >= 2 {
        result.Version = segments[1]
    } else {
        result.Version = "latest"
    }
    
    // Module path starts after /{crate}/{version}/{crate}/
    if len(segments) >= 4 {
        result.ModulePath = strings.Join(segments[3:], "/")
    }
    
    return result, nil
}
```

#### CanHandle

```go
func (s *DocsRSStrategy) CanHandle(rawURL string) bool {
    parsed, err := parseDocsRSPath(rawURL)
    if err != nil {
        return false
    }
    
    // Excluir source views
    if parsed.IsSourceView {
        return false
    }
    
    // Deve ter pelo menos o nome do crate
    return parsed.CrateName != ""
}
```

### 3.2 Etapa 2: Scope Filtering

#### shouldCrawl

```go
var (
    docsRSExcludePaths = []string{
        "/src/",
        "/source/",
        "/all.html",
        "/-/rustdoc.static/",
        "/-/static/",
    }
    
    docsRSExcludeExtensions = []string{
        ".js", ".css", ".svg", ".png", ".ico", 
        ".woff", ".woff2", ".ttf",
    }
    
    docsRSExcludeFiles = []string{
        "search-index.js", "sidebar-items.js", "crates.js",
        "aliases.js", "source-script.js", "storage.js", "settings.js",
    }
)

func (s *DocsRSStrategy) shouldCrawl(targetURL string, baseInfo *DocsRSURL) bool {
    u, err := url.Parse(targetURL)
    if err != nil {
        return false
    }
    
    // Deve ser mesmo host
    if !strings.EqualFold(u.Host, "docs.rs") {
        return false
    }
    
    path := u.Path
    
    // Check excluded paths
    for _, excluded := range docsRSExcludePaths {
        if strings.Contains(path, excluded) {
            return false
        }
    }
    
    // Check excluded extensions
    for _, ext := range docsRSExcludeExtensions {
        if strings.HasSuffix(strings.ToLower(path), ext) {
            return false
        }
    }
    
    // Check excluded files
    baseName := filepath.Base(path)
    for _, file := range docsRSExcludeFiles {
        if baseName == file {
            return false
        }
    }
    
    // Parse target URL
    targetInfo, err := parseDocsRSPath(targetURL)
    if err != nil {
        return false
    }
    
    // Excluir source views
    if targetInfo.IsSourceView {
        return false
    }
    
    // Deve ser mesmo crate (não seguir links para std, core, etc)
    if targetInfo.CrateName != baseInfo.CrateName {
        return false
    }
    
    // Deve ser mesma versão (ou latest)
    if targetInfo.Version != baseInfo.Version && 
       targetInfo.Version != "latest" && 
       baseInfo.Version != "latest" {
        return false
    }
    
    return true
}
```

### 3.3 Etapa 3: Crawling BFS

#### discoverPages

```go
func (s *DocsRSStrategy) discoverPages(ctx context.Context, startURL string, baseInfo *DocsRSURL, opts Options) ([]string, error) {
    visited := &sync.Map{}
    var pages []string
    var mu sync.Mutex
    
    queue := []struct {
        url   string
        depth int
    }{{startURL, 0}}
    
    visited.Store(startURL, true)
    pages = append(pages, startURL)
    
    for len(queue) > 0 {
        select {
        case <-ctx.Done():
            return pages, ctx.Err()
        default:
        }
        
        current := queue[0]
        queue = queue[1:]
        
        // Check depth limit
        if opts.MaxDepth > 0 && current.depth >= opts.MaxDepth {
            continue
        }
        
        // Check page limit
        mu.Lock()
        if opts.Limit > 0 && len(pages) >= opts.Limit {
            mu.Unlock()
            break
        }
        mu.Unlock()
        
        // Fetch page
        resp, err := s.fetcher.Get(ctx, current.url)
        if err != nil {
            s.logger.Debug().Err(err).Str("url", current.url).Msg("Failed to fetch for discovery")
            continue
        }
        
        // Parse HTML
        doc, err := goquery.NewDocumentFromReader(strings.NewReader(string(resp.Body)))
        if err != nil {
            continue
        }
        
        // Extract links from sidebar and main content
        doc.Find(".sidebar a[href], #main-content a[href]").Each(func(_ int, sel *goquery.Selection) {
            href, exists := sel.Attr("href")
            if !exists || href == "" {
                return
            }
            
            // Skip anchors and special protocols
            if strings.HasPrefix(href, "#") ||
               strings.HasPrefix(href, "javascript:") ||
               strings.HasPrefix(href, "mailto:") {
                return
            }
            
            // Resolve relative URL
            absoluteURL, err := utils.ResolveURL(current.url, href)
            if err != nil {
                return
            }
            
            // Normalize
            normalizedURL, _ := utils.NormalizeURLWithoutQuery(absoluteURL)
            
            // Check if should crawl
            if !s.shouldCrawl(normalizedURL, baseInfo) {
                return
            }
            
            // Check if already visited
            if _, exists := visited.LoadOrStore(normalizedURL, true); exists {
                return
            }
            
            mu.Lock()
            pages = append(pages, normalizedURL)
            mu.Unlock()
            
            queue = append(queue, struct {
                url   string
                depth int
            }{normalizedURL, current.depth + 1})
        })
    }
    
    return pages, nil
}
```

### 3.4 Etapa 4: Extração de Conteúdo

#### processPage

```go
func (s *DocsRSStrategy) processPage(ctx context.Context, pageURL string, baseInfo *DocsRSURL, opts Options) error {
    // Rate limiting: random delay between 500-1500ms
    delay := time.Duration(500+rand.Intn(1000)) * time.Millisecond
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(delay):
    }
    
    // Check if already exists
    if !opts.Force && s.writer.Exists(pageURL) {
        return nil
    }
    
    // Fetch page
    resp, err := s.fetcher.Get(ctx, pageURL)
    if err != nil {
        s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to fetch page")
        return nil // Continue with other pages
    }
    
    // Parse HTML for metadata extraction
    htmlDoc, err := goquery.NewDocumentFromReader(strings.NewReader(string(resp.Body)))
    if err != nil {
        s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to parse HTML")
        return nil
    }
    
    // Extract docs.rs-specific metadata
    meta := s.extractMetadata(htmlDoc, pageURL, baseInfo)
    
    // Convert to document using pipeline
    doc, err := s.converter.Convert(ctx, string(resp.Body), pageURL)
    if err != nil {
        s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to convert page")
        return nil
    }
    
    // Apply metadata
    s.applyMetadata(doc, meta)
    doc.SourceStrategy = s.Name()
    doc.CacheHit = resp.FromCache
    doc.FetchedAt = time.Now()
    
    // Write document
    if !opts.DryRun {
        if err := s.deps.WriteDocument(ctx, doc); err != nil {
            s.logger.Warn().Err(err).Str("url", pageURL).Msg("Failed to write document")
            return nil
        }
    }
    
    return nil
}
```

#### extractMetadata

```go
func (s *DocsRSStrategy) extractMetadata(doc *goquery.Document, pageURL string, baseInfo *DocsRSURL) *DocsRSMetadata {
    meta := &DocsRSMetadata{
        CrateName:  baseInfo.CrateName,
        Version:    baseInfo.Version,
        ModulePath: baseInfo.ModulePath,
        SourceURL:  pageURL,
    }
    
    // Extract title from main heading
    title := doc.Find(".main-heading h1").First().Text()
    meta.Title = strings.TrimSpace(title)
    
    // Detect item type from body class
    bodyClass, _ := doc.Find("body").Attr("class")
    switch {
    case strings.Contains(bodyClass, "struct"):
        meta.ItemType = "struct"
    case strings.Contains(bodyClass, "enum"):
        meta.ItemType = "enum"
    case strings.Contains(bodyClass, "trait"):
        meta.ItemType = "trait"
    case strings.Contains(bodyClass, "fn"):
        meta.ItemType = "function"
    case strings.Contains(bodyClass, "mod"):
        meta.ItemType = "module"
    case strings.Contains(bodyClass, "macro"):
        meta.ItemType = "macro"
    case strings.Contains(bodyClass, "type"):
        meta.ItemType = "type"
    case strings.Contains(bodyClass, "constant"):
        meta.ItemType = "constant"
    default:
        meta.ItemType = "page"
    }
    
    // Check stability from badges
    if doc.Find(".portability.nightly-only").Length() > 0 {
        meta.Stability = "nightly"
    } else if doc.Find(".stab.deprecated").Length() > 0 {
        meta.Stability = "deprecated"
    } else if doc.Find(".stab.unstable").Length() > 0 {
        meta.Stability = "unstable"
    } else {
        meta.Stability = "stable"
    }
    
    return meta
}

func (s *DocsRSStrategy) applyMetadata(doc *domain.Document, meta *DocsRSMetadata) {
    if meta.Title != "" {
        doc.Title = meta.Title
    }
    doc.SourceURL = meta.SourceURL
    doc.Metadata = map[string]interface{}{
        "crate":       meta.CrateName,
        "version":     meta.Version,
        "module_path": meta.ModulePath,
        "item_type":   meta.ItemType,
        "stability":   meta.Stability,
        "source":      "docs.rs",
    }
}
```

### 3.5 Etapa 5: Execute Principal

```go
func (s *DocsRSStrategy) Execute(ctx context.Context, rawURL string, opts Options) error {
    s.logger.Info().Str("url", rawURL).Msg("Starting docs.rs extraction")
    
    // Validate dependencies
    if s.fetcher == nil {
        return fmt.Errorf("docsrs strategy fetcher is nil")
    }
    if s.converter == nil {
        return fmt.Errorf("docsrs strategy converter is nil")
    }
    if s.writer == nil {
        return fmt.Errorf("docsrs strategy writer is nil")
    }
    
    // Parse entry URL
    baseInfo, err := parseDocsRSPath(rawURL)
    if err != nil {
        return fmt.Errorf("invalid docs.rs URL: %w", err)
    }
    
    // Build normalized start URL
    startURL := s.buildStartURL(baseInfo)
    s.logger.Info().
        Str("crate", baseInfo.CrateName).
        Str("version", baseInfo.Version).
        Str("start_url", startURL).
        Msg("Parsed docs.rs URL")
    
    // Discover all pages via BFS crawl
    pages, err := s.discoverPages(ctx, startURL, baseInfo, opts)
    if err != nil {
        return fmt.Errorf("discovery failed: %w", err)
    }
    
    s.logger.Info().Int("count", len(pages)).Msg("Discovered pages")
    
    // Apply limit
    if opts.Limit > 0 && len(pages) > opts.Limit {
        pages = pages[:opts.Limit]
        s.logger.Info().Int("limit", opts.Limit).Msg("Applied page limit")
    }
    
    // Create progress bar
    bar := progressbar.NewOptions(len(pages),
        progressbar.OptionSetDescription("Extracting docs.rs"),
        progressbar.OptionShowCount(),
        progressbar.OptionShowIts(),
    )
    
    // Process pages concurrently
    errors := utils.ParallelForEach(ctx, pages, opts.Concurrency, func(ctx context.Context, pageURL string) error {
        defer bar.Add(1)
        return s.processPage(ctx, pageURL, baseInfo, opts)
    })
    
    if err := utils.FirstError(errors); err != nil {
        return err
    }
    
    s.logger.Info().Int("pages", len(pages)).Msg("docs.rs extraction completed")
    return nil
}

func (s *DocsRSStrategy) buildStartURL(info *DocsRSURL) string {
    if info.IsCratePage {
        return fmt.Sprintf("https://docs.rs/crate/%s/%s", info.CrateName, info.Version)
    }
    return fmt.Sprintf("https://docs.rs/%s/%s/%s/", info.CrateName, info.Version, info.CrateName)
}
```

---

## 4. Integração com Detector

### 4.1 Modificações em `internal/app/detector.go`

```go
// Adicionar constante
const (
    StrategyLLMS    StrategyType = "llms"
    StrategyPkgGo   StrategyType = "pkggo"
    StrategyDocsRS  StrategyType = "docsrs"  // NOVO
    StrategySitemap StrategyType = "sitemap"
    StrategyWiki    StrategyType = "wiki"
    StrategyGit     StrategyType = "git"
    StrategyCrawler StrategyType = "crawler"
    StrategyUnknown StrategyType = "unknown"
)

// Em DetectStrategy(), adicionar após pkg.go.dev e antes de sitemap:
func DetectStrategy(rawURL string) StrategyType {
    // ... código existente para llms.txt e pkg.go.dev ...
    
    // Check for docs.rs
    if strings.Contains(lower, "docs.rs") {
        // Excluir source views
        if !strings.Contains(lowerPath, "/src/") && 
           !strings.Contains(lowerPath, "/source/") {
            return StrategyDocsRS
        }
    }
    
    // ... resto do código para sitemap, wiki, git, crawler ...
}

// Em CreateStrategy():
func CreateStrategy(strategyType StrategyType, deps *strategies.Dependencies) strategies.Strategy {
    switch strategyType {
    case StrategyDocsRS:
        return strategies.NewDocsRSStrategy(deps)
    // ... outros cases ...
    }
}

// Em GetAllStrategies() - ordem de prioridade:
func GetAllStrategies(deps *strategies.Dependencies) []strategies.Strategy {
    return []strategies.Strategy{
        strategies.NewLLMSStrategy(deps),
        strategies.NewPkgGoStrategy(deps),
        strategies.NewDocsRSStrategy(deps),   // NOVO - após PkgGo, antes de Sitemap
        strategies.NewSitemapStrategy(deps),
        strategies.NewWikiStrategy(deps),
        strategies.NewGitStrategy(deps),
        strategies.NewCrawlerStrategy(deps),
    }
}
```

---

## 5. Testes

### 5.1 Estrutura de Arquivos de Teste

```
tests/
├── fixtures/
│   └── docsrs/
│       ├── serde_crate_root.html       # Página raiz típica
│       ├── serde_module.html           # Página de módulo (de/)
│       ├── serde_struct.html           # Página de struct
│       ├── tokio_async_trait.html      # Página de trait async
│       ├── with_deprecated.html        # Itens deprecated
│       ├── nightly_only.html           # Itens nightly-only
│       └── minimal.html                # HTML mínimo válido
├── unit/
│   └── strategies/
│       └── docsrs_strategy_test.go
└── integration/
    └── strategies/
        └── docsrs_integration_test.go
```

### 5.2 Casos de Teste Unitário

```go
// tests/unit/strategies/docsrs_strategy_test.go

func TestDocsRSStrategy_CanHandle(t *testing.T) {
    tests := []struct {
        name     string
        url      string
        expected bool
    }{
        // Should handle
        {"crate root", "https://docs.rs/serde", true},
        {"crate with version", "https://docs.rs/serde/1.0.0", true},
        {"crate latest", "https://docs.rs/serde/latest/serde/", true},
        {"module path", "https://docs.rs/serde/1.0.0/serde/de", true},
        {"struct page", "https://docs.rs/tokio/1.0.0/tokio/net/struct.TcpStream.html", true},
        {"trait page", "https://docs.rs/serde/1.0.0/serde/trait.Serialize.html", true},
        {"crate info page", "https://docs.rs/crate/serde/1.0.0", true},
        
        // Should NOT handle
        {"source view", "https://docs.rs/serde/1.0.0/src/serde/lib.rs.html", false},
        {"github", "https://github.com/serde-rs/serde", false},
        {"crates.io", "https://crates.io/crates/serde", false},
        {"pkg.go.dev", "https://pkg.go.dev/encoding/json", false},
        {"empty", "", false},
        {"malformed", "not-a-url", false},
    }
    // ...
}

func TestParseDocsRSPath(t *testing.T) {
    tests := []struct {
        name        string
        url         string
        wantCrate   string
        wantVersion string
        wantModule  string
        wantErr     bool
    }{
        {"simple crate", "https://docs.rs/serde", "serde", "latest", "", false},
        {"with version", "https://docs.rs/serde/1.0.0", "serde", "1.0.0", "", false},
        {"full doc path", "https://docs.rs/serde/1.0.0/serde/de/", "serde", "1.0.0", "de/", false},
        {"with struct", "https://docs.rs/serde/1.0.0/serde/de/struct.Deserializer.html", "serde", "1.0.0", "de/struct.Deserializer.html", false},
        {"crate info", "https://docs.rs/crate/serde/1.0.0", "serde", "1.0.0", "", false},
        {"not docs.rs", "https://example.com/serde", "", "", "", true},
        {"empty path", "https://docs.rs/", "", "", "", true},
    }
    // ...
}

func TestDocsRSStrategy_ShouldCrawl(t *testing.T) {
    baseInfo := &DocsRSURL{CrateName: "serde", Version: "1.0.0"}
    
    tests := []struct {
        name     string
        url      string
        expected bool
    }{
        // Should crawl
        {"same crate module", "https://docs.rs/serde/1.0.0/serde/de/", true},
        {"same crate struct", "https://docs.rs/serde/1.0.0/serde/struct.Serialize.html", true},
        
        // Should NOT crawl
        {"different crate", "https://docs.rs/tokio/1.0.0/tokio/", false},
        {"std library", "https://docs.rs/std/", false},
        {"source view", "https://docs.rs/serde/1.0.0/src/serde/lib.rs.html", false},
        {"js file", "https://docs.rs/serde/1.0.0/search-index.js", false},
        {"different host", "https://github.com/serde-rs/serde", false},
        {"all.html", "https://docs.rs/serde/1.0.0/serde/all.html", false},
    }
    // ...
}

func TestDocsRSStrategy_ExtractMetadata(t *testing.T) {
    tests := []struct {
        name          string
        fixtureFile   string
        wantItemType  string
        wantStability string
    }{
        {"struct page", "serde_struct.html", "struct", "stable"},
        {"module page", "serde_module.html", "module", "stable"},
        {"deprecated item", "with_deprecated.html", "function", "deprecated"},
        {"nightly item", "nightly_only.html", "trait", "nightly"},
    }
    // ...
}
```

### 5.3 Casos de Teste de Integração

```go
// tests/integration/strategies/docsrs_integration_test.go

func TestDocsRSStrategy_Execute_MockServer(t *testing.T) {
    // Criar servidor mock que simula docs.rs
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Servir fixtures baseado no path
        switch {
        case strings.Contains(r.URL.Path, "/serde/1.0.0/serde/de"):
            serveFixture(w, "serde_module.html")
        case strings.Contains(r.URL.Path, "/serde/1.0.0/serde/"):
            serveFixture(w, "serde_crate_root.html")
        default:
            w.WriteHeader(http.StatusNotFound)
        }
    }))
    defer server.Close()
    
    // Testar extração completa
    // ...
}

func TestDocsRSStrategy_Execute_RateLimiting(t *testing.T) {
    // Verificar que delays são respeitados entre requisições
}

func TestDocsRSStrategy_Execute_ErrorHandling(t *testing.T) {
    // Testar comportamento com 404, 429, 503
}

func TestDocsRSStrategy_Execute_ContextCancellation(t *testing.T) {
    // Testar cancelamento gracioso
}
```

---

## 6. Configuração do Converter

Para docs.rs, o converter deve usar seletores específicos:

```go
func (s *DocsRSStrategy) getConverterPipeline() *converter.Pipeline {
    return converter.NewPipeline(converter.PipelineOptions{
        ContentSelector: "#main-content",
        ExcludeSelector: strings.Join([]string{
            ".sidebar",
            ".nav-container", 
            ".sidebar-elems",
            ".search-form",
            ".search-results-title",
            "#search",
            ".mobile-topbar",
            ".out-of-band",
            ".since",
            ".srclink",
            "script",
            "style",
            "link[rel=stylesheet]",
        }, ", "),
    })
}
```

---

## 7. Checklist de Implementação

### Fase 1: Core
- [ ] Criar `internal/strategies/docsrs.go` com estruturas básicas
- [ ] Implementar `parseDocsRSPath()` com todos os padrões de URL
- [ ] Implementar `DocsRSStrategy` struct e constructor
- [ ] Implementar `Name()` e `CanHandle()`
- [ ] Implementar `shouldCrawl()` com todos os filtros

### Fase 2: Crawling
- [ ] Implementar `discoverPages()` com BFS
- [ ] Implementar `buildStartURL()` para normalização
- [ ] Adicionar deduplicação via `sync.Map`
- [ ] Respeitar `MaxDepth` e `Limit`

### Fase 3: Extração
- [ ] Implementar `processPage()` com rate limiting
- [ ] Implementar `extractMetadata()` para item type e stability
- [ ] Implementar `applyMetadata()` para frontmatter
- [ ] Integrar com `converter.Pipeline`

### Fase 4: Integração
- [ ] Adicionar `StrategyDocsRS` em `detector.go`
- [ ] Atualizar `DetectStrategy()`
- [ ] Atualizar `CreateStrategy()`
- [ ] Atualizar `GetAllStrategies()`

### Fase 5: Testes
- [ ] Criar fixtures HTML em `tests/fixtures/docsrs/`
- [ ] Criar testes unitários para parsing
- [ ] Criar testes unitários para CanHandle
- [ ] Criar testes unitários para shouldCrawl
- [ ] Criar testes de integração com mock server
- [ ] Verificar com `make test && make lint`

### Fase 6: Documentação
- [ ] Atualizar README com exemplo de uso
- [ ] Adicionar entrada no help do CLI

---

## 8. Exemplo de Uso

Após implementação, o comando funcionará assim:

```bash
# Extrair documentação do crate serde (versão latest)
repodocs https://docs.rs/serde

# Extrair versão específica
repodocs https://docs.rs/tokio/1.32.0

# Com opções
repodocs https://docs.rs/serde/1.0.0 -o ./serde-docs -j 3 --limit 50

# Dry run para ver o que seria extraído
repodocs https://docs.rs/axum --dry-run
```

---

## 9. Considerações Finais

### Performance
- Rate limiting de 500-1500ms entre requisições
- Concorrência padrão de 3-5 workers
- Cache habilitado com TTL de 24h (docs raramente mudam)

### Limitações Conhecidas
- Versão "latest" não é resolvida para versão real (pode ser adicionado depois)
- Páginas `/all.html` são excluídas (muito grandes, pouco útil)
- Source view excluído (código, não documentação)

### Extensões Futuras
- Suporte a extração de source code (`/src/`) como opção
- Resolução de "latest" para versão semântica real
- Extração de dependências do crate
- Suporte a `<link rel="canonical">` para melhor deduplicação
