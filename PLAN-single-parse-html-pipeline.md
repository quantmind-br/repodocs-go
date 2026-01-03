# PLAN: Single-Parse HTML Pipeline

## Executive Summary

Reduzir o custo de parsing no `Pipeline.Convert` para, idealmente, **1 parse por documento** quando o seletor CSS é usado e **2 parses** quando a rota de Readability é necessária. A abordagem introduz variantes *doc-aware* (operam sobre `*goquery.Document`/`*goquery.Selection`) para extração, sanitização, headers e links, eliminando re-parses redundantes.

---

## Problema

### Estado atual (parsing repetido)
Em `internal/converter/pipeline.go`, um único documento HTML pode ser parseado 4-6 vezes:
1. `ExtractContent` (goquery quando há seletor) **ou** Readability (novo parse interno).
2. `Sanitizer.Sanitize` (goquery).
3. `ExtractDescription` (goquery em doc do HTML original).
4. `ExtractHeaders` (goquery no HTML sanitizado).
5. `ExtractLinks` (goquery no HTML sanitizado).
6. `removeExcluded` (goquery adicional quando `ExcludeSelector` é usado).

### Impacto
- Parsing HTML é O(n); em documentos grandes, cada parse amplifica custo de CPU e memória.
- A pipeline cria múltiplas árvores DOM sem reaproveitamento.

---

## Objetivos

1. **Consolidar parsing** em uma única árvore DOM sempre que possível.
2. **Reaproveitar DOM** para sanitização e extrações (headers, links) sem reparse.
3. **Manter compatibilidade** de saída (Markdown/metadata) com mudanças mínimas.
4. **Preservar Readability** como fallback, mas reduzir seus parses adjacentes.

### Não objetivos
- Reescrever Readability.
- Alterar o formato final do Markdown ou metadata.
- Trocar goquery por outro parser.

---

## Abordagem Geral

### Ideia central
Criar versões *doc-aware* das funções de extração e sanitização para trabalhar diretamente em `*goquery.Document` ou `*goquery.Selection`, permitindo:
- 1 parse do HTML original quando `ContentSelector` está presente.
- 2 parses quando Readability é usado (1 para metadata do HTML original + 1 para sanitização/extras do conteúdo extraído).

### Novo fluxo proposto (alto nível)

**Caso A: ContentSelector presente**
1. Parse HTML original **uma vez** → `origDoc`.
2. Extrair conteúdo via seleção no `origDoc`.
3. Aplicar `ExcludeSelector` e sanitização diretamente no DOM/Selection.
4. Extrair headers/links do DOM sanitizado (sem reparse).
5. Converter HTML sanitizado em Markdown.
6. Extrair descrição do `origDoc`.

**Caso B: Readability (fallback)**
1. Parse HTML original **uma vez** → `origDoc` (para descrição).
2. Readability parse (interno) para conteúdo + título.
3. Parse **apenas uma vez** o HTML extraído → `contentDoc`.
4. Sanitizar `contentDoc` e extrair headers/links do mesmo DOM.

---

## Mudanças de API (detalhadas)

### 1) ExtractContent (doc-aware)
**Arquivo:** `internal/converter/readability.go`

Adicionar método novo:
- `ExtractFromDocument(doc *goquery.Document, sourceURL string) (contentHTML, title string, err error)`

Comportamento:
- Se `selector` definido: usar `doc.Find(selector)`; se vazio, sinaliza que deve seguir Readability.
- Retorna **HTML combinado** dos nós (sem reparse).
- Retorna `title` via `extractTitle(doc)`.

Notas:
- O método antigo `Extract(html, sourceURL)` permanece, mas passa a chamar o novo método quando possível.
- Quando `selector` não encontra elementos, retorna erro específico (ex.: `ErrSelectorNotFound`) para o pipeline decidir fallback.

### 2) Sanitizer doc-aware
**Arquivo:** `internal/converter/sanitizer.go`

