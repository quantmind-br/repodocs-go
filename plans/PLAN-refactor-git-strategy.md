# Plano de Implementacao: Refactor GitStrategy into Specialized Components

## Resumo Executivo

Refatorar o arquivo `internal/strategies/git.go` (688 linhas) de um "God Object" com 18+ funcoes e 5 responsabilidades distintas para um pacote modular `internal/strategies/git/` com componentes especializados. A refatoracao mantera compatibilidade total com a interface `Strategy` existente e todos os testes atuais.

## Analise de Requisitos

### Requisitos Funcionais
- [ ] Manter compatibilidade com a interface `Strategy` (Name, CanHandle, Execute)
- [ ] Preservar comportamento de fallback: archive download -> git clone
- [ ] Suportar todas as plataformas atuais (GitHub, GitLab, Bitbucket)
- [ ] Manter parsing de URLs com path/branch (ex: `/tree/main/docs`)
- [ ] Preservar logica de deteccao automatica de branch default
- [ ] Manter filtro de diretorio (DocumentExtensions, IgnoreDirs)
- [ ] Preservar processamento paralelo de arquivos
- [ ] Manter integracao com MetadataEnhancer e Writer

### Requisitos Nao-Funcionais
- [ ] **Testabilidade**: Cada componente deve ser testavel isoladamente via interfaces
- [ ] **Manutencao**: Eliminar duplicacao de regex patterns entre funcoes
- [ ] **Extensibilidade**: Facilitar adicao de novas plataformas (ex: Gitea, Codeberg)
- [ ] **Performance**: Manter ou melhorar performance atual
- [ ] **Cobertura**: Manter 100% dos testes existentes passando

## Analise Tecnica

### Arquitetura Proposta

```
internal/strategies/git/
  |
  +-- strategy.go      # Coordinator: implements Strategy interface
  |                    # Orquestra parser, fetcher(s), processor
  |
  +-- parser.go        # URL parsing, platform detection
  |                    # Centraliza todos os regex patterns
  |
  +-- fetcher.go       # Interface RepoFetcher + implementations
  |                    # ArchiveFetcher, CloneFetcher
  |
  +-- archive.go       # ArchiveFetcher: HTTP download, tar.gz extraction
  |                    # Responsabilidade unica: obter repo via archive
  |
  +-- clone.go         # CloneFetcher: go-git clone operations
  |                    # Responsabilidade unica: obter repo via clone
  |
  +-- processor.go     # File discovery and conversion
  |                    # DocumentExtensions, IgnoreDirs, processFiles
  |
  +-- types.go         # Shared types: RepoInfo, GitURLInfo, FetchResult
  |
  +-- doc.go           # Package documentation
```

### Interface Principal (RepoFetcher)

```go
// fetcher.go
package git

import "context"

// FetchResult contains the result of a repository fetch operation
type FetchResult struct {
    LocalPath string // Path to extracted/cloned repo
    Branch    string // Detected or specified branch
    Method    string // "archive" or "clone"
}

// RepoFetcher defines the interface for fetching repository contents
type RepoFetcher interface {
    // Fetch downloads/clones repository to destDir
    // Returns FetchResult with branch info and method used
    Fetch(ctx context.Context, info *RepoInfo, destDir string) (*FetchResult, error)
    
    // Name returns the fetcher name for logging
    Name() string
}
```

### Componentes Afetados

| Arquivo/Modulo | Tipo de Mudanca | Descricao |
|----------------|-----------------|-----------|
| `internal/strategies/git.go` | **Deletar** | Conteudo movido para o novo pacote |
| `internal/strategies/git/strategy.go` | **Criar** | Coordinator que implementa Strategy |
| `internal/strategies/git/parser.go` | **Criar** | URL parsing e platform detection |
| `internal/strategies/git/fetcher.go` | **Criar** | Interface RepoFetcher |
| `internal/strategies/git/archive.go` | **Criar** | ArchiveFetcher implementation |
| `internal/strategies/git/clone.go` | **Criar** | CloneFetcher implementation |
| `internal/strategies/git/processor.go` | **Criar** | File discovery e conversion |
| `internal/strategies/git/types.go` | **Criar** | Shared types |
| `internal/strategies/strategy.go` | **Modificar** | Import do novo pacote (se necessario) |
| `internal/app/orchestrator.go` | **Verificar** | Garantir compatibilidade |
| `tests/unit/strategies/git_strategy_test.go` | **Criar** | Testes para novo pacote |

