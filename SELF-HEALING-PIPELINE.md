# Self-Healing Extraction Pipeline — Plano de Design

**Status:** Em execução — Phase 0 ✅ · Phase 1 ✅ · Phase 2 ✅ (MVP funcional) · Phase 3 ✅ · Phases 4–5 ⬜
**Autor:** Claude
**Data:** 2026-05-06 · **Status atualizado:** 2026-06-26 (main @ 26806ee)
**Escopo:** `internal/app`, `internal/strategies`, `internal/domain`, `internal/recovery` (novo), `cmd/repodocs`

---

## 1. Sumário executivo

O repodocs hoje toma uma decisão **one-shot** de strategy baseada em sinais probabilísticos (URL pattern, robots.txt, probes de sitemap). Quando esses sinais mentem — caso real: `doc.rust-lang.org/sitemap.txt` lista 3 URLs-índice e `--filter` reduz isso a zero — o pipeline retorna `nil` (sucesso) com 0 documentos escritos e sai em ~1 segundo.

Este documento propõe transformar a extração em um **pipeline auto-recuperante** com:

1. **Telemetria de outcome** (`StrategyResult`) substituindo `error`/`nil` como único sinal.
2. **Validação de outcome** com critérios explícitos e testáveis.
3. **Plano de fallback** construído proativamente (Plano A + B + C).
4. **Probes diagnósticos** que refinam o plano após uma falha.
5. **Cache de receitas por domínio** que acelera runs subsequentes.

Entrega em **5 phases incrementais**, ~10–15 dias de dev. Phase 0+1 sozinhas (3–4d) já fecham o sintoma do caso reportado.

---

## 2. Contexto

### 2.1 Caso reproduzível

```bash
$ repodocs https://doc.rust-lang.org/ --filter https://doc.rust-lang.org/book/
INF Discovered sitemap, switching from crawler to sitemap strategy
INF URLs after filter filtered_count=0
INF Processing URLs from sitemap count=0
INF Sitemap extraction completed
INF Documentation extraction completed duration=961ms
```

Resultado: 0 arquivos no diretório de saída. Exit code 0. Nenhum warning.

### 2.2 Diagnóstico estrutural

Três defeitos compostos produzem o sintoma:

| # | Defeito | Local |
|---|---------|-------|
| D1 | Sucesso vazio = sucesso. `nil` quando 0 docs escritos. | `internal/strategies/sitemap.go:106-113` |
| D2 | Auto-descoberta de sitemap ignora path da URL/filtro. | `internal/strategies/sitemap_discovery.go:104` |
| D3 | Sem laço de feedback: strategy errada não é detectada nem corrigida. | `internal/app/orchestrator.go:130-220` |

D1 e D2 isoladas são heurísticas frágeis. D3 é o problema **de fundo**: não existe loop reativo. Resolver D1+D2 sem D3 só atrasa o próximo caso patológico (sitemap raso + sem filtro, blocking 403, redirect chain quebrada, etc.).

### 2.3 Por que não basta um patch ad-hoc

- "Adicionar warn quando count=0" → não resolve, só explicita. User ainda fica com 0 docs.
- "Validar prefix do sitemap antes de switchar" → resolve este caso mas é pattern-match infinito em casos novos.
- "Adicionar flag `--strategy crawler`" → joga decisão pro user, não escala.

A solução **estrutural** é fechar o loop: detectar outcome ruim, diagnosticar, tentar alternativa, parar quando satisfatório.

---

## 3. Princípios de design

| Princípio | Implicação |
|-----------|-----------|
| **Outcome > Execution** | Validar o que foi entregue, não só se errou. |
| **Plano explícito > Mágica reativa** | Plano A+B+C construído antes da execução, logado. |
| **Probes baratos** | Cada probe ≤ 2s, total ≤ 10s/run, sempre cacheado. |
| **Loud por default** | Todo fallback é `Info`, não `Debug`. User sempre entende por que rodou N strategies. |
| **Override vence** | `--strategy X` ou manifest com strategy explícita desliga fallback. |
| **Bounded budget** | Max 3 attempts/run, max 2× tempo da estratégia original. |
| **Memorize, mas valide** | Recipe cache acelera runs futuras, mas é validado por probe leve antes de aplicar. |
| **Fallback ≠ Retry** | Erros transitórios (rede, rate limit, 5xx temporário) são responsabilidade do fetcher. Fallback é para erros **lógicos** (strategy errada). |

---

## 4. Arquitetura proposta

### 4.1 Fluxo atual vs. proposto

```
ATUAL
─────
URL → Detect → [Sitemap.Execute] ──► error/nil ──► done

PROPOSTO
────────
                ┌────────────────────────────────┐
                │ DomainRecipeCache.Lookup(url)  │
                └──────────────┬─────────────────┘
                               │  hit/miss
                               ▼
URL ─► Detect ─► FallbackPlanner.Build ──► Plan{A,B,C}
                                                │
                                                ▼
            ┌────────  loop (max 3 attempts) ─────────┐
            │                                          │
            │   Strategy.Execute(attempt)              │
            │           │                              │
            │           ▼                              │
            │   StrategyResult                         │
            │           │                              │
            │           ▼                              │
            │   OutcomeValidator.Validate              │
            │           │                              │
            │   ┌───────┼───────┐                      │
            │   ▼       ▼       ▼                      │
            │  OK   Retry   HardFail                   │
            │   │     │        │                       │
            │   │     ▼        ▼                       │
            │   │   Probes  return UserError          │
            │   │   refine                            │
            │   │   plan                              │
            │   ▼                                      │
            │  Save Recipe → Summary → done            │
            └──────────────────────────────────────────┘
```

