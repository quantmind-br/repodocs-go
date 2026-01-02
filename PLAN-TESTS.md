# Plano de Aumento de Cobertura de Testes

**Objetivo:** Aumentar cobertura de testes de 20.2% para no mínimo 80% por pacote.

**Threshold Ajustados:** Alguns pacotes terão thresholds menores devido à complexidade extrema ou dependências externas.

---

## Estado Atual

| Pacote | Cobertura | Status |
|--------|-----------|--------|
| pkg/version | 100.0% | ✅ Concluído |
| internal/utils | 94.3% | ✅ Concluído |
| internal/strategies | 31.0% | ⚠️ Precisa de trabalho |
| internal/llm | 6.0% | ❌ Crítico |
| cmd/repodocs | 0.0% | ❌ Sem testes |
| internal/app | 0.0% | ❌ Sem testes |
| internal/cache | 0.0% | ❌ Sem testes |
| internal/config | 0.0% | ❌ Sem testes |
| internal/converter | 0.0% | ❌ Crítico |
| internal/domain | 0.0% | ❌ Sem testes |
| internal/fetcher | 0.0% | ❌ Alta complexidade |
| internal/git | 0.0% | ❌ Sem testes |
| internal/output | 0.0% | ❌ Sem testes |
| internal/renderer | 0.0% | ❌ Muito alta complexidade (40-50%) |

---

## Estratégia de Testes

### Abordagem Híbrida (para pacotes de alta complexidade)
- **Testes de Unidade**: Usar mocks para testes rápidos e isolados
- **Testes de Integração Selecionados**: Para casos críticos com dependências reais
- **Testes de Contrato**: Verificar conformidade com interfaces

### Infraestrutura Disponível
- Test utilities: `tests/testutil/` (temp dirs, cache, HTTP servers, assertions)
- Mocks: `tests/mocks/` (gerados com go.uber.org/mock)
- Fixtures: `tests/fixtures/`, `tests/testdata/fixtures/`

---

## Fases de Implementação (Criticidade Primeiro)

### FASE 1: Componentes Críticos de Negócio
**Meta:** Cobertura 80%+ | Estimativa: 3-4 semanas

#### 1.1 internal/strategies (31% → 85%)
**Arquivos a criar:**
- `tests/unit/strategies/git_strategy_test.go` - Testes de unidade para GitStrategy
- `tests/unit/strategies/crawler_strategy_test.go` - Expandir cobertura existente
- `tests/unit/strategies/llms_strategy_test.go` - Testes completos para LLMS
- `tests/unit/strategies/pkggo_strategy_test.go` - Expandir cobertura existente
- `tests/unit/strategies/strategy_base_test.go` - Testes para métodos base

**Funções a cobrir:**
- Git: `parseGitURL`, `tryArchiveDownload`, `downloadAndExtract`, `extractTarGz`, `findDocumentationFiles`, `processFiles`, `processFile`, `detectDefaultBranch`
- Crawler: `Execute`, `isHTMLContentType`, `crawling logic`
- LLMS: `Execute`, `parseLLMSLinks`, `filterLLMSLinks`
- PkgGo: `Execute`, `extractSections`
- Base: `DefaultOptions`, `FlushMetadata`, `SetStrategy`, `SetSourceURL`, `WriteDocument`

**Mocks necessários:** Expandir mocks para GitClient, HTTP responses

#### 1.2 internal/converter (0% → 85%)
**Arquivos a criar:**
- `tests/unit/converter/pipeline_test.go` - Pipeline orchestration
- `tests/unit/converter/sanitizer_test.go` - HTML sanitization
- `tests/unit/converter/readability_test.go` - Content extraction
- `tests/unit/converter/markdown_test.go` - Markdown conversion
- `tests/unit/converter/encoding_test.go` - Encoding normalization

