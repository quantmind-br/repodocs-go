# Plano de Implementacao: Project Manifest / Batch Processing

## Resumo Executivo

Implementar suporte a arquivos de manifesto YAML/JSON que definem multiplas fontes de documentacao com configuracoes individuais. Permite ingestao reprodutivel de dados para pipelines RAG complexos com um unico comando.

**Valor entregue**: Usuarios podem definir multiplas fontes em um arquivo versionavel e executar `repodocs --manifest sources.yaml` para extrair documentacao de todas as fontes de forma reprodutivel.

---

## Analise de Requisitos

### Requisitos Funcionais

- [ ] RF01: Suportar formato YAML para definicao de manifesto
- [ ] RF02: Suportar formato JSON para definicao de manifesto
- [ ] RF03: Permitir multiplas fontes (`sources`) em um unico manifesto
- [ ] RF04: Permitir configuracao individual por fonte:
  - `url` (obrigatorio)
  - `strategy` (opcional, auto-detect se omitido)
  - `content_selector` (opcional)
  - `exclude_selector` (opcional)
  - `exclude` patterns (opcional)
  - `max_depth` (opcional)
  - `include` patterns para git (opcional)
  - `render_js` (opcional)
- [ ] RF05: Permitir opcoes globais no manifesto:
  - `continue_on_error` (default: false)
  - `output` directory
  - `concurrency`
- [ ] RF06: Processar fontes sequencialmente
- [ ] RF07: Reportar progresso por fonte
- [ ] RF08: Implementar `continue_on_error: true` para falhas parciais
- [ ] RF09: Implementar `continue_on_error: false` para fail-fast

### Requisitos Nao-Funcionais

- [ ] RNF01: Validacao de schema do manifesto com mensagens de erro claras
- [ ] RNF02: Tempo de carregamento do manifesto < 100ms
- [ ] RNF03: Sem dependencias externas novas (usar gopkg.in/yaml.v3 existente)
- [ ] RNF04: Compatibilidade com flags CLI existentes (manifest pode sobrescrever)

---

## Analise Tecnica

### Arquitetura Proposta

```
+------------------+     +-------------------+     +------------------+
|   CLI (main.go)  | --> | manifest.Loader   | --> | manifest.Config  |
|  --manifest flag |     | Load(path)        |     | Sources[]        |
+------------------+     +-------------------+     | Options          |
                                                   +------------------+
                                                            |
                                                            v
                              +---------------------------+
                              |   app.Orchestrator        |
                              |   RunManifest(ctx, cfg)   |
                              |   - Itera sobre sources   |
                              |   - Aplica opcoes por src |
                              |   - Reporta progresso     |
                              +---------------------------+
```

### Componentes Afetados

| Arquivo/Modulo | Tipo de Mudanca | Descricao |
|----------------|-----------------|-----------|
| `internal/manifest/types.go` | **Criar** | Definir tipos Config, Source, Options |
| `internal/manifest/loader.go` | **Criar** | Loader YAML/JSON com validacao |
| `internal/manifest/loader_test.go` | **Criar** | Testes unitarios do loader |
| `cmd/repodocs/main.go` | **Modificar** | Adicionar flag `--manifest` |
| `internal/app/orchestrator.go` | **Modificar** | Adicionar metodo `RunManifest()` |
| `tests/unit/app/orchestrator_test.go` | **Modificar** | Testes para RunManifest |
| `tests/integration/manifest_test.go` | **Criar** | Testes de integracao |

### Dependencias

- **Internas**: `internal/app`, `internal/strategies`, `internal/config`
- **Externas**: `gopkg.in/yaml.v3` (ja existente via viper)
- **Nenhuma dependencia nova necessaria**

---

## Plano de Implementacao

### Fase 1: Definir Tipos do Manifesto

**Objetivo**: Criar estruturas de dados que representam o schema do manifesto

#### Tarefa 1.1: Criar `internal/manifest/types.go`

**Arquivos**: `internal/manifest/types.go`