### 4.2 Componentes

#### 4.2.1 `domain.StrategyResult`

Substitui `error`-only como contrato de strategy. Definido em `internal/domain/`.

```go
// StrategyResult reports the outcome of a Strategy.Execute call.
// All counters are populated even when err is non-nil — partial progress matters.
type StrategyResult struct {
    Strategy       string
    EntryURL       string        // URL the strategy was invoked with
    URLsDiscovered int           // total URLs the strategy found before filters
    URLsAttempted  int           // URLs that survived filters and were tried
    DocsWritten    int           // documents successfully written to disk
    DocsSkipped    int           // skipped due to incremental sync, force=false, etc.
    DocsFailed     int           // attempted but errored mid-pipeline
    BytesWritten   int64
    Diagnostics    []Diagnostic  // structured signals for the validator
    Duration       time.Duration
}

type Diagnostic struct {
    Code    DiagnosticCode  // enum: filter_zeroed, sitemap_shallow, all_fetches_blocked, ...
    Message string          // human-readable
    Hint    string          // suggested next step (consumed by Planner)
}

type DiagnosticCode string

const (
    DiagFilterZeroed       DiagnosticCode = "filter_zeroed"
    DiagSitemapShallow     DiagnosticCode = "sitemap_shallow"
    DiagAllFetchesFailed   DiagnosticCode = "all_fetches_failed"
    DiagAllFetchesBlocked  DiagnosticCode = "all_fetches_blocked"  // 403/429
    DiagEmptyContent       DiagnosticCode = "empty_content"
    DiagRedirectLoop       DiagnosticCode = "redirect_loop"
    DiagJSRequired         DiagnosticCode = "js_required"           // SPA without --render-js
)
```

**Mudança de assinatura** (breaking, mas bounded):

```go
// internal/strategies/strategy.go
type Strategy interface {
    Name() string
    CanHandle(url string) bool
    Execute(ctx context.Context, url string, opts Options) (*domain.StrategyResult, error)
}
```

Strategies que ainda não foram migradas usam `domain.NewBasicResult(strategyName, url)` e populam o mínimo (`DocsWritten`, `Duration`). Validator trata resultado parcial como "telemetria insuficiente, não dispara fallback inteligente — mas ainda detecta `DocsWritten == 0`".

#### 4.2.2 `recovery.OutcomeValidator`

Componente puro (sem I/O). Recebe `(*StrategyResult, error, Options)` e devolve `Verdict`.

```go
// internal/recovery/validator.go
package recovery

type Verdict interface{ verdict() }

type VerdictOK struct{}
type VerdictRetryAlternative struct {
    Reason     string
    Diagnostics []domain.Diagnostic
}
type VerdictHardFail struct {
    Reason string
    Cause  error
}
type VerdictPropagate struct{ // erro transitório, não é nosso lugar de tratar
    Cause error
}

func (VerdictOK) verdict()              {}
func (VerdictRetryAlternative) verdict() {}
func (VerdictHardFail) verdict()         {}
func (VerdictPropagate) verdict()        {}

type OutcomeCriteria struct {
    MinDocsWritten   int     // default 1
    MinSuccessRatio  float64 // default 0.1 (DocsWritten / URLsAttempted)
    MaxAttempts      int     // default 3
}

func (v *Validator) Validate(r *domain.StrategyResult, err error, opts Options) Verdict {
    // 1. Transient errors propagate (fetcher already retried)
    if err != nil && IsTransient(err) { return VerdictPropagate{Cause: err} }

    // 2. Logical errors → check if alternatives exist
    if err != nil { return VerdictRetryAlternative{Reason: err.Error()} }

    // 3. Empty success
    if r.URLsAttempted == 0 && opts.FilterURL != "" {
        return VerdictRetryAlternative{Reason: "filter_zeroed", Diagnostics: r.Diagnostics}
    }
    if r.DocsWritten == 0 && r.URLsAttempted < 5 {
        return VerdictRetryAlternative{Reason: "strategy_misdetected", Diagnostics: r.Diagnostics}
    }
    if r.URLsAttempted > 20 && float64(r.DocsWritten)/float64(r.URLsAttempted) < v.criteria.MinSuccessRatio {
        return VerdictRetryAlternative{Reason: "high_failure_ratio", Diagnostics: r.Diagnostics}
    }

    // 4. Met threshold
    if r.DocsWritten >= v.criteria.MinDocsWritten { return VerdictOK{} }

    return VerdictHardFail{Reason: "below threshold", Cause: ErrInsufficientOutput}
}
```

Critérios são **explícitos**, **testáveis**, e **sobrescritíveis** por flags (`--min-docs N`, `--min-success-ratio F`).

#### 4.2.3 `recovery.FallbackPlanner`

Constrói plano A+B+C *antes* de executar. Recebe sinais coletados durante detecção; produz lista ordenada de tentativas.

