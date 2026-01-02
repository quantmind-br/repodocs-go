# PLAN: Simplificação de Metadados JSON

## Objetivo

Simplificar a estrutura de metadados JSON gerada pelo `repodocs-go` para criar um formato mais conciso e adequado para avaliação por modelos de LLM. Os metadados do JSON devem espelhar os metadados inseridos no frontmatter do Markdown.

## Estado Atual vs. Estado Desejado

### Campos Atuais no JSON (domain.Metadata - 14 campos)

```json
{
  "file_path": "docs/page.md",
  "url": "https://example.com/page",
  "title": "Page Title",
  "description": "Page description",
  "fetched_at": "2025-01-01T12:00:00Z",
  "content_hash": "abc123...",           // REMOVER
  "word_count": 500,                      // REMOVER
  "char_count": 3000,                     // REMOVER
  "links": ["link1", "link2"],            // REMOVER
  "headers": {"h1": [...], "h2": [...]},  // REMOVER
  "rendered_with_js": false,              // REMOVER
  "source_strategy": "crawler",           // RENOMEAR -> "source"
  "cache_hit": false,                     // REMOVER
  "summary": "AI summary",
  "tags": ["tag1", "tag2"],
  "category": "guide"
}
```

### Campos Atuais no Frontmatter (domain.Frontmatter - 9 campos)

```yaml
---
title: "Page Title"
url: "https://example.com/page"
source: "crawler"
fetched_at: 2025-01-01T12:00:00Z
rendered_js: false                        # MANTER APENAS NO MD (não no JSON)
word_count: 500                           # MANTER APENAS NO MD (não no JSON)
summary: "AI summary"
tags: ["tag1", "tag2"]
category: "guide"
---
```

### Nova Estrutura JSON Desejada (9 campos)

```json
{
  "file_path": "docs/page.md",
  "title": "Page Title",
  "url": "https://example.com/page",
  "source": "crawler",
  "fetched_at": "2025-01-01T12:00:00Z",
  "description": "Page description",
  "summary": "AI summary",
  "tags": ["tag1", "tag2"],
  "category": "guide"
}
```

### Nova Estrutura MetadataIndex

```json
{
  "generated_at": "2025-01-01T12:00:00Z",
  "source_url": "https://example.com",
  "strategy": "crawler",
  "total_documents": 10,
  "documents": [...]
}
```

**Removidos do MetadataIndex:**
- `total_word_count` (não relevante para avaliação LLM)
- `total_char_count` (não relevante para avaliação LLM)

---

## Arquitetura de Mudanças

### Arquivos a Modificar

| Arquivo | Mudanças |
|---------|----------|
| `internal/domain/models.go` | Criar `SimpleMetadata`, modificar `MetadataIndex`, atualizar métodos de conversão |
| `internal/output/collector.go` | Usar nova estrutura `SimpleMetadata` |
| `tests/unit/domain/models_test.go` | Atualizar testes para nova estrutura |
| `tests/unit/collector_test.go` | Atualizar testes do collector |
| `tests/unit/writer_test.go` | Atualizar testes que validam JSON |
| `tests/e2e/full_pipeline_test.go` | Atualizar validação de JSON |

### Impacto

- **Backward Incompatible**: Sim - JSON output muda de formato
- **Breaking para usuários**: Usuários que parseiam o JSON atual precisarão atualizar
- **Mitigação**: Documentar mudança no CHANGELOG

---

## Epics e Tasks

### Epic 1: Refatorar Estruturas de Domínio

**Objetivo**: Criar nova estrutura de metadados simplificada e manter a antiga para compatibilidade interna.

#### Task 1.1: Criar SimpleMetadata e SimpleDocumentMetadata
- **Arquivo**: `internal/domain/models.go`
- **Ação**: Criar novos tipos:
  ```go
  // SimpleMetadata represents simplified document metadata for JSON output
  type SimpleMetadata struct {
      Title       string    `json:"title"`
      URL         string    `json:"url"`
      Source      string    `json:"source"`
      FetchedAt   time.Time `json:"fetched_at"`
      Description string    `json:"description,omitempty"`
      Summary     string    `json:"summary,omitempty"`
      Tags        []string  `json:"tags,omitempty"`
      Category    string    `json:"category,omitempty"`
  }
  
  // SimpleDocumentMetadata adds file_path to SimpleMetadata
  type SimpleDocumentMetadata struct {
      FilePath string `json:"file_path"`
      *SimpleMetadata
  }
  ```
