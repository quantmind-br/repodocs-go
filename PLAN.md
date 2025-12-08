# RepoDocs-Go - Plano de Desenvolvimento

## Informações do Projeto

- **Repositório:** `github.com/quantmind-br/repodocs-go`
- **Linguagem:** Go (Golang) 1.21+
- **Objetivo:** CLI de extração de documentação com capacidade stealth (anti-bot detection) e suporte a páginas renderizadas com JavaScript.

---

## Decisões Arquiteturais

| Decisão         | Escolha                   | Justificativa                                   |
| --------------- | ------------------------- | ----------------------------------------------- |
| robots.txt      | Ignorar                   | Modo stealth agressivo para evitar detecção     |
| Git Auth        | Não suportado             | Apenas repositórios públicos                    |
| JS Rendering    | rod (headless)            | Suporte completo a SPAs                         |
| Rod Concurrency | Múltiplas tabs            | Um browser, várias tabs (menor uso de memória)  |
| Cache           | BadgerDB                  | Key-value nativo Go, alta performance           |
| Config File     | `~/.repodocs/config.yaml` | Configurações persistentes do usuário           |
| Testes          | Cobertura completa        | Unit + Integration + E2E                        |
| Output          | MD + JSON                 | Markdown + metadados estruturados               |
| Docker          | Não                       | Apenas binário estático, usuário instala Chrome |

---

## Fase 1: Estrutura e Fundação

### 1.1. Inicialização do Módulo

```bash
mkdir repodocs-go && cd repodocs-go
go mod init github.com/quantmind-br/repodocs-go
```

### 1.2. Arquitetura de Diretórios

```text
cmd/
  └── repodocs/
      └── main.go                 # Entrypoint
internal/
  ├── app/
  │   ├── orchestrator.go         # Orquestrador principal
  │   └── detector.go             # Detecção automática de tipo de URL
  ├── config/
  │   ├── config.go               # Struct de configuração
  │   ├── loader.go               # Carregamento de config (viper)
  │   └── defaults.go             # Valores padrão e constantes
  ├── domain/
  │   ├── interfaces.go           # Strategy, Fetcher, Cache interfaces
  │   ├── models.go               # Page, Document, Metadata structs
  │   └── errors.go               # Erros customizados tipados
  ├── fetcher/
  │   ├── client.go               # tls-client wrapper
  │   ├── transport.go            # http.RoundTripper customizado
  │   ├── stealth.go              # User-Agent rotation, fingerprinting
  │   └── retry.go                # Retry com backoff exponencial
  ├── renderer/
  │   ├── rod.go                  # Integração com rod (headless)
  │   ├── pool.go                 # Pool de tabs para concorrência
  │   ├── stealth.go              # Plugin stealth para rod
  │   └── detector.go             # Detecta se página precisa de JS
  ├── cache/
  │   ├── interface.go            # Cache interface
  │   ├── badger.go               # BadgerDB implementation
  │   └── keys.go                 # Geração de chaves (SHA256)
  ├── converter/
  │   ├── pipeline.go             # Orquestrador do pipeline
  │   ├── readability.go          # Extração de conteúdo principal
  │   ├── markdown.go             # Conversão HTML → Markdown
  │   ├── sanitizer.go            # Limpeza de HTML residual
  │   └── encoding.go             # Detecção e conversão de charset
  ├── strategies/
  │   ├── strategy.go             # Interface base Strategy
  │   ├── crawler.go              # Web crawling com Colly
  │   ├── git.go                  # Clone de repositórios
  │   ├── sitemap.go              # Parser de sitemap XML
  │   ├── pkggo.go                # Extrator pkg.go.dev
  │   └── llms.go                 # Parser llms.txt
  ├── output/
  │   ├── writer.go               # Orquestrador de output
  │   ├── markdown.go             # Gerador de arquivos .md
  │   ├── json.go                 # Gerador de metadados .json
  │   └── filesystem.go           # Operações de arquivo (flat/nested)
  └── utils/
      ├── fs.go                   # Sanitização de filenames
      ├── logger.go               # Zerolog wrapper
      ├── url.go                  # Normalização de URLs
      └── workerpool.go           # Worker pool com context
pkg/
  └── version/
      └── version.go              # Informações de versão
tests/
  ├── unit/
  │   ├── converter_test.go
  │   ├── sanitizer_test.go
  │   ├── cache_test.go
  │   └── retry_test.go
  ├── integration/
  │   ├── fetcher_test.go
  │   ├── renderer_test.go
  │   └── strategies_test.go
  ├── e2e/
  │   ├── crawl_test.go
  │   └── sitemap_test.go
  └── testdata/
      ├── fixtures/               # HTML/XML de teste
      └── golden/                 # Expected outputs

# Arquivos de configuração na raiz
.github/
  └── workflows/
      └── ci.yml                  # GitHub Actions CI/CD
.golangci.yml                     # Configuração do linter
Makefile                          # Build system
go.mod                            # Módulo Go
go.sum                            # Checksums de dependências
README.md                         # Documentação
```

### 1.3. Dependências