### Dependencias

**Internas (existentes)**:
- `internal/domain` - Document, interfaces
- `internal/output` - Writer
- `internal/utils` - Logger, ParallelForEach, ProgressBar

**Externas (existentes)**:
- `github.com/go-git/go-git/v5` - Git operations
- `archive/tar`, `compress/gzip` - Archive extraction

**Nota**: O pacote `internal/git` existente define `Client` interface para go-git. Podemos integrar ou manter separado.

## Plano de Implementacao

### Fase 1: Preparacao e Estrutura Base
**Objetivo**: Criar estrutura do pacote e tipos compartilhados

#### Tarefas:

1. **Criar diretorio `internal/strategies/git/`**
   - Descricao: Criar o diretorio do novo pacote
   - Comando: `mkdir -p internal/strategies/git`

2. **Criar `internal/strategies/git/doc.go`**
   - Descricao: Documentacao do pacote
   - Arquivo: `internal/strategies/git/doc.go`
   ```go
   // Package git implements the git repository extraction strategy.
   //
   // It supports extracting documentation from GitHub, GitLab, and Bitbucket
   // repositories using either archive download (faster) or git clone (fallback).
   //
   // Architecture:
   //   - Strategy: Coordinator implementing strategies.Strategy interface
   //   - Parser: URL parsing and platform detection
   //   - ArchiveFetcher: HTTP-based tar.gz download and extraction
   //   - CloneFetcher: go-git based repository cloning
   //   - Processor: File discovery and document conversion
   package git
   ```

3. **Criar `internal/strategies/git/types.go`**
   - Descricao: Tipos compartilhados entre componentes
   - Arquivo: `internal/strategies/git/types.go`
   ```go
   package git
   
   // Platform represents a git hosting platform
   type Platform string
   
   const (
       PlatformGitHub    Platform = "github"
       PlatformGitLab    Platform = "gitlab"
       PlatformBitbucket Platform = "bitbucket"
       PlatformGeneric   Platform = "generic"
   )
   
   // RepoInfo contains parsed repository information
   type RepoInfo struct {
       Platform Platform
       Owner    string
       Repo     string
       URL      string // Original URL
   }
   
   // GitURLInfo contains parsed Git URL information including optional path
   type GitURLInfo struct {
       RepoURL  string   // Clean repository URL (without /tree/... suffix)
       Platform Platform
       Owner    string
       Repo     string
       Branch   string   // Branch from URL (empty if not specified)
       SubPath  string   // Subdirectory path (empty if root)
   }
   
   // FetchResult contains the result of a repository fetch operation
   type FetchResult struct {
       LocalPath string // Path to extracted/cloned repo
       Branch    string // Detected or specified branch
       Method    string // "archive" or "clone"
   }
   
   // DocumentExtensions are file extensions to process (markdown only)
   var DocumentExtensions = map[string]bool{
       ".md":  true,
       ".mdx": true,
   }
   
   // IgnoreDirs are directories to skip during file discovery
   var IgnoreDirs = map[string]bool{
       ".git":         true,
       "node_modules": true,
       "vendor":       true,
       "__pycache__":  true,
       ".venv":        true,
       "venv":         true,
       "dist":         true,
       "build":        true,
       ".next":        true,
       ".nuxt":        true,
   }
   ```

### Fase 2: Parser Component
**Objetivo**: Extrair e centralizar toda logica de parsing de URLs

#### Tarefas:

1. **Criar `internal/strategies/git/parser.go`**
   - Descricao: Centralizar parsing de URLs e deteccao de plataforma
   - Arquivo: `internal/strategies/git/parser.go`
   - Funcoes a extrair de `git.go`:
     - `parseGitURL` (linhas 363-385)
     - `parseGitURLWithPath` (linhas 257-319)
     - `normalizeFilterPath` (linhas 321-340)
     - `extractPathFromTreeURL` (linhas 342-361)
   
   ```go
   package git
   
   import (
       "fmt"
       "net/url"
       "path/filepath"
       "regexp"
       "strings"
   )
   
   // Parser handles URL parsing and platform detection
   type Parser struct {
       // Compiled regex patterns for each platform
       patterns []platformPattern
   }
   
   type platformPattern struct {
       platform    Platform
       repoPattern *regexp.Regexp
       treePattern *regexp.Regexp
   }
   
   // NewParser creates a new URL parser with pre-compiled regex patterns
   func NewParser() *Parser {
       return &Parser{
           patterns: []platformPattern{
               {
                   platform:    PlatformGitHub,
                   repoPattern: regexp.MustCompile(`^(https?://github\.com/([^/]+)/([^/]+?))(\.git)?(/|$)`),
                   treePattern: regexp.MustCompile(`/tree/([^/]+)(?:/(.+))?$`),
               },
               {
                   platform:    PlatformGitLab,
                   repoPattern: regexp.MustCompile(`^(https?://gitlab\.com/([^/]+)/([^/]+?))(\.git)?(/|$)`),
                   treePattern: regexp.MustCompile(`/-/tree/([^/]+)(?:/(.+))?$`),
               },
               {
                   platform:    PlatformBitbucket,
                   repoPattern: regexp.MustCompile(`^(https?://bitbucket\.org/([^/]+)/([^/]+?))(\.git)?(/|$)`),
                   treePattern: regexp.MustCompile(`/src/([^/]+)(?:/(.+))?$`),
               },
           },
       }
   }
   
   // ParseURL parses a git URL and extracts repository information
   func (p *Parser) ParseURL(rawURL string) (*RepoInfo, error) {
       // Implementation moved from parseGitURL
   }
   
   // ParseURLWithPath parses a git URL including optional branch and subpath
   func (p *Parser) ParseURLWithPath(rawURL string) (*GitURLInfo, error) {
       // Implementation moved from parseGitURLWithPath
   }
   
   // NormalizeFilterPath normalizes a filter path for consistent comparison
   func NormalizeFilterPath(path string) string {
       // Implementation moved from normalizeFilterPath
   }
   
   // extractPathFromTreeURL extracts subdirectory path from tree/blob URLs
   func extractPathFromTreeURL(rawURL string) string {
       // Implementation moved from extractPathFromTreeURL
   }
   ```

2. **Criar testes `internal/strategies/git/parser_test.go`**
   - Descricao: Testes unitarios para o parser
   - Migrar testes de `git_strategy_test.go`:
     - `TestParseGitURLWithPath_GitHub`
     - `TestParseGitURLWithPath_GitLab`
     - `TestParseGitURLWithPath_InvalidURL`
     - `TestNormalizeFilterPath`
     - `TestNormalizeFilterPath_WithFullURLs`

### Fase 3: Fetcher Interface e Archive Implementation
**Objetivo**: Criar interface de fetching e implementacao via archive

#### Tarefas:

1. **Criar `internal/strategies/git/fetcher.go`**
   - Descricao: Interface RepoFetcher
   - Arquivo: `internal/strategies/git/fetcher.go`
   ```go
   package git
   
   import "context"
   
   // RepoFetcher defines the interface for fetching repository contents
   type RepoFetcher interface {
       // Fetch downloads/clones repository to destDir
       Fetch(ctx context.Context, info *RepoInfo, branch, destDir string) (*FetchResult, error)
       
       // Name returns the fetcher name for logging
       Name() string
   }
   ```

2. **Criar `internal/strategies/git/archive.go`**
   - Descricao: ArchiveFetcher implementation
   - Arquivo: `internal/strategies/git/archive.go`
   - Funcoes a extrair de `git.go`:
     - `tryArchiveDownload` (linhas 212-255)
     - `buildArchiveURL` (linhas 413-430)
     - `downloadAndExtract` (linhas 432-461)
     - `extractTarGz` (linhas 463-520)
   
   ```go
   package git
   
   import (
       "archive/tar"
       "compress/gzip"
       "context"
       "fmt"
       "io"
       "net/http"
       "os"
       "path/filepath"
       "strings"
       
       "github.com/quantmind-br/repodocs-go/internal/utils"
   )
   
   // ArchiveFetcher fetches repositories via HTTP archive download
   type ArchiveFetcher struct {
       httpClient *http.Client
       logger     *utils.Logger
   }
   
   // ArchiveFetcherOptions contains options for creating an ArchiveFetcher
   type ArchiveFetcherOptions struct {
       HTTPClient *http.Client
       Logger     *utils.Logger
   }
   
   // NewArchiveFetcher creates a new ArchiveFetcher
   func NewArchiveFetcher(opts ArchiveFetcherOptions) *ArchiveFetcher {
       // Implementation
   }
   
   // Fetch implements RepoFetcher
   func (f *ArchiveFetcher) Fetch(ctx context.Context, info *RepoInfo, branch, destDir string) (*FetchResult, error) {
       // Implementation moved from tryArchiveDownload
   }
   
   // Name implements RepoFetcher
   func (f *ArchiveFetcher) Name() string {
       return "archive"
   }
   
   // buildArchiveURL constructs the archive download URL for the platform
   func (f *ArchiveFetcher) buildArchiveURL(info *RepoInfo, branch string) string {
       // Implementation moved from buildArchiveURL
   }
   
   // downloadAndExtract downloads and extracts a tar.gz archive
   func (f *ArchiveFetcher) downloadAndExtract(ctx context.Context, archiveURL, destDir string) error {
       // Implementation moved from downloadAndExtract
   }
   
   // extractTarGz extracts a tar.gz archive to destDir
   func (f *ArchiveFetcher) extractTarGz(r io.Reader, destDir string) error {
       // Implementation moved from extractTarGz
   }
   ```

3. **Criar testes `internal/strategies/git/archive_test.go`**
   - Migrar testes:
     - `TestBuildArchiveURL_GitHub`
     - `TestBuildArchiveURL_GitLab`
     - `TestBuildArchiveURL_Custom`
     - `TestDownloadAndExtract_Success`
     - `TestDownloadAndExtract_Error`
     - `TestDownloadAndExtract_Unauthorized`
     - `TestDownloadAndExtract_OtherErrorStatus`
     - `TestExtractTarGz_Invalid`

### Fase 4: Clone Fetcher Implementation
**Objetivo**: Extrair logica de git clone para componente dedicado

#### Tarefas:

1. **Criar `internal/strategies/git/clone.go`**
   - Descricao: CloneFetcher implementation
   - Arquivo: `internal/strategies/git/clone.go`
   - Funcoes a extrair de `git.go`:
     - `cloneRepository` (linhas 522-555)
     - `detectDefaultBranch` (linhas 387-411)
   
   ```go
   package git
   
   import (
       "bufio"
       "context"
       "fmt"
       "os"
       "os/exec"
       "strings"
       
       "github.com/go-git/go-git/v5"
       githttp "github.com/go-git/go-git/v5/plumbing/transport/http"
       "github.com/quantmind-br/repodocs-go/internal/utils"
   )
   
   // CloneFetcher fetches repositories via git clone
   type CloneFetcher struct {
       logger *utils.Logger
   }
   
   // CloneFetcherOptions contains options for creating a CloneFetcher
   type CloneFetcherOptions struct {
       Logger *utils.Logger
   }
   
   // NewCloneFetcher creates a new CloneFetcher
   func NewCloneFetcher(opts CloneFetcherOptions) *CloneFetcher {
       return &CloneFetcher{logger: opts.Logger}
   }
   
   // Fetch implements RepoFetcher
   func (f *CloneFetcher) Fetch(ctx context.Context, info *RepoInfo, branch, destDir string) (*FetchResult, error) {
       // Implementation moved from cloneRepository
   }
   
   // Name implements RepoFetcher
   func (f *CloneFetcher) Name() string {
       return "clone"
   }
   
   // DetectDefaultBranch uses git ls-remote to find the default branch
   func DetectDefaultBranch(ctx context.Context, url string) (string, error) {
       // Implementation moved from detectDefaultBranch
   }
   ```

2. **Criar testes `internal/strategies/git/clone_test.go`**
   - Migrar testes:
     - `TestDetectDefaultBranch_Main`
     - `TestDetectDefaultBranch_Master`
     - `TestDetectDefaultBranch_Custom`
     - `TestDetectDefaultBranch_Error`
     - `TestCloneRepository_WithGitHubToken`
     - `TestCloneRepository_HeadRefParsing`

### Fase 5: Processor Component
**Objetivo**: Extrair logica de processamento de arquivos

#### Tarefas:

1. **Criar `internal/strategies/git/processor.go`**
   - Descricao: File discovery e document conversion
   - Arquivo: `internal/strategies/git/processor.go`
   - Funcoes a extrair de `git.go`:
     - `findDocumentationFiles` (linhas 557-599)
     - `processFiles` (linhas 601-623)
     - `processFile` (linhas 625-668)
     - `extractTitleFromPath` (linhas 670-687)
   
   ```go
   package git
   
   import (
       "context"
       "io/fs"
       "os"
       "path/filepath"
       "strings"
       "time"
       
       "github.com/quantmind-br/repodocs-go/internal/domain"
       "github.com/quantmind-br/repodocs-go/internal/utils"
   )
   
   // Processor handles file discovery and document conversion
   type Processor struct {
       logger *utils.Logger
   }
   
   // ProcessorOptions contains options for creating a Processor
   type ProcessorOptions struct {
       Logger *utils.Logger
   }
   
   // NewProcessor creates a new Processor
   func NewProcessor(opts ProcessorOptions) *Processor {
       return &Processor{logger: opts.Logger}
   }
   
   // ProcessOptions contains options for processing files
   type ProcessOptions struct {
       RepoURL     string
       Branch      string
       FilterPath  string
       Concurrency int
       Limit       int
       DryRun      bool
       WriteFunc   func(ctx context.Context, doc *domain.Document) error
   }
   
   // FindDocumentationFiles finds all documentation files in the given directory
   func (p *Processor) FindDocumentationFiles(dir string, filterPath string) ([]string, error) {
       // Implementation moved from findDocumentationFiles
   }
   
   // ProcessFiles processes all documentation files
   func (p *Processor) ProcessFiles(ctx context.Context, files []string, tmpDir string, opts ProcessOptions) error {
       // Implementation moved from processFiles
   }
   
   // processFile processes a single documentation file
   func (p *Processor) processFile(ctx context.Context, path, tmpDir string, opts ProcessOptions) error {
       // Implementation moved from processFile
   }
   
   // ExtractTitleFromPath extracts a title from a file path
   func ExtractTitleFromPath(path string) string {
       // Implementation moved from extractTitleFromPath
   }
   ```

2. **Criar testes `internal/strategies/git/processor_test.go`**
   - Migrar testes:
     - `TestFindDocumentationFiles_Markdown`
     - `TestFindDocumentationFiles_MDX`
     - `TestFindDocumentationFiles_Empty`
     - `TestFindDocumentationFiles_Nested`
     - `TestFindDocumentationFiles_WithFilter`
     - `TestFindDocumentationFiles_NonExistentPath`
     - `TestFindDocumentationFiles_PathIsFile`
     - `TestProcessFiles_Success`
     - `TestProcessFiles_Invalid`
     - `TestProcessFiles_Empty`
     - `TestExtractTitleFromPath_*`

### Fase 6: Strategy Coordinator
**Objetivo**: Criar o coordenador que implementa a interface Strategy

#### Tarefas:

1. **Criar `internal/strategies/git/strategy.go`**
   - Descricao: Coordinator que implementa Strategy interface
   - Arquivo: `internal/strategies/git/strategy.go`
   
   ```go
   package git
   
   import (
       "context"
       "fmt"
       "net/http"
       "os"
       "strings"
       "time"
       
       "github.com/quantmind-br/repodocs-go/internal/strategies"
       "github.com/quantmind-br/repodocs-go/internal/utils"
   )
   
   // Strategy extracts documentation from git repositories
   // Uses archive download as primary method (faster) with git clone as fallback
   type Strategy struct {
       deps             *strategies.Dependencies
       parser           *Parser
       archiveFetcher   *ArchiveFetcher
       cloneFetcher     *CloneFetcher
       processor        *Processor
       logger           *utils.Logger
       skipBranchDetect bool
   }
   
   // NewStrategy creates a new git strategy
   func NewStrategy(deps *strategies.Dependencies) *Strategy {
       httpClient := createHTTPClient(deps)
       logger := deps.Logger
       
       return &Strategy{
           deps:   deps,
           parser: NewParser(),
           archiveFetcher: NewArchiveFetcher(ArchiveFetcherOptions{
               HTTPClient: httpClient,
               Logger:     logger,
           }),
           cloneFetcher: NewCloneFetcher(CloneFetcherOptions{
               Logger: logger,
           }),
           processor: NewProcessor(ProcessorOptions{
               Logger: logger,
           }),
           logger:           logger,
           skipBranchDetect: deps.HTTPClient != nil, // Custom client = testing
       }
   }
   
   // Name returns the strategy name
   func (s *Strategy) Name() string {
       return "git"
   }
   
   // CanHandle returns true if this strategy can handle the given URL
   func (s *Strategy) CanHandle(url string) bool {
       // Implementation moved from CanHandle
   }
   
   // Execute runs the git extraction strategy
   func (s *Strategy) Execute(ctx context.Context, rawURL string, opts strategies.Options) error {
       // Orchestration logic:
       // 1. Parse URL
       // 2. Try archive fetch, fallback to clone
       // 3. Find documentation files
       // 4. Process files
   }
   
   // createHTTPClient creates an HTTP client with appropriate settings
   func createHTTPClient(deps *strategies.Dependencies) *http.Client {
       // Implementation
   }
   ```

2. **Atualizar exportacao em `internal/strategies/`**
   - Descricao: Criar alias ou re-export para manter compatibilidade
   - Arquivo: `internal/strategies/git_alias.go` (opcional)
   ```go
   package strategies
   
   import "github.com/quantmind-br/repodocs-go/internal/strategies/git"
   
   // NewGitStrategy creates a new git strategy (alias for backward compatibility)
   func NewGitStrategy(deps *Dependencies) *git.Strategy {
       return git.NewStrategy(deps)
   }
   ```

3. **Criar testes `internal/strategies/git/strategy_test.go`**
   - Migrar testes:
     - `TestNewGitStrategy_Success`
     - `TestNewGitStrategy_WithOptions`
     - `TestCanHandle_WithTreeURL`
     - `TestExecute_*`

### Fase 7: Migracao e Limpeza
**Objetivo**: Migrar codigo, atualizar imports e remover arquivo antigo

#### Tarefas:

1. **Atualizar imports em arquivos dependentes**
   - Verificar `internal/app/orchestrator.go`
   - Verificar `internal/app/detector.go`
   - Atualizar para usar `strategies/git.NewStrategy` ou alias

2. **Remover `internal/strategies/git.go`**
   - Somente apos todos os testes passarem
   - Mover `IsWikiURL` para local apropriado (wiki.go ou helpers.go)

3. **Atualizar `internal/strategies/git_strategy_test.go`**
   - Renomear para `internal/strategies/git_test.go` (testes de integracao)
   - Ou mover testes para o novo pacote

4. **Executar suite completa de testes**
   ```bash
   make test
   make test-integration
   ```

## Estrategia de Testes

### Testes Unitarios

| Componente | Arquivo de Teste | Cobertura |
|------------|------------------|-----------|
| Parser | `parser_test.go` | URL parsing, platform detection, path normalization |
| ArchiveFetcher | `archive_test.go` | Download, extraction, error handling |
| CloneFetcher | `clone_test.go` | Git clone, branch detection, auth |
| Processor | `processor_test.go` | File discovery, document conversion |
| Strategy | `strategy_test.go` | CanHandle, Execute orchestration |

### Testes de Integracao

- [ ] `TestExecute_ArchiveDownload` - Fluxo completo via archive
- [ ] `TestExecute_CloneFallback` - Fallback para clone quando archive falha
- [ ] `TestExecute_WithFilterPath` - Filtragem de subdiretorio
- [ ] `TestExecute_WithBranchFromURL` - Branch especificado na URL

### Casos de Teste Especificos

| ID | Cenario | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | GitHub repo basico | `https://github.com/owner/repo` | Parse correto, platform=github |
| TC02 | GitHub com tree/path | `https://github.com/o/r/tree/main/docs` | branch=main, subPath=docs |
| TC03 | GitLab com tree | `https://gitlab.com/o/r/-/tree/dev/src` | platform=gitlab, branch=dev |
| TC04 | Archive 404 fallback | Repo sem archive | Fallback para clone |
| TC05 | Clone com token | GITHUB_TOKEN set | Auth headers corretos |
| TC06 | Filter path inexistente | filterPath=nonexistent | Erro claro |