```go
package manifest

import "time"

// Config representa a configuracao completa do manifesto
type Config struct {
    Sources []Source `yaml:"sources" json:"sources"`
    Options Options  `yaml:"options" json:"options"`
}

// Source representa uma fonte individual de documentacao
type Source struct {
    URL             string   `yaml:"url" json:"url"`
    Strategy        string   `yaml:"strategy,omitempty" json:"strategy,omitempty"`
    ContentSelector string   `yaml:"content_selector,omitempty" json:"content_selector,omitempty"`
    ExcludeSelector string   `yaml:"exclude_selector,omitempty" json:"exclude_selector,omitempty"`
    Exclude         []string `yaml:"exclude,omitempty" json:"exclude,omitempty"`
    Include         []string `yaml:"include,omitempty" json:"include,omitempty"`
    MaxDepth        int      `yaml:"max_depth,omitempty" json:"max_depth,omitempty"`
    RenderJS        *bool    `yaml:"render_js,omitempty" json:"render_js,omitempty"`
    Limit           int      `yaml:"limit,omitempty" json:"limit,omitempty"`
}

// Options representa opcoes globais do manifesto
type Options struct {
    ContinueOnError bool          `yaml:"continue_on_error" json:"continue_on_error"`
    Output          string        `yaml:"output,omitempty" json:"output,omitempty"`
    Concurrency     int           `yaml:"concurrency,omitempty" json:"concurrency,omitempty"`
    CacheTTL        time.Duration `yaml:"cache_ttl,omitempty" json:"cache_ttl,omitempty"`
}

// Validate valida a configuracao do manifesto
func (c *Config) Validate() error {
    if len(c.Sources) == 0 {
        return ErrNoSources
    }
    for i, src := range c.Sources {
        if src.URL == "" {
            return fmt.Errorf("source %d: %w", i, ErrEmptyURL)
        }
    }
    return nil
}

// DefaultOptions retorna opcoes padrao
func DefaultOptions() Options {
    return Options{
        ContinueOnError: false,
        Output:          "./docs",
        Concurrency:     5,
    }
}
```

#### Tarefa 1.2: Criar erros sentinela

**Arquivos**: `internal/manifest/errors.go`

```go
package manifest

import "errors"

var (
    ErrNoSources      = errors.New("manifest must contain at least one source")
    ErrEmptyURL       = errors.New("source URL cannot be empty")
    ErrInvalidFormat  = errors.New("manifest must be valid YAML or JSON")
    ErrFileNotFound   = errors.New("manifest file not found")
    ErrUnsupportedExt = errors.New("unsupported file extension (use .yaml, .yml, or .json)")
)
```

---

### Fase 2: Implementar Loader do Manifesto

**Objetivo**: Criar loader que le e valida arquivos YAML/JSON

#### Tarefa 2.1: Criar `internal/manifest/loader.go`

**Arquivos**: `internal/manifest/loader.go`

```go
package manifest

import (
    "encoding/json"
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "gopkg.in/yaml.v3"
)

// Loader carrega e valida manifestos
type Loader struct{}

// NewLoader cria um novo loader
func NewLoader() *Loader {
    return &Loader{}
}

// Load carrega um manifesto de um arquivo
func (l *Loader) Load(path string) (*Config, error) {
    // Verificar se arquivo existe
    if _, err := os.Stat(path); os.IsNotExist(err) {
        return nil, fmt.Errorf("%w: %s", ErrFileNotFound, path)
    }

    // Ler conteudo
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read manifest: %w", err)
    }

    // Determinar formato pelo extensao
    ext := strings.ToLower(filepath.Ext(path))
    
    var cfg Config
    switch ext {
    case ".yaml", ".yml":
        if err := yaml.Unmarshal(data, &cfg); err != nil {
            return nil, fmt.Errorf("%w: %v", ErrInvalidFormat, err)
        }
    case ".json":
        if err := json.Unmarshal(data, &cfg); err != nil {
            return nil, fmt.Errorf("%w: %v", ErrInvalidFormat, err)
        }
    default:
        return nil, fmt.Errorf("%w: %s", ErrUnsupportedExt, ext)
    }

    // Aplicar defaults
    if cfg.Options.Output == "" {
        cfg.Options.Output = DefaultOptions().Output
    }
    if cfg.Options.Concurrency == 0 {
        cfg.Options.Concurrency = DefaultOptions().Concurrency
    }

    // Validar
    if err := cfg.Validate(); err != nil {
        return nil, err
    }

    return &cfg, nil
}

// LoadFromBytes carrega um manifesto de bytes (util para testes)
func (l *Loader) LoadFromBytes(data []byte, format string) (*Config, error) {
    var cfg Config
    switch format {
    case "yaml":
        if err := yaml.Unmarshal(data, &cfg); err != nil {
            return nil, fmt.Errorf("%w: %v", ErrInvalidFormat, err)
        }
    case "json":
        if err := json.Unmarshal(data, &cfg); err != nil {
            return nil, fmt.Errorf("%w: %v", ErrInvalidFormat, err)
        }
    default:
        return nil, fmt.Errorf("%w: %s", ErrUnsupportedExt, format)
    }

    if cfg.Options.Output == "" {
        cfg.Options.Output = DefaultOptions().Output
    }
    if cfg.Options.Concurrency == 0 {
        cfg.Options.Concurrency = DefaultOptions().Concurrency
    }

    if err := cfg.Validate(); err != nil {
        return nil, err
    }

    return &cfg, nil
}
```

