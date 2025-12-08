# Plano de Melhoria da Cobertura de Testes

## Resumo Executivo

**Cobertura Atual:** 47.6% (statements em `./internal/...`)

Este plano identifica as lacunas críticas na cobertura de testes e prioriza as áreas que mais necessitam de atenção.

---

## 1. Análise por Pacote

### 1.1 Pacotes com Cobertura Crítica (0-30%)

| Pacote | Cobertura Estimada | Prioridade |
|--------|-------------------|------------|
| `internal/renderer` | ~15% | ALTA |
| `internal/strategies/git.go` | ~10% | ALTA |
| `internal/strategies/pkggo.go` | 0% | MÉDIA |
| `internal/config/loader.go` | 0% | MÉDIA |
| `internal/utils/url.go` | ~20% | ALTA |
| `internal/utils/workerpool.go` | ~25% | MÉDIA |
| `internal/output/writer.go` | ~30% | MÉDIA |

### 1.2 Pacotes com Cobertura Moderada (30-70%)

| Pacote | Cobertura Estimada | Prioridade |
|--------|-------------------|------------|
| `internal/strategies/crawler.go` | ~60% | MÉDIA |
| `internal/strategies/sitemap.go` | ~55% | MÉDIA |
| `internal/strategies/llms.go` | ~35% | MÉDIA |
| `internal/converter/*` | ~70% | BAIXA |
| `internal/fetcher/client.go` | ~65% | BAIXA |

### 1.3 Pacotes com Boa Cobertura (70%+)

| Pacote | Cobertura | Status |
|--------|-----------|--------|
| `internal/cache/badger.go` | ~95% | OK |
| `internal/fetcher/retry.go` | ~95% | OK |
| `internal/fetcher/stealth.go` | ~90% | OK |
| `internal/utils/fs.go` | ~85% | OK |
| `internal/app/detector.go` | ~80% | OK |

---

## 2. Tarefas Priorizadas

### PRIORIDADE ALTA

#### 2.1 Renderer Package (`internal/renderer/`)
**Cobertura atual:** ~15%

**Funções sem cobertura:**
- `Render()` - 0%
- `Close()` - 0%
- `Acquire()` / `Release()` (pool) - 0%
- `scrollToEnd()` - 0%
- `setCookies()` - 0%
- `ApplyStealthMode()` - 0%
- `DetectFramework()` - 0%
- `HasDynamicContent()` - 0%

**Testes recomendados:**
```go
// tests/unit/renderer_test.go
func TestRenderer_NewRenderer_Success(t *testing.T)
func TestRenderer_NewRenderer_InvalidOptions(t *testing.T)
func TestRenderer_Render_BasicHTML(t *testing.T)
func TestRenderer_Render_WithJavaScript(t *testing.T)
func TestRenderer_Render_Timeout(t *testing.T)
func TestRenderer_Close(t *testing.T)

func TestTabPool_AcquireRelease(t *testing.T)
func TestTabPool_Concurrent(t *testing.T)
func TestTabPool_MaxSize(t *testing.T)

func TestDetectFramework_React(t *testing.T)
func TestDetectFramework_Vue(t *testing.T)
func TestDetectFramework_Angular(t *testing.T)
func TestHasDynamicContent(t *testing.T)
```

**Complexidade:** ALTA (requer Chrome/Chromium)
**Estratégia:** Usar mocks para browser ou testes de integração com servidor HTTP local

---

#### 2.2 Git Strategy (`internal/strategies/git.go`)
**Cobertura atual:** ~10%

**Funções sem cobertura:**
- `Execute()` - 0%
- `tryArchiveDownload()` - 0%
- `parseGitURL()` - 0% (testado em unit mas não coberto)
- `detectDefaultBranch()` - 0% (testado em unit mas não coberto)
- `buildArchiveURL()` - 0% (testado em unit mas não coberto)
- `downloadAndExtract()` - 0%
- `extractTarGz()` - 0%
- `cloneRepository()` - 0%
- `findDocumentationFiles()` - 0%
- `processFiles()` - 0%
- `processFile()` - 0%

**Testes recomendados:**
```go
// tests/unit/git_strategy_test.go (adicionar)
func TestGitStrategy_Execute_ArchiveDownload(t *testing.T)
func TestGitStrategy_Execute_CloneFallback(t *testing.T)
func TestGitStrategy_ExtractTarGz(t *testing.T)
func TestGitStrategy_FindDocumentationFiles(t *testing.T)
func TestGitStrategy_ProcessFile_Markdown(t *testing.T)
func TestGitStrategy_ProcessFile_ReStructuredText(t *testing.T)

// tests/integration/git_strategy_test.go
func TestGitStrategy_Execute_RealRepository(t *testing.T)
func TestGitStrategy_Execute_PrivateRepository(t *testing.T)
```