## Riscos e Mitigacoes

| Risco | Probabilidade | Impacto | Mitigacao |
|-------|---------------|---------|-----------|
| Quebra de compatibilidade | Media | Alto | Criar alias `NewGitStrategy` no pacote strategies |
| Regressao em testes | Baixa | Alto | Executar testes a cada fase, manter testes existentes |
| Dependencia circular | Baixa | Medio | Tipos em `types.go` separado, interfaces claras |
| Performance degradada | Baixa | Medio | Benchmark antes/depois, evitar allocations extras |
| Duplicacao de codigo | Media | Baixo | Code review ao final, centralizar em helpers |

## Checklist de Conclusao

### Por Fase

**Fase 1: Preparacao**
- [ ] Diretorio `internal/strategies/git/` criado
- [ ] `doc.go` com documentacao do pacote
- [ ] `types.go` com tipos compartilhados
- [ ] Compila sem erros

**Fase 2: Parser**
- [ ] `parser.go` implementado
- [ ] `parser_test.go` com testes migrados
- [ ] Todos os testes de parsing passando
- [ ] Nenhum regex duplicado

**Fase 3: Archive Fetcher**
- [ ] `fetcher.go` com interface
- [ ] `archive.go` implementado
- [ ] `archive_test.go` com testes migrados
- [ ] Testes de download/extraction passando

