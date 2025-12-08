# Plano de Cobertura de Testes - Repodocs-Go

## Resumo Executivo

**Status dos Testes:**
- ✅ Unit Tests: APROVADOS (54.6% cobertura)
- ✅ Integration Tests: APROVADOS (38.0% cobertura)
- ✅ Teste com falha: CORRIGIDO (TestClient_GetCookies/invalid_URL)

**Objetivo:** Elevar cobertura de 54.6% para 90%+ através de implementação em 3 fases.

---

## 📊 Status Atual por Pacote

### Cobertura por Pacote
| Pacote | Cobertura | Status |
|--------|-----------|--------|
| tests/unit | 54.6% | ⚠️ |
| tests/unit/app | 2.0% | 🔴 Crítico |
| tests/unit/cache | 16.1% | 🔴 Baixo |
| tests/unit/fetcher | 48.7% | ⚠️ |
| tests/unit/renderer | 24.9% | 🔴 Baixo |
| tests/unit/utils | 6.0% | 🔴 Crítico |
| tests/integration | 38.0% | ⚠️ |
| tests/integration/strategies | 25.3% | 🔴 Baixo |

### Estatísticas Gerais
- **Total de arquivos Go:** 35
- **Arquivos de teste unitário:** 29
- **Arquivos de teste de integração:** 8
- **Funções com 0% cobertura:** 15
- **Funções com < 30% cobertura:** 17

---

## 🚨 Funções com 0% de Cobertura (CRÍTICO)

### 1. internal/config/defaults.go
- `ConfigFilePath`: 0.0%

### 2. internal/converter/readability.go
- `extractBody`: 0.0%

### 3. internal/fetcher/client.go
- `DefaultClientOptions`: 0.0%
- `saveToCache`: 0.0%

### 4. internal/fetcher/stealth.go
- `RandomDelay`: 0.0%

### 5. internal/strategies/git.go (8 FUNÇÕES)
- `detectDefaultBranch`: 0.0%
- `buildArchiveURL`: 0.0%
- `downloadAndExtract`: 0.0%
- `extractTarGz`: 0.0%
- `findDocumentationFiles`: 0.0%
- `processFiles`: 0.0%
- `processFile`: 0.0%
- `extractTitleFromPath`: 0.0%

### 6. internal/strategies/sitemap.go
- `processSitemapIndex`: 0.0%
- `decompressGzip`: 0.0%

### 7. internal/cache/interface.go
- `DefaultOptions`: 0.0%

---

## 📉 Funções com Baixa Cobertura (< 30%)

### internal/strategies/git.go
- `tryArchiveDownload`: 21.1%
- `NewGitStrategy`: 25.0%

### internal/app/detector.go
- `CreateStrategy`: 57.1%

### internal/utils/fs.go
- `GeneratePathFromRelative`: 50.0%
- `ExpandPath`: 27.3%

### internal/converter/readability.go
- `extractTitle`: 60.0%

---

## 🎯 Estratégia de Implementação (3 Fases)

### Fase 1: Crítico (1-2 semanas) - Meta: 70%
**Prioridade Máxima**

#### 1.1 Corrigir Teste com Falha ✅
- [x] `TestClient_GetCookies/invalid_URL` - CORRIGIDO
- [x] Teste esperava `nil` mas recebia lista vazia
- [x] Atualizado para usar `assert.Empty()` ao invés de `assert.Nil()`

#### 1.2 internal/app (2% → 70%)
**Criar:** `tests/unit/app/orchestrator_test.go`

**Casos de teste:**
```go
// orchestrator.go
- TestNewOrchestrator_Success - Configuração válida
- TestNewOrchestrator_WithNilConfig - Erro de configuração
- TestRun_Success - Execução completa bem-sucedida
- TestRun_InvalidURL - Erro de validação de URL
- TestRun_StrategyError - Erro durante execução da estratégia
- TestRun_ContextCancellation - Cancelamento durante execução
- TestClose_Success - Fechamento adequado dos recursos
- TestGetStrategyName - Retorna nome da estratégia atual
- TestValidateURL_Valid - URL válida aceita
- TestValidateURL_Invalid - URL inválida rejeitada
- TestValidateURL_Empty - URL vazia rejeitada

// detector.go
- TestCreateStrategy_ValidURL - Cria estratégia para URL válida
- TestCreateStrategy_InvalidURL - Erro para URL inválida
- TestCreateStrategy_UnknownType - Erro para tipo desconhecido
- TestDetectStrategy_Crawler - Detecta estratégia crawler
- TestDetectStrategy_Git - Detecta estratégia git
- TestDetectStrategy_Sitemap - Detecta estratégia sitemap
- TestDetectStrategy_PkgGo - Detecta estratégia pkg.go.dev
- TestDetectStrategy_LLMS - Detecta estratégia LLMS
- TestGetAllStrategies - Retorna todas as estratégias registradas
- TestFindMatchingStrategy - Encontra estratégia correspondente
```