```go
// internal/recovery/planner.go
type StrategyAttempt struct {
    Strategy domain.StrategyType
    URL      string                 // pode diferir da URL original (ex: FilterURL como entry)
    Reason   string                 // "primary detection", "filter URL fallback", "git probe"
    Budget   time.Duration
}

type FallbackPlan struct {
    Original   StrategyAttempt
    Alternatives []StrategyAttempt
    cursor     int
}

func (p *FallbackPlan) Next() (StrategyAttempt, bool) { /* ... */ }
func (p *FallbackPlan) RefineWith(probeResults []ProbeResult) { /* reorder/insert */ }

type Planner struct {
    probes  *ProbeRunner
    rules   []HeuristicRule  // regras compiladas, não YAML externo (yet)
}

func (pl *Planner) Build(ctx context.Context, url string, opts Options, signals DetectionSignals) *FallbackPlan {
    plan := &FallbackPlan{
        Original: StrategyAttempt{ Strategy: signals.PrimaryStrategy, URL: url, Reason: "primary detection" },
    }

    // Rule: FilterURL é sempre uma alternative entry implícita
    if opts.FilterURL != "" && opts.FilterURL != url {
        plan.Alternatives = append(plan.Alternatives, StrategyAttempt{
            Strategy: domain.StrategyCrawler,
            URL:      opts.FilterURL,
            Reason:   "filter URL as alternative entry",
        })
    }

    // Rule: Sitemap raso (< 10 URLs) com filter setado é suspeito
    if signals.SitemapURLCount > 0 && signals.SitemapURLCount < 10 && opts.FilterURL != "" {
        // promover crawler@FilterURL para 1ª alternativa (já adicionado acima)
    }

    // Rule: GitHub Pages → repo Git pode ser último recurso
    if signals.IsGitHubPages {
        plan.Alternatives = append(plan.Alternatives, StrategyAttempt{
            Strategy: domain.StrategyGit,
            URL:      signals.GitHubRepoURL,
            Reason:   "github pages backed by git repo",
        })
    }

    return plan
}
```

`HeuristicRule` é estrutura simples; rules ficam em `internal/recovery/rules.go` como Go puro. Não exteriorizar para YAML até passar de ~15 regras.

#### 4.2.4 `recovery.Probes`

Probes baratos para diferenciar causas pós-falha. Cada probe implementa:

```go
// internal/recovery/probes.go
type Probe interface {
    Name() string
    Cost() time.Duration   // upper bound
    Run(ctx context.Context, target string) ProbeResult
}

type ProbeResult struct {
    Probe   string
    Target  string
    Outcome ProbeOutcome  // success/failure/inconclusive
    Data    map[string]any
}
```

Set inicial de probes:

| Probe | Custo | Detecta |
|-------|-------|---------|
| `URLAlive` | 1× HEAD | URL responde, content-type, redirect chain |
| `HasOwnSitemap` | 1× GET | `<url>/sitemap.xml` ou `/sitemap.txt` existe |
| `LooksLikeMdBook` | reutiliza HTML do URLAlive | sinaliza crawler shallow é viável |
| `LooksLikeIndexPage` | mesmo HTML | muitos `<a>` para subpáginas (>30) |
| `LLMSTxtOnAncestor` | até 3 GETs | `/llms.txt` em qualquer ancestral do path |
| `IsGitHubPagesBacked` | 1× lookup CNAME + HTML scrape | GitHub Pages → tem repo Git |
| `RobotsAllowsCrawl` | 1× GET (cacheado) | robots.txt permite User-Agent atual |

Probes rodam **sob demanda** — só os que diferenciam alternativas restantes do plano. Resultados são cacheados em-memória por run e (opcionalmente) em Badger por TTL curto (1h).

#### 4.2.5 `recovery.RecipeCache`

Memória de longo prazo por domínio. Bucket no Badger existente com prefix `recipe:`.

```go
// internal/recovery/recipes.go
type Recipe struct {
    Domain          string
    PreferredEntry  string         // URL real que funcionou (pode ser diferente da informada)
    PreferredStrategy domain.StrategyType
    SignalOverrides map[string]any // ex: {"ignore_root_sitemap": true}
    ValidatedAt     time.Time
    DocsAvg         int
    Version         int            // schema version para migração
}

type RecipeCache struct {
    db  *badger.DB
    ttl time.Duration  // default 7d
}

func (rc *RecipeCache) Lookup(domain string) (*Recipe, bool) { /* ... */ }
func (rc *RecipeCache) Save(r *Recipe) error                 { /* ... */ }
func (rc *RecipeCache) Invalidate(domain string) error       { /* ... */ }
```

Regras de aplicação:
- Lookup acontece **antes** da detecção. Se HIT e não-stale, planner usa `PreferredEntry` + `PreferredStrategy` como Plano A.
- Antes de aplicar, um probe leve (`URLAlive(PreferredEntry)`) confirma que ainda funciona. Se 404 ou redirect inesperado → invalida e cai pra detecção normal.
- TTL 7d. Salvar só após `VerdictOK` com `DocsWritten >= 5` (evita memorizar runs marginais).
- Flag `--ignore-recipes` desliga lookup; útil para CI determinístico e debugging.

#### 4.2.6 Loop integrado no Orchestrator

Substitui `internal/app/orchestrator.go:130-220`:

```go
func (o *Orchestrator) Execute(ctx context.Context, url string, opts ExecuteOptions) error {
    // 1. Recipe cache lookup
    if !opts.IgnoreRecipes {
        if recipe, ok := o.recipes.Lookup(domainOf(url)); ok && !o.recipes.IsStale(recipe) {
            if probeOK := o.probes.QuickValidate(ctx, recipe); probeOK {
                url = recipe.PreferredEntry
                opts.StrategyOverride = string(recipe.PreferredStrategy)
                o.logger.Info().Str("recipe", recipe.Domain).Msg("Applied cached domain recipe")
            } else {
                o.recipes.Invalidate(recipe.Domain)
            }
        }
    }

    // 2. Detection (existing logic, encapsulado em DetectionSignals)
    signals := o.detector.Detect(ctx, url, opts)

    // 3. Build plan (only if no explicit override)
    var plan *recovery.FallbackPlan
    if opts.StrategyOverride != "" || opts.NoFallback {
        plan = recovery.SinglePlan(signals.PrimaryStrategy, url)
    } else {
        plan = o.planner.Build(ctx, url, opts, signals)
    }

    // 4. Loop
    var lastResult *domain.StrategyResult
    var lastErr error
    for attempt, hasMore := plan.Next(); hasMore; attempt, hasMore = plan.Next() {
        if err := ctx.Err(); err != nil { return err }

        result, err := o.runAttempt(ctx, attempt, opts)
        lastResult, lastErr = result, err

        switch v := o.validator.Validate(result, err, opts).(type) {
        case recovery.VerdictOK:
            o.recipes.Save(recipeFromAttempt(attempt, result))
            o.summary.Emit(plan, lastResult)
            return nil

        case recovery.VerdictRetryAlternative:
            o.logger.Warn().
                Str("strategy", attempt.Strategy.String()).
                Str("reason", v.Reason).
                Interface("diagnostics", v.Diagnostics).
                Msg("Strategy outcome unsatisfactory, falling back")
            plan.RefineWith(o.probes.RunRelevant(ctx, plan.Remaining(), v.Diagnostics))
            continue

        case recovery.VerdictPropagate:
            return v.Cause

        case recovery.VerdictHardFail:
            return o.errors.UserFacingError(v, plan, lastResult)
        }
    }

    return o.errors.PlanExhausted(plan, lastResult, lastErr)
}
```

---

## 5. Plano de execução faseado

### Phase 0 — Contratos e infra (fundação) — ✅ Concluída

**Objetivo:** Definir tipos novos sem mudar comportamento.

**Arquivos:**
- `internal/domain/strategy_result.go` — `StrategyResult`, `Diagnostic`, codes.
- `internal/domain/errors.go` — `ErrInsufficientOutput`, `ErrPlanExhausted`, `IsTransient(err) bool`.
- `internal/recovery/doc.go` — pacote stub com sumário.
- `internal/strategies/strategy.go` — atualizar interface `Strategy` (assinatura nova).

**Migração das 8 strategies:**
- crawler, sitemap, llms, pkggo, docsrs, wiki, github_pages, git
- Cada uma: instanciar `result := domain.NewBasicResult(s.Name(), url)`, popular contadores, retornar `(result, nil)`.
- Mock em `tests/mocks/domain.go` regenerado via `go generate`.

**Critério de aceite:**
- `make build && make test` passa.
- Comportamento idêntico ao atual (telemetria coletada mas não consumida).
- Nenhuma flag nova.

**Risco:** baixo. Refactor mecânico.

**Esforço:** 1–2 dias.

---

### Phase 1 — Validator + erro útil quando vazio — ✅ Concluída

**Objetivo:** Resolver o sintoma imediato. Saída silenciosa vira erro claro.

**Arquivos:**
- `internal/recovery/validator.go` — `OutcomeValidator`, verdicts.
- `internal/recovery/errors.go` — formatadores de erro user-facing.
- `internal/app/orchestrator.go` — chamar validator após `Execute`. Se `VerdictHardFail`, retornar erro útil.

**Sem fallback ainda.** Apenas validação + mensagem útil:

```
Error: extraction produced 0 documents

Strategy:    sitemap (https://doc.rust-lang.org/sitemap.txt)
Discovered:  3 URLs
After filter: 0 URLs (filter excluded all)
Diagnostics: filter_zeroed, sitemap_shallow

Suggestions:
  - Try a different entry point: repodocs https://doc.rust-lang.org/book/
  - Force the crawler strategy: repodocs ... --strategy crawler
  - Check filter URL spelling: --filter https://doc.rust-lang.org/book/
```

**Critério de aceite:**
- Caso rust-book retorna erro com sugestões, exit code != 0.
- Tests: integration test reproduzindo `filter_zeroed`.
- Nenhuma run que antes funcionava regride.

**Esforço:** 1–2 dias.

---

### Phase 2 — Fallback de 1 nível + flags CLI — ✅ Concluída