```go
// Core HTTP Stealth
"github.com/bogdanfinn/tls-client"          // TLS fingerprinting

// Web Crawling
"github.com/gocolly/colly/v2"               // Crawler framework

// Headless Browser
"github.com/go-rod/rod"                     // Chrome DevTools Protocol
"github.com/go-rod/stealth"                 // Stealth plugin

// HTML Processing
"github.com/go-shiori/go-readability"       // Extração de conteúdo
"github.com/JohannesKaufmann/html-to-markdown/v2" // Conversão MD
"github.com/PuerkitoBio/goquery"            // Manipulação DOM
"golang.org/x/net/html/charset"             // Detecção de encoding

// Git
"github.com/go-git/go-git/v5"               // Git operations

// Cache
"github.com/dgraph-io/badger/v4"            // Key-value store

// CLI
"github.com/spf13/cobra"                    // CLI framework
"github.com/spf13/viper"                    // Configuração

// Utilities
"github.com/cenkalti/backoff/v4"            // Retry logic
"github.com/schollz/progressbar/v3"         // Progress bar
"github.com/rs/zerolog"                     // Structured logging

// Testing
"github.com/stretchr/testify"               // Assertions
"github.com/jarcoal/httpmock"               // HTTP mocking
```

---

## Fase 2: Módulo HTTP Stealth (`internal/fetcher`)

### 2.1. Cliente Base

```go
// internal/fetcher/client.go
type Client struct {
    tlsClient  tls_client.HttpClient
    userAgents []string
    retrier    *Retrier
    cache      cache.Cache
}

type ClientOptions struct {
    Timeout       time.Duration  // Default: 30s
    MaxRetries    int            // Default: 3
    EnableCache   bool           // Default: true
    CacheTTL      time.Duration  // Default: 24h
}

func NewClient(opts ClientOptions) (*Client, error)
func (c *Client) Get(ctx context.Context, url string) (*Response, error)
func (c *Client) GetWithHeaders(ctx context.Context, url string, headers map[string]string) (*Response, error)
func (c *Client) GetCookies(url string) []*http.Cookie // Compartilhar sessão com Rod
```

### 2.2. Stealth Features

- **TLS Fingerprinting:** Usar perfil `Chrome_131` (atualizar trimestralmente)
- **User-Agent Rotation:** Pool de 20+ User-Agents reais (Chrome, Firefox, Safari)
- **Header Randomization:** Variar ordem de Accept, Accept-Language, Accept-Encoding
- **Connection Pooling:** Reutilizar conexões TCP (MaxIdleConns: 100)
- **JA3 Fingerprint:** Garantido pelo tls-client

### 2.3. Retry com Backoff

```go
// internal/fetcher/retry.go
type Retrier struct {
    maxRetries int
    backoff    backoff.BackOff  // Exponential: 1s → 2s → 4s
}

// Retry em casos de:
// - Status 429 (Too Many Requests)
// - Status 503 (Service Unavailable)
// - Status 520-530 (Cloudflare errors)
// - Timeout de conexão
// - Reset de conexão
```

### 2.4. RoundTripper para Colly

```go
// internal/fetcher/transport.go
type StealthTransport struct {
    client *Client
}

func (t *StealthTransport) RoundTrip(req *http.Request) (*http.Response, error)

// Permite injetar o tls-client no Colly:
// c := colly.NewCollector()
// c.WithTransport(&StealthTransport{client})
```

---

## Fase 3: Módulo Renderer (`internal/renderer`)

### 3.1. Integração Rod

```go
// internal/renderer/rod.go
type Renderer struct {
    browser  *rod.Browser
    timeout  time.Duration
    stealth  bool
}

type RenderOptions struct {
    Timeout     time.Duration  // Default: 60s
    WaitFor     string         // CSS selector para aguardar
    WaitStable  time.Duration  // Aguardar network idle (ex: 2s)
    ScrollToEnd bool           // Scroll para carregar lazy content
    Cookies     []*http.Cookie // Cookies compartilhados do Fetcher
}

func NewRenderer() (*Renderer, error)
func (r *Renderer) Render(ctx context.Context, url string, opts RenderOptions) (string, error)
func (r *Renderer) Close() error
```

### 3.2. Detecção de SPA

```go
// internal/renderer/detector.go
func NeedsJSRendering(html string) bool {
    // Detecta padrões de SPA:
    // - <div id="root"></div> vazio (React)
    // - <div id="app"></div> vazio (Vue)
    // - <div id="__next"></div> (Next.js)
    // - <script>window.__NUXT__</script> (Nuxt)
    // - Conteúdo principal < 500 chars com muitos scripts
}
```

### 3.3. Stealth Mode

```go
// internal/renderer/stealth.go
func (r *Renderer) applyStealthMode(page *rod.Page) {
    // Usar github.com/go-rod/stealth
    // - Remove webdriver flag
    // - Emula plugins reais
    // - Emula WebGL vendor
    // - Define screen/viewport realistas
}
```

### 3.4. Estratégia de Concorrência (Múltiplas Tabs)

