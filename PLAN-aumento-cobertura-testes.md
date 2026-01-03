# Plano de Aumento de Cobertura de Testes

**Data**: 3 de Janeiro de 2026
**Cobertura Atual**: 52.7%
**Meta**: 80%+ de cobertura

---

## Resumo Executivo

Este plano visa aumentar a cobertura de testes de **52.7% para 80%+**, focando nas áreas críticas que atualmente possuem baixa ou nenhuma cobertura. O projeto possui 131 arquivos de teste, mas algumas funcionalidades importantes permanecem sem testes adequados.

### Situação Atual por Pacote

| Pacote | Cobertura | Status |
|--------|-----------|--------|
| domain, git, version | 100% | ✅ Completo |
| cache, config, output, app, utils | 90%+ | ✅ Bom |
| converter, fetcher | 84-86% | ✅ Aceitável |
| llm, cmd | 67-73% | ⚠️ Melhorias necessárias |
| **renderer** | **30.3%** | 🔴 Crítico |
| **strategies** | **35.1%** | 🔴 Crítico |

---

## Fase 1: Correção de Testes Existentes (Semana 1)

### 1.1 Corrigir Testes Falhando

**Arquivo**: `tests/integration/fetcher/fetcher_integration_test.go`

| Teste | Problema | Solução |
|-------|----------|---------|
| `TestFetcherIntegration_Timeout` | Timeout não está sendo respeitado | Ajustar configurações de timeout no teste |
| `TestFetcherIntegration_WithRealServer` | EOF inesperado | Adicionar retry ou mock mais robusto |

**Arquivo**: `tests/unit/app/detector_test.go`

- Investigar e corrigir testes do detector que estão falhando
- Verificar se mocks estão atualizados com as estratégias novas

---

## Fase 2: Cobertura do Pacote `renderer` (Semana 2-3)

### Situação Atual: 30.3% → Meta: 75%+

**Arquivos a serem testados**:
- `internal/renderer/pool.go` (0%)
- `internal/renderer/rod.go` (0% no Render/stealth)
- `internal/renderer/stealth.go` (0%)

### 2.1 Testes para TabPool (`pool.go`)

```go
// tests/unit/renderer/pool_test.go (novo arquivo)
func TestNewTabPool(t *testing.T)
func TestTabPool_Acquire(t *testing.T)
func TestTabPool_Release(t *testing.T)
func TestTabPool_Close(t *testing.T)
func TestTabPool_Size(t *testing.T)
func TestTabPool_MaxSize(t *testing.T)
func TestTabPool_Concurrency(t *testing.T)
```

**Abordagem**: Usar mocks do Rod browser interface para evitar depender de Chrome real

### 2.2 Testes para RodRenderer (`rod.go`)

```go
// tests/unit/renderer/rod_test.go (ampliar)
func TestRodRenderer_Render(t *testing.T)
func TestRodRenderer_Render_WithStealth(t *testing.T)
func TestRodRenderer_Render_WithCookies(t *testing.T)
func TestRodRenderer_Render_ScrollToEnd(t *testing.T)
func TestRodRenderer_NewRenderer_CI_Mode(t *testing.T)
```

**Abordagem**:
- Criar HTML fixtures para renderização
- Mock de browser responses
- Testar CI mode (sem Chrome)

### 2.3 Testes para Stealth Mode (`stealth.go`)

```go
// tests/unit/renderer/stealth_test.go (novo arquivo)
func TestStealthPage(t *testing.T)
func TestApplyStealthMode(t *testing.T)
func TestApplyStealthMode_WithOptions(t *testing.T)
```

---

## Fase 3: Cobertura do Pacote `strategies` (Semana 4-6)

### Situação Atual: 35.1% → Meta: 70%+

#### 3.1 Estratégia DocsRS (~1000 linhas, 0% cobertura)

**Arquivos**:
- `internal/strategies/docsrs.go`
- `internal/strategies/docsrs_json.go`
- `internal/strategies/docsrs_renderer.go`
- `internal/strategies/docsrs_types.go`

**Testes necessários**:

```go
// tests/unit/strategies/docsrs_strategy_test.go (ampliar)
func TestDocsRSStrategy_CanHandle(t *testing.T)
func TestDocsRSStrategy_Execute_Integration(t *testing.T)
func TestDocsRSStrategy_parseDocsRSPath(t *testing.T)
func TestDocsRSStrategy_buildItemURL(t *testing.T)
func TestDocsRSStrategy_buildItemTitle(t *testing.T)

// tests/unit/strategies/docsrs_json_test.go (ampliar)
func TestDocsRSJSONEndpoint(t *testing.T)
func TestParseRustdocJSON(t *testing.T)
func TestFetchRustdocJSON(t *testing.T)
func TestCollectItems(t *testing.T)

// tests/unit/strategies/docsrs_renderer_test.go (novo)
func TestRenderItem_Struct(t *testing.T)
func TestRenderItem_Function(t *testing.T)
func TestRenderItem_Trait(t *testing.T)
func TestRenderSignature(t *testing.T)
func TestRenderType(t *testing.T)
func TestResolveCrossRefs(t *testing.T)
```

**Abordagem**:
- Criar fixtures JSON de rustdoc reais
- Mock de fetcher para retornar JSON fixtures
- Testar renderização de diferentes tipos de itens Rust

#### 3.2 Estratégia GitHub Pages (~500 linhas, 0% cobertura)

**Arquivos**:
- `internal/strategies/github_pages.go`
- `internal/strategies/github_pages_discovery.go`

**Testes necessários**:

```go
// tests/unit/strategies/github_pages_strategy_test.go (novo)
func TestGitHubPagesStrategy_CanHandle(t *testing.T)
func TestGitHubPagesStrategy_IsGitHubPagesURL(t *testing.T)
func TestGitHubPagesStrategy_Execute_Integration(t *testing.T)
func TestGitHubPagesStrategy_discoverURLs(t *testing.T)
func TestGitHubPagesStrategy_extractLinksFromRenderedPage(t *testing.T)

// tests/unit/strategies/github_pages_discovery_test.go (novo)
func TestDiscoverViaHTTPProbes(t *testing.T)
func TestDiscoverViaBrowser(t *testing.T)
func TestExtractLinksWithGoquery(t *testing.T)
```

**Abordagem**:
- Criar HTML fixtures de páginas GitHub Pages
- Mock de renderer para retornar HTML fixture
- Testar lógica de descoberta de URLs

---

## Fase 4: Cobertura de LLM Providers (Semana 7)

### Situação Atual: 73.8% → Meta: 85%+

**Arquivos sem cobertura**:
- `internal/llm/anthropic.go` (Complete: 0%)
- `internal/llm/google.go` (Complete: 0%)

### Testes necessários:

```go
// tests/unit/llm/anthropic_test.go (novo)
func TestAnthropicProvider_Complete(t *testing.T)
func TestAnthropicProvider_Complete_WithError(t *testing.T)
func TestAnthropicProvider_Close(t *testing.T)
func TestAnthropicProvider_handleHTTPError(t *testing.T)

// tests/unit/llm/google_test.go (novo)
func TestGoogleProvider_Complete(t *testing.T)
func TestGoogleProvider_Complete_WithError(t *testing.T)
func TestGoogleProvider_Close(t *testing.T)
func TestGoogleProvider_handleHTTPError(t *testing.T)
```

**Abordagem**:
- Criar HTTP server mock (httptest.Server)
- Testar diferentes cenários de resposta
- Testar retry e error handling

---

## Fase 5: Melhorias no Converter (Semana 8)

### Situação Atual: 86.8% → Meta: 95%+

**Funções com 0% cobertura**:
- `extractWithSelector`
- `normalizeURLs`
- `removeEmptyElements`

```go
// tests/unit/converter/pipeline_test.go (ampliar)
func TestExtractWithSelector(t *testing.T)
func TestNormalizeURLs(t *testing.T)
func TestRemoveEmptyElements(t *testing.T)
```

---

## Estratégias de Teste

### 1. Testes Unitários vs Integração