Adicionar métodos:
- `SanitizeDocument(doc *goquery.Document) (*goquery.Document, error)`
- `SanitizeSelection(sel *goquery.Selection) (*goquery.Selection, error)`

Comportamento:
- Reutilizar a lógica existente (remoção de tags/classes/ids, normalize URLs) operando no DOM já parseado.
- Retornar o mesmo doc/selection (mutado), evitando reparse.

### 3) Header/Link extraction doc-aware
**Arquivo:** `internal/converter/readability.go`

Adicionar funções:
- `ExtractHeadersFromDoc(doc *goquery.Document) map[string][]string`
- `ExtractLinksFromDoc(doc *goquery.Document, baseURL string) []string`

Manter funções atuais como wrappers:
- `ExtractHeaders(html string)` → parse + `ExtractHeadersFromDoc` (compat).
- `ExtractLinks(html, baseURL string)` → parse + `ExtractLinksFromDoc` (compat).

### 4) Exclude selector doc-aware
**Arquivo:** `internal/converter/pipeline.go`

Alterar `removeExcluded` para operar sobre `*goquery.Selection` ou `*goquery.Document`:
- `removeExcludedFromSelection(sel *goquery.Selection) *goquery.Selection`
- ou `removeExcludedFromDoc(doc *goquery.Document) *goquery.Document`

---

## Plano de Implementação Detalhado

### Fase 0 - Preparação (contexto e testes atuais)
1. Abrir `internal/converter/pipeline.go` e mapear o fluxo atual passo a passo.
2. Localizar testes relevantes em `internal/converter/*` e `tests/unit/converter/*`.
3. Identificar dependências diretas de `ExtractHeaders`/`ExtractLinks` e `Sanitizer.Sanitize` fora da pipeline.

### Fase 1 - Funções doc-aware
1. **Adicionar `ExtractHeadersFromDoc` e `ExtractLinksFromDoc`** em `internal/converter/readability.go`.
2. Manter `ExtractHeaders` e `ExtractLinks` como wrappers (compatibilidade).
3. Adicionar testes unitários novos para as variantes doc-aware, reusando fixtures existentes.

### Fase 2 - Sanitizer doc-aware
1. Implementar `SanitizeDocument` e `SanitizeSelection` em `internal/converter/sanitizer.go`.
2. Refatorar `Sanitize(html string)` para:
   - parse → `SanitizeDocument` → `doc.Html()`.
3. Adicionar testes que assegurem equivalência entre `Sanitize` e `SanitizeDocument`.

### Fase 3 - Extractor doc-aware
1. Implementar `ExtractFromDocument` em `internal/converter/readability.go`.
2. Atualizar `Extract` para:
   - Se `selector` presente: parse externo e delegar para `ExtractFromDocument`.
   - Se `selector` ausente: manter Readability.
3. Adicionar erro sentinel `ErrSelectorNotFound` para fallback previsível.

### Fase 4 - Pipeline.Convert refatorado
**Arquivo:** `internal/converter/pipeline.go`

#### 4.1 Estrutura sugerida (pseudocódigo)
```go
// Convert(...) (novo fluxo)
htmlBytes, err := ConvertToUTF8(...)
html = string(htmlBytes)

// Parse original uma única vez
origDoc, err := goquery.NewDocumentFromReader(strings.NewReader(html))

// Extract content
if p.extractor.HasSelector() {
    contentHTML, title, err := p.extractor.ExtractFromDocument(origDoc, sourceURL)
    if errors.Is(err, ErrSelectorNotFound) { fallback to readability }
} else {
    contentHTML, title, err := p.extractor.Extract(html, sourceURL) // readability
}

// Exclude selector (sobre selection ou doc)
// Preferência: operar sobre selection dentro do origDoc quando selector presente

// Parse apenas se necessário (readability output)
contentDoc := ensureDoc(contentHTML)

// Sanitizar em doc
sanitizedDoc, err := p.sanitizer.SanitizeDocument(contentDoc)

// Extrair headers/links do sanitizedDoc
headers := ExtractHeadersFromDoc(sanitizedDoc)
links := ExtractLinksFromDoc(sanitizedDoc, sourceURL)

// Description do origDoc
description := ExtractDescription(origDoc)

// HTML sanitizado para converter
sanitizedHTML, _ := sanitizedDoc.Html()
markdown := p.mdConverter.Convert(sanitizedHTML)
```