```go
// internal/renderer/pool.go
type TabPool struct {
    browser    *rod.Browser
    maxTabs    int
    activeTabs chan *rod.Page
    mu         sync.Mutex
}

func NewTabPool(maxTabs int) (*TabPool, error) {
    browser := rod.New().MustConnect()

    pool := &TabPool{
        browser:    browser,
        maxTabs:    maxTabs,
        activeTabs: make(chan *rod.Page, maxTabs),
    }

    // Pré-criar tabs
    for i := 0; i < maxTabs; i++ {
        page := browser.MustPage("")
        pool.activeTabs <- page
    }

    return pool, nil
}

func (p *TabPool) Acquire(ctx context.Context) (*rod.Page, error) {
    select {
    case page := <-p.activeTabs:
        return page, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

func (p *TabPool) Release(page *rod.Page) {
    // Limpar estado da página antes de reutilizar
    page.MustNavigate("about:blank")
    p.activeTabs <- page
}

func (p *TabPool) Close() error {
    return p.browser.Close()
}
```

**Configuração:**

- `maxTabs` = flag `--concurrency` (default: 5)
- Cada tab é reutilizada após navegação para `about:blank`
- Menor consumo de memória comparado a múltiplos browsers
- Todas as tabs compartilham cookies/storage do browser

---

## Fase 4: Módulo Cache (`internal/cache`)

### 4.1. Interface

```go
// internal/cache/interface.go
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
    Has(ctx context.Context, key string) bool
    Delete(ctx context.Context, key string) error
    Close() error
}

type CacheEntry struct {
    URL         string
    Content     []byte
    ContentType string
    FetchedAt   time.Time
    ExpiresAt   time.Time
}
```

### 4.2. BadgerDB Implementation

```go
// internal/cache/badger.go
type BadgerCache struct {
    db *badger.DB
}

func NewBadgerCache(path string) (*BadgerCache, error)

// Chave: SHA256(normalized_url)
// Valor: gob-encoded CacheEntry
```

### 4.3. Estratégias de Cache

- **Cache Hit:** Retorna conteúdo se não expirado
- **Cache Miss:** Faz request, armazena resultado
- **Cache Bypass:** Flag `--no-cache` ignora cache
- **Cache Refresh:** Flag `--refresh-cache` força novo download
- **Limpeza:** GC automático do BadgerDB para entradas expiradas

---

## Fase 5: Pipeline de Conversão (`internal/converter`)

### 5.1. Fluxo do Pipeline

```
HTML Bruto + URL
       │
       ▼
┌──────────────────┐
│ Encoding         │ → Detecta charset, converte para UTF-8
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Readability      │ → Extrai conteúdo (via seletor ou heurística)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Sanitizer        │ → Remove script, style, iframe, noscript
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ URL Normalizer   │ → Converte links relativos para absolutos
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Asset Downloader │ → Baixa imagens e reescreve links (local)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Markdown         │ → Converte HTML → Markdown
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Frontmatter      │ → Adiciona YAML header
└──────────────────┘
         │
         ▼
   Markdown Final
```

### 5.2. Configuração do Conversor Markdown

````go
// internal/converter/markdown.go
converter := md.NewConverter(
    md.WithDomain(baseURL),
    md.WithCodeBlockStyle(md.Fenced),  // ```code```
    md.WithHeadingStyle(md.ATX),       // # Heading
    md.WithBulletListMarker('-'),
    md.WithTableSupport(),
)

// Regras customizadas:
// - Preservar syntax highlighting em code blocks
// - Converter <details> para seções colapsáveis
// - Manter atributos de imagem (alt, title)
````

### 5.3. Frontmatter YAML

```yaml
---
title: "Page Title"
url: "https://example.com/docs/page"
source: "crawler" # crawler|sitemap|git|pkggo|llms
fetched_at: "2024-01-15T10:30:00Z"
rendered_js: false
word_count: 1523
---
```

---

## Fase 6: Implementação das Estratégias

### 6.1. Interface Base

```go
// internal/strategies/strategy.go
type Strategy interface {
    Name() string
    CanHandle(url string) bool
    Execute(ctx context.Context, url string, opts Options) error
}

type Options struct {
    Output      string
    Concurrency int
    Limit       int
    MaxDepth    int
    Exclude     []string
    NoFolders   bool
    DryRun      bool
    Verbose     bool
}
```

### 6.2. Crawler (`strategies/crawler.go`)

```go
type CrawlerStrategy struct {
    fetcher   *fetcher.Client
    renderer  *renderer.Renderer
    converter *converter.Pipeline
    output    *output.Writer
}

func (s *CrawlerStrategy) Execute(ctx context.Context, url string, opts Options) error {
    c := colly.NewCollector(
        colly.Async(true),
        colly.MaxDepth(opts.MaxDepth),
    )

    // Injetar transport stealth
    c.WithTransport(s.fetcher.Transport())

    // Configurar rate limiting
    c.Limit(&colly.LimitRule{
        DomainGlob:  "*",
        Parallelism: opts.Concurrency,
        RandomDelay: 2 * time.Second,  // 1-3s random
    })

    // Deduplicação de URLs
    visited := sync.Map{}

    // Handler de páginas
    c.OnHTML("a[href]", s.handleLink)
    c.OnResponse(s.handleResponse)

    // Graceful shutdown
    go func() {
        <-ctx.Done()
        c.Wait()
    }()

    return c.Visit(url)
}
```

**Features:**