#### 1.3 internal/utils (6% → 75%)
**Criar:** `tests/unit/utils/fs_test.go`
**Expandir:** `tests/unit/utils/url_test.go`

**Casos de teste:**
```go
// fs.go
- TestSanitizeFilename - Remove caracteres inválidos
- TestGeneratePath - Gera caminho absoluto válido
- TestGeneratePathFromRelative - Converte path relativo (50% → 100%)
- TestExpandPath - Expande paths com ~ e variáveis (27% → 100%)
- TestEnsureDir - Cria diretórios com diferentes permissões
- TestEnsureDir_ExistingDir - Não recria diretório existente
- TestURLToFilename - Converte URL para nome de arquivo
- TestURLToPath - Converte URL para caminho completo
- TestIsValidFilename - Valida nomes de arquivo
- TestJSONPath - Gera caminho para arquivo JSON

// url.go (expandir testes existentes)
- TestNormalizeURL_WithQuery - Mantém query parameters
- TestNormalizeURL_WithoutQuery - Remove query parameters
- TestNormalizeURL_Invalid - Erro para URL inválida
- TestResolveURL_Valid - Resolve URL relativa com base
- TestResolveURL_InvalidBase - Erro para base inválida
- TestExtractLinks_HTML - Extrai links de HTML
- TestExtractLinks_Complex - Extrai de HTML complexo
- TestFilterLinks_Include - Filtra links incluídos
- TestFilterLinks_Exclude - Filtra links excluídos
- TestGetDomain - Extrai domínio de URL
- TestGetBaseDomain - Extrai domínio base
- TestIsSameDomain - Compara domínios
- TestIsSameBaseDomain - Compara domínios base
```

#### 1.4 internal/strategies/git.go (50% → 80%)
**Criar/Expandir:** `tests/unit/git_strategy_test.go`

**Casos de teste:**
```go
// Funções críticas sem testes (0% → 80%+)
- TestDetectDefaultBranch_Main - Detecta branch 'main'
- TestDetectDefaultBranch_Master - Detecta branch 'master'
- TestDetectDefaultBranch_Custom - Detecta branch customizado
- TestDetectDefaultBranch_Error - Erro ao detectar branch
- TestBuildArchiveURL_GitHub - Constrói URL do GitHub
- TestBuildArchiveURL_GitLab - Constrói URL do GitLab
- TestBuildArchiveURL_Custom - Constrói URL customizada
- TestDownloadAndExtract_Success - Download e extração bem-sucedidos
- TestDownloadAndExtract_Gzip - Processa arquivo .tar.gz
- TestDownloadAndExtract_Error - Erro durante download
- TestExtractTarGz_Success - Extração bem-sucedida
- TestExtractTarGz_Invalid - Erro para arquivo inválido
- TestFindDocumentationFiles_Markdown - Encontra arquivos .md
- TestFindDocumentationFiles_AsciiDoc - Encontra arquivos .adoc
- TestFindDocumentationFiles_Empty - Lista vazia para repo sem docs
- TestFindDocumentationFiles_Nested - Encontra em subdiretórios
- TestProcessFiles_Success - Processa múltiplos arquivos
- TestProcessFiles_Invalid - Erro para arquivo inválido
- TestProcessFiles_Empty - Lista vazia
- TestProcessFile_Markdown - Processa arquivo Markdown
- TestProcessFile_HTML - Converte HTML para Markdown
- TestProcessFile_Error - Erro ao processar arquivo
- TestExtractTitleFromPath_Readme - Extrai título do README
- TestExtractTitleFromPath_Custom - Extrai de path customizado
- TestExtractTitleFromPath_Index - Extrai de index.*
- TestTryArchiveDownload_Success - Download via archive bem-sucedido
- TestTryArchiveDownload_Error - Erro no download via archive
- TestTryArchiveDownload_Fallback - Fallback para clone
- TestNewGitStrategy_Success - Inicialização bem-sucedida
- TestNewGitStrategy_WithOptions - Inicialização com opções
```