#### Tarefa 2.2: Criar testes unitarios do loader

**Arquivos**: `tests/unit/manifest/loader_test.go`

```go
package manifest_test

import (
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/quantmind-br/repodocs-go/internal/manifest"
)

func TestLoader_Load_YAML(t *testing.T) {
    tests := []struct {
        name    string
        content string
        wantErr bool
        check   func(*testing.T, *manifest.Config)
    }{
        {
            name: "valid manifest with multiple sources",
            content: `
sources:
  - url: https://docs.example.com
    strategy: crawler
    content_selector: "article.main"
    exclude:
      - "*/changelog/*"
    max_depth: 3
  - url: https://github.com/org/repo
    strategy: git
    include:
      - "docs/**/*.md"
options:
  continue_on_error: true
  output: ./knowledge-base
`,
            wantErr: false,
            check: func(t *testing.T, cfg *manifest.Config) {
                assert.Len(t, cfg.Sources, 2)
                assert.Equal(t, "https://docs.example.com", cfg.Sources[0].URL)
                assert.Equal(t, "crawler", cfg.Sources[0].Strategy)
                assert.Equal(t, "article.main", cfg.Sources[0].ContentSelector)
                assert.Equal(t, 3, cfg.Sources[0].MaxDepth)
                assert.Equal(t, []string{"*/changelog/*"}, cfg.Sources[0].Exclude)
                
                assert.Equal(t, "https://github.com/org/repo", cfg.Sources[1].URL)
                assert.Equal(t, "git", cfg.Sources[1].Strategy)
                assert.Equal(t, []string{"docs/**/*.md"}, cfg.Sources[1].Include)
                
                assert.True(t, cfg.Options.ContinueOnError)
                assert.Equal(t, "./knowledge-base", cfg.Options.Output)
            },
        },
        {
            name: "minimal valid manifest",
            content: `
sources:
  - url: https://example.com
`,
            wantErr: false,
            check: func(t *testing.T, cfg *manifest.Config) {
                assert.Len(t, cfg.Sources, 1)
                assert.Equal(t, "https://example.com", cfg.Sources[0].URL)
                assert.Equal(t, "./docs", cfg.Options.Output) // default
                assert.Equal(t, 5, cfg.Options.Concurrency)   // default
            },
        },
        {
            name:    "empty sources",
            content: `sources: []`,
            wantErr: true,
        },
        {
            name: "source without URL",
            content: `
sources:
  - strategy: crawler