**Complexidade:** MÉDIA
**Estratégia:** Usar servidor HTTP mock para simular GitHub/GitLab archives

---

#### 2.3 URL Utils (`internal/utils/url.go`)
**Cobertura atual:** ~20%

**Funções sem cobertura:**
- `NormalizeURL()` - 0%
- `NormalizeURLWithoutQuery()` - 0%
- `ResolveURL()` - 0%
- `GetDomain()` - 0%
- `GetBaseDomain()` - 0%
- `IsSameDomain()` - 0%
- `IsSameBaseDomain()` - 0%
- `IsAbsoluteURL()` - 0%
- `IsGitURL()` - 0%
- `IsSitemapURL()` - 0%
- `IsLLMSURL()` - 0%
- `IsPkgGoDevURL()` - 0%
- `ExtractLinks()` - 0%
- `FilterLinks()` - 0%

**Testes recomendados:**
```go
// tests/unit/url_utils_test.go
func TestNormalizeURL(t *testing.T)
func TestNormalizeURLWithoutQuery(t *testing.T)
func TestResolveURL(t *testing.T)
func TestGetDomain(t *testing.T)
func TestGetBaseDomain(t *testing.T)
func TestIsSameDomain(t *testing.T)
func TestIsSameBaseDomain(t *testing.T)
func TestIsAbsoluteURL(t *testing.T)
func TestIsGitURL(t *testing.T)
func TestIsSitemapURL(t *testing.T)
func TestIsLLMSURL(t *testing.T)
func TestIsPkgGoDevURL(t *testing.T)
func TestExtractLinks(t *testing.T)
func TestFilterLinks(t *testing.T)
```

**Complexidade:** BAIXA
**Estratégia:** Testes unitários puros, table-driven tests

---

### PRIORIDADE MÉDIA

#### 2.4 PkgGo Strategy (`internal/strategies/pkggo.go`)
**Cobertura atual:** 0%

**Funções sem cobertura:**
- `NewPkgGoStrategy()` - 0%
- `Name()` - 0%
- `CanHandle()` - 0%
- `Execute()` - 0%
- `extractSections()` - 0%

**Testes recomendados:**
```go
// tests/unit/pkggo_strategy_test.go
func TestPkgGoStrategy_CanHandle(t *testing.T)
func TestPkgGoStrategy_Name(t *testing.T)

// tests/integration/pkggo_strategy_test.go
func TestPkgGoStrategy_Execute_StandardPackage(t *testing.T)
func TestPkgGoStrategy_Execute_ThirdParty(t *testing.T)
func TestPkgGoStrategy_ExtractSections(t *testing.T)
```

**Complexidade:** MÉDIA
**Estratégia:** Mock HTTP responses ou testes de integração com pkg.go.dev real

---

#### 2.5 Config Loader (`internal/config/loader.go`)
**Cobertura atual:** 0%

**Funções sem cobertura:**
- `Load()` - 0%
- `LoadWithViper()` - 0%
- `setDefaults()` - 0%
- `setDefaultsIfNotSet()` - 0%
- `EnsureConfigDir()` - 0%
- `EnsureCacheDir()` - 0%

**Testes recomendados:**
```go
// tests/unit/config_loader_test.go
func TestLoad_DefaultConfig(t *testing.T)
func TestLoad_FromFile(t *testing.T)
func TestLoad_FromEnvironment(t *testing.T)
func TestLoad_MergeConfigs(t *testing.T)
func TestEnsureConfigDir(t *testing.T)
func TestEnsureCacheDir(t *testing.T)
```

**Complexidade:** BAIXA
**Estratégia:** Usar diretórios temporários, arquivos de configuração de teste

---

#### 2.6 Worker Pool (`internal/utils/workerpool.go`)
**Cobertura atual:** ~25%

**Funções sem cobertura:**
- `NewPool()` - 0%
- `Start()` - 0%
- `runWorker()` - 0%
- `Submit()` - 0%
- `Results()` - 0%
- `Stop()` - 0%
- `Process()` - 0%
- `NewSimplePool()` - 0%
- `Run()` - 0%
- `CollectErrors()` - 0%