- Deduplicação de URLs visitadas
- Normalização de URLs (remove fragments, query params opcionais)
- Filtro por regex (`--exclude`)
- Detecção automática de SPA → fallback para renderer
- Respeita `--max-depth` e `--limit`

### 6.3. Sitemap (`strategies/sitemap.go`)

```go
type SitemapStrategy struct {
    fetcher   *fetcher.Client
    converter *converter.Pipeline
    output    *output.Writer
    pool      *workerpool.Pool
}

func (s *SitemapStrategy) Execute(ctx context.Context, url string, opts Options) error {
    // 1. Fetch sitemap
    content, err := s.fetcher.Get(ctx, url)

    // 2. Detectar tipo (Index ou URLs)
    if isSitemapIndex(content) {
        return s.processSitemapIndex(ctx, content, opts)
    }

    // 3. Parsear URLs
    urls := parseSitemapURLs(content)

    // 4. Aplicar limit
    if opts.Limit > 0 && len(urls) > opts.Limit {
        urls = urls[:opts.Limit]
    }

    // 5. Processar em paralelo via worker pool
    return s.pool.Process(ctx, urls, s.processURL)
}
```

**Features:**

- Suporte a Sitemap Index (sitemaps aninhados)
- Suporte a sitemaps comprimidos (.xml.gz)
- Ordenação por `<lastmod>` (mais recentes primeiro)
- Worker pool com context cancellation

### 6.4. Git (`strategies/git.go`)

```go
type GitStrategy struct {
    converter *converter.Pipeline
    output    *output.Writer
}

func (s *GitStrategy) Execute(ctx context.Context, url string, opts Options) error {
    // 1. Criar diretório temporário
    tmpDir, err := os.MkdirTemp("", "repodocs-git-*")
    defer os.RemoveAll(tmpDir)

    // 2. Clone com progress
    repo, err := git.PlainCloneContext(ctx, tmpDir, false, &git.CloneOptions{
        URL:      url,
        Depth:    1,  // Shallow clone
        Progress: s.progressWriter(),
    })

    // 3. Walk com filtros
    return filepath.WalkDir(tmpDir, func(path string, d fs.DirEntry, err error) error {
        // Ignorar: .git, node_modules, vendor, __pycache__
        // Ignorar: binários, arquivos > 50MB
        // Processar: .md, .txt, .rst, .adoc
        return s.processFile(path, opts)
    })
}
```

**Features:**

- Shallow clone (depth=1) para velocidade
- Usa `filepath.WalkDir` (mais eficiente que Walk)
- Filtros configuráveis por extensão
- Suporte a `--include-assets` para copiar imagens referenciadas

### 6.5. Pkg.go.dev (`strategies/pkggo.go`)

```go
type PkgGoStrategy struct {
    fetcher   *fetcher.Client
    converter *converter.Pipeline
    output    *output.Writer
}

func (s *PkgGoStrategy) Execute(ctx context.Context, url string, opts Options) error {
    // 1. Fetch página
    html, err := s.fetcher.Get(ctx, url)

    // 2. Extrair documentação com goquery
    doc, _ := goquery.NewDocumentFromReader(strings.NewReader(html))
    content := doc.Find("div.Documentation-content").First()

    // 3. Se --split, dividir por seções
    if opts.Split {
        return s.splitBySections(content, opts)
    }

    // 4. Converter e salvar
    return s.convertAndSave(content, opts)
}
```

**Features:**

- Seletor específico para pkg.go.dev
- Opção `--split` divide por `<h3>` (tipos, funções, etc.)
- Preserva syntax highlighting de exemplos

### 6.6. LLMS.txt (`strategies/llms.go`)

```go
type LLMSStrategy struct {
    fetcher   *fetcher.Client
    converter *converter.Pipeline
    output    *output.Writer
    pool      *workerpool.Pool
}

func (s *LLMSStrategy) Execute(ctx context.Context, url string, opts Options) error {
    // 1. Baixar llms.txt
    content, err := s.fetcher.Get(ctx, url)

    // 2. Parsear links markdown: [Title](url)
    links := parseLLMSLinks(content)

    // 3. Aplicar limit
    if opts.Limit > 0 && len(links) > opts.Limit {
        links = links[:opts.Limit]
    }

    // 4. Download em paralelo
    return s.pool.Process(ctx, links, s.downloadAndConvert)
}

// Regex para links: \[([^\]]+)\]\(([^)]+)\)
```

---

## Fase 7: Output (`internal/output`)

### 7.1. Writer

```go
// internal/output/writer.go
type Writer struct {
    baseDir   string
    flat      bool      // --nofolders
    jsonMeta  bool      // --json-meta
}

func (w *Writer) Write(ctx context.Context, doc *domain.Document) error {
    // 1. Gerar path do arquivo
    path := w.generatePath(doc.URL)

    // 2. Criar diretórios se necessário
    if !w.flat {
        os.MkdirAll(filepath.Dir(path), 0755)
    }

    // 3. Escrever .md
    if err := w.writeMarkdown(path, doc); err != nil {
        return err
    }

    // 4. Escrever .json se habilitado
    if w.jsonMeta {
        return w.writeJSON(path, doc)
    }

    return nil
}
```

### 7.2. Estrutura de Metadados JSON

