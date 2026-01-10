# Plano de Implementação: Consolidate Configuration Options

## Resumo Executivo

Consolidar campos duplicados de configuração (`Verbose`, `DryRun`, `Force`, `Limit`, `RenderJS`, etc.) que existem em `OrchestratorOptions`, `strategies.Options`, `domain.StrategyOptions`, e `DependencyOptions` usando o padrão de struct embedding. Isso reduzirá o acoplamento, eliminará a cópia manual de campos, e tornará a adição de novas opções comuns um processo de 1-2 alterações em vez de 7+.

## Análise de Requisitos

### Requisitos Funcionais
- [ ] Criar struct `CommonOptions` em `internal/domain/options.go` com campos compartilhados
- [ ] `OrchestratorOptions` deve embutir `CommonOptions`
- [ ] `strategies.Options` deve embutir `CommonOptions`
- [ ] `DependencyOptions` deve embutir `CommonOptions`
- [ ] Eliminar `domain.StrategyOptions` (duplicado de `strategies.Options`)
- [ ] CLI e comportamento de runtime devem permanecer inalterados
- [ ] Adicionar nova opção comum deve requerer mudança em apenas 1-2 locais

### Requisitos Não-Funcionais
- [ ] Manter compatibilidade com testes existentes
- [ ] Não quebrar a interface `Strategy` em `internal/domain`
- [ ] Preservar padrões de inicialização existentes

## Análise Técnica

### Campos Duplicados Identificados

| Campo | OrchestratorOptions | strategies.Options | domain.StrategyOptions | DependencyOptions |
|-------|:-------------------:|:------------------:|:----------------------:|:-----------------:|
| Verbose | ✅ | ✅ | ✅ | ✅ |
| DryRun | ✅ | ✅ | ✅ | ✅ |
| Force | ✅ | ✅ | ✅ | ✅ |
| Limit | ✅ | ✅ | ✅ | ❌ |
| RenderJS | ✅ | ✅ | ✅ | ❌ |
| Split | ✅ | ✅ | ✅ | ❌ |
| IncludeAssets | ✅ | ✅ | ✅ | ❌ |
| ContentSelector | ✅ | ✅ | ✅ | ✅ |
| ExcludeSelector | ✅ | ✅ | ❌ | ✅ |
| Concurrency | ❌ | ✅ | ✅ | ✅ |

### Arquitetura Proposta

```
┌─────────────────────────────────────────────────────────────────┐
│                    internal/domain/options.go                    │
├─────────────────────────────────────────────────────────────────┤
│  CommonOptions struct {                                          │
│      Verbose  bool                                               │
│      DryRun   bool                                               │
│      Force    bool                                               │
│      RenderJS bool                                               │
│      Limit    int                                                │
│  }                                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│ OrchestratorOpts │ │ strategies.Opts  │ │  DependencyOptions   │
│ (app/orchestrator│ │ (strategies/     │ │  (strategies/        │
│  .go)            │ │  strategy.go)    │ │   strategy.go)       │
├──────────────────┤ ├──────────────────┤ ├──────────────────────┤
│ CommonOptions    │ │ CommonOptions    │ │ CommonOptions        │
│ Config           │ │ Output           │ │ Timeout              │
│ Split            │ │ Concurrency      │ │ EnableCache          │
│ IncludeAssets    │ │ MaxDepth         │ │ CacheTTL             │
│ ContentSelector  │ │ Exclude          │ │ CacheDir             │
│ ExcludeSelector  │ │ NoFolders        │ │ UserAgent            │
│ ExcludePatterns  │ │ Split            │ │ EnableRenderer       │
│ FilterURL        │ │ IncludeAssets    │ │ RendererTimeout      │
│ StrategyFactory  │ │ ContentSelector  │ │ Concurrency          │
└──────────────────┘ │ ExcludeSelector  │ │ ContentSelector      │
                     │ CacheTTL         │ │ ExcludeSelector      │
                     │ FilterURL        │ │ OutputDir            │
                     └──────────────────┘ │ Flat                 │
                                          │ JSONMetadata         │
                                          │ LLMConfig            │
                                          │ SourceURL            │
                                          └──────────────────────┘
```