**Testes recomendados:**
```go
// tests/unit/workerpool_test.go (adicionar)
func TestPool_BasicProcessing(t *testing.T)
func TestPool_ConcurrentSubmissions(t *testing.T)
func TestPool_Stop(t *testing.T)
func TestSimplePool_Run(t *testing.T)
func TestCollectErrors(t *testing.T)
```

**Complexidade:** MÉDIA
**Estratégia:** Testes com tasks simuladas, verificar concorrência

---

#### 2.7 Output Writer (`internal/output/writer.go`)
**Cobertura atual:** ~30%

**Funções sem cobertura:**
- `writeJSON()` - 0%
- `WriteMultiple()` - 0%
- `EnsureBaseDir()` - 0%
- `Clean()` - 0%
- `Stats()` - 0%

**Testes recomendados:**
```go
// tests/unit/writer_test.go
func TestWriter_Write_Success(t *testing.T)
func TestWriter_Write_WithMetadata(t *testing.T)
func TestWriter_WriteJSON(t *testing.T)
func TestWriter_WriteMultiple(t *testing.T)
func TestWriter_Clean(t *testing.T)
func TestWriter_Stats(t *testing.T)
func TestWriter_DryRun(t *testing.T)
```

**Complexidade:** BAIXA
**Estratégia:** Usar diretórios temporários

---

### PRIORIDADE BAIXA

#### 2.8 Domain Errors (`internal/domain/errors.go`)
**Cobertura atual:** ~40%

**Funções sem cobertura:**
- `NewFetchError()` - 0%
- `FetchError.Error()` - 0%
- `FetchError.Unwrap()` - 0%
- `NewValidationError()` - 0%
- `ValidationError.Error()` - 0%
- `NewStrategyError()` - 0%
- `StrategyError.Error()` - 0%
- `StrategyError.Unwrap()` - 0%

**Testes recomendados:**
```go
// tests/unit/errors_test.go
func TestFetchError(t *testing.T)
func TestValidationError(t *testing.T)
func TestStrategyError(t *testing.T)
func TestIsRetryable(t *testing.T)
```

**Complexidade:** BAIXA

---

## 3. Melhorias Arquiteturais Necessárias

### 3.1 Dependency Injection para Strategies

**Problema:** Os testes do Orchestrator criam mocks que nunca são injetados.

**Solução proposta:**
```go
// internal/app/orchestrator.go
type OrchestratorOptions struct {
    // ... campos existentes
    StrategyFactory func(StrategyType, *strategies.Dependencies) domain.Strategy
}
```

Isso permitiria injetar estratégias mock em testes unitários.

### 3.2 Interface para Browser/Renderer

**Problema:** O renderer está fortemente acoplado ao rod/Chrome.

**Solução proposta:**
```go
// internal/domain/interfaces.go
type Browser interface {
    Render(ctx context.Context, url string, opts RenderOptions) (string, error)
    Close() error
}
```

Isso permitiria usar um mock browser em testes.

---

## 4. Cronograma Sugerido

### Fase 1 - Fundamentos (1-2 semanas)
- [ ] Testes para `internal/utils/url.go`
- [ ] Testes para `internal/config/loader.go`
- [ ] Testes para `internal/domain/errors.go`

### Fase 2 - Infraestrutura (2-3 semanas)
- [ ] Testes para `internal/output/writer.go`
- [ ] Testes para `internal/utils/workerpool.go`
- [ ] Refatorar DI no Orchestrator

### Fase 3 - Strategies (3-4 semanas)
- [ ] Testes para `internal/strategies/pkggo.go`
- [ ] Testes para `internal/strategies/git.go`
- [ ] Testes adicionais para crawler/sitemap/llms

### Fase 4 - Renderer (2-3 semanas)
- [ ] Interface Browser para DI
- [ ] Testes para `internal/renderer/`
- [ ] Testes de integração com Chrome

---

## 5. Meta de Cobertura

| Fase | Meta |
|------|------|
| Atual | 47.6% |
| Após Fase 1 | 55% |
| Após Fase 2 | 65% |
| Após Fase 3 | 75% |
| Após Fase 4 | 80%+ |

---

## 6. Comandos Úteis

```bash
# Verificar cobertura atual
go test -coverprofile=coverage.out -coverpkg=./internal/... ./tests/...
go tool cover -func=coverage.out

# Gerar relatório HTML
go tool cover -html=coverage.out -o coverage.html

# Verificar cobertura de um pacote específico
go test -coverprofile=coverage.out -coverpkg=./internal/renderer/... ./tests/...

# Rodar testes específicos
go test -v -run TestRenderer ./tests/unit/...
```