`,
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Create temp file
            tmpDir := t.TempDir()
            path := filepath.Join(tmpDir, "manifest.yaml")
            err := os.WriteFile(path, []byte(tt.content), 0644)
            require.NoError(t, err)

            // Load
            loader := manifest.NewLoader()
            cfg, err := loader.Load(path)

            if tt.wantErr {
                assert.Error(t, err)
                return
            }

            require.NoError(t, err)
            require.NotNil(t, cfg)
            if tt.check != nil {
                tt.check(t, cfg)
            }
        })
    }
}

func TestLoader_Load_JSON(t *testing.T) {
    content := `{
        "sources": [
            {"url": "https://example.com", "strategy": "crawler"}
        ],
        "options": {
            "output": "./output"
        }
    }`

    tmpDir := t.TempDir()
    path := filepath.Join(tmpDir, "manifest.json")
    err := os.WriteFile(path, []byte(content), 0644)
    require.NoError(t, err)

    loader := manifest.NewLoader()
    cfg, err := loader.Load(path)

    require.NoError(t, err)
    assert.Len(t, cfg.Sources, 1)
    assert.Equal(t, "https://example.com", cfg.Sources[0].URL)
    assert.Equal(t, "./output", cfg.Options.Output)
}

func TestLoader_Load_FileNotFound(t *testing.T) {
    loader := manifest.NewLoader()
    _, err := loader.Load("/nonexistent/manifest.yaml")

    assert.ErrorIs(t, err, manifest.ErrFileNotFound)
}

func TestLoader_Load_UnsupportedExtension(t *testing.T) {
    tmpDir := t.TempDir()
    path := filepath.Join(tmpDir, "manifest.txt")
    err := os.WriteFile(path, []byte("content"), 0644)
    require.NoError(t, err)

    loader := manifest.NewLoader()
    _, err = loader.Load(path)

    assert.ErrorIs(t, err, manifest.ErrUnsupportedExt)
}
```

---

### Fase 3: Integrar com CLI

**Objetivo**: Adicionar flag `--manifest` ao CLI e processar manifesto

#### Tarefa 3.1: Adicionar flag `--manifest` em `cmd/repodocs/main.go`

**Arquivos**: `cmd/repodocs/main.go`

Modificacoes necessarias:

```go
// Adicionar na secao de imports:
import (
    // ... imports existentes ...
    "github.com/quantmind-br/repodocs-go/internal/manifest"
)

// Adicionar variavel global:
var manifestPath string

// Modificar func init() - adicionar flag:
func init() {
    // ... flags existentes ...
    
    // Manifest flag
    rootCmd.PersistentFlags().StringVar(&manifestPath, "manifest", "", "Path to manifest file (YAML/JSON)")
    
    // ... resto da funcao ...
}

// Modificar func run() - tratar manifesto:
func run(cmd *cobra.Command, args []string) error {
    // ... inicializacao do logger ...
    
    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        return fmt.Errorf("failed to load config: %w", err)
    }

    // Check if manifest was provided
    if manifestPath != "" {
        return runManifest(cmd, cfg)
    }

    // ... resto do codigo existente para URL unica ...
}

// Nova funcao para processar manifesto:
func runManifest(cmd *cobra.Command, cfg *config.Config) error {
    // Load manifest
    loader := manifest.NewLoader()
    manifestCfg, err := loader.Load(manifestPath)
    if err != nil {
        return fmt.Errorf("failed to load manifest: %w", err)
    }

    // Override config with manifest options
    if manifestCfg.Options.Output != "" {
        cfg.Output.Directory = manifestCfg.Options.Output
    }
    if manifestCfg.Options.Concurrency > 0 {
        cfg.Concurrency.Workers = manifestCfg.Options.Concurrency
    }

    // Create context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle graceful shutdown
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigCh
        log.Info().Msg("Shutting down gracefully...")
        cancel()
    }()

    // Get common flags
    dryRun, _ := cmd.Flags().GetBool("dry-run")
    renderJS, _ := cmd.Flags().GetBool("render-js")
    force, _ := cmd.Flags().GetBool("force")

    // Create orchestrator
    orchOpts := app.OrchestratorOptions{
        CommonOptions: domain.CommonOptions{
            Verbose:  verbose,
            DryRun:   dryRun,
            Force:    force,
            RenderJS: renderJS,
        },
        Config: cfg,
    }

    orchestrator, err := app.NewOrchestrator(orchOpts)
    if err != nil {
        return fmt.Errorf("failed to create orchestrator: %w", err)
    }
    defer orchestrator.Close()

    // Run manifest
    return orchestrator.RunManifest(ctx, manifestCfg, orchOpts)
}
```

---

### Fase 4: Implementar Execucao do Manifesto no Orchestrator

**Objetivo**: Adicionar metodo `RunManifest()` que itera sobre fontes

#### Tarefa 4.1: Adicionar `RunManifest()` em `internal/app/orchestrator.go`