> **Status (2026-06-26):** entregue. O orchestrator agora aciona auto-recuperação no `VerdictRetryAlternative` em vez de só retornar `OutcomeError`. Decisões de implementação (revisadas adversarialmente):
> - **Planner puro em `internal/recovery/planner.go`** (importa só `domain`) — testável isolado.
> - **Loop em `internal/app/fallback.go`** (`runWithFallback` + `execAttempt`), não em `recovery/runner.go`: o loop já precisa de `CreateStrategy`/`Options`/`Execute` (tudo concern de `app`), então a abstração `Runner`/`ExecFunc` foi descartada por over-engineering.
> - **Só gatilhos de zero-escrita disparam fallback** (`filter_zeroed`, `no_urls_attempted`); `high_failure_ratio` fica de fora (pode já ter escrito docs → risco de overwrite).
> - **`--strategy`/manifest forçado suprime fallback** (respeita a escolha explícita).

**Objetivo (entregue):** Auto-recuperação para os casos de zero-output mais comuns:
- (a) filtro zerou (sitemap) → crawler escopado ao subtree filtrado (mantém o filtro). **R1**
- (b) sitemap raso sem filtro → crawler na origem do site. **R3**

**Arquivos:**
- `internal/recovery/planner.go` — Planner puro (R1 + R3) + `planner_test.go` (tabela).
- `internal/app/fallback.go` — `runWithFallback` (loop de 1 nível) + `execAttempt` (caminho de execução único).
- `internal/app/orchestrator.go` — campo `planner`, `NoFallback` em `OrchestratorOptions`, `Run` chama `runWithFallback` e reusa o switch de veredito.
- `cmd/repodocs/main.go` — flags:
  - `--strategy {crawler|sitemap|...}` — força strategy, desliga fallback (já existia).
  - `--no-fallback` — desliga fallback explicitamente.
  - `--min-docs N` (0 = default de 1).

**Critério de aceite (atendido):**
- Caso rust-book auto-recupera (crawler escopado em `/book/` escreve N>0 docs). ✅
- Manifests/`--strategy` explícito mantêm comportamento determinístico (sem fallback). ✅
- E2E `TestE2E_SelfHealingFallback_RustBook` (`tests/e2e/self_healing_test.go`): positivo + `--no-fallback` + override. ✅

**Esforço:** 2–3 dias (entregue).

---

### Phase 3 — Probes diagnósticos + planner refinado — ✅ Concluída

> **Status (2026-06-26):** entregue. Quando o Plano B estático (R1/R3) também não recupera, o orchestrator roda **probes diagnósticos baratos** e refina o plano (Plano C). Decisões de implementação (revisadas adversarialmente, no mesmo espírito da Phase 2):
> - **Pureza preservada:** `internal/recovery` continua importando só `domain` + stdlib. Os probes recebem `domain.Fetcher` injetado (mesmo seam do `DiscoverSitemap`); checagens de conteúdo (`looksLikeSitemap`, `countAnchors`) foram **reimplementadas localmente** em vez de importar `internal/strategies`, para não quebrar o invariante.
> - **7 → 4 probes (cada um consumido).** Mantidos apenas os probes que produzem ou habilitam um `Attempt` real em `RefineWith`, evitando dead code: `llms_txt_on_ancestor` → `llms`; `has_own_sitemap` → `sitemap`; `looks_like_index_page` → `crawler`; `github_pages_backed` → `git`. Descartados `URLAlive` (redundante: os probes acionáveis já confirmam liveness), `LooksLikeMdBook` (redundante com index) e `RobotsAllowsCrawl` (apenas consultivo; o crawler já respeita robots internamente).
> - **Budget de tentativas duro:** `maxFallbackAttempts = 2` → no máximo 3 execuções de estratégia por run (inicial + 2 fallbacks). Probes são limitados (2s/probe, 6s total) e **nunca escrevem em disco**.
> - **Dedup entre tiers:** um `tried` set (`strategy\x00url\x00filter`) impede re-executar um candidato já tentado (ex.: `crawler@origin` proposto tanto pela R3 quanto pelo probe de index).
> - **`--no-fallback`/`--strategy` continuam suprimindo tudo** (incluindo probes) — sem novas flags na Phase 3.

**Arquivos:**
- `internal/recovery/probes.go` — `ProbeRunner` (paralelo, com budget) + 4 probes + helpers puros (`llmsCandidates`, `joinPath`, `githubRepoFromPagesHost`, …).
- `internal/recovery/probe_cache.go` — `fetchCache`: single-flight em-memória por run sobre `Fetcher.Get` (deduplica fetches compartilhados entre probes).
- `internal/recovery/planner.go` — método `RefineWith([]ProbeResult)` mapeando probes vencedores → `Attempt`s ordenados (mais barato primeiro: llms < sitemap < crawl < git).
- `internal/app/fallback.go` — `runWithFallback` agora tem 2 tiers (estático + probes) com budget + dedup; helpers `tryFallback`, `validationOpts`, `logProbes`, `attemptKey`.
- `internal/app/orchestrator.go` — campo `probeRunner`, inicializado com `recovery.NewProbeRunner(deps.Fetcher)`.

**Critério de aceite (atendido):**
- Probes e helpers com unit tests usando `httptest.Server` + `domain.Fetcher` stub (`internal/recovery/probes_test.go`, `probe_cache_test.go`); `RefineWith` com tabela (`planner_test.go`). ✅
- Caso sintético end-to-end: sitemap com `--filter` zerado **e** `/book/` 404 (Plano B falha) → probe acha `/llms.txt` na origem → fallback LLMS escreve docs (`TestE2E_SelfHealingProbeRecovery`). ✅
- Budget total de probing logado (`Diagnostic probes completed … probe_budget=…`) e ≤ 6s/run. ✅