#### 1.5 internal/config (75% → 95%)
**Expandir:** `tests/unit/config_loader_test.go`

**Casos de teste:**
```go
// defaults.go
- TestConfigFilePath_Default - Retorna caminho padrão
- TestConfigFilePath_Custom - Retorna caminho customizado
- TestConfigFilePath_Empty - Trata caminho vazio

// loader.go
- TestLoad_WithConfigFile - Carrega de arquivo específico
- TestLoad_WithoutConfigFile - Usa valores padrão
- TestEnsureConfigDir_Success - Cria diretório de config
- TestEnsureConfigDir_Existing - Não recria diretório existente
- TestEnsureCacheDir_Success - Cria diretório de cache
- TestEnsureCacheDir_Existing - Não recria diretório existente
```

**Meta Fase 1:** 70% de cobertura

---

### Fase 2: Alto (2-3 semanas) - Meta: 80%

#### 2.1 internal/fetcher (80% → 95%)
**Expandir:** `tests/unit/fetcher/client_cache_test.go`

**Casos de teste:**
```go
// client.go
- TestDefaultClientOptions - Verifica opções padrão
- TestSaveToCache_Success - Salva no cache com sucesso
- TestSaveToCache_Disabled - Não salva com cache desabilitado
- TestSaveToCache_Error - Erro ao salvar no cache
- TestGet_WithCache - Busca com cache habilitado
- TestGet_WithoutCache - Busca com cache desabilitado
- TestGetWithHeaders_CustomHeaders - Headers customizados

// stealth.go
- TestRandomDelay_Generate - Gera delay aleatório
- TestRandomDelay_WithinRange - Delay dentro do intervalo
- TestRandomDelay_Zero - Delay zero quando configurado
```

#### 2.2 internal/strategies/sitemap (80% → 95%)
**Expandir:** `tests/unit/sitemap_strategy_test.go`

**Casos de teste:**
```go
// sitemap.go
- TestProcessSitemapIndex_Success - Processa índice de sitemaps
- TestProcessSitemapIndex_Empty - Lista vazia
- TestProcessSitemapIndex_Nested - Processa sitemaps aninhados
- TestDecompressGzip_Success - Descompacta arquivo .gz
- TestDecompressGzip_Invalid - Erro para arquivo inválido
- TestDecompressGzip_NotGzipped - Erro para arquivo não compactado
- TestParseLastMod_WithDate - Analisa data válida
- TestParseLastMod_Invalid - Ignora data inválida
```

#### 2.3 internal/converter/readability (71.4% → 90%)
**Expandir:** `tests/unit/readability_test.go`

**Casos de teste:**
```go
// readability.go
- TestExtractBody_WithSelector - Extrai com seletor específico
- TestExtractBody_WithoutSelector - Extrai sem seletor
- TestExtractBody_Empty - Retorna vazio para conteúdo vazio
- TestExtractBody_ComplexHTML - Extrai de HTML complexo
- TestExtractTitle_FromH1 - Extrai título de <h1>
- TestExtractTitle_FromTitle - Extrai de <title>
- TestExtractTitle_Empty - Retorna vazio sem título
- TestExtractDescription_Meta - Extrai de meta description
- TestExtractDescription_Content - Extrai do conteúdo
- TestExtractDescription_Empty - Retorna vazio sem description
```

#### 2.4 internal/renderer (24.9% → 75%)
**Expandir:** `tests/unit/renderer/pool_test.go`
**Criar:** `tests/unit/renderer/rod_test.go`

**Casos de teste:**
```go
// pool.go (70% → 90%)
- TestNewTabPool_WithOptions - Cria pool com opções
- TestNewTabPool_DefaultOptions - Usa opções padrão
- TestAcquire_Success - Adquire tab do pool
- TestAcquire_Timeout - Timeout ao adquirir tab
- TestRelease_Success - Libera tab para o pool
- TestRelease_Invalid - Erro ao liberar tab inválida
- TestClose_ClosesAllTabs - Fecha todas as tabs
- TestSize_ReturnsCurrentSize - Retorna tamanho atual
- TestMaxSize_ReturnsMaxSize - Retorna tamanho máximo

// rod.go (80% → 90%)
- TestNewRenderer_Success - Inicialização bem-sucedida
- TestNewRenderer_WithOptions - Inicialização com opções
- TestClose_ClosesBrowser - Fecha navegador adequadamente
- TestIsAvailable_CheckAvailability - Verifica disponibilidade
- TestGetBrowserPath_FindsChrome - Encontra caminho do Chrome
- TestGetBrowserPath_NotFound - Erro se Chrome não encontrado
```