**Arquivos**: `internal/app/orchestrator.go`

```go
// Adicionar import:
import (
    // ... imports existentes ...
    "github.com/quantmind-br/repodocs-go/internal/manifest"
)

// Adicionar metodo RunManifest:

// ManifestResult representa o resultado da execucao de uma fonte do manifesto
type ManifestResult struct {
    Source manifest.Source
    Error  error
}

// RunManifest executa todas as fontes definidas no manifesto
func (o *Orchestrator) RunManifest(ctx context.Context, manifestCfg *manifest.Config, baseOpts OrchestratorOptions) error {
    startTime := time.Now()
    totalSources := len(manifestCfg.Sources)
    
    o.logger.Info().
        Int("sources", totalSources).
        Bool("continue_on_error", manifestCfg.Options.ContinueOnError).
        Str("output", manifestCfg.Options.Output).
        Msg("Starting manifest execution")

    var results []ManifestResult
    var firstError error

    for i, source := range manifestCfg.Sources {
        // Check for cancellation
        if ctx.Err() != nil {
            o.logger.Warn().Msg("Manifest execution cancelled")
            return ctx.Err()
        }

        o.logger.Info().
            Int("source", i+1).
            Int("total", totalSources).
            Str("url", source.URL).
            Str("strategy", source.Strategy).
            Msg("Processing source")

        // Build options for this source
        opts := o.buildSourceOptions(source, baseOpts)

        // Run extraction for this source
        err := o.Run(ctx, source.URL, opts)
        
        result := ManifestResult{Source: source, Error: err}
        results = append(results, result)

        if err != nil {
            o.logger.Error().
                Err(err).
                Str("url", source.URL).
                Msg("Source extraction failed")

            if firstError == nil {
                firstError = err
            }

            if !manifestCfg.Options.ContinueOnError {
                o.logger.Warn().Msg("Stopping execution due to continue_on_error=false")
                return fmt.Errorf("source %s failed: %w", source.URL, err)
            }
        } else {
            o.logger.Info().
                Str("url", source.URL).
                Msg("Source extraction completed")
        }
    }

    // Log summary
    duration := time.Since(startTime)
    successCount := 0
    for _, r := range results {
        if r.Error == nil {
            successCount++
        }
    }

    o.logger.Info().
        Dur("duration", duration).
        Int("total", totalSources).
        Int("success", successCount).
        Int("failed", totalSources-successCount).
        Msg("Manifest execution completed")

    // If continue_on_error was true and there were errors, return the first error
    if firstError != nil {
        return fmt.Errorf("manifest completed with errors: %w", firstError)
    }

    return nil
}

// buildSourceOptions constroi OrchestratorOptions a partir de uma Source
func (o *Orchestrator) buildSourceOptions(source manifest.Source, baseOpts OrchestratorOptions) OrchestratorOptions {
    opts := baseOpts

    // Apply source-specific overrides
    if source.ContentSelector != "" {
        opts.ContentSelector = source.ContentSelector
    }
    if source.ExcludeSelector != "" {
        opts.ExcludeSelector = source.ExcludeSelector
    }
    if len(source.Exclude) > 0 {
        opts.ExcludePatterns = append(opts.ExcludePatterns, source.Exclude...)
    }
    if source.RenderJS != nil {
        opts.RenderJS = *source.RenderJS
    }
    if source.Limit > 0 {
        opts.Limit = source.Limit
    }

    // Note: MaxDepth needs to be passed through config
    // This would require modifying how we handle per-source config

    return opts
}
```

#### Tarefa 4.2: Modificar `Run()` para suportar strategy override

Para respeitar o campo `strategy` do manifesto quando especificado:

```go
// Modificar Run() para aceitar strategy override
func (o *Orchestrator) Run(ctx context.Context, url string, opts OrchestratorOptions) error {
    startTime := time.Now()

    o.logger.Info().
        Str("url", url).
        Str("output", o.config.Output.Directory).
        Int("concurrency", o.config.Concurrency.Workers).
        Msg("Starting documentation extraction")

    // Detect or use specified strategy
    var strategyType StrategyType
    if opts.StrategyOverride != "" {
        strategyType = StrategyType(opts.StrategyOverride)
        o.logger.Debug().
            Str("strategy", string(strategyType)).
            Msg("Using specified strategy from manifest")
    } else {
        strategyType = DetectStrategy(url)
        o.logger.Debug().
            Str("strategy", string(strategyType)).
            Msg("Detected strategy type")
    }

    // ... resto do metodo permanece igual ...
}
```