```go
// internal/output/json.go
type Metadata struct {
    URL            string            `json:"url"`
    Title          string            `json:"title"`
    Description    string            `json:"description,omitempty"`
    FetchedAt      time.Time         `json:"fetched_at"`
    ContentHash    string            `json:"content_hash"`  // SHA256
    WordCount      int               `json:"word_count"`
    CharCount      int               `json:"char_count"`
    Links          []string          `json:"links"`
    Headers        map[string][]string `json:"headers"`  // h1, h2, h3...
    RenderedWithJS bool              `json:"rendered_with_js"`
    SourceStrategy string            `json:"source_strategy"`
    CacheHit       bool              `json:"cache_hit"`
}
```

### 7.3. Sanitização de Filenames

```go
// internal/utils/fs.go
func SanitizeFilename(url string) string {
    // 1. Extrair path da URL
    // 2. Remover caracteres inválidos: < > : " | ? * \ /
    // 3. Limitar a 200 caracteres
    // 4. Garantir extensão .md
    // 5. Evitar nomes reservados Windows: CON, PRN, AUX, NUL, COM1-9, LPT1-9
}

func GeneratePath(baseDir, url string, flat bool) string {
    if flat {
        // docs-api-v1-auth.md
        return filepath.Join(baseDir, sanitizeFlat(url))
    }
    // docs/api/v1/auth.md
    return filepath.Join(baseDir, sanitizeNested(url))
}
```

---

## Fase 8: CLI (`cmd/repodocs`)

### 8.1. Estrutura de Comandos

```go
// cmd/repodocs/main.go
func main() {
    rootCmd := &cobra.Command{
        Use:   "repodocs [url]",
        Short: "Extract documentation from any source",
    }

    // Flags globais
    rootCmd.PersistentFlags().StringP("output", "o", "./docs", "Output directory")
    rootCmd.PersistentFlags().IntP("concurrency", "j", 5, "Number of concurrent workers")
    rootCmd.PersistentFlags().IntP("limit", "l", 0, "Max pages to process (0=unlimited)")
    rootCmd.PersistentFlags().IntP("max-depth", "d", 3, "Max crawl depth")
    rootCmd.PersistentFlags().StringSlice("exclude", nil, "Regex patterns to exclude")
    rootCmd.PersistentFlags().Bool("nofolders", false, "Flat output structure")
    rootCmd.PersistentFlags().Bool("force", false, "Overwrite existing files")
    rootCmd.PersistentFlags().BoolP("verbose", "v", false, "Verbose output")

    // Flags de cache
    rootCmd.PersistentFlags().Bool("no-cache", false, "Disable caching")
    rootCmd.PersistentFlags().Duration("cache-ttl", 24*time.Hour, "Cache TTL")
    rootCmd.PersistentFlags().Bool("refresh-cache", false, "Force cache refresh")

    // Flags de rendering
    rootCmd.PersistentFlags().Bool("render-js", false, "Force JS rendering")
    rootCmd.PersistentFlags().Duration("timeout", 30*time.Second, "Request timeout")

    // Flags de output
    rootCmd.PersistentFlags().Bool("json-meta", false, "Generate JSON metadata files")
    rootCmd.PersistentFlags().Bool("dry-run", false, "Simulate without writing files")

    // Flags específicas
    rootCmd.PersistentFlags().Bool("split", false, "Split output by sections (pkg.go.dev)")
    rootCmd.PersistentFlags().Bool("include-assets", false, "Include referenced images (git)")
    rootCmd.PersistentFlags().String("user-agent", "", "Custom User-Agent")
    rootCmd.PersistentFlags().String("content-selector", "", "CSS selector for main content")

    rootCmd.AddCommand(doctorCmd) // Verifica dependências (browser, etc)

    rootCmd.Execute()
}
```

### 8.2. Tabela de Flags

| Flag               | Curto | Default  | Descrição                       |
| ------------------ | ----- | -------- | ------------------------------- |
| `--output`         | `-o`  | `./docs` | Diretório de saída              |
| `--concurrency`    | `-j`  | `5`      | Workers paralelos               |
| `--limit`          | `-l`  | `0`      | Limite de páginas (0=ilimitado) |
| `--max-depth`      | `-d`  | `3`      | Profundidade máxima de crawl    |
| `--exclude`        | -     | `[]`     | Patterns regex para excluir     |
| `--nofolders`      | -     | `false`  | Output flat (sem subpastas)     |
| `--force`          | -     | `false`  | Sobrescrever existentes         |
| `--verbose`        | `-v`  | `false`  | Logs detalhados                 |
| `--no-cache`       | -     | `false`  | Desabilitar cache               |
| `--cache-ttl`      | -     | `24h`    | Tempo de vida do cache          |
| `--refresh-cache`  | -     | `false`  | Forçar refresh do cache         |
| `--render-js`      | -     | `false`  | Forçar renderização JS          |
| `--timeout`        | -     | `30s`    | Timeout por requisição          |
| `--json-meta`      | -     | `false`  | Gerar arquivos .json            |
| `--dry-run`        | -     | `false`  | Simular sem salvar              |
| `--split`          | -     | `false`  | Dividir por seções              |
| `--include-assets` | -     | `false`  | Baixar imagens (git/web)        |
| `--user-agent`     | -     | `""`     | User-Agent customizado          |
| `--content-selector`| -    | `""`     | Seletor CSS do conteúdo         |