**Meta Fase 2:** 80% de cobertura

---

### Fase 3: Médio (1-2 semanas) - Meta: 90%+

#### 3.1 internal/cache (16.1% → 80%)
**Expandir:** `tests/unit/cache/keys_test.go`
**Criar:** `tests/unit/cache/badger_test.go`

**Casos de teste:**
```go
// badger.go
- TestNewBadgerCache_Success - Inicialização bem-sucedida
- TestNewBadgerCache_WithOptions - Inicialização com opções
- TestGet_Found - Encontra chave no cache
- TestGet_NotFound - Não encontra chave inexistente
- TestGet_Expired - Remove entrada expirada
- TestSet_Success - Define valor no cache
- TestSet_Update - Atualiza valor existente
- TestHas_Exists - Verifica existência
- TestHas_NotExists - Verifica não-existência
- TestDelete_Success - Remove chave
- TestDelete_NotExists - Erro ao remover inexistente
- TestClear_Success - Limpa todo o cache
- TestSize_ReturnsCount - Retorna número de entradas
- TestStats_ReturnsStatistics - Retorna estatísticas
- TestClose_Success - Fecha cache adequadamente

// keys.go (80% → 100%)
- TestGenerateKey_Simple - Gera chave simples
- TestGenerateKey_WithPrefix - Gera chave com prefixo
- TestNormalizeForKey_SpecialChars - Normaliza caracteres especiais
- TestPageKey_GeneratesCorrectKey - Gera chave de página
- TestSitemapKey_GeneratesCorrectKey - Gera chave de sitemap
- TestMetadataKey_GeneratesCorrectKey - Gera chave de metadados
```

#### 3.2 internal/output (81% → 95%)
**Expandir:** `tests/unit/writer_test.go`

**Casos de teste:**
```go
// writer.go
- TestWrite_Success - Escreve documento único
- TestWrite_WithMetadata - Inclui metadados
- TestWrite_EmptyContent - Trata conteúdo vazio
- TestWrite_InvalidPath - Erro para caminho inválido
- TestWriteMultiple_Success - Escreve múltiplos documentos
- TestWriteMultiple_Partial - Falha parcial em escrita múltipla
- TestWriteJSON_Success - Escreve JSON válido
- TestWriteJSON_Indent - Escreve JSON formatado
- TestGetPath_ReturnsPath - Retorna caminho configurado
- TestExists_CheckExistence - Verifica existência de arquivo
- TestEnsureBaseDir_CreatesDir - Cria diretório base
- TestEnsureBaseDir_Existing - Não recria diretório existente
- TestClean_RemovesFiles - Remove arquivos e diretórios
- TestClean_EmptyDir - Não falha ao limpar diretório vazio
- TestStats_ReturnsStatistics - Retorna estatísticas
```

#### 3.3 internal/strategies/llms (51.4% → 85%)
**Expandir:** `tests/unit/llms_strategy_test.go`

**Casos de teste:**
```go
// llms.go
- TestParseLLMSLinks_Success - Extrai links LLMS
- TestParseLLMSLinks_Empty - Lista vazia para HTML sem LLMS
- TestParseLLMSLinks_Complex - Processa HTML complexo
- TestExecute_WithValidLLMS - Executa com LLMS válido
- TestExecute_WithEmptyLLMS - Trata LLMS vazio
- TestExecute_WithInvalidHTML - Erro para HTML inválido
- TestExecute_FetchError - Erro durante fetch
- TestNewLLMSStrategy_Success - Inicialização bem-sucedida
- TestCanHandle_LLMSURL - Reconhece URL LLMS
- TestCanHandle_NonLLMSURL - Rejeita URL não-LLMS
```

#### 3.4 Testes de Integração (38% → 70%)
**Expandir:** `tests/integration/orchestrator_test.go`
**Criar:** `tests/e2e/full_pipeline_test.go`