Adicionar campo em OrchestratorOptions:

```go
type OrchestratorOptions struct {
    domain.CommonOptions
    Config           *config.Config
    Split            bool
    IncludeAssets    bool
    ContentSelector  string
    ExcludeSelector  string
    ExcludePatterns  []string
    FilterURL        string
    StrategyFactory  func(StrategyType, *strategies.Dependencies) strategies.Strategy
    StrategyOverride string // NEW: Override detected strategy
}
```

---

### Fase 5: Implementar Progress Reporting

**Objetivo**: Reportar progresso por fonte durante execucao do manifesto

#### Tarefa 5.1: Melhorar logging de progresso

Ja implementado na Fase 4 com logs estruturados. Adicionar barra de progresso opcional:

```go
// Em RunManifest, antes do loop:
bar := utils.NewProgressBar(int64(totalSources), utils.DescExtracting)
defer bar.Finish()

// Dentro do loop, apos processar cada fonte:
bar.Add(1)
```

---

### Fase 6: Testes

**Objetivo**: Garantir cobertura de testes para todas as funcionalidades

#### Tarefa 6.1: Testes unitarios do manifest package

**Arquivos**: `tests/unit/manifest/loader_test.go` (ja definido na Tarefa 2.2)

#### Tarefa 6.2: Testes do orchestrator com manifesto

**Arquivos**: `tests/unit/app/orchestrator_manifest_test.go`

```go
package app_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/quantmind-br/repodocs-go/internal/app"
    "github.com/quantmind-br/repodocs-go/internal/config"
    "github.com/quantmind-br/repodocs-go/internal/domain"
    "github.com/quantmind-br/repodocs-go/internal/manifest"
    "github.com/quantmind-br/repodocs-go/internal/strategies"
)

func TestOrchestrator_RunManifest_AllSuccess(t *testing.T) {
    // Arrange
    cfg := config.Default()
    cfg.Cache.Enabled = false

    manifestCfg := &manifest.Config{
        Sources: []manifest.Source{
            {URL: "https://example1.com"},
            {URL: "https://example2.com"},
        },
        Options: manifest.Options{
            ContinueOnError: false,
            Output:          t.TempDir(),
        },
    }

    executedURLs := []string{}
    mockStrategy := &testStrategy{
        name:      "mock",
        canHandle: true,
        execFunc: func(ctx context.Context, url string, opts strategies.Options) error {
            executedURLs = append(executedURLs, url)
            return nil
        },
    }

    orchestrator := createTestOrchestrator(t, cfg, mockStrategy)
    defer orchestrator.Close()

    // Act
    err := orchestrator.RunManifest(context.Background(), manifestCfg, app.OrchestratorOptions{Config: cfg})

    // Assert
    require.NoError(t, err)
    assert.Len(t, executedURLs, 2)
    assert.Contains(t, executedURLs, "https://example1.com")
    assert.Contains(t, executedURLs, "https://example2.com")
}

func TestOrchestrator_RunManifest_ContinueOnError_True(t *testing.T) {
    // Arrange
    cfg := config.Default()
    cfg.Cache.Enabled = false

    manifestCfg := &manifest.Config{
        Sources: []manifest.Source{
            {URL: "https://fail.com"},
            {URL: "https://success.com"},
        },
        Options: manifest.Options{
            ContinueOnError: true,
            Output:          t.TempDir(),
        },
    }

    executedURLs := []string{}
    mockStrategy := &testStrategy{
        name:      "mock",
        canHandle: true,
        execFunc: func(ctx context.Context, url string, opts strategies.Options) error {
            executedURLs = append(executedURLs, url)
            if url == "https://fail.com" {
                return fmt.Errorf("simulated failure")
            }
            return nil
        },
    }

    orchestrator := createTestOrchestrator(t, cfg, mockStrategy)
    defer orchestrator.Close()

    // Act
    err := orchestrator.RunManifest(context.Background(), manifestCfg, app.OrchestratorOptions{Config: cfg})

    // Assert
    assert.Error(t, err) // Should report error but continue
    assert.Contains(t, err.Error(), "manifest completed with errors")
    assert.Len(t, executedURLs, 2) // Both sources were attempted
}

func TestOrchestrator_RunManifest_ContinueOnError_False(t *testing.T) {
    // Arrange
    cfg := config.Default()
    cfg.Cache.Enabled = false

    manifestCfg := &manifest.Config{
        Sources: []manifest.Source{
            {URL: "https://fail.com"},
            {URL: "https://success.com"},
        },
        Options: manifest.Options{
            ContinueOnError: false,
            Output:          t.TempDir(),
        },
    }

    executedURLs := []string{}
    mockStrategy := &testStrategy{
        name:      "mock",
        canHandle: true,
        execFunc: func(ctx context.Context, url string, opts strategies.Options) error {
            executedURLs = append(executedURLs, url)
            if url == "https://fail.com" {
                return fmt.Errorf("simulated failure")
            }
            return nil
        },
    }

    orchestrator := createTestOrchestrator(t, cfg, mockStrategy)
    defer orchestrator.Close()

    // Act
    err := orchestrator.RunManifest(context.Background(), manifestCfg, app.OrchestratorOptions{Config: cfg})

    // Assert
    assert.Error(t, err)
    assert.Len(t, executedURLs, 1) // Only first source was attempted
}

func TestOrchestrator_RunManifest_ContextCancellation(t *testing.T) {
    // Arrange
    cfg := config.Default()
    cfg.Cache.Enabled = false

    manifestCfg := &manifest.Config{
        Sources: []manifest.Source{
            {URL: "https://example1.com"},
            {URL: "https://example2.com"},
        },
        Options: manifest.Options{
            Output: t.TempDir(),
        },
    }

    ctx, cancel := context.WithCancel(context.Background())
    cancel() // Cancel immediately

    orchestrator := createTestOrchestrator(t, cfg, nil)
    defer orchestrator.Close()

    // Act
    err := orchestrator.RunManifest(ctx, manifestCfg, app.OrchestratorOptions{Config: cfg})

    // Assert
    assert.ErrorIs(t, err, context.Canceled)
}
```