### 8.3. Detecção Automática de Modo

```go
// internal/app/detector.go
func DetectStrategy(url string) Strategy {
    switch {
    case strings.HasSuffix(url, "/llms.txt"):
        return &LLMSStrategy{}
    case strings.HasSuffix(url, "sitemap.xml"), strings.Contains(url, "sitemap"):
        return &SitemapStrategy{}
    case strings.HasPrefix(url, "git@"), strings.HasSuffix(url, ".git"):
        return &GitStrategy{}
    case strings.Contains(url, "pkg.go.dev"):
        return &PkgGoStrategy{}
    case strings.HasPrefix(url, "http://"), strings.HasPrefix(url, "https://"):
        return &CrawlerStrategy{}
    default:
        return nil
    }
}

### 8.4. Comandos Utilitários

```go
// cmd/repodocs/doctor.go
var doctorCmd = &cobra.Command{
    Use:   "doctor",
    Short: "Check system dependencies",
    Run: func(cmd *cobra.Command, args []string) {
        // 1. Check Internet connection
        // 2. Check Chrome/Chromium availability for Rod
        // 3. Check Write permissions in output/cache dirs
        // 4. Validate config file format
    },
}
```
```

---

## Fase 9: Fluxo Principal

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLI Input                                 │
│                    repodocs [url] [flags]                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     URL Detector                                 │
│         Identifica: crawler|sitemap|git|pkggo|llms               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Strategy Executor                             │
│              Executa estratégia apropriada                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │   Cache   │   │  Fetcher  │   │ Renderer  │
    │   Check   │   │  Stealth  │   │    Rod    │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │               │               │
          │  Miss         │               │ (se SPA)
          └───────────────┴───────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Converter Pipeline                             │
│     Encoding → Readability → Sanitize → Markdown → Frontmatter   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Output Writer                                │
│              .md files + .json metadata (opcional)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Fase 10: Testes

### 10.1. Unit Tests

```go
// tests/unit/converter_test.go
func TestPipeline_BasicHTML(t *testing.T) { ... }
func TestPipeline_WithCodeBlocks(t *testing.T) { ... }
func TestPipeline_WithTables(t *testing.T) { ... }
func TestPipeline_EncodingDetection(t *testing.T) { ... }

// tests/unit/sanitizer_test.go
func TestSanitizeFilename_SpecialChars(t *testing.T) { ... }
func TestSanitizeFilename_WindowsReserved(t *testing.T) { ... }
func TestSanitizeFilename_LongNames(t *testing.T) { ... }

// tests/unit/cache_test.go
func TestBadgerCache_SetGet(t *testing.T) { ... }
func TestBadgerCache_Expiration(t *testing.T) { ... }
func TestBadgerCache_Concurrent(t *testing.T) { ... }
```

### 10.2. Integration Tests

```go
// tests/integration/fetcher_test.go
func TestFetcher_RealRequest(t *testing.T) {
    // Testa contra httpbin.org
}

func TestFetcher_StealthHeaders(t *testing.T) {
    // Verifica headers enviados
}

// tests/integration/renderer_test.go
func TestRenderer_SPAPage(t *testing.T) {
    // Serve página React local, testa renderização
}

// tests/integration/strategies_test.go
func TestCrawlerStrategy_MockServer(t *testing.T) {
    // Servidor HTTP local com páginas de teste
}
```

### 10.3. E2E Tests

```go
// tests/e2e/crawl_test.go
func TestE2E_CrawlRealSite(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping E2E test")
    }
    // Crawl de site real (ex: golang.org/doc)
}

// tests/e2e/sitemap_test.go
func TestE2E_SitemapParsing(t *testing.T) {
    // Parse de sitemap real
}
```

### 10.4. Golden Files

```text
tests/testdata/
├── fixtures/
│   ├── basic.html
│   ├── spa_react.html
│   ├── with_tables.html
│   └── sitemap.xml
└── golden/
    ├── basic.md
    ├── spa_react.md
    ├── with_tables.md
    └── sitemap_urls.json
```

---

## Fase 11: Logging e Progress

### 11.1. Zerolog Setup

```go
// internal/utils/logger.go
func NewLogger(verbose bool) zerolog.Logger {
    if verbose {
        return zerolog.New(zerolog.ConsoleWriter{Out: os.Stderr}).
            With().Timestamp().Caller().Logger()
    }
    return zerolog.New(os.Stderr).With().Timestamp().Logger()
}

// Uso:
// log.Info().Str("url", url).Int("status", 200).Msg("fetched")
// log.Error().Err(err).Str("url", url).Msg("failed to fetch")
```

### 11.2. Progress Bar

```go
// internal/utils/progress.go
func NewProgressBar(total int, description string) *progressbar.ProgressBar {
    return progressbar.NewOptions(total,
        progressbar.OptionSetDescription(description),
        progressbar.OptionShowCount(),
        progressbar.OptionShowIts(),
        progressbar.OptionSetTheme(progressbar.Theme{
            Saucer:        "█",
            SaucerPadding: "░",
            BarStart:      "[",
            BarEnd:        "]",
        }),
    )
}
```

---