- **Dependências**: Nenhuma

#### Task 1.2: Criar método ToSimpleMetadata em Document
- **Arquivo**: `internal/domain/models.go`
- **Ação**: Adicionar método de conversão:
  ```go
  func (d *Document) ToSimpleMetadata() *SimpleMetadata {
      return &SimpleMetadata{
          Title:       d.Title,
          URL:         d.URL,
          Source:      d.SourceStrategy,
          FetchedAt:   d.FetchedAt,
          Description: d.Description,
          Summary:     d.Summary,
          Tags:        d.Tags,
          Category:    d.Category,
      }
  }
  
  func (d *Document) ToSimpleDocumentMetadata(filePath string) *SimpleDocumentMetadata {
      return &SimpleDocumentMetadata{
          FilePath:       filePath,
          SimpleMetadata: d.ToSimpleMetadata(),
      }
  }
  ```
- **Dependências**: Task 1.1

#### Task 1.3: Atualizar SimpleMetadataIndex
- **Arquivo**: `internal/domain/models.go`
- **Ação**: Criar versão simplificada do índice:
  ```go
  // SimpleMetadataIndex represents the consolidated JSON output
  type SimpleMetadataIndex struct {
      GeneratedAt    time.Time                `json:"generated_at"`
      SourceURL      string                   `json:"source_url"`
      Strategy       string                   `json:"strategy"`
      TotalDocuments int                      `json:"total_documents"`
      Documents      []SimpleDocumentMetadata `json:"documents"`
  }
  ```
- **Dependências**: Task 1.1

---

### Epic 2: Atualizar MetadataCollector

**Objetivo**: Modificar o collector para usar a nova estrutura simplificada.

#### Task 2.1: Atualizar tipo documents no MetadataCollector
- **Arquivo**: `internal/output/collector.go`
- **Ação**: Mudar tipo de `[]*domain.DocumentMetadata` para `[]*domain.SimpleDocumentMetadata`
- **Dependências**: Epic 1

#### Task 2.2: Atualizar método Add
- **Arquivo**: `internal/output/collector.go`
- **Ação**: Usar `doc.ToSimpleDocumentMetadata(relPath)` em vez de `doc.ToDocumentMetadata(relPath)`
- **Dependências**: Task 2.1

#### Task 2.3: Atualizar método buildIndex
- **Arquivo**: `internal/output/collector.go`
- **Ação**: 
  - Retornar `*domain.SimpleMetadataIndex` em vez de `*domain.MetadataIndex`
  - Remover cálculo de `totalWords` e `totalChars`
- **Dependências**: Task 2.1, Task 2.2

#### Task 2.4: Atualizar método GetIndex
- **Arquivo**: `internal/output/collector.go`
- **Ação**: Atualizar tipo de retorno para `*domain.SimpleMetadataIndex`
- **Dependências**: Task 2.3

---

### Epic 3: Atualizar Testes Unitários

**Objetivo**: Garantir que todos os testes passem com a nova estrutura.

#### Task 3.1: Criar testes para SimpleMetadata
- **Arquivo**: `tests/unit/domain/models_test.go`
- **Ação**: Adicionar testes para:
  - `ToSimpleMetadata()`
  - `ToSimpleDocumentMetadata()`
  - Verificar campos corretos no JSON
- **Dependências**: Epic 1

#### Task 3.2: Atualizar testes do collector
- **Arquivo**: `tests/unit/collector_test.go`
- **Ação**: Atualizar assertions para usar `SimpleMetadataIndex` e `SimpleDocumentMetadata`
- **Dependências**: Epic 2

#### Task 3.3: Atualizar testes do writer
- **Arquivo**: `tests/unit/writer_test.go`
- **Ação**: Atualizar testes que validam estrutura JSON
- **Dependências**: Epic 2