**Casos de teste:**
```go
// Integração completa
- TestFullPipeline_Website - Pipeline completo para website
- TestFullPipeline_GitRepo - Pipeline completo para repositório Git
- TestFullPipeline_Sitemap - Pipeline completo para sitemap
- TestFullPipeline_PkgGo - Pipeline completo para pkg.go.dev
- TestCache_Integration - Testa cache entre execuções
- TestConcurrency_MultipleURLs - Executa URLs em paralelo
- TestContextCancellation_FullFlow - Cancelamento durante pipeline
- TestErrorHandling_Graceful - Tratamento gracioso de erros
- TestPerformance_LargeSite - Performance com site grande
- TestRenderer_PoolExhaustion - Exaustão e renovação de pool
```

#### 3.5 Testes End-to-End (0% → 50%)
**Criar:** `tests/e2e/`

```go
// e2e tests
- TestCrawl_RealWebsite - Crawl de website real
- TestCrawl_GitHubRepo - Crawl de repositório GitHub
- TestCrawl_PkgGoDev - Crawl de pkg.go.dev
- TestCrawl_Sitemap - Crawl via sitemap
- TestOutput_ValidMarkdown - Valida Markdown gerado
- TestMetadata_ValidJSON - Valida JSON de metadados
- TestCache_PersistsBetweenRuns - Cache persiste entre execuções
- TestConfig_Overrides - Testa sobrescrita de configurações
- TestCLI_Integration - Testa integração via CLI
```

**Meta Fase 3:** 90%+ de cobertura

---

## 📋 Lista de Verificação por Fase

### Fase 1 - Crítico (1-2 semanas)
- [ ] 1.1 Corrigir `TestClient_GetCookies/invalid_URL` ✅
- [ ] 1.2 Criar `tests/unit/app/orchestrator_test.go` (11 casos)
- [ ] 1.3 Expandir `tests/unit/app/detector_test.go` (10 casos)
- [ ] 1.4 Criar `tests/unit/utils/fs_test.go` (10 casos)
- [ ] 1.5 Expandir `tests/unit/utils/url_test.go` (15 casos)
- [ ] 1.6 Expandir `tests/unit/git_strategy_test.go` (25 casos)
- [ ] 1.7 Expandir `tests/unit/config_loader_test.go` (10 casos)
- [ ] **Meta:** 70% cobertura
- [ ] **Duração:** 1-2 semanas

### Fase 2 - Alto (2-3 semanas)
- [ ] 2.1 Expandir `tests/unit/fetcher/client_cache_test.go` (10 casos)
- [ ] 2.2 Expandir `tests/unit/fetcher/stealth_test.go` (3 casos)
- [ ] 2.3 Expandir `tests/unit/sitemap_strategy_test.go` (10 casos)
- [ ] 2.4 Expandir `tests/unit/readability_test.go` (10 casos)
- [ ] 2.5 Expandir `tests/unit/renderer/pool_test.go` (10 casos)
- [ ] 2.6 Criar `tests/unit/renderer/rod_test.go` (8 casos)
- [ ] **Meta:** 80% cobertura
- [ ] **Duração:** 2-3 semanas

### Fase 3 - Médio (1-2 semanas)
- [ ] 3.1 Criar `tests/unit/cache/badger_test.go` (15 casos)
- [ ] 3.2 Expandir `tests/unit/cache/keys_test.go` (7 casos)
- [ ] 3.3 Expandir `tests/unit/writer_test.go` (15 casos)
- [ ] 3.4 Expandir `tests/unit/llms_strategy_test.go` (10 casos)
- [ ] 3.5 Expandir `tests/integration/` (10 casos)
- [ ] 3.6 Criar `tests/e2e/` (9 casos)
- [ ] **Meta:** 90%+ cobertura
- [ ] **Duração:** 1-2 semanas

**Total:** 4-7 semanas para alcançar 90%+ de cobertura

---

## 🛠️ Ferramentas e Comandos

### Comandos de Teste
```bash
# Gerar relatório de cobertura completo
go test -coverprofile=coverage.out -coverpkg=./internal/... ./tests/unit/... ./tests/integration/...

# Visualizar cobertura por função
go tool cover -func=coverage.out

# Ver funções com 0% cobertura
go tool cover -func=coverage.out | grep "0.0%"

# Ver funções com baixa cobertura (< 30%)
go tool cover -func=coverage.out | awk -F: 'NF==4 && $3+0 < 30 && $3+0 > 0 {print $0}'

# Executar todos os testes
make test

# Executar testes unitários
make test-unit

# Executar testes de integração
make test-integration

# Executar testes end-to-end
make test-e2e

# Gerar HTML de cobertura
go tool cover -html=coverage.out -o coverage.html

# Ver cobertura de um pacote específico
go test -coverprofile=/tmp/pkg.out ./tests/unit/fetcher/ && go tool cover -func=/tmp/pkg.out

# Executar teste específico
go test -v ./tests/unit/app/orchestrator_test.go

# Executar com race detection
go test -race ./tests/unit/...

# Executar com timeout
go test -timeout 5m ./tests/integration/...
```

