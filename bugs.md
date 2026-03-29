# Analise de Bugs - Rate Limit e Circuit Breaker

**Data:** 2026-03-29  
**Branch:** main

Encontrados **6 bugs/problemas** nas implementacoes de rate limit e circuit breaker. Organizados por severidade.

---

## BUG 1 (Severidade Alta) - Token de rate limit consumido ANTES da verificacao do circuit breaker

**Ficheiro:** `internal/llm/provider_wrapper.go:107-118`

```go
// Primeiro: consome o token do rate limiter
if err := p.rateLimiter.Wait(ctx); err != nil {
    return nil, fmt.Errorf("rate limit wait cancelled: %w", err)
}

// Depois: verifica o circuit breaker
if !p.circuitBreaker.Allow() {
    return nil, domain.ErrLLMCircuitOpen  // Token ja foi desperdicado!
}
```

**Problema:** A ordem das operacoes esta invertida. Quando o circuit breaker esta aberto (estado `StateOpen`), o token do rate limiter ja foi consumido e desperdicado. Durante um periodo de falha prolongada, todos os tokens vao sendo drenados inutilmente. Quando o servico recupera e o circuit breaker fecha, ha poucos tokens disponiveis, causando delays desnecessarios.

**Correcao:** Inverter a ordem -- verificar o circuit breaker primeiro, e so depois consumir o token.

---

## BUG 2 (Severidade Media) - `JitterFactor` ausente na config, sempre `0.0` em producao

**Ficheiros:**
- `internal/config/config.go:38-47` -- `RateLimitConfig` nao tem campo `JitterFactor`
- `internal/strategies/strategy.go:167-179` -- `JitterFactor` nao e mapeado
- `internal/llm/provider_wrapper.go:20` -- `RateLimitedProviderConfig` tem o campo, mas recebe `0.0`

O `RateLimitConfig` no pacote `config` nao inclui `JitterFactor`. Na hora de construir o `RateLimitedProviderConfig` em `internal/strategies/strategy.go:167-179`, o campo `JitterFactor` nunca e atribuido, ficando com o zero value (`0.0`).

O `DefaultRateLimitedProviderConfig()` (`provider_wrapper.go:28-42`) define `JitterFactor: 0.1`, mas essa funcao **nunca e usada** no caminho de producao -- so nos testes.

**Impacto:** Sem jitter, todos os retries de multiplos clientes concorrentes acontecem exatamente nos mesmos instantes, criando um efeito de "thundering herd" que sobrecarrega o provider LLM.

**Correcao:** Adicionar `JitterFactor float64` ao `RateLimitConfig` em `config.go`, definir o default em `defaults.go`, e mapear o valor em `strategy.go`.

---

## BUG 3 (Severidade Media) - Half-Open permite requests ilimitados

**Ficheiro:** `internal/llm/circuit_breaker.go:96-97`

```go
case StateHalfOpen:
    return true  // Permite TODOS os requests
```

No padrao classico de circuit breaker, o estado half-open deve permitir apenas um numero limitado de requests de "sonda" para testar se o servico recuperou. Aqui, uma vez em `StateHalfOpen`, `Allow()` retorna `true` para **todas** as chamadas concorrentes.

**Impacto:** Se houver muitas goroutines a espera, todas passam simultaneamente quando o timeout expira e o breaker transiciona para half-open, potencialmente inundando um servico que acabou de recuperar e causando nova abertura do circuito.

**Correcao:** Adicionar um contador de requests permitidos em half-open (ex: `halfOpenAllowed int`) e limitar com base em `SuccessThresholdHalfOpen`, rejeitando requests adicionais ate que os probes retornem sucesso.

---

## BUG 4 (Severidade Media) - Header `Retry-After` parseado mas nunca respeitado

**Ficheiros:**
- `internal/fetcher/client.go:164` -- parseia o header e guarda em `RetryableError.RetryAfter`
- `internal/fetcher/retry.go:73-90` -- usa `cenkalti/backoff` que ignora o campo `RetryAfter`
- `internal/domain/errors.go:77` -- campo `RetryAfter int` definido

O fetcher parseia o header `Retry-After` e guarda o valor no `RetryableError`, mas o `Retrier` do fetcher usa `cenkalti/backoff` que calcula o backoff independentemente. O campo `RetryAfter` nunca e consultado em nenhum lugar.

**Impacto:** Se um servidor diz "retry after 60s" mas o backoff exponencial calcula 2s, o cliente faz retry muito cedo e recebe outro 429. O valor do servidor e completamente ignorado.

**Correcao:** No `Retrier.Retry()`, verificar se o erro e `RetryableError` com `RetryAfter > 0` e usar `max(backoff_calculado, RetryAfter)` como intervalo de espera.

---

## BUG 5 (Severidade Baixa) - Retries nao consomem tokens adicionais de rate limit

**Ficheiro:** `internal/llm/provider_wrapper.go:100-125`

```go
func (p *RateLimitedProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // 1 token consumido aqui
    if err := p.rateLimiter.Wait(ctx); err != nil { ... }

    // Retries acontecem DENTRO de Execute, sem consumir mais tokens
    err := p.retrier.Execute(ctx, func() error {
        response, err = p.provider.Complete(ctx, req)
        return err
    })
}
```

Com `MaxRetries=3` e `BurstSize=10`, um burst de 10 requests pode gerar ate 40 chamadas reais ao LLM (10 iniciais + 30 retries), ultrapassando o rate limit configurado.

**Correcao:** Mover o `rateLimiter.Wait()` para dentro da closure do `retrier.Execute()`, de modo que cada tentativa (incluindo retries) consuma um token.

---

## BUG 6 (Severidade Baixa) - `Validate()` nao valida configuracao de rate limit

**Ficheiro:** `internal/config/config.go:105-129`

O metodo `Validate()` valida concurrency, cache, rendering e git, mas **ignora completamente** os campos de `LLM.RateLimit` e `LLM.RateLimit.CircuitBreaker`. Valores invalidos como `RequestsPerMinute: -5` ou `BurstSize: 0` passam a validacao silenciosamente.

Embora `NewTokenBucket()` (`internal/llm/ratelimit.go:27-32`) e `NewCircuitBreaker()` (`internal/llm/circuit_breaker.go:64-73`) apliquem defaults para valores invalidos, o utilizador nao recebe feedback de que a sua configuracao esta incorreta.

**Correcao:** Adicionar validacao dos campos de rate limit e circuit breaker no metodo `Validate()`, retornando erros descritivos para valores invalidos.

---

## Resumo

| # | Severidade | Bug | Localizacao |
|---|-----------|-----|-------------|
| 1 | **Alta** | Token consumido antes de verificar circuit breaker | `provider_wrapper.go:107-118` |
| 2 | **Media** | `JitterFactor` sempre `0.0` (thundering herd) | `config.go:38-47`, `strategy.go:167-179` |
| 3 | **Media** | Half-open permite requests ilimitados | `circuit_breaker.go:96-97` |
| 4 | **Media** | `Retry-After` parseado mas nunca respeitado | `client.go:164`, `retry.go:73-90` |
| 5 | **Baixa** | Retries nao consomem tokens de rate limit | `provider_wrapper.go:100-125` |
| 6 | **Baixa** | Sem validacao de config de rate limit/circuit breaker | `config.go:105-129` |
