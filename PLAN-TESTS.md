# Plano de Aumento de Cobertura de Testes (Test Coverage Improvement Plan)

Baseado na análise de cobertura realizada em 01/01/2026.
**STATUS: COMPLETED** - All epics closed on 01/01/2026.

## Estratégia Geral
O foco é mitigar riscos nas áreas "Core" do sistema que possuem 0% ou baixa cobertura. A prioridade é garantir que a execução (`Execute`) das estratégias e a orquestração (`Run`) estejam blindadas contra regressões.

---

## Epic 1: Core Strategy Engine (Prioridade: Crítica) ✅ CLOSED
**Objetivo:** Garantir que o motor principal de crawling e processamento funcione corretamente, já que é o fallback padrão para a maioria das documentações.

### Task 1.1: CrawlerStrategy `Execute` ✅
- **Alvo:** `internal/strategies/crawler.go`
- **Tests:** `tests/unit/crawler_strategy_test.go` (850 lines)
- **Cenários Cobertos:**
    - Crawl simples de uma página (Seed -> Fetch -> Convert -> Save).
    - Respeito ao `MaxDepth`.
    - Deduplicação de URLs (Visited check).
    - Tratamento de erros de Fetch (404, 500).
    - Respeito a domínios externos (não seguir links fora do escopo).

### Task 1.2: SitemapStrategy `Execute` ✅
- **Alvo:** `internal/strategies/sitemap.go`
- **Tests:** `tests/unit/sitemap_strategy_test.go` (968 lines)
- **Cenários Cobertos:**
    - Parsing de Sitemap Index (sitemap de sitemaps).
    - Tratamento de Sitemaps comprimidos (.gz).
    - Filtragem por `LastMod`.

---

## Epic 2: Orchestration Logic (Prioridade: Alta) ✅ CLOSED
**Objetivo:** Garantir que o sistema escolha a estratégia correta e gerencie o ciclo de vida corretamente.

### Task 2.1: Orchestrator `Run` ✅
- **Alvo:** `internal/app/orchestrator.go`
- **Tests:** `tests/unit/app/orchestrator_test.go` (639 lines)
- **Cenários Cobertos:**
    - Seleção correta de estratégia baseada na URL.
    - Propagação de contexto (cancelamento).
    - Tratamento de erro fatal na inicialização da estratégia.
    - Verificação de fluxo completo (Start -> Execute -> Close).
    - Mock injection via StrategyFactory pattern.

---

## Epic 3: Specialized Strategies (Prioridade: Média) ✅ CLOSED
**Objetivo:** Cobrir estratégias específicas que falham silenciosamente hoje por falta de testes.

### Task 3.1: WikiStrategy ✅
- **Alvo:** `internal/strategies/wiki.go`
- **Tests:** `tests/unit/strategies/wiki_strategy_test.go` (65 lines)
- **Cenários Cobertos:**
    - Detecção de estrutura de Wiki (sidebar).
    - Crawl sequencial de tópicos.
    - Mock git client injection.

### Task 3.2: PkgGoStrategy ✅
- **Alvo:** `internal/strategies/pkggo.go`
- **Tests:** `tests/unit/pkggo_strategy_test.go` (1233 lines)
- **Cenários Cobertos:**
    - Extração de README e documentação de pacote Go.
    - Comprehensive Execute flow testing.

### Task 3.3: LLMsStrategy ✅ (Bonus)
- **Alvo:** `internal/strategies/llms.go`
- **Tests:** `tests/unit/llms_strategy_test.go` (1088 lines)
- **Cenários Cobertos:**
    - Full LLM documentation extraction workflow.