| Tipo | Uso | Proporção |
|------|-----|-----------|
| Unit | Lógica pura, sem dependências externas | 70% |
| Integration | Com mocks de fetcher/renderer | 20% |
| E2E | URLs reais, em ambiente controlado | 10% |

### 2. Fixtures

Criar diretório `tests/fixtures/` com:

```
tests/fixtures/
├── html/              # HTML pages para crawler
├── rustdoc/           # JSON responses do DocsRS
├── github-pages/      # HTML de GitHub Pages
└── renderer/          # HTML para renderer tests
```

### 3. Mocks e Interfaces

Verificar que dependências externas estão através de interfaces:
- `domain.Fetcher` → mock em `tests/mocks/fetcher.go`
- `domain.Renderer` → mock em `tests/mocks/renderer.go`
- `domain.Cache` → mock em `tests/mocks/cache.go`

### 4. Testes Parametrizados (Table-Driven Tests)

Usar pattern table-driven para múltiplos cenários:

```go
func TestDocsRSStrategy_CanHandle(t *testing.T) {
    tests := []struct {
        name string
        url  string
        want bool
    }{
        {"docs.rs valid url", "https://docs.rs/serde/latest/serde/", true},
        {"crates.io", "https://crates.io/crates/serde", false},
        // ...
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) { /* ... */ })
    }
}
```

---

## Ordem de Prioridade

### 🔴 Prioridade Crítica (Semana 1-4)
1. Corrigir testes falhando (fetcher timeout)
2. Estratégia DocsRS (recém adicionada, sem cobertura)
3. Estratégia GitHub Pages (recém adicionada, sem cobertura)

### 🟡 Prioridade Alta (Semana 5-7)
4. Pacote renderer (30% → 75%)
5. LLM providers (Anthropic, Google)

### 🟢 Prioridade Média (Semana 8)
6. Converter (86% → 95%)
7. Melhorias em pacotes já cobertos

---

## Métricas de Sucesso

| Fase | Meta de Cobertura | Data |
|------|-------------------|------|
| Início | 52.7% | - |
| Fase 1 | 55% | Semana 1 |
| Fase 2 | 62% | Semana 3 |
| Fase 3 | 75% | Semana 6 |
| Fase 4 | 80% | Semana 7 |
| Fase 5 | 82%+ | Semana 8 |

---

## Checklist de Implementação

### Para cada novo teste:
- [ ] Teste específico e isolado
- [ ] Usa fixtures ou mocks apropriados
- [ ] Cobre happy path e error cases
- [ ] Testa edge cases (nil, vazio, limites)
- [ ] Não depende de rede externa (exceto E2E)
- [ ] Documenta comportamento esperado

### Para cada teste corrigido:
- [ ] Identifica causa da falha
- [ ] Corrige sem alterar comportamento esperado
- [ ] Adiciona testes de regressão se necessário
- [ ] Atualiza documentação se comportamento mudou

---

## Ferramentas e Comandos

```bash
# Rodar todos os testes
make test

# Testes com cobertura
go test -coverprofile=coverage.out -covermode=atomic ./...
go tool cover -html=coverage.out -o coverage.html

# Testes específicos
go test -v ./tests/unit/renderer/...
go test -v -run TestDocsRSStrategy ./tests/unit/strategies/...

# Ver cobertura por pacote
go tool cover -func=coverage.out | grep total

# Testes de integração (com rede)
make test-integration

# Testes E2E
make test-e2e
```

---

## Próximos Passos

1. **Aprovação**: Revisar e aprovar este plano
2. **Setup**: Criar estrutura de fixtures
3. **Fase 1**: Começar corrigindo testes falhando
4. **Tracking**: Usar `bd` para tracking de tarefas
5. **Review**: Code review para cada PR de testes

---

## Notas

- Alguns testes de renderer dependem de Chrome/Chromium → considerar usar headless mode ou mocks
- Testes de integração devem ter flag para pular em CI/CD sem dependências
- Manter testes rápidos (< 100ms para unitários)
- Documentar bugs encontrados durante criação de testes