### Componentes Afetados

| Arquivo/Módulo | Tipo de Mudança | Descrição |
|----------------|-----------------|-----------|
| `internal/domain/options.go` | **Criar** | Novo arquivo com `CommonOptions` struct |
| `internal/domain/interfaces.go` | **Modificar** | Remover `StrategyOptions`, atualizar interface `Strategy` |
| `internal/app/orchestrator.go` | **Modificar** | `OrchestratorOptions` embute `CommonOptions` |
| `internal/strategies/strategy.go` | **Modificar** | `Options` e `DependencyOptions` embutem `CommonOptions` |
| `cmd/repodocs/main.go` | **Modificar** | Atualizar criação de `OrchestratorOptions` |
| `tests/mocks/domain.go` | **Modificar** | Atualizar mock para nova interface |
| `internal/domain/models_test.go` | **Modificar** | Atualizar/remover `TestStrategyOptions` |
| `tests/unit/renderer_test.go` | **Modificar** | Atualizar `MockBrowserRenderer` |
| Vários testes de strategy | **Modificar** | Atualizar imports e uso de opções |

### Dependências
- Nenhuma dependência externa nova
- Dependência interna: `internal/domain` → usado por `internal/app` e `internal/strategies`

## Plano de Implementação

### Fase 1: Criar CommonOptions e Refatorar Domain
**Objetivo**: Estabelecer a base com `CommonOptions` e limpar a interface `Strategy`

#### Tarefa 1.1: Criar `internal/domain/options.go`

**Descrição**: Criar novo arquivo com struct `CommonOptions` contendo campos verdadeiramente comuns a todas as option structs.

**Arquivos envolvidos**: `internal/domain/options.go` (novo)

**Código**:
```go
// internal/domain/options.go
package domain

// CommonOptions contains configuration options shared across orchestrator,
// strategies, and dependencies. Embed this struct to inherit common fields.
type CommonOptions struct {
    // Verbose enables detailed logging output
    Verbose bool
    // DryRun simulates execution without writing files
    DryRun bool
    // Force overwrites existing files without prompting
    Force bool
    // RenderJS enables JavaScript rendering via headless browser
    RenderJS bool
    // Limit caps the maximum number of pages to process (0=unlimited)
    Limit int
}

// DefaultCommonOptions returns sensible defaults for CommonOptions
func DefaultCommonOptions() CommonOptions {
    return CommonOptions{
        Verbose:  false,
        DryRun:   false,
        Force:    false,
        RenderJS: false,
        Limit:    0,
    }
}
```

#### Tarefa 1.2: Atualizar `internal/domain/interfaces.go`

**Descrição**: Remover `StrategyOptions` duplicado e atualizar a interface `Strategy` para usar `strategies.Options` diretamente (via import cycle workaround ou definição inline).

**NOTA IMPORTANTE**: Existe um problema de import cycle aqui. `domain.Strategy` usa `StrategyOptions`, mas `strategies.Options` está em `internal/strategies`. A solução é:
- Opção A: Manter `domain.StrategyOptions` mas fazer embed de `CommonOptions`
- Opção B: Mover a interface `Strategy` para `internal/strategies`

**Decisão**: Opção A - manter `domain.StrategyOptions` com embed de `CommonOptions` para evitar quebra de interface pública.

**Arquivos envolvidos**: `internal/domain/interfaces.go`