**Fase 4: Clone Fetcher**
- [ ] `clone.go` implementado
- [ ] `clone_test.go` com testes migrados
- [ ] Testes de clone/branch detection passando

**Fase 5: Processor**
- [ ] `processor.go` implementado
- [ ] `processor_test.go` com testes migrados
- [ ] Testes de file discovery passando

**Fase 6: Strategy**
- [ ] `strategy.go` implementado
- [ ] `strategy_test.go` com testes migrados
- [ ] Alias de compatibilidade criado (se necessario)
- [ ] Testes de integracao passando

**Fase 7: Migracao**
- [ ] Imports atualizados
- [ ] `git.go` original removido
- [ ] `make test` passa
- [ ] `make test-integration` passa
- [ ] `make lint` passa

### Final

- [ ] Codigo implementado e revisado
- [ ] Todos os testes passando (unit + integration)
- [ ] Documentacao atualizada (AGENTS.md se necessario)
- [ ] Code review realizado
- [ ] Nenhum regex duplicado entre arquivos
- [ ] Performance validada (nao degradou)

## Notas Adicionais

### Ordem de Execucao Recomendada

1. **Fases 1-2** podem ser feitas juntas (preparacao + parser)
2. **Fases 3-4** podem ser paralelizadas (archive e clone sao independentes)
3. **Fase 5** depende de 1-4 (processor usa tipos de todos)
4. **Fase 6** depende de todas anteriores
5. **Fase 7** somente apos todos os testes passarem