**Esforço:** 3–4 dias (entregue).

---

### Phase 4 — Recipe cache

**Objetivo:** Acelerar runs subsequentes no mesmo domínio. Lembrar que `doc.rust-lang.org` quer crawler@/book/.

**Arquivos:**
- `internal/recovery/recipes.go` — `RecipeCache` sobre Badger existente.
- `internal/cache/recipe_codec.go` — encode/decode JSON.
- `cmd/repodocs/main.go` — flag `--ignore-recipes`.

**Critério de aceite:**
- Recipe salva após `VerdictOK` com `DocsWritten >= 5`.
- Lookup pre-detection com probe rápido de validação (`URLAlive`).
- TTL 7d, expiração testada.
- Recipe inválida (URL agora 404) é descartada silenciosamente, run prossegue.
- Comando `repodocs config recipes` para listar/limpar (subcomando opcional).

**Esforço:** 2 dias.

---

### Phase 5 — UX e observability

**Objetivo:** User entende o que aconteceu sem ler código.

**Arquivos:**
- `internal/recovery/summary.go` — formatador com lipgloss.
- `cmd/repodocs/main.go` — flag `--explain`.

**Output `--explain`:**

```
Extraction plan for https://doc.rust-lang.org/

  Recipe lookup        MISS
  Detection            crawler (default), then sitemap auto-discovered
  Plan
    [A] sitemap   /sitemap.txt              (primary detection)
    [B] crawler   /book/                    (filter URL as entry)
    [C] git       github.com/rust-lang/book (probe-discovered, deferred)

Execution
  [A] sitemap                               0 docs / 0 attempts        180ms
       diagnostics: filter_zeroed, sitemap_shallow
       verdict: RetryAlternative
  Probes
    URLAlive(/book/)         200 text/html  (320 KB)        420ms
    HasOwnSitemap(/book/)    not found                       110ms
    LooksLikeMdBook          true                              0ms
  [B] crawler /book/                        412 docs / 412 attempts    2m17s
       verdict: OK

Recipe saved
  doc.rust-lang.org → crawler@/book/   (TTL 7d)

Summary: 412 docs, 8.4 MB, 2m18s total
```

**Output default (sem `--explain`):**

```
Extraction completed via fallback (1 alternative tried)
  ✗ sitemap        0 docs   filter excluded all 3 URLs
  ✓ crawler /book/ 412 docs 2m17s 8.4 MB
  Recipe cached for doc.rust-lang.org
```

**Critério de aceite:**
- Saída legível com cor (TTY) e plain (não-TTY).
- Modo `--explain` produz árvore completa.
- Telemetria sumarizada em `output.json` quando `--json-meta`.

**Esforço:** 1 dia.

---

### Resumo de phases

| Phase | Entrega | Esforço | Status |
|-------|---------|---------|--------|
| 0 | StrategyResult + interface migrada | 1–2d | ✅ Concluída |
| 1 | Validator + erro útil | 1–2d | ✅ Concluída |
| 2 | Fallback 1 nível + flags CLI | 2–3d | ✅ Concluída — planner puro + loop em `app/fallback.go` + `--no-fallback`/`--min-docs` |
| 3 | Probes + planner refinado | 3–4d | ✅ Concluída — `ProbeRunner` (4 probes) + `fetchCache` + `RefineWith`; Tier 2 em `app/fallback.go` (budget 3, dedup) |
| 4 | Recipe cache | 2d | ⬜ Pendente |
| 5 | UX/observability | 1d | ⬜ Pendente |

**MVP funcional entregue (Phase 0+1+2).** Phases 3–5 são polish progressivo. **Estado atual: Phases 0, 1, 2 e 3 entregues — o MVP de auto-recuperação está fechado e a recuperação por probes (Plano C) está ativa.**

---

## 6. Trade-offs e alternativas consideradas

### 6.1 Mudar assinatura de `Strategy.Execute` vs interface opcional

**Considerada:** adicionar `interface ResultReporter` que strategies *podem* implementar.

**Rejeitada porque:** validator não pode confiar em telemetria opcional. Forçar a contract dá poder ao loop. Migração das 8 strategies é mecânica e bounded.

### 6.2 Pacote `recovery` separado vs dentro de `app`

**Considerada:** colocar tudo em `internal/app/`.

**Escolhida:** pacote separado `internal/recovery/`. Justificativa: recovery tem responsabilidade focada (validar+planejar+fallback), pode ser testado isoladamente, e cresce no tempo. App fica fino: só coordena.

### 6.3 Heurísticas em Go vs YAML externo

**Considerada:** YAML em `~/.repodocs/heuristics.yaml` editável pelo user.

**Adiada:** começar com Go interno. Externalizar quando passar de 15 regras. YAML traz schema, validação, evolução — overhead prematuro.

### 6.4 Recipe cache: bucket no Badger existente vs DB separado

**Considerada:** segundo BadgerDB para isolar.

**Escolhida:** prefix `recipe:` no Badger existente. Reutiliza config, evita segunda lifecycle, schema-versioning trivial via field `Version`.

### 6.5 Probes em paralelo vs sequenciais

**Escolhida:** probes do mesmo plan-refinement em paralelo (`errgroup`), com timeout total de 5s. Cada probe individual com timeout próprio.