### Comandos de Desenvolvimento
```bash
# Instalar dependências de teste
make deps

# Executar linter
make lint

# Formatar código
make fmt

# Verificar código
make vet

# Build do projeto
make build

# Executar CLI
make run ARGS="https://example.com -o ./output"
```

---

## 📚 Recursos de Referência

### Estrutura de Testes Recomendada
```
tests/
├── unit/                  # Testes unitários
│   ├── app/
│   │   ├── detector_test.go
│   │   └── orchestrator_test.go  # CRIAR
│   ├── cache/
│   │   ├── keys_test.go
│   │   └── badger_test.go  # CRIAR
│   ├── fetcher/
│   │   ├── client_cache_test.go
│   │   └── stealth_test.go  # EXPANDIR
│   ├── renderer/
│   │   ├── pool_test.go
│   │   └── rod_test.go  # CRIAR
│   ├── strategies/
│   │   ├── git_strategy_test.go  # EXPANDIR
│   │   ├── sitemap_strategy_test.go  # EXPANDIR
│   │   └── llms_strategy_test.go  # EXPANDIR
│   └── utils/
│       ├── fs_test.go  # CRIAR
│       ├── url_test.go  # EXPANDIR
│       └── logger_test.go
├── integration/          # Testes de integração
│   ├── orchestrator_test.go  # EXPANDIR
│   ├── strategies/
│   └── ...
└── e2e/                  # Testes end-to-end
    ├── full_pipeline_test.go  # CRIAR
    └── ...
```

### Padrões de Teste
- **Arrange-Act-Assert:** Estrutura clara para cada teste
- **Table-driven tests:** Para múltiplos cenários
- **Mock interfaces:** Isolar dependências
- **Golden files:** Para outputs complexos (Markdown)
- **TempDir:** Para testes que escrevem arquivos

---

## 🎯 Critérios de Aceitação

### Para cada pacote/pacote:
- [ ] Cobertura ≥ 70% (Fase 1)
- [ ] Cobertura ≥ 80% (Fase 2)
- [ ] Cobertura ≥ 90% (Fase 3)
- [ ] Todos os testes passing
- [ ] Sem race conditions
- [ ] Sem data races

### Para cada função crítica:
- [ ] Testes de sucesso
- [ ] Testes de erro
- [ ] Testes de edge cases
- [ ] Testes de contexto (cancelamento)

### Para integração:
- [ ] Testes end-to-end passing
- [ ] Performance acceptable
- [ ] Cache funcionando
- [ ] Concorrência segura

---

## 📊 Métricas de Sucesso

### Fase 1
- **Cobertura geral:** 70%
- **Pacotes críticos (app, utils):** ≥ 70%
- **Funções 0%:** Reduzir de 15 para ≤ 5

### Fase 2
- **Cobertura geral:** 80%
- **Todos os pacotes:** ≥ 75%
- **Funções 0%:** Reduzir para ≤ 2

### Fase 3
- **Cobertura geral:** 90%+
- **Todos os pacotes:** ≥ 85%
- **Funções 0%:** ≤ 1

---

## 🚦 Status do Projeto

### Atual
- ✅ Unit Tests: APROVADOS (54.6%)
- ✅ Integration Tests: APROVADOS (38.0%)
- ✅ Teste falhando: CORRIGIDO
- 🔄 Execução de testes: PASSOU

### Próximos Passos
1. **Imediato:** Iniciar Fase 1 (internal/app, internal/utils, internal/strategies/git)
2. **Semana 1:** Completar testes para internal/app
3. **Semana 2:** Completar testes para internal/utils
4. **Semana 3:** Completar testes para internal/strategies/git
5. **Semana 4:** Verificar 70% de cobertura

---

## 📞 Contato e Suporte

Para dúvidas sobre implementação de testes:
- Verificar: `/home/diogo/dev/repodocs-go/repodocs-go/CLAUDE.md`
- Consultar: Documentação de testes existente
- Executar: `make test` para verificar status

---

**Última atualização:** 2025-12-08
**Versão do plano:** 2.0
**Responsável:** Equipe de Desenvolvimento