**Código**:
```go
// Em internal/domain/interfaces.go, atualizar StrategyOptions:

// StrategyOptions contains options for strategy execution.
// This is the public interface version; internal/strategies.Options
// extends this with additional fields.
type StrategyOptions struct {
    CommonOptions         // Embedded common options
    Output          string
    Concurrency     int
    MaxDepth        int
    Exclude         []string
    NoFolders       bool
    Split           bool
    IncludeAssets   bool
    ContentSelector string
    ExcludeSelector string
    FilterURL       string
}
```

### Fase 2: Refatorar OrchestratorOptions
**Objetivo**: Fazer `OrchestratorOptions` embutir `CommonOptions`

#### Tarefa 2.1: Atualizar `internal/app/orchestrator.go`

**Descrição**: Modificar `OrchestratorOptions` para embutir `domain.CommonOptions` em vez de declarar campos duplicados.

**Arquivos envolvidos**: `internal/app/orchestrator.go`

**Antes**:
```go
type OrchestratorOptions struct {
    Config          *config.Config
    Verbose         bool      // DUPLICADO
    DryRun          bool      // DUPLICADO
    Force           bool      // DUPLICADO
    RenderJS        bool      // DUPLICADO
    Split           bool
    IncludeAssets   bool
    Limit           int       // DUPLICADO
    ContentSelector string
    ExcludeSelector string
    ExcludePatterns []string
    FilterURL       string
    StrategyFactory func(StrategyType, *strategies.Dependencies) strategies.Strategy
}
```

**Depois**:
```go
type OrchestratorOptions struct {
    domain.CommonOptions              // Embedded: Verbose, DryRun, Force, RenderJS, Limit
    Config          *config.Config
    Split           bool
    IncludeAssets   bool
    ContentSelector string
    ExcludeSelector string
    ExcludePatterns []string
    FilterURL       string
    StrategyFactory func(StrategyType, *strategies.Dependencies) strategies.Strategy
}
```

#### Tarefa 2.2: Atualizar uso em `NewOrchestrator` e `Run`

**Descrição**: Ajustar código que referencia campos diretamente para usar campos embebidos.

**Arquivos envolvidos**: `internal/app/orchestrator.go`

**Mudanças**:
- `opts.Verbose` continua funcionando (campos embebidos são promovidos)
- `opts.DryRun` continua funcionando
- Atualizar passagem para `DependencyOptions`:
  ```go
  deps, err := strategies.NewDependencies(strategies.DependencyOptions{
      CommonOptions:   opts.CommonOptions,  // Passar bloco inteiro
      Timeout:         cfg.Concurrency.Timeout,
      // ... resto
  })
  ```

### Fase 3: Refatorar strategies.Options e DependencyOptions
**Objetivo**: Fazer ambas structs embutirem `CommonOptions`

#### Tarefa 3.1: Atualizar `strategies.Options`

**Descrição**: Modificar `Options` em `internal/strategies/strategy.go` para embutir `domain.CommonOptions`.

**Arquivos envolvidos**: `internal/strategies/strategy.go`

**Antes**:
```go
type Options struct {
    Output          string
    Concurrency     int
    Limit           int       // DUPLICADO
    MaxDepth        int
    Exclude         []string
    NoFolders       bool
    DryRun          bool      // DUPLICADO
    Verbose         bool      // DUPLICADO
    Force           bool      // DUPLICADO
    RenderJS        bool      // DUPLICADO
    Split           bool
    IncludeAssets   bool
    ContentSelector string
    ExcludeSelector string
    CacheTTL        string
    FilterURL       string
}
```

**Depois**:
```go
type Options struct {
    domain.CommonOptions              // Embedded: Verbose, DryRun, Force, RenderJS, Limit
    Output          string
    Concurrency     int
    MaxDepth        int
    Exclude         []string
    NoFolders       bool
    Split           bool
    IncludeAssets   bool
    ContentSelector string
    ExcludeSelector string
    CacheTTL        string
    FilterURL       string
}
```

#### Tarefa 3.2: Atualizar `DependencyOptions`

**Descrição**: Modificar `DependencyOptions` para embutir `domain.CommonOptions`.