#### 4.2 Regras de parsing
- **Sempre** parsear `origDoc` uma vez (para descrição e selector path).
- **Somente** parsear `contentDoc` quando o conteúdo vem de Readability (HTML extraído).
- **Nunca** reparsear para headers/links.

#### 4.3 Garantias de equivalência
- `description` continua vindo do HTML original.
- `headers/links` continuam vindo do HTML sanitizado.
- `excludeSelector` mantém o mesmo comportamento.

### Fase 5 - Atualização de testes
1. Atualizar `internal/converter/pipeline_test.go` para cobrir:
   - Selector path sem reparse (comparar resultado).
   - Readability path com 2 parses (verificar saída idêntica).
2. Ajustar `readability_test.go` para novos métodos e `ErrSelectorNotFound`.
3. Garantir que `sanitizer_test.go` cobre `SanitizeDocument`.

### Fase 6 - Ajustes finais
1. Verificar import order (std/external/internal).
2. Rodar `make test` e `make lint`.
3. Revisar logs e possíveis regressões de comportamento.

---

## Checklist de Arquivos

- `internal/converter/pipeline.go` (refatorar fluxo principal)
- `internal/converter/readability.go` (doc-aware + wrappers + erro sentinel)
- `internal/converter/sanitizer.go` (doc-aware)
- `internal/converter/pipeline_test.go`
- `internal/converter/readability_test.go`
- `internal/converter/sanitizer_test.go`
- `tests/unit/converter/*` (mirror conforme necessidade)

---

## Plano de Testes

1. **Unit tests**
   - `ExtractHeadersFromDoc` e `ExtractLinksFromDoc` com HTML simples.
   - `SanitizeDocument` vs `Sanitize` (mesma saída).
   - `ExtractFromDocument` com selector válido e inválido.

2. **Pipeline tests**
   - Verificar `Title`, `Description`, `Headers`, `Links` em casos com selector.
   - Verificar fallback de Readability preserva campos esperados.

3. **Regressão**
   - Rodar `make test`.
   - Opcional: `go test -v -run TestPipeline_Convert_Metadata ./internal/converter/...`.

---

## Riscos e Mitigações

| Risco | Impacto | Mitigação |
|------|---------|-----------|
| Sanitizer muta DOM usado por metadata | Medium | Extrair `description` antes da sanitização (origDoc) |
| Readability ainda parseia internamente | Low | Reduzir parses adjacentes e manter fallback |
| Alteração de comportamento com `excludeSelector` | Medium | Testes específicos com `ExcludeSelector` |
| Quebra de API em funções existentes | Medium | Manter wrappers para compatibilidade |

---

## Critérios de Sucesso

1. **Número de parses reduzido** conforme os dois fluxos propostos.
2. **Mesma saída de Markdown** para casos testados.
3. **Metadados consistentes** com o comportamento atual.
4. **Testes existentes e novos passam**.

---

## Sequência de Entrega (passo-a-passo)

1. Implementar `ExtractHeadersFromDoc` e `ExtractLinksFromDoc`.
2. Implementar `SanitizeDocument`/`SanitizeSelection`.
3. Implementar `ExtractFromDocument` e erro sentinel.
4. Refatorar `Pipeline.Convert` para novo fluxo.
5. Atualizar testes unitários e de pipeline.
6. Rodar `make test` e `make lint`.