---

### Epic 4: Atualizar Testes E2E

**Objetivo**: Garantir que o pipeline completo funcione com a nova estrutura.

#### Task 4.1: Atualizar TestJSONMetadata_Output
- **Arquivo**: `tests/e2e/full_pipeline_test.go`
- **Ação**: 
  - Verificar novos campos: `file_path`, `title`, `url`, `source`, `fetched_at`
  - Remover verificações de campos antigos: `content_hash`, `word_count`, etc.
- **Dependências**: Epics 1-3

---

### Epic 5: Limpeza e Documentação

**Objetivo**: Remover código obsoleto e documentar mudanças.

#### Task 5.1: Remover tipos obsoletos (opcional - deprecate first)
- **Arquivo**: `internal/domain/models.go`
- **Ação**: Adicionar comentário `// Deprecated:` aos tipos `Metadata`, `DocumentMetadata`, `MetadataIndex`
- **Nota**: Manter para backward compatibility ou remover completamente
- **Dependências**: Epics 1-4

#### Task 5.2: Atualizar memória Serena
- **Ação**: Atualizar `project_overview` com nova estrutura de metadados
- **Dependências**: Task 5.1

---

## Ordem de Execução

```
Epic 1 (Domínio)
    ├── Task 1.1: Criar SimpleMetadata
    ├── Task 1.2: Criar ToSimpleMetadata
    └── Task 1.3: Criar SimpleMetadataIndex
           │
           ▼
Epic 2 (Collector)
    ├── Task 2.1: Atualizar tipo documents
    ├── Task 2.2: Atualizar método Add
    ├── Task 2.3: Atualizar buildIndex
    └── Task 2.4: Atualizar GetIndex
           │
           ▼
Epic 3 (Testes Unitários)
    ├── Task 3.1: Testes SimpleMetadata
    ├── Task 3.2: Testes collector
    └── Task 3.3: Testes writer
           │
           ▼
Epic 4 (Testes E2E)
    └── Task 4.1: TestJSONMetadata_Output
           │
           ▼
Epic 5 (Limpeza)
    ├── Task 5.1: Deprecar tipos antigos
    └── Task 5.2: Atualizar documentação
```

---

## Critérios de Aceitação

1. **JSON Output**
   - [ ] Contém apenas: `file_path`, `title`, `url`, `source`, `fetched_at`, `description`, `summary`, `tags`, `category`
   - [ ] Campos opcionais (`description`, `summary`, `tags`, `category`) são omitidos quando vazios
   - [ ] `source` reflete o nome da estratégia usada

2. **MetadataIndex**
   - [ ] Não contém `total_word_count` ou `total_char_count`
   - [ ] `documents` usa estrutura simplificada

3. **Frontmatter (MD)**
   - [ ] Continua incluindo `word_count` e `rendered_js` (não afetado)
   - [ ] Campos LLM (`summary`, `tags`, `category`) presentes quando disponíveis

4. **Testes**
   - [ ] Todos os testes unitários passam
   - [ ] Todos os testes de integração passam
   - [ ] Todos os testes E2E passam

5. **Qualidade**
   - [ ] `make lint` sem erros
   - [ ] `make build` com sucesso

---

## Riscos e Mitigações

| Risco | Impacto | Mitigação |
|-------|---------|-----------|
| Breaking change para usuários | Alto | Documentar no CHANGELOG, bump minor version |
| Testes falhando | Médio | Executar testes após cada task |
| Perda de dados úteis | Baixo | Dados removidos são técnicos, não relevantes para LLM eval |

---

## Estimativa de Esforço

| Epic | Tasks | Complexidade | Estimativa |
|------|-------|--------------|------------|
| Epic 1 | 3 | Baixa | 30 min |
| Epic 2 | 4 | Média | 45 min |
| Epic 3 | 3 | Média | 45 min |
| Epic 4 | 1 | Baixa | 20 min |
| Epic 5 | 2 | Baixa | 15 min |
| **Total** | **13** | - | **~2.5 horas** |