### 6.6 Fallback automático sempre on vs opt-in

**Escolhida:** sempre on. Opt-out via `--no-fallback`, override explícito (`--strategy`, manifest), e ambientes CI (variável `REPODOCS_NO_FALLBACK=1`).

Justificativa: o problema que motiva o trabalho **só aparece** em uso interativo. Auto-on resolve UX. Determinismo continua possível para quem precisa.

### 6.7 Strategy pattern atual vs Pipeline pattern (radical)

**Considerada:** refatorar para pipeline declarativo (Discover → Filter → Fetch → Convert → Write como steps separados).

**Rejeitada:** escopo grande demais. Strategies atuais já encapsulam pipelines internos. Manter strategy como unidade. Recovery age **acima** delas.

---

## 7. Anti-padrões a evitar

| Anti-padrão | Mitigação |
|-------------|-----------|
| Mágica silenciosa: fallback sem log loud | Toda transição é `Info` log com motivo estruturado |
| Fallback agressivo em erros transitórios | `IsTransient(err)` curto-circuita com `VerdictPropagate` |
| Override implícito vs explícito | `--strategy` e manifest `strategy:` desligam fallback |
| Probes caros sem cache | Budget duro 2s/probe, cache em-memória por run |
| Receitas que apodrecem | TTL 7d + probe `URLAlive` antes de aplicar |
| Recovery loop infinito | Max 3 attempts, mesma strategy não retenta com mesma config |
| Heurística virando lookup table de 200 entradas | Externalizar para YAML quando passar de 15 regras (ainda não) |
| Mudança breaking sem migração | Strategy interface mudada com adapter `BasicResult` para minimizar churn |
| Fallback executando trabalho duplicado | Cada attempt usa cache do fetcher; URLs já fetched são re-aproveitadas |

---

## 8. Backward compatibility

### 8.1 CLI

- Nenhuma flag existente muda comportamento.
- Flags novas (`--strategy`, `--no-fallback`, `--min-docs`, `--explain`, `--ignore-recipes`) são aditivas.
- Exit codes: `0` para sucesso, `1` para erro genérico (igual hoje), `2` adicionado para `ErrPlanExhausted` (novo).

### 8.2 Manifests

- Manifests existentes funcionam sem mudança.
- Manifest com `strategy:` explícito → fallback desligado (preserva determinismo de batches).
- Campo opcional novo: `recovery: {disabled: true, min_docs: 5}` por source.

### 8.3 Config (`~/.repodocs/config.yaml`)

Seção nova opcional:

```yaml
recovery:
  enabled: true             # default
  max_attempts: 3
  min_docs: 1
  min_success_ratio: 0.1
  probe_budget: 10s
  recipes:
    enabled: true
    ttl: 168h               # 7d
```

### 8.4 API interna

- `Strategy.Execute` muda assinatura — **breaking** para forks/embeddings externos.
- Mitigação: documento de migração + adapter shim em `internal/strategies/legacy_adapter.go` (opcional, descartar após release).

### 8.5 Cache (Badger)

- Bucket `recipe:` é novo. Caches existentes intactos.
- Schema do `Recipe` versionado (`Version int`). Migração futura via `RecipeCodec`.

---

## 9. Métricas de sucesso e validação

### 9.1 Métricas funcionais

| Métrica | Antes | Meta após Phase 5 |
|---------|-------|-------------------|
| Caso rust-book (`--filter book/`) baixa docs | ✗ 0 docs | ✓ ≥ 400 docs |
| Erro silencioso (exit 0, 0 docs) | possível | impossível |
| Saída inteligível em falha | log linha única | árvore com sugestões |
| Run repetida no mesmo domínio | redescobre tudo | recipe hit, ~2× mais rápido |

### 9.2 Métricas técnicas

- **Test coverage:** ≥ 80% em `internal/recovery/`.
- **Regression:** suite e2e existente passa 100% sem mudança.
- **Latência:** overhead de fallback path quando Plano A funciona ≤ 50ms (probe + recipe lookup).
- **Budget probes:** total ≤ 10s/run mensurado em e2e.

### 9.3 Casos de teste obrigatórios

| Cenário | Phase | Tipo |
|---------|-------|------|
| Sitemap raso + filter zerado → crawler@filter | 2 | E2E |
| Sitemap rico + filter funciona → no fallback | 2 | E2E |
| Crawler @ root retorna 0 → fallback sitemap | 2 | E2E |
| Site bloqueado 403 em todas → HardFail útil | 2 | E2E |
| Erro transitório 503 + Retry-After → não dispara fallback | 2 | Integration |
| `--strategy crawler` com filter zerado → erro, sem fallback | 2 | E2E |
| GitHub Pages quebrado + repo Git OK → fallback git | 3 | E2E |
| Recipe stale (URL 404 hoje) → invalidado, run normal | 4 | Integration |
| `--explain` produz árvore parseável | 5 | Unit (snapshot) |

---

## 10. Open questions

1. **Composição com `--sync`/`--full-sync`/`--prune`:** se Phase 1 falha mas escreveu N docs, sync state preserva ou descarta? Proposta: preservar. Recovery não invalida progresso parcial.

2. **Composição com `--limit`:** limit divide entre attempts ou cada attempt tem limit cheio? Proposta: limit é global, contadores agregam entre attempts.