## Fase 12: Graceful Shutdown

```go
// cmd/repodocs/main.go
func run(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    // Captura SIGINT/SIGTERM
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigCh
        log.Info().Msg("Shutting down gracefully...")
        cancel()
    }()

    // Executar com context
    return orchestrator.Run(ctx, config)
}
```

---

## Fase 13: Arquivo de Configuração

### 13.1. Estrutura de Diretórios

```text
~/.repodocs/
├── config.yaml          # Configurações do usuário
├── cache/               # Diretório de cache BadgerDB
│   └── badger/
└── logs/                # Logs persistentes (opcional)
```

### 13.2. Formato do Config File

```yaml
# ~/.repodocs/config.yaml

# Configurações de output
output:
  directory: "./docs" # Diretório padrão de saída
  flat: false # Estrutura flat (--nofolders)
  json_metadata: true # Gerar .json junto com .md
  overwrite: false # Sobrescrever existentes (--force)

# Configurações de concorrência
concurrency:
  workers: 5 # Número de workers paralelos
  timeout: 30s # Timeout por requisição
  max_depth: 3 # Profundidade máxima de crawl

# Configurações de cache
cache:
  enabled: true # Habilitar cache
  ttl: 24h # Tempo de vida do cache
  directory: "~/.repodocs/cache"

# Configurações de rendering
rendering:
  force_js: false # Forçar renderização JS
  js_timeout: 60s # Timeout para renderização JS
  scroll_to_end: true # Scroll para carregar lazy content

# Configurações de stealth
stealth:
  user_agent: "" # User-Agent customizado (vazio = rotação)
  random_delay_min: 1s # Delay mínimo entre requisições
  random_delay_max: 3s # Delay máximo entre requisições

# Padrões de exclusão globais
exclude:
  - ".*\\.pdf$"
  - ".*/login.*"
  - ".*/logout.*"
  - ".*/admin.*"

# Logging
logging:
  level: "info" # debug, info, warn, error
  format: "pretty" # pretty, json
```

### 13.3. Carregamento de Configuração

```go
// internal/config/loader.go
func LoadConfig() (*Config, error) {
    v := viper.New()

    // 1. Defaults
    setDefaults(v)

    // 2. Config file
    v.SetConfigName("config")
    v.SetConfigType("yaml")
    v.AddConfigPath("$HOME/.repodocs")
    v.AddConfigPath(".")

    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, err
        }
        // Config não encontrado = usar defaults
    }

    // 3. Environment variables (REPODOCS_*)
    v.SetEnvPrefix("REPODOCS")
    v.AutomaticEnv()

    // 4. Unmarshal
    var cfg Config
    if err := v.Unmarshal(&cfg); err != nil {
        return nil, err
    }

    return &cfg, nil
}
```

**Prioridade de configuração:**

1. Flags CLI (maior prioridade)
2. Environment variables (`REPODOCS_*`)
3. Config file (`~/.repodocs/config.yaml`)
4. Defaults (menor prioridade)

---

## Fase 14: Build System

### 14.1. Makefile

```makefile
# Makefile

# Variáveis
BINARY_NAME=repodocs
VERSION=$(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
BUILD_TIME=$(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
COMMIT=$(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
LDFLAGS=-ldflags "-X main.Version=$(VERSION) -X main.BuildTime=$(BUILD_TIME) -X main.Commit=$(COMMIT) -s -w"

# Go
GOCMD=go
GOBUILD=$(GOCMD) build
GOTEST=$(GOCMD) test
GOMOD=$(GOCMD) mod
GOVET=$(GOCMD) vet
GOFMT=gofmt

# Diretórios
CMD_DIR=./cmd/repodocs
BUILD_DIR=./build
COVERAGE_DIR=./coverage

.PHONY: all build clean test coverage lint fmt vet deps help

## Comandos principais

all: deps lint test build ## Executa todos os passos

build: ## Compila o binário
 @echo "Building $(BINARY_NAME)..."
 @mkdir -p $(BUILD_DIR)
 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME) $(CMD_DIR)

build-all: ## Compila para todas as plataformas
 @echo "Building for all platforms..."
 @mkdir -p $(BUILD_DIR)
 GOOS=linux GOARCH=amd64 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)-linux-amd64 $(CMD_DIR)
 GOOS=linux GOARCH=arm64 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)-linux-arm64 $(CMD_DIR)
 GOOS=darwin GOARCH=amd64 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)-darwin-amd64 $(CMD_DIR)
 GOOS=darwin GOARCH=arm64 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)-darwin-arm64 $(CMD_DIR)
 GOOS=windows GOARCH=amd64 CGO_ENABLED=0 $(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)-windows-amd64.exe $(CMD_DIR)

clean: ## Remove artefatos de build
 @echo "Cleaning..."
 @rm -rf $(BUILD_DIR)
 @rm -rf $(COVERAGE_DIR)

## Testes

test: ## Executa testes unitários
 @echo "Running tests..."
 $(GOTEST) -v -race -short ./...

test-integration: ## Executa testes de integração
 @echo "Running integration tests..."
 $(GOTEST) -v -race -run Integration ./...

test-e2e: ## Executa testes E2E
 @echo "Running E2E tests..."
 $(GOTEST) -v -race -run E2E ./tests/e2e/...

test-all: ## Executa todos os testes
 @echo "Running all tests..."
 $(GOTEST) -v -race ./...

coverage: ## Gera relatório de cobertura
 @echo "Generating coverage report..."
 @mkdir -p $(COVERAGE_DIR)
 $(GOTEST) -coverprofile=$(COVERAGE_DIR)/coverage.out ./...
 $(GOCMD) tool cover -html=$(COVERAGE_DIR)/coverage.out -o $(COVERAGE_DIR)/coverage.html
 @echo "Coverage report: $(COVERAGE_DIR)/coverage.html"

## Qualidade de código

lint: ## Executa linters
 @echo "Running linters..."
 @which golangci-lint > /dev/null || (echo "Installing golangci-lint..." && go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest)
 golangci-lint run ./...

fmt: ## Formata código
 @echo "Formatting code..."
 $(GOFMT) -s -w .

vet: ## Executa go vet
 @echo "Running go vet..."
 $(GOVET) ./...

## Dependências

deps: ## Baixa dependências
 @echo "Downloading dependencies..."
 $(GOMOD) download
 $(GOMOD) tidy

deps-update: ## Atualiza dependências
 @echo "Updating dependencies..."
 $(GOMOD) get -u ./...
 $(GOMOD) tidy

## Desenvolvimento

run: ## Executa em modo desenvolvimento
 @$(GOCMD) run $(CMD_DIR) $(ARGS)

dev: ## Watch mode (requer air)
 @which air > /dev/null || (echo "Installing air..." && go install github.com/cosmtrek/air@latest)
 air

## Instalação

install: build ## Instala no sistema
 @echo "Installing $(BINARY_NAME)..."
 @cp $(BUILD_DIR)/$(BINARY_NAME) $(GOPATH)/bin/

## Help

help: ## Mostra esta ajuda
 @grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
```