**Funções a cobrir:**
- Pipeline: `Convert`, `ConvertHTML`, `ConvertHTMLWithSelector`, `removeExcluded`
- Sanitizer: `Sanitize`, `normalizeURLs`, `resolveURL`, `normalizeSrcset`, `removeEmptyElements`
- Readability: `Extract`, `extractWithSelector`, `extractWithReadability`, `ExtractDescription`, `ExtractHeaders`, `ExtractLinks`
- Markdown: `Convert`, `cleanMarkdown`, `GenerateFrontmatter`, `AddFrontmatter`, `StripMarkdown`, `CountWords`, `CountChars`
- Encoding: `DetectEncoding`, `ConvertToUTF8`, `IsUTF8`, `GetEncoder`

**Fixtures necessárias:** HTML samples com diversos cenários (SPA, tabelas, code blocks, etc.)

#### 1.3 internal/app (0% → 85%)
**Arquivo a criar/modificar:**
- `tests/unit/app/detector_test.go` - Expandir testes existentes
- `tests/unit/app/orchestrator_test.go` - Já existe, expandir cobertura

**Funções a cobrir:**
- Detector: `DetectStrategy`, `CreateStrategy`, `GetAllStrategies`, `FindMatchingStrategy`
- Orchestrator: `NewOrchestrator`, `Run`, `Close`, `GetStrategyName`, `ValidateURL`

**Mocks necessários:** Strategy factory injection já existe

---

### FASE 2: LLM e Configuração
**Meta:** Cobertura 80%+ | Estimativa: 2-3 semanas

#### 2.1 internal/llm (6% → 80%)
**Arquivos a criar:**
- `tests/unit/llm/provider_test.go` - Provider factory e interfaces
- `tests/unit/llm/circuit_breaker_test.go` - Circuit breaker state machine
- `tests/unit/llm/retry_test.go` - Retry logic e backoff
- `tests/unit/llm/ratelimit_test.go` - Rate limiter com timing controlado
- `tests/integration/llm/provider_integration_test.go` - Testes de integração com HTTP mocking

**Funções a cobrir:**
- Providers: `NewAnthropicProvider`, `NewGoogleProvider`, `NewOpenAIProvider`, `Complete`, `Close`, `handleHTTPError`
- Circuit Breaker: `NewCircuitBreaker`, `Allow`, `RecordSuccess`, `RecordFailure`, `State`, `transitionTo`
- Retry: `NewRetrier`, `Execute`, `calculateBackoff`, `IsRetryableError`, `ShouldRetryStatusCode`
- Rate Limiter: `NewTokenBucket`, `Wait`, `TryAcquire`, `Available`, `refill`
- Metadata: `Enhance`, `EnhanceAll`, `applyMetadata`

**Abordagem:** Unit tests com mocks + integração para HTTP

#### 2.2 internal/config (0% → 85%)
**Arquivos a criar:**
- `tests/unit/config/config_test.go` - Config validation
- `tests/unit/config/loader_test.go` - Config loading com Viper mocking
- `tests/unit/config/defaults_test.go` - Default values

**Funções a cobrir:**
- Config: `Validate`, métodos de validação
- Loader: `Load`, `LoadWithViper`, `setDefaults`, `EnsureConfigDir`, `EnsureCacheDir`
- Defaults: `Default`, `ConfigDir`, `CacheDir`, `ConfigFilePath`

---

### FASE 3: Output e Cache
**Meta:** Cobertura 75%+ | Estimativa: 2 semanas

#### 3.1 internal/output (0% → 80%)
**Arquivos a criar:**
- `tests/unit/output/writer_test.go` - Writer operations
- `tests/unit/output/collector_test.go` - MetadataCollector

**Funções a cobrir:**
- Writer: `Write`, `WriteMultiple`, `FlushMetadata`, `Exists`, `EnsureBaseDir`, `Clean`, `Stats`
- Collector: `Add`, `Flush`, `buildIndex`, `GetIndex`, métodos de configuração

#### 3.2 internal/cache (0% → 75%)
**Arquivos a criar:**
- `tests/unit/cache/badger_test.go` - BadgerCache operations
- `tests/unit/cache/keys_test.go` - Key generation
- `tests/integration/cache/cache_integration_test.go` - Cache com BadgerDB real