#### Tarefa 6.3: Teste de integracao

**Arquivos**: `tests/integration/manifest_test.go`

```go
package integration_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/quantmind-br/repodocs-go/internal/app"
    "github.com/quantmind-br/repodocs-go/internal/config"
    "github.com/quantmind-br/repodocs-go/internal/manifest"
)

func TestManifest_Integration_MultipleWebSources(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test in short mode")
    }

    // Setup mock servers
    server1 := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(`<html><head><title>Site 1</title></head><body><p>Content 1</p></body></html>`))
    }))
    defer server1.Close()

    server2 := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(`<html><head><title>Site 2</title></head><body><p>Content 2</p></body></html>`))
    }))
    defer server2.Close()

    // Create manifest
    outputDir := t.TempDir()
    manifestContent := fmt.Sprintf(`
sources:
  - url: %s
    strategy: crawler
  - url: %s
    strategy: crawler
options:
  output: %s
  continue_on_error: true
`, server1.URL, server2.URL, outputDir)

    manifestPath := filepath.Join(t.TempDir(), "manifest.yaml")
    err := os.WriteFile(manifestPath, []byte(manifestContent), 0644)
    require.NoError(t, err)

    // Load and run
    loader := manifest.NewLoader()
    manifestCfg, err := loader.Load(manifestPath)
    require.NoError(t, err)

    cfg := config.Default()
    cfg.Cache.Enabled = false
    cfg.Output.Directory = outputDir

    orchestrator, err := app.NewOrchestrator(app.OrchestratorOptions{Config: cfg})
    require.NoError(t, err)
    defer orchestrator.Close()

    err = orchestrator.RunManifest(context.Background(), manifestCfg, app.OrchestratorOptions{Config: cfg})

    // Assert
    require.NoError(t, err)
    
    // Check output files exist
    files, err := filepath.Glob(filepath.Join(outputDir, "**/*.md"))
    assert.NoError(t, err)
    assert.GreaterOrEqual(t, len(files), 2)
}
```