### Integracao com `internal/git` Existente

O pacote `internal/git` existente define uma interface `Client` para go-git. Opcoes:
1. **Integrar**: Usar `git.Client` em `CloneFetcher` para facilitar mocking
2. **Manter separado**: CloneFetcher usa go-git diretamente (atual)

Recomendacao: **Integrar** - permite melhor testabilidade do CloneFetcher.

### Padroes a Seguir

Conforme `AGENTS.md`:
- Imports em 3 grupos (stdlib, external, internal)
- Interfaces: `Parser`, `RepoFetcher`, `Processor`
- Constructors: `NewParser()`, `NewArchiveFetcher(opts)`
- Options pattern: `ArchiveFetcherOptions`, `ProcessorOptions`
- Erros com contexto: `fmt.Errorf("failed to parse URL: %w", err)`
- Logging via zerolog: `s.logger.Info().Str("url", url).Msg("message")`

### Estimativa de Esforco

| Fase | Estimativa | Complexidade |
|------|------------|--------------|
| Fase 1 | 30 min | Baixa |
| Fase 2 | 1-2 horas | Media |
| Fase 3 | 1-2 horas | Media |
| Fase 4 | 1 hora | Media |
| Fase 5 | 1-2 horas | Media |
| Fase 6 | 1-2 horas | Media-Alta |
| Fase 7 | 1 hora | Baixa |
| **Total** | **7-11 horas** | |