**Funções a cobrir:**
- BadgerCache: `NewBadgerCache`, `Get`, `Set`, `Has`, `Delete`, `Close`, `Clear`, `Size`, `Stats`
- Keys: `GenerateKey`, `GenerateKeyWithPrefix`, `normalizeForKey`, `PageKey`, `SitemapKey`, `MetadataKey`

**Abordagem:** Unit tests com in-memory cache + integração para persistência

---

### FASE 4: Fetcher e Git
**Meta:** Cobertura 70-75% | Estimativa: 2 semanas

#### 4.1 internal/fetcher (0% → 70%)
**Arquivos a criar:**
- `tests/unit/fetcher/client_test.go` - HTTP client operations
- `tests/unit/fetcher/retry_test.go` - Retry logic
- `tests/unit/fetcher/stealth_test.go` - Stealth headers
- `tests/integration/fetcher/fetcher_integration_test.go` - HTTP requests reais

**Funções a cobrir:**
- Client: `NewClient`, `Get`, `GetWithHeaders`, `doRequest`, `GetCookies`, `Close`, cache operations
- Retry: `NewRetrier`, `Retry`, `RetryWithValue`, `ShouldRetryStatus`, `ParseRetryAfter`
- Stealth: `RandomUserAgent`, `RandomAcceptLanguage`, `StealthHeaders`, `RandomDelay`
- Transport: `NewStealthTransport`, `RoundTrip`

**Abordagem:** Híbrida - mocks para unit tests + integração para HTTP real

#### 4.2 internal/git (0% → 80%)
**Arquivos a criar:**
- `tests/unit/git/client_test.go` - Git client wrapper

**Funções a cobrir:**
- `NewClient`, `PlainCloneContext`

**Abordagem:** Mock de go-git operations

---

### FASE 5: CLI e Domain
**Meta:** Cobertura 80%+ | Estimativa: 1-2 semanas

#### 5.1 cmd/repodocs (0% → 80%)
**Arquivos a criar:**
- `cmd/repodocs/main_test.go` - CLI operations

**Funções a cobrir:**
- `run`, `initConfig`, `checkInternet`, `checkChrome`, `checkWritePermissions`, `checkCacheDir`

**Abordagem:** Mock de dependencies e testes de CLI

#### 5.2 internal/domain (0% → 85%)
**Arquivos a criar:**
- `tests/unit/domain/models_test.go` - Model methods
- `tests/unit/domain/errors_test.go` - Error types

**Funções a cobrir:**
- Models: `ToMetadata`, `ToFrontmatter`, `ToDocumentMetadata`, `ToSimpleMetadata`, `ToSimpleDocumentMetadata`
- Errors: Construtores e métodos de erro para todos os tipos

---

### FASE 6: Renderer (Alta Complexidade)
**Meta:** Cobertura 40-50% | Estimativa: 2 semanas

#### 6.1 internal/renderer (0% → 45%)
**Arquivos a criar:**
- `tests/unit/renderer/detector_test.go` - Framework detection
- `tests/unit/renderer/pool_test.go` - Tab pool management
- `tests/integration/renderer/renderer_integration_test.go` - Browser rendering real

**Funções a cobrir (selecionadas):**
- Detector: `NeedsJSRendering`, `DetectFramework`, `HasDynamicContent`, `hasSPAPattern`
- Pool: `NewTabPool`, `Acquire`, `Release`, `Close`, `Size`, `MaxSize`
- Rod: `DefaultRendererOptions`, `IsAvailable`, `GetBrowserPath`, `Close`
- Stealth: `DefaultStealthOptions`, `StealthPage`

**Threshold reduzido (40-50%) devido a:**
- Dependência de Chrome/Chromium instalado
- Operações de browser que difíceis de mockar
- Timing sensitivity em JavaScript rendering

**Abordagem:** Unit tests para detecção e pool + integração limitada com browser real

---

## Novos Fixtures e Mocks Necessários