**Arquivos envolvidos**: `internal/strategies/strategy.go`

**Antes**:
```go
type DependencyOptions struct {
    Timeout         time.Duration
    EnableCache     bool
    CacheTTL        time.Duration
    CacheDir        string
    UserAgent       string
    EnableRenderer  bool
    RendererTimeout time.Duration
    Concurrency     int
    ContentSelector string
    ExcludeSelector string
    OutputDir       string
    Flat            bool
    JSONMetadata    bool
    Force           bool      // DUPLICADO
    DryRun          bool      // DUPLICADO
    Verbose         bool      // DUPLICADO
    LLMConfig       *config.LLMConfig
    SourceURL       string
}
```

**Depois**:
```go
type DependencyOptions struct {
    domain.CommonOptions              // Embedded: Verbose, DryRun, Force, RenderJS, Limit
    Timeout         time.Duration
    EnableCache     bool
    CacheTTL        time.Duration
    CacheDir        string
    UserAgent       string
    EnableRenderer  bool
    RendererTimeout time.Duration
    Concurrency     int
    ContentSelector string
    ExcludeSelector string
    OutputDir       string
    Flat            bool
    JSONMetadata    bool
    LLMConfig       *config.LLMConfig
    SourceURL       string
}
```

#### Tarefa 3.3: Atualizar `DefaultOptions()`

**Descrição**: Modificar a função para usar `DefaultCommonOptions()`.

**Arquivos envolvidos**: `internal/strategies/strategy.go`

**Código**:
```go
func DefaultOptions() Options {
    return Options{
        CommonOptions: domain.DefaultCommonOptions(),
        Output:        "./docs",
        Concurrency:   5,
        MaxDepth:      3,
        NoFolders:     false,
        Split:         false,
    }
}
```

### Fase 4: Atualizar Orchestrator Run Method
**Objetivo**: Simplificar a passagem de opções entre orchestrator e strategy

#### Tarefa 4.1: Atualizar `Orchestrator.Run()`

**Descrição**: Simplificar a construção de `strategyOpts` usando spread do `CommonOptions` embebido.

**Arquivos envolvidos**: `internal/app/orchestrator.go`

**Antes (linhas 148-164)**:
```go
strategyOpts := strategies.Options{
    Output:          o.config.Output.Directory,
    Concurrency:     o.config.Concurrency.Workers,
    Limit:           opts.Limit,
    MaxDepth:        o.config.Concurrency.MaxDepth,
    Exclude:         append(o.config.Exclude, opts.ExcludePatterns...),
    NoFolders:       o.config.Output.Flat,
    DryRun:          opts.DryRun,
    Verbose:         opts.Verbose,
    Force:           opts.Force || o.config.Output.Overwrite,
    RenderJS:        opts.RenderJS || o.config.Rendering.ForceJS,
    Split:           opts.Split,
    IncludeAssets:   opts.IncludeAssets,
    ContentSelector: opts.ContentSelector,
    ExcludeSelector: opts.ExcludeSelector,
    FilterURL:       opts.FilterURL,
}
```

**Depois**:
```go
strategyOpts := strategies.Options{
    CommonOptions: domain.CommonOptions{
        Verbose:  opts.Verbose,
        DryRun:   opts.DryRun,
        Force:    opts.Force || o.config.Output.Overwrite,
        RenderJS: opts.RenderJS || o.config.Rendering.ForceJS,
        Limit:    opts.Limit,
    },
    Output:          o.config.Output.Directory,
    Concurrency:     o.config.Concurrency.Workers,
    MaxDepth:        o.config.Concurrency.MaxDepth,
    Exclude:         append(o.config.Exclude, opts.ExcludePatterns...),
    NoFolders:       o.config.Output.Flat,
    Split:           opts.Split,
    IncludeAssets:   opts.IncludeAssets,
    ContentSelector: opts.ContentSelector,
    ExcludeSelector: opts.ExcludeSelector,
    FilterURL:       opts.FilterURL,
}
```