3. **Recovery em manifests batch:** se source 5/10 dispara fallback, deve continuar ou abortar manifest inteiro? Proposta: continuar, marcar source como "recovered" no relatório final.

4. **Tracking de "auto-corrected" para feedback ao desenvolvedor:** salvar local opt-in de runs que precisaram de fallback, para identificar padrões de regras heurísticas a adicionar? Adiar para pós-Phase 5.

5. **Métricas de telemetria:** considerar export Prometheus/OTel das `Diagnostic` codes? Adiar — fora do escopo atual.

6. **Recovery composição com LLM enhance:** se LLM enhance falha (rate limit), fallback dispara? Não — LLM falha é transient/independent, já tem circuit breaker próprio. Recovery vê só `DocsWritten`, não `DocsEnhanced`.

---

## 11. Próximos passos sugeridos

**Concluído na `main`:** Phase 0 (telemetria `StrategyResult` + interface migrada nas 8 strategies) e Phase 1 (validator + `OutcomeError` útil quando vazio) — commits 95ff61e, 614aed1, 26806ee. A flag `--strategy` veio no commit 80d24d7.

**Concluído (Phase 2 — fallback automático de 1 nível):** o `VerdictRetryAlternative` agora aciona auto-recuperação. Entregue em:
- `internal/recovery/planner.go` (+ `planner_test.go`) — Planner puro: R1 `filter_zeroed`/`no_urls_attempted` → crawler escopado ao subtree filtrado; R3 sitemap raso sem filtro → crawler na origem.
- `internal/app/fallback.go` — `runWithFallback` (loop de 1 nível, sem recursão) + `execAttempt` (caminho de execução único, inicial e fallback).
- `internal/app/orchestrator.go` — `Run` chama `runWithFallback` e reusa o switch de veredito; `NoFallback` em `OrchestratorOptions`.
- `cmd/repodocs/main.go` — flags `--no-fallback` e `--min-docs N`.
- `tests/e2e/self_healing_test.go` — `TestE2E_SelfHealingFallback_RustBook` (positivo + `--no-fallback` + override).

**Concluído (Phase 3 — probes diagnósticos + planner refinado):** quando o Plano B estático também falha, o `ProbeRunner` roda 4 probes baratos e o `Planner.RefineWith` sugere um Plano C. Entregue em:
- `internal/recovery/probes.go` — `ProbeRunner` (paralelo, budget 2s/probe, 6s total) + probes `llms_txt_on_ancestor` → llms, `has_own_sitemap` → sitemap, `looks_like_index_page` → crawler, `github_pages_backed` → git.
- `internal/recovery/probe_cache.go` — `fetchCache` single-flight por run.
- `internal/recovery/planner.go` — `RefineWith` mapeando probes → `Attempt`s (mais barato primeiro).
- `internal/app/fallback.go` — Tier 2 no `runWithFallback` (budget 3 execuções, dedup por `attemptKey`).
- `tests/e2e/self_healing_test.go` — `TestE2E_SelfHealingProbeRecovery` (recuperação via llms.txt quando o fallback estático falha).

**Próximo passo — Phase 4 (recipe cache):** lembrar por domínio o que funcionou (`doc.rust-lang.org` → crawler@/book/) para acelerar runs subsequentes. Arquivos: `internal/recovery/recipes.go` (sobre o Badger existente, prefix `recipe:`), `internal/cache/recipe_codec.go`, flag `--ignore-recipes`. Depois: Phase 5 (UX/observability com `--explain`).

---

## Apêndice A — Referências de código atual

| Arquivo | Linhas | Papel hoje | Mudança proposta |
|---------|--------|-----------|-------------------|
| `internal/strategies/strategy.go` | inteiro | Define `Strategy` interface | Assinatura `Execute` muda |
| `internal/strategies/sitemap.go` | 67-114 | `Execute` da sitemap strategy | Retorna `*StrategyResult` |
| `internal/strategies/sitemap_discovery.go` | 104-212 | Probes de sitemap por domínio | Mantém, mas planner usa sinais |
| `internal/app/orchestrator.go` | 130-220 | Detect + execute one-shot | Reescrito como loop de plan |
| `internal/cache/` | inteiro | Page cache sobre Badger | Adiciona prefix `recipe:` |
| `tests/mocks/domain.go` | 24-90 | Mock de `Strategy` | Regenerado via `go generate` |
| `cmd/repodocs/main.go` | 62-110 | Flags do CLI | Adiciona 5 flags |

---

## Apêndice B — Glossário

- **Outcome:** o que foi entregue (docs escritos, bytes), em oposição a **execution** (errou ou não).
- **Plano (FallbackPlan):** lista ordenada de `StrategyAttempt` com cursor, refinável durante execução.
- **Probe:** consulta barata (≤ 2s) que diferencia causas de falha. Não escreve disco.
- **Receita (Recipe):** registro persistente "para o domínio X, o que funcionou foi Y".
- **Verdict:** decisão do validator: OK / RetryAlternative / HardFail / Propagate.
- **Diagnostic:** sinal estruturado emitido por strategy durante execução (filter_zeroed, etc.).
- **Transient error:** erro de rede/rate-limit/5xx temporário — não dispara fallback.
- **Logical error:** strategy errada ou sucesso vazio — dispara fallback.