### Fixtures
- `tests/testdata/fixtures/html/` - Amostras HTML variadas
- `tests/testdata/fixtures/git/` - Repositórios Git de exemplo
- `tests/testdata/fixtures/sitemap/` - Sitemaps variados
- `tests/testdata/fixtures/llms/` - Arquivos llms.txt de exemplo

### Mocks a Gerar/Expandir
```bash
# Gerar mocks a partir de interfaces
mockgen -source=internal/git/client.go -destination=tests/mocks/git.go
mockgen -source=internal/domain/interfaces.go -destination=tests/mocks/domain.go
```

---

## Testes de Integração Adicionais

### tests/integration/
- `converter_integration_test.go` - Pipeline completo
- `fetcher_integration_test.go` - HTTP requests
- `cache_integration_test.go` - BadgerDB persistência
- `strategies_integration_test.go` - Estratégias com dependencies reais

---

## Resumo de Esforço

| Fase | Componentes | Cobertura Alvo | Estimativa |
|------|-------------|----------------|------------|
| 1 | strategies, converter, app | 80-85% | 3-4 semanas |
| 2 | llm, config | 80-85% | 2-3 semanas |
| 3 | output, cache | 75-80% | 2 semanas |
| 4 | fetcher, git | 70-80% | 2 semanas |
| 5 | cmd, domain | 80-85% | 1-2 semanas |
| 6 | renderer | 40-50% | 2 semanas |

**Total Estimado:** 12-15 semanas

---

## Ajustes de Threshold por Pacote

| Pacote | Threshold | Justificativa |
|--------|-----------|---------------|
| internal/renderer | 40-50% | Browser automation, Chrome dependency |
| internal/fetcher | 70% | Network operations, complex retry logic |
| internal/cache | 75% | Database persistence, concurrent operations |
| internal/converter | 85% | HTML processing é testável com fixtures |
| internal/llm | 80% | HTTP requests podem ser mockados |
| Demais pacotes | 80%+ | Lógica de negócio principal |

---

## Próximos Passos

1. **Iniciar pela FASE 1** - Componentes críticos de negócio (strategies, converter, app)
2. **Executar testes incrementalmente** após cada pacote concluído
3. **Atualizar Makefile** para incluir novos testes
4. **Configurar CI** para reportar cobertura por pacote
5. **Documentar** padrões de testes em CONTRIBUTING.md

---

## Arquivos Críticos a Modificar/Criar

### Criar
- `tests/unit/converter/pipeline_test.go`
- `tests/unit/converter/sanitizer_test.go`
- `tests/unit/converter/readability_test.go`
- `tests/unit/converter/markdown_test.go`
- `tests/unit/converter/encoding_test.go`
- `tests/unit/strategies/git_strategy_test.go`
- `tests/unit/strategies/llms_strategy_test.go`
- `tests/unit/llm/provider_test.go`
- `tests/unit/llm/circuit_breaker_test.go`
- `tests/unit/llm/retry_test.go`
- `tests/unit/llm/ratelimit_test.go`
- `tests/unit/config/config_test.go`
- `tests/unit/output/writer_test.go`
- `tests/unit/output/collector_test.go`
- `tests/unit/cache/badger_test.go`
- `tests/unit/fetcher/client_test.go`
- `tests/unit/git/client_test.go`
- `tests/unit/domain/models_test.go`
- `tests/unit/domain/errors_test.go`
- `tests/unit/renderer/detector_test.go`
- `tests/unit/renderer/pool_test.go`
- `cmd/repodocs/main_test.go`

### Expandir
- `tests/unit/app/orchestrator_test.go`
- `tests/unit/strategies/crawler_strategy_test.go`
- `tests/unit/strategies/pkggo_strategy_test.go`
- `tests/unit/strategies/sitemap_strategy_test.go`

### Integração
- `tests/integration/converter_integration_test.go`
- `tests/integration/fetcher_integration_test.go`
- `tests/integration/cache_integration_test.go`
- `tests/integration/llm_integration_test.go`
- `tests/integration/renderer_integration_test.go`