**NOTA**: Após esta refatoração, adicionar uma nova opção comum (ex: `Quiet bool`) requer apenas:
1. Adicionar em `CommonOptions`
2. Passar do CLI para `OrchestratorOptions`
(Propagação automática via embedding)

### Fase 5: Atualizar CLI
**Objetivo**: Ajustar `cmd/repodocs/main.go` para a nova estrutura

#### Tarefa 5.1: Atualizar criação de `OrchestratorOptions`

**Descrição**: Ajustar a construção de opções no main.go.

**Arquivos envolvidos**: `cmd/repodocs/main.go`

**Antes (linhas 172-186)**:
```go
orchOpts := app.OrchestratorOptions{
    Config:          cfg,
    Verbose:         verbose,
    DryRun:          dryRun,
    Force:           force,
    RenderJS:        renderJS,
    Split:           split,
    IncludeAssets:   includeAssets,
    Limit:           limit,
    ContentSelector: contentSelector,
    ExcludeSelector: excludeSelector,
    ExcludePatterns: excludePatterns,
    FilterURL:       filterURL,
}
```

**Depois**:
```go
orchOpts := app.OrchestratorOptions{
    CommonOptions: domain.CommonOptions{
        Verbose:  verbose,
        DryRun:   dryRun,
        Force:    force,
        RenderJS: renderJS,
        Limit:    limit,
    },
    Config:          cfg,
    Split:           split,
    IncludeAssets:   includeAssets,
    ContentSelector: contentSelector,
    ExcludeSelector: excludeSelector,
    ExcludePatterns: excludePatterns,
    FilterURL:       filterURL,
}
```

**Import adicional**:
```go
import (
    // ...existing imports...
    "github.com/quantmind-br/repodocs-go/internal/domain"
)
```

### Fase 6: Atualizar Testes e Mocks
**Objetivo**: Garantir que todos os testes passem com a nova estrutura

#### Tarefa 6.1: Atualizar `tests/mocks/domain.go`

**Descrição**: Atualizar o mock `MockStrategy` para usar a nova assinatura com `domain.StrategyOptions` atualizado.

**Arquivos envolvidos**: `tests/mocks/domain.go`

**Mudança**: A interface não muda assinatura, apenas a struct `StrategyOptions` internamente tem embed. Mocks devem continuar funcionando.

#### Tarefa 6.2: Atualizar `internal/domain/models_test.go`

**Descrição**: Atualizar `TestStrategyOptions` para testar a nova estrutura com embed.

**Arquivos envolvidos**: `internal/domain/models_test.go`

**Código de teste atualizado**:
```go
func TestStrategyOptions(t *testing.T) {
    t.Run("empty options have zero values", func(t *testing.T) {
        opts := StrategyOptions{}
        assert.False(t, opts.Verbose)
        assert.False(t, opts.DryRun)
        assert.False(t, opts.Force)
        assert.Equal(t, 0, opts.Limit)
    })

    t.Run("embedded CommonOptions is accessible", func(t *testing.T) {
        opts := StrategyOptions{
            CommonOptions: CommonOptions{
                Verbose: true,
                DryRun:  true,
            },
        }
        assert.True(t, opts.Verbose)
        assert.True(t, opts.DryRun)
    })
}
```

#### Tarefa 6.3: Atualizar testes de estratégias

**Descrição**: Atualizar testes que criam `DependencyOptions` e `Options` para usar a nova estrutura.

**Arquivos envolvidos**:
- `internal/strategies/crawler_strategy_test.go`
- `internal/strategies/llms_strategy_test.go`
- `internal/strategies/pkggo_strategy_test.go`
- `internal/strategies/sitemap_strategy_test.go`
- `internal/strategies/strategy_test.go`
- `internal/app/detector_test.go`
- `tests/unit/strategies/strategy_base_test.go`
- `tests/unit/llms_strategy_test.go`