---

## Estrategia de Testes

### Testes Unitarios

| Componente | Arquivo de Teste | Cobertura |
|------------|------------------|-----------|
| Loader YAML/JSON | `tests/unit/manifest/loader_test.go` | Parsing, validacao, erros |
| Config validation | `tests/unit/manifest/types_test.go` | Validate(), defaults |
| Orchestrator.RunManifest | `tests/unit/app/orchestrator_manifest_test.go` | Todos os cenarios |

### Testes de Integracao

| Cenario | Arquivo |
|---------|---------|
| Multi-source web | `tests/integration/manifest_test.go` |
| Mixed strategies | `tests/integration/manifest_test.go` |
| Error handling | `tests/integration/manifest_test.go` |

### Casos de Teste Especificos

| ID | Cenario | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | Manifesto YAML valido com 2 fontes | YAML com 2 URLs | Ambas processadas |
| TC02 | Manifesto JSON valido | JSON com 1 URL | Processada com sucesso |
| TC03 | Manifesto sem fontes | `sources: []` | ErrNoSources |
| TC04 | Fonte sem URL | `- strategy: crawler` | ErrEmptyURL |
| TC05 | Arquivo nao encontrado | Path invalido | ErrFileNotFound |
| TC06 | Extensao invalida | `.txt` | ErrUnsupportedExt |
| TC07 | continue_on_error=true | 1 fonte falha | Continua, retorna erro |
| TC08 | continue_on_error=false | 1 fonte falha | Para imediatamente |
| TC09 | Contexto cancelado | ctx.Done() | context.Canceled |
| TC10 | Strategy override | `strategy: git` | Usa git, nao auto-detect |

---

## Riscos e Mitigacoes

| Risco | Probabilidade | Impacto | Mitigacao |
|-------|---------------|---------|-----------|
| Conflito entre flags CLI e manifesto | Medio | Baixo | Definir precedencia clara: manifesto > CLI > config |
| Performance com muitas fontes | Baixo | Medio | Processamento sequencial evita sobrecarga |
| Validacao de schema incompleta | Medio | Medio | Validar todos os campos obrigatorios e tipos |
| Erros silenciosos em continue_on_error | Medio | Alto | Log claro de todas as falhas, retornar primeiro erro |

---

## Checklist de Conclusao

### Implementacao
- [ ] `internal/manifest/types.go` criado e testado
- [ ] `internal/manifest/errors.go` criado
- [ ] `internal/manifest/loader.go` criado e testado
- [ ] Flag `--manifest` adicionada ao CLI
- [ ] `RunManifest()` implementado no orchestrator
- [ ] `buildSourceOptions()` implementado
- [ ] Progress reporting implementado

### Testes
- [ ] Testes unitarios para manifest package
- [ ] Testes unitarios para orchestrator.RunManifest
- [ ] Testes de integracao para multi-source
- [ ] Cobertura >= 80%

### Documentacao
- [ ] README atualizado com exemplo de manifesto
- [ ] Comentarios godoc em funcoes publicas

### Qualidade
- [ ] `make lint` passa
- [ ] `make test` passa
- [ ] `make build` passa
- [ ] Code review realizado

---

## Notas Adicionais

### Precedencia de Configuracao

1. **Mais alta**: Flags CLI explicitas (`--output`, `--concurrency`)
2. **Media**: Campos do manifesto (`options.output`, `sources[].content_selector`)
3. **Mais baixa**: Config file (`~/.repodocs/config.yaml`)

### Extensibilidade Futura

O design permite adicionar facilmente:
- Processamento paralelo de fontes (via flag `parallel: true`)
- Estrategias de retry por fonte
- Webhook notifications ao completar
- Dry-run por fonte individual

### Exemplo de Uso

```bash
# Criar manifesto
cat > sources.yaml << EOF
sources:
  - url: https://docs.example.com
    content_selector: "article.main"
    max_depth: 3
  - url: https://github.com/org/repo
    strategy: git
    include:
      - "docs/**/*.md"
options:
  output: ./knowledge-base
  continue_on_error: true
EOF

# Executar
repodocs --manifest sources.yaml

# Com flags adicionais
repodocs --manifest sources.yaml --verbose --dry-run
```