### 14.2. Arquivo .golangci.yml

```yaml
# .golangci.yml
run:
  timeout: 5m
  tests: true

linters:
  enable:
    - gofmt
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - typecheck
    - misspell
    - gocyclo
    - dupl
    - gosec
    - unconvert
    - goconst
    - gocognit

linters-settings:
  gocyclo:
    min-complexity: 15
  dupl:
    threshold: 100
  gocognit:
    min-complexity: 20
  goconst:
    min-len: 3
    min-occurrences: 3

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - dupl
        - gosec
```

### 14.3. GitHub Actions (CI)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: Install dependencies
        run: make deps

      - name: Lint
        run: make lint

      - name: Test
        run: make test

      - name: Build
        run: make build

  release:
    needs: test
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: Build all platforms
        run: make build-all

      - name: Release
        uses: softprops/action-gh-release@v1
        with:
          files: build/*
```

---

## Cronograma de Execução

### Sprint 1: Fundação & Build System

1. Setup: `go.mod`, estrutura de pastas
2. Build System: Makefile, .golangci.yml, GitHub Actions
3. Domain: Definir interfaces e models
4. Config: Structs de configuração, loader (viper), defaults

### Sprint 2: Core HTTP

5. Fetcher: Implementar tls-client wrapper
6. Retry: Backoff exponencial
7. Transport: RoundTripper para Colly

### Sprint 3: Cache & Rendering

8. Cache: BadgerDB implementation
9. Renderer: Integração rod + stealth
10. Tab Pool: Pool de tabs para concorrência
11. Detector: Detecção de SPA

### Sprint 4: Converter

12. Pipeline: Orquestrador
13. Readability: Extração de conteúdo
14. Markdown: Conversão HTML→MD
15. Encoding: Detecção e conversão de charset

### Sprint 5: Strategies (Parte 1)

16. LLMS: Implementar llms.txt parser
17. PkgGo: Implementar pkg.go.dev extractor
18. Sitemap: Implementar sitemap parser

### Sprint 6: Strategies (Parte 2)

19. Git: Implementar git clone
20. Crawler: Implementar web crawler
21. Worker Pool: Pool genérico com context

### Sprint 7: CLI & Output

22. Cobra: Setup CLI completo com todas as flags
23. Config File: Integração com ~/.repodocs/config.yaml
24. Output: Writer + JSON metadata
25. Progress: Barras de progresso

### Sprint 8: Testes & Polish

26. Unit Tests: Converter, Cache, Sanitizer, Config
27. Integration Tests: Fetcher, Renderer, Tab Pool
28. E2E Tests: Crawl, Sitemap
29. Documentação: README, exemplos de uso

---

## Validação de Stealth

### Testes Recomendados

```bash
# Testar TLS fingerprint
repodocs https://tls.browserleaks.com/json --verbose

# Testar bypass Cloudflare
repodocs https://nowsecure.nl --verbose

# Testar detecção de bot
repodocs https://bot.sannysoft.com --render-js --verbose
```

### Métricas de Sucesso

- [ ] TLS fingerprint indistinguível de Chrome real
- [ ] Bypass de Cloudflare Challenge (modo managed)
- [ ] Bypass de rate limiting básico
- [ ] Renderização correta de SPAs React/Vue/Angular
- [ ] Cache funcional com resume de downloads
- [ ] Testes com cobertura > 80%