**Padrão de mudança**: Em cada teste que cria `DependencyOptions`:

**Antes**:
```go
deps, err := NewDependencies(DependencyOptions{
    Timeout:        5 * time.Second,
    EnableCache:    false,
    Verbose:        false,
    DryRun:         false,
    Force:          false,
    OutputDir:      tmpDir,
})
```

**Depois**:
```go
deps, err := NewDependencies(DependencyOptions{
    CommonOptions: domain.CommonOptions{
        Verbose:  false,
        DryRun:   false,
        Force:    false,
    },
    Timeout:     5 * time.Second,
    EnableCache: false,
    OutputDir:   tmpDir,
})
```

#### Tarefa 6.4: Atualizar `tests/unit/renderer_test.go`

**Descrição**: Atualizar `MockBrowserRenderer.Execute` para usar nova `StrategyOptions`.

**Arquivos envolvidos**: `tests/unit/renderer_test.go`

**Mudança mínima**: A assinatura permanece compatível com `domain.StrategyOptions`.

### Fase 7: Criar Testes para CommonOptions
**Objetivo**: Adicionar testes específicos para o novo arquivo

#### Tarefa 7.1: Criar `internal/domain/options_test.go`

**Descrição**: Adicionar testes para `CommonOptions` e `DefaultCommonOptions()`.

**Arquivos envolvidos**: `internal/domain/options_test.go` (novo)

**Código**:
```go
package domain

import (
    "testing"

    "github.com/stretchr/testify/assert"
)

func TestCommonOptions_ZeroValue(t *testing.T) {
    opts := CommonOptions{}
    
    assert.False(t, opts.Verbose)
    assert.False(t, opts.DryRun)
    assert.False(t, opts.Force)
    assert.False(t, opts.RenderJS)
    assert.Equal(t, 0, opts.Limit)
}

func TestDefaultCommonOptions(t *testing.T) {
    opts := DefaultCommonOptions()
    
    assert.False(t, opts.Verbose)
    assert.False(t, opts.DryRun)
    assert.False(t, opts.Force)
    assert.False(t, opts.RenderJS)
    assert.Equal(t, 0, opts.Limit)
}

func TestCommonOptions_WithValues(t *testing.T) {
    opts := CommonOptions{
        Verbose:  true,
        DryRun:   true,
        Force:    true,
        RenderJS: true,
        Limit:    100,
    }
    
    assert.True(t, opts.Verbose)
    assert.True(t, opts.DryRun)
    assert.True(t, opts.Force)
    assert.True(t, opts.RenderJS)
    assert.Equal(t, 100, opts.Limit)
}
```

## Estratégia de Testes

### Testes Unitários
- [ ] `internal/domain/options_test.go` - Testar `CommonOptions` e defaults
- [ ] `internal/domain/models_test.go` - Atualizar para nova `StrategyOptions`
- [ ] `internal/strategies/strategy_test.go` - Verificar `Options` e `DependencyOptions` com embed
- [ ] Todos os testes de strategy individuais

### Testes de Integração
- [ ] `tests/integration/orchestrator_test.go` - Verificar fluxo completo
- [ ] `tests/integration/strategies_test.go` - Verificar todas as estratégias

### Testes E2E
- [ ] `tests/e2e/full_pipeline_test.go` - Pipeline completo
- [ ] `tests/e2e/crawl_test.go` - Crawler com novas opções

### Casos de Teste Específicos

| ID | Cenário | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | Zero value CommonOptions | `CommonOptions{}` | Todos campos false/0 |
| TC02 | Embed em OrchestratorOptions | `opts.Verbose` | Campo promovido acessível |
| TC03 | Embed em strategies.Options | `opts.DryRun` | Campo promovido acessível |
| TC04 | Passagem via CommonOptions block | Criar `DependencyOptions{CommonOptions: ...}` | Valores preservados |
| TC05 | DefaultCommonOptions | `DefaultCommonOptions()` | Valores padrão corretos |
| TC06 | CLI → Orchestrator → Strategy flow | Flags CLI | Valores chegam à estratégia |

### Comandos de Verificação

```bash
# Executar todos os testes
make test

# Testes unitários apenas
go test -v -short ./internal/...

# Testes de integração
make test-integration

# Verificar que CLI funciona
./build/repodocs --help
./build/repodocs --dry-run --verbose https://example.com

# Lint
make lint
```

## Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Import cycle domain ↔ strategies | Alto | Alto | Manter `StrategyOptions` em domain, fazer embed de `CommonOptions` |
| Quebra de API pública | Médio | Alto | Esta é breaking change documentada; testar extensivamente |
| Testes falhando por mudança de struct | Alto | Médio | Atualizar todos os testes na mesma PR |
| Campos promovidos ambíguos | Baixo | Baixo | Go resolve automaticamente; documentar padrão |
| Performance de embedding | Muito Baixo | Baixo | Structs pequenas, impacto negligível |

## Checklist de Conclusão

### Código
- [ ] `internal/domain/options.go` criado com `CommonOptions`
- [ ] `internal/domain/interfaces.go` atualizado (`StrategyOptions` com embed)
- [ ] `internal/app/orchestrator.go` atualizado (`OrchestratorOptions` com embed)
- [ ] `internal/strategies/strategy.go` atualizado (`Options` e `DependencyOptions` com embed)
- [ ] `cmd/repodocs/main.go` atualizado

### Testes
- [ ] `internal/domain/options_test.go` criado
- [ ] Testes existentes atualizados para nova estrutura
- [ ] Todos os testes unitários passando (`make test`)
- [ ] Testes de integração passando (`make test-integration`)
- [ ] Testes E2E passando (`make test-e2e`)

### Qualidade
- [ ] Lint passando (`make lint`)
- [ ] Build passando (`make build`)
- [ ] CLI funcionando normalmente

### Documentação
- [ ] Comentários em `CommonOptions` explicam propósito
- [ ] AGENTS.md não precisa atualização (patterns internos)

## Notas Adicionais

### Ordem de Execução Recomendada

1. **Criar `internal/domain/options.go`** - Não quebra nada, apenas adiciona
2. **Atualizar `internal/domain/interfaces.go`** - Modificar `StrategyOptions`
3. **Atualizar `internal/strategies/strategy.go`** - Modificar `Options` e `DependencyOptions`
4. **Atualizar `internal/app/orchestrator.go`** - Modificar `OrchestratorOptions`
5. **Atualizar `cmd/repodocs/main.go`** - Ajustar criação de options
6. **Atualizar todos os testes** - Garantir compatibilidade
7. **Rodar full test suite** - Verificar tudo funciona

### Benefício Pós-Implementação

**Antes**: Adicionar `--quiet` flag requeria:
1. Adicionar em `OrchestratorOptions`
2. Adicionar em `strategies.Options`
3. Adicionar em `domain.StrategyOptions`
4. Adicionar em `DependencyOptions`
5. Copiar em `NewOrchestrator()`
6. Copiar em `Run()`
7. Atualizar testes

**Depois**: Adicionar `--quiet` flag requer:
1. Adicionar em `CommonOptions`
2. Passar do CLI para `OrchestratorOptions.CommonOptions`
(Embedding propaga automaticamente)

### Considerações de Breaking Change

Esta é uma **breaking change** porque:
- `domain.StrategyOptions` muda estrutura (embed vs flat)
- Código externo que cria essas structs precisa atualizar

Mitigação:
- Campos promovidos continuam acessíveis da mesma forma (`opts.Verbose`)
- Apenas inicialização muda (`CommonOptions{Verbose: true}` vs `Verbose: true`)
