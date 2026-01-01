# PLAN: Fix LLM Metadata Enhancement JSON Extraction

**Issue**: LLM metadata enhancement fails for certain document types (e.g., prompt templates) when the LLM generates responses mimicking document content structure instead of the expected metadata format.

**Error Example**:
```
Failed to enhance metadata error="failed to extract JSON from LLM response: no valid JSON found in: ```json\n{\n  \"multilingual_messages\": ..."
```

**Affected File**: `docs-en-resources-prompt-library-babels-broadcasts.md`

---

## Table of Contents

1. [Root Cause Analysis](#1-root-cause-analysis)
2. [Solution Architecture](#2-solution-architecture)
3. [Implementation Tasks](#3-implementation-tasks)
4. [File Changes](#4-file-changes)
5. [Testing Strategy](#5-testing-strategy)
6. [Rollout Plan](#6-rollout-plan)

---

## 1. Root Cause Analysis

### 1.1 Problem Chain

```
Document Content (prompt template with multilingual examples)
    ↓
LLM receives ambiguous prompt
    ↓
LLM generates response matching document's example format ("multilingual_messages")
    ↓
extractJSON() finds JSON but wrong structure OR truncated/malformed
    ↓
Regex fallback fails (requires "summary", "tags", "category" keys)
    ↓
Returns empty string → Error logged → Document written without metadata
```

### 1.2 Specific Code Issues

| Location | Issue | Severity |
|----------|-------|----------|
| `metadata.go:27-39` | Prompt is too permissive, doesn't constrain output format strongly | HIGH |
| `metadata.go:83-105` | `extractJSON()` validates syntax only, not structure | HIGH |
| `metadata.go:107-113` | `stripMarkdownCodeBlocks()` doesn't handle all markdown fence variations | MEDIUM |
| `metadata.go:59` | `MaxTokens: 32000` is excessive for metadata (3 fields) | LOW |
| `metadata.go:41-81` | No retry mechanism for malformed responses | MEDIUM |

### 1.3 Document Types at Risk

1. **Prompt Library** - Contains example outputs that LLM may mimic
2. **API Reference** - Contains JSON schemas/examples
3. **Code Tutorials** - Contains code blocks with JSON
4. **Configuration Guides** - Contains YAML/JSON config examples

---

## 2. Solution Architecture

### 2.1 Defense in Depth Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: Prompt Engineering                   │
│  - Explicit format instructions with XML delimiters             │
│  - Negative examples ("Do NOT generate...")                     │
│  - Constrained output length                                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2: Extraction Hardening                 │
│  - Multi-strategy JSON extraction                               │
│  - Structure validation (required keys)                         │
│  - Improved markdown fence handling                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 3: Retry Mechanism                      │
│  - Retry with simplified prompt on failure                      │
│  - Exponential backoff                                          │
│  - Max 2 retries                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 4: Graceful Degradation                 │
│  - Structured logging of failures                               │
│  - Continue pipeline (current behavior)                         │
│  - Metrics for monitoring                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Keep current warning-only behavior | Don't break pipeline for non-critical feature |
| Retry max 2 times | Balance between success rate and latency/cost |
| Validate structure in extraction | Fail fast, don't attempt unmarshal on wrong structure |
| Use XML delimiters in prompt | Clearer boundaries for LLM, less ambiguity |
| Reduce MaxTokens to 1024 | Metadata is small; prevents runaway responses |

---

## 3. Implementation Tasks

### Phase 1: Prompt Engineering (Priority: HIGH)

#### Task 1.1: Create Improved Metadata Prompt

**File**: `internal/llm/metadata.go`

**Current** (lines 27-39):
```go
const metadataSystemPrompt = `You are a strict JSON generator. Output valid JSON only. No text. No thinking.`

const metadataPrompt = `Generate a JSON object for this document.

Format:
{"summary": "string", "tags": ["string"], "category": "string"}

Categories: api, tutorial, guide, reference, concept, configuration, other

Document Content:
%s

JSON Output:`
```

**New**:
```go
const metadataSystemPrompt = `You are a metadata extraction system. You analyze documents and output ONLY valid JSON with exactly three fields: summary, tags, and category. Never output anything else.`

const metadataPrompt = `<task>
Extract metadata from the document below. Output ONLY a JSON object.
</task>

<format>
{
  "summary": "1-2 sentence description of what this document explains or teaches",
  "tags": ["3-8 lowercase hyphenated keywords relevant to the content"],
  "category": "one of: api, tutorial, guide, reference, concept, configuration, other"
}
</format>

<rules>
- Output ONLY the JSON object, no other text
- Do NOT include markdown code fences
- Do NOT generate content that matches examples shown in the document
- Do NOT create fields other than summary, tags, category
- Summary should describe the document's PURPOSE, not its examples
- Tags should be lowercase with hyphens (e.g., "api-reference", "error-handling")
</rules>

<document>
%s
</document>

<output>`
```

**Rationale**:
- XML delimiters provide clear structure
- Explicit negative rules prevent mimicking document content
- Format example shows expected structure
- Tags format specified (lowercase, hyphenated)

#### Task 1.2: Add Retry Prompt Variant

**File**: `internal/llm/metadata.go`

Add a simplified retry prompt for when first attempt fails:

```go
const metadataRetryPrompt = `The previous attempt failed. Output ONLY this exact JSON structure with your values:

{"summary": "brief description here", "tags": ["tag1", "tag2", "tag3"], "category": "guide"}

Document title: %s

Your JSON:`
```

**Rationale**: Simpler prompt with example reduces chance of format errors on retry.

---

### Phase 2: Extraction Hardening (Priority: HIGH)

#### Task 2.1: Improve `stripMarkdownCodeBlocks()`

**File**: `internal/llm/metadata.go`

**Current** (lines 107-113):
```go
func stripMarkdownCodeBlocks(text string) string {
	text = strings.TrimSpace(text)
	text = strings.TrimPrefix(text, "```json")
	text = strings.TrimPrefix(text, "```")
	text = strings.TrimSuffix(text, "```")
	return strings.TrimSpace(text)
}
```

**New**:
```go
// stripMarkdownCodeBlocks removes markdown code fences from LLM responses.
// Handles variations: ```json, ```JSON, ``` json, ```\n, etc.
func stripMarkdownCodeBlocks(text string) string {
	text = strings.TrimSpace(text)
	
	// Pattern: ```json or ```JSON or ``` json (with optional whitespace)
	// followed by optional newline, then content, then closing ```
	codeBlockRegex := regexp.MustCompile("(?s)^```(?:json|JSON)?\\s*\\n?(.*?)\\n?```$")
	if matches := codeBlockRegex.FindStringSubmatch(text); len(matches) == 2 {
		return strings.TrimSpace(matches[1])
	}
	
	// Fallback: just strip prefixes/suffixes
	for _, prefix := range []string{"```json", "```JSON", "``` json", "```"} {
		text = strings.TrimPrefix(text, prefix)
	}
	text = strings.TrimSuffix(text, "```")
	text = strings.TrimSpace(text)
	
	// Handle case where there's a newline after the opening fence
	if strings.HasPrefix(text, "\n") {
		text = strings.TrimPrefix(text, "\n")
	}
	
	return strings.TrimSpace(text)
}
```

**Rationale**: Handles more edge cases including newlines after fence.

#### Task 2.2: Add Structure Validation to `extractJSON()`

**File**: `internal/llm/metadata.go`

**Current** (lines 83-105):
```go
func extractJSON(text string) string {
	text = strings.TrimSpace(text)

	if json.Valid([]byte(text)) && strings.HasPrefix(text, "{") {
		return text
	}

	text = stripMarkdownCodeBlocks(text)
	if json.Valid([]byte(text)) && strings.HasPrefix(text, "{") {
		return text
	}

	if jsonObj := findJSONObjectByBraceMatching(text); jsonObj != "" {
		return jsonObj
	}

	re := regexp.MustCompile(`\{[^{}]*"summary"[^{}]*"tags"[^{}]*"category"[^{}]*\}`)
	if match := re.FindString(text); match != "" && json.Valid([]byte(match)) {
		return match
	}

	return ""
}
```

**New**:
```go
// extractJSON attempts to extract a valid metadata JSON object from LLM response.
// Returns empty string if no valid JSON with required structure is found.
func extractJSON(text string) string {
	text = strings.TrimSpace(text)
	
	// Try direct parse
	if candidate := tryExtractAndValidate(text); candidate != "" {
		return candidate
	}
	
	// Try after stripping markdown
	stripped := stripMarkdownCodeBlocks(text)
	if candidate := tryExtractAndValidate(stripped); candidate != "" {
		return candidate
	}
	
	// Try brace matching to find JSON object
	if jsonObj := findJSONObjectByBraceMatching(stripped); jsonObj != "" {
		if candidate := tryExtractAndValidate(jsonObj); candidate != "" {
			return candidate
		}
	}
	
	// Try brace matching on original (in case stripping broke something)
	if jsonObj := findJSONObjectByBraceMatching(text); jsonObj != "" {
		if candidate := tryExtractAndValidate(jsonObj); candidate != "" {
			return candidate
		}
	}
	
	return ""
}

// tryExtractAndValidate checks if text is valid JSON with required metadata structure.
func tryExtractAndValidate(text string) string {
	if !strings.HasPrefix(text, "{") {
		return ""
	}
	
	if !json.Valid([]byte(text)) {
		return ""
	}
	
	// Validate structure: must have summary, tags, and category
	var check map[string]interface{}
	if err := json.Unmarshal([]byte(text), &check); err != nil {
		return ""
	}
	
	// Check for required keys
	if _, ok := check["summary"]; !ok {
		return ""
	}
	if _, ok := check["tags"]; !ok {
		return ""
	}
	if _, ok := check["category"]; !ok {
		return ""
	}
	
	// Validate types
	if _, ok := check["summary"].(string); !ok {
		return ""
	}
	if _, ok := check["tags"].([]interface{}); !ok {
		return ""
	}
	if _, ok := check["category"].(string); !ok {
		return ""
	}
	
	return text
}
```

**Rationale**: 
- Validates structure before returning
- Fails fast on wrong JSON structure (e.g., `multilingual_messages`)
- Clearer separation of concerns

#### Task 2.3: Remove Unused Regex Fallback

The regex fallback that looks for keys within non-nested braces is now redundant since we validate structure. Remove it to simplify code.

---

### Phase 3: Retry Mechanism (Priority: MEDIUM)

#### Task 3.1: Implement Retry Logic in `Enhance()`

**File**: `internal/llm/metadata.go`

**New function and modified `Enhance()`**:

```go
const (
	maxRetries      = 2
	retryBaseDelay  = 500 * time.Millisecond
)

func (e *MetadataEnhancer) Enhance(ctx context.Context, doc *domain.Document) error {
	if doc == nil {
		return fmt.Errorf("document is nil")
	}

	content := doc.Content
	if len(content) > 8000 {
		content = content[:8000] + "\n...[truncated]"
	}

	var lastErr error
	
	// First attempt with full prompt
	metadata, err := e.tryEnhance(ctx, content, false)
	if err == nil {
		e.applyMetadata(doc, metadata)
		return nil
	}
	lastErr = err
	
	// Retry with simplified prompt
	for attempt := 1; attempt <= maxRetries; attempt++ {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(retryBaseDelay * time.Duration(attempt)):
		}
		
		metadata, err := e.tryEnhance(ctx, doc.Title, true) // Use title only for retry
		if err == nil {
			e.applyMetadata(doc, metadata)
			return nil
		}
		lastErr = err
	}
	
	return fmt.Errorf("metadata enhancement failed after %d attempts: %w", maxRetries+1, lastErr)
}

func (e *MetadataEnhancer) tryEnhance(ctx context.Context, content string, isRetry bool) (*enhancedMetadata, error) {
	var prompt string
	if isRetry {
		prompt = fmt.Sprintf(metadataRetryPrompt, content)
	} else {
		prompt = fmt.Sprintf(metadataPrompt, content)
	}

	req := &domain.LLMRequest{
		Messages: []domain.LLMMessage{
			{Role: domain.RoleSystem, Content: metadataSystemPrompt},
			{Role: domain.RoleUser, Content: prompt},
		},
		MaxTokens: 1024, // Reduced from 32000 - metadata is small
	}

	resp, err := e.provider.Complete(ctx, req)
	if err != nil {
		return nil, fmt.Errorf("LLM completion failed: %w", err)
	}

	jsonStr := extractJSON(resp.Content)
	if jsonStr == "" {
		return nil, fmt.Errorf("no valid JSON structure found in response: %s", truncateForError(resp.Content))
	}

	var metadata enhancedMetadata
	if err := json.Unmarshal([]byte(jsonStr), &metadata); err != nil {
		return nil, fmt.Errorf("JSON unmarshal failed: %w (extracted: %s)", err, truncateForError(jsonStr))
	}

	return &metadata, nil
}

func (e *MetadataEnhancer) applyMetadata(doc *domain.Document, metadata *enhancedMetadata) {
	doc.Summary = metadata.Summary
	doc.Tags = metadata.Tags
	doc.Category = metadata.Category
}
```

**Rationale**:
- Retry with simpler prompt increases success rate
- Exponential backoff prevents rate limiting
- Clear separation between attempt logic and metadata application

---

### Phase 4: Testing (Priority: HIGH)

#### Task 4.1: Create Unit Tests for Edge Cases

**File**: `tests/unit/llm/metadata_test.go` (new file)

```go
package llm_test

import (
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	
	"github.com/quantmind-br/repodocs-go/internal/llm"
)

func TestExtractJSON_ValidMetadata(t *testing.T) {
	tests := []struct {
		name     string
		input    string
		expected string
	}{
		{
			name:     "plain JSON",
			input:    `{"summary": "test", "tags": ["a"], "category": "guide"}`,
			expected: `{"summary": "test", "tags": ["a"], "category": "guide"}`,
		},
		{
			name:     "with markdown fence",
			input:    "```json\n{\"summary\": \"test\", \"tags\": [\"a\"], \"category\": \"guide\"}\n```",
			expected: `{"summary": "test", "tags": ["a"], "category": "guide"}`,
		},
		{
			name:     "with markdown fence no newline",
			input:    "```json{\"summary\": \"test\", \"tags\": [\"a\"], \"category\": \"guide\"}```",
			expected: `{"summary": "test", "tags": ["a"], "category": "guide"}`,
		},
		{
			name:     "with surrounding text",
			input:    "Here is the metadata:\n{\"summary\": \"test\", \"tags\": [\"a\"], \"category\": \"guide\"}\nDone.",
			expected: `{"summary": "test", "tags": ["a"], "category": "guide"}`,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := llm.ExtractJSON(tt.input)
			assert.JSONEq(t, tt.expected, result)
		})
	}
}

func TestExtractJSON_InvalidStructure(t *testing.T) {
	tests := []struct {
		name  string
		input string
	}{
		{
			name:  "wrong keys",
			input: `{"multilingual_messages": {"en": "hello"}}`,
		},
		{
			name:  "missing summary",
			input: `{"tags": ["a"], "category": "guide"}`,
		},
		{
			name:  "missing tags",
			input: `{"summary": "test", "category": "guide"}`,
		},
		{
			name:  "missing category",
			input: `{"summary": "test", "tags": ["a"]}`,
		},
		{
			name:  "wrong type for summary",
			input: `{"summary": 123, "tags": ["a"], "category": "guide"}`,
		},
		{
			name:  "wrong type for tags",
			input: `{"summary": "test", "tags": "a", "category": "guide"}`,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := llm.ExtractJSON(tt.input)
			assert.Empty(t, result, "should reject invalid structure")
		})
	}
}

func TestExtractJSON_MalformedJSON(t *testing.T) {
	tests := []struct {
		name  string
		input string
	}{
		{
			name:  "truncated",
			input: `{"summary": "test", "tags": ["a"], "category": "gu`,
		},
		{
			name:  "missing closing brace",
			input: `{"summary": "test", "tags": ["a"], "category": "guide"`,
		},
		{
			name:  "invalid syntax",
			input: `{"summary": test, "tags": ["a"], "category": "guide"}`,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := llm.ExtractJSON(tt.input)
			assert.Empty(t, result, "should reject malformed JSON")
		})
	}
}

func TestStripMarkdownCodeBlocks(t *testing.T) {
	tests := []struct {
		name     string
		input    string
		expected string
	}{
		{
			name:     "json fence with newlines",
			input:    "```json\n{\"key\": \"value\"}\n```",
			expected: `{"key": "value"}`,
		},
		{
			name:     "JSON uppercase",
			input:    "```JSON\n{\"key\": \"value\"}\n```",
			expected: `{"key": "value"}`,
		},
		{
			name:     "plain fence",
			input:    "```\n{\"key\": \"value\"}\n```",
			expected: `{"key": "value"}`,
		},
		{
			name:     "no fence",
			input:    `{"key": "value"}`,
			expected: `{"key": "value"}`,
		},
		{
			name:     "fence with space",
			input:    "``` json\n{\"key\": \"value\"}\n```",
			expected: `{"key": "value"}`,
		},
	}
	
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := llm.StripMarkdownCodeBlocks(tt.input)
			assert.Equal(t, tt.expected, result)
		})
	}
}
```

#### Task 4.2: Create Integration Test with Mock LLM

**File**: `tests/integration/llm/metadata_integration_test.go` (new file)

```go
package llm_test

import (
	"context"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	
	"github.com/quantmind-br/repodocs-go/internal/domain"
	"github.com/quantmind-br/repodocs-go/internal/llm"
)

// mockLLMProvider simulates LLM responses for testing
type mockLLMProvider struct {
	responses []string
	callCount int
}

func (m *mockLLMProvider) Name() string { return "mock" }

func (m *mockLLMProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
	idx := m.callCount
	if idx >= len(m.responses) {
		idx = len(m.responses) - 1
	}
	m.callCount++
	return &domain.LLMResponse{Content: m.responses[idx]}, nil
}

func (m *mockLLMProvider) Close() error { return nil }

func TestMetadataEnhancer_RetryOnWrongStructure(t *testing.T) {
	mock := &mockLLMProvider{
		responses: []string{
			// First attempt: wrong structure
			`{"multilingual_messages": {"en": "hello"}}`,
			// Retry: correct structure
			`{"summary": "A document", "tags": ["test"], "category": "guide"}`,
		},
	}
	
	enhancer := llm.NewMetadataEnhancer(mock)
	doc := &domain.Document{
		Title:   "Test Doc",
		Content: "Some content here",
	}
	
	err := enhancer.Enhance(context.Background(), doc)
	require.NoError(t, err)
	
	assert.Equal(t, "A document", doc.Summary)
	assert.Equal(t, []string{"test"}, doc.Tags)
	assert.Equal(t, "guide", doc.Category)
	assert.Equal(t, 2, mock.callCount, "should have retried once")
}

func TestMetadataEnhancer_FailAfterMaxRetries(t *testing.T) {
	mock := &mockLLMProvider{
		responses: []string{
			`{"wrong": "structure"}`,
			`{"also": "wrong"}`,
			`{"still": "wrong"}`,
		},
	}
	
	enhancer := llm.NewMetadataEnhancer(mock)
	doc := &domain.Document{
		Title:   "Test Doc",
		Content: "Some content",
	}
	
	err := enhancer.Enhance(context.Background(), doc)
	require.Error(t, err)
	assert.Contains(t, err.Error(), "failed after")
	assert.Equal(t, 3, mock.callCount, "should have tried 1 + 2 retries")
}
```

#### Task 4.3: Add Test for Problematic Document Type

**File**: `tests/integration/llm/metadata_prompt_library_test.go` (new file)

```go
package llm_test

import (
	"context"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	
	"github.com/quantmind-br/repodocs-go/internal/domain"
	"github.com/quantmind-br/repodocs-go/internal/llm"
)

// This test uses a real prompt template document that caused issues
const problematicDocument = `# Babel's broadcasts
Create compelling product announcement tweets in the world's 10 most spoken languages.

> Copy this prompt into our developer Console to try it for yourself!

| | Content |
| ---- | ---- |
| User | Write me a series of product announcement tweets in the 10 most commonly spoken languages. |

### Example output

> English:
> Introducing the future of bird/wildlife watching! Our new AI binoculars use cutting-edge vision tech.
>
> Mandarin Chinese:
> 令人兴奋的新品上市!我们的 AI 双筒望远镜融合了尖端视觉技术。
`

func TestMetadataEnhancer_PromptLibraryDocument(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping integration test in short mode")
	}
	
	// This test requires a real LLM provider - skip if not configured
	provider, err := llm.NewProviderFromConfig(&config.LLMConfig{
		Provider: "anthropic",
		Model:    "claude-3-haiku-20240307", // Use fast model for tests
	})
	if err != nil {
		t.Skipf("LLM provider not configured: %v", err)
	}
	defer provider.Close()
	
	enhancer := llm.NewMetadataEnhancer(provider)
	doc := &domain.Document{
		Title:   "Babel's broadcasts",
		URL:     "https://example.com/prompt-library/babels-broadcasts.md",
		Content: problematicDocument,
	}
	
	err = enhancer.Enhance(context.Background(), doc)
	require.NoError(t, err, "should succeed for prompt library document")
	
	// Validate metadata was extracted correctly
	assert.NotEmpty(t, doc.Summary)
	assert.NotContains(t, doc.Summary, "multilingual", "should not mimic document content")
	assert.NotEmpty(t, doc.Tags)
	assert.NotEmpty(t, doc.Category)
	
	// Category should be one of the valid options
	validCategories := []string{"api", "tutorial", "guide", "reference", "concept", "configuration", "other"}
	assert.Contains(t, validCategories, doc.Category)
}
```

---

## 4. File Changes

### Summary of Changes

| File | Action | Changes |
|------|--------|---------|
| `internal/llm/metadata.go` | MODIFY | New prompts, improved extraction, retry logic |
| `internal/llm/extract.go` | CREATE | Move extraction functions to separate file |
| `tests/unit/llm/metadata_test.go` | CREATE | Unit tests |
| `tests/integration/llm/metadata_integration_test.go` | CREATE | Integration tests |

### Detailed Changes for `internal/llm/metadata.go`

```diff
 package llm
 
 import (
 	"context"
 	"encoding/json"
 	"fmt"
-	"regexp"
 	"strings"
+	"time"
 
 	"github.com/quantmind-br/repodocs-go/internal/domain"
 )
 
+const (
+	maxRetries     = 2
+	retryBaseDelay = 500 * time.Millisecond
+)
+
 type MetadataEnhancer struct {
 	provider domain.LLMProvider
 }
 
 func NewMetadataEnhancer(provider domain.LLMProvider) *MetadataEnhancer {
 	return &MetadataEnhancer{provider: provider}
 }
 
 type enhancedMetadata struct {
 	Summary  string   `json:"summary"`
 	Tags     []string `json:"tags"`
 	Category string   `json:"category"`
 }
 
-const metadataSystemPrompt = `You are a strict JSON generator. Output valid JSON only. No text. No thinking.`
+const metadataSystemPrompt = `You are a metadata extraction system. You analyze documents and output ONLY valid JSON with exactly three fields: summary, tags, and category. Never output anything else.`
 
-const metadataPrompt = `Generate a JSON object for this document.
+const metadataPrompt = `<task>
+Extract metadata from the document below. Output ONLY a JSON object.
+</task>
 
-Format:
-{"summary": "string", "tags": ["string"], "category": "string"}
+<format>
+{
+  "summary": "1-2 sentence description of what this document explains or teaches",
+  "tags": ["3-8 lowercase hyphenated keywords relevant to the content"],
+  "category": "one of: api, tutorial, guide, reference, concept, configuration, other"
+}
+</format>
 
-Categories: api, tutorial, guide, reference, concept, configuration, other
+<rules>
+- Output ONLY the JSON object, no other text
+- Do NOT include markdown code fences
+- Do NOT generate content that matches examples shown in the document
+- Do NOT create fields other than summary, tags, category
+- Summary should describe the document's PURPOSE, not its examples
+- Tags should be lowercase with hyphens (e.g., "api-reference", "error-handling")
+</rules>
 
-Document Content:
+<document>
 %s
+</document>
 
-JSON Output:`
+<output>`
+
+const metadataRetryPrompt = `The previous attempt failed. Output ONLY this JSON structure:
+
+{"summary": "brief description", "tags": ["keyword1", "keyword2"], "category": "guide"}
+
+Document title: %s
+
+JSON:`
 
 func (e *MetadataEnhancer) Enhance(ctx context.Context, doc *domain.Document) error {
 	if doc == nil {
 		return fmt.Errorf("document is nil")
 	}
 
 	content := doc.Content
 	if len(content) > 8000 {
 		content = content[:8000] + "\n...[truncated]"
 	}
 
-	prompt := fmt.Sprintf(metadataPrompt, content)
+	var lastErr error
+
+	// First attempt with full content
+	metadata, err := e.tryEnhance(ctx, content, false)
+	if err == nil {
+		e.applyMetadata(doc, metadata)
+		return nil
+	}
+	lastErr = err
+
+	// Retry with simplified prompt using title only
+	for attempt := 1; attempt <= maxRetries; attempt++ {
+		select {
+		case <-ctx.Done():
+			return ctx.Err()
+		case <-time.After(retryBaseDelay * time.Duration(attempt)):
+		}
+
+		metadata, err := e.tryEnhance(ctx, doc.Title, true)
+		if err == nil {
+			e.applyMetadata(doc, metadata)
+			return nil
+		}
+		lastErr = err
+	}
+
+	return fmt.Errorf("metadata enhancement failed after %d attempts: %w", maxRetries+1, lastErr)
+}
 
-	req := &domain.LLMRequest{
+func (e *MetadataEnhancer) tryEnhance(ctx context.Context, content string, isRetry bool) (*enhancedMetadata, error) {
+	var prompt string
+	if isRetry {
+		prompt = fmt.Sprintf(metadataRetryPrompt, content)
+	} else {
+		prompt = fmt.Sprintf(metadataPrompt, content)
+	}
+
+	req := &domain.LLMRequest{
 		Messages: []domain.LLMMessage{
 			{Role: domain.RoleSystem, Content: metadataSystemPrompt},
 			{Role: domain.RoleUser, Content: prompt},
 		},
-		MaxTokens: 32000,
+		MaxTokens: 1024,
 	}
 
 	resp, err := e.provider.Complete(ctx, req)
 	if err != nil {
-		return fmt.Errorf("LLM completion failed: %w", err)
+		return nil, fmt.Errorf("LLM completion failed: %w", err)
 	}
 
-	var metadata enhancedMetadata
 	jsonStr := extractJSON(resp.Content)
 	if jsonStr == "" {
-		return fmt.Errorf("failed to extract JSON from LLM response: no valid JSON found in: %s", truncateForError(resp.Content))
+		return nil, fmt.Errorf("no valid JSON structure found: %s", truncateForError(resp.Content))
 	}
 
+	var metadata enhancedMetadata
 	if err := json.Unmarshal([]byte(jsonStr), &metadata); err != nil {
-		return fmt.Errorf("failed to parse LLM response JSON: %w (extracted: %s)", err, truncateForError(jsonStr))
+		return nil, fmt.Errorf("JSON unmarshal failed: %w", err)
 	}
 
-	doc.Summary = metadata.Summary
-	doc.Tags = metadata.Tags
-	doc.Category = metadata.Category
+	return &metadata, nil
+}
 
-	return nil
+func (e *MetadataEnhancer) applyMetadata(doc *domain.Document, metadata *enhancedMetadata) {
+	doc.Summary = metadata.Summary
+	doc.Tags = metadata.Tags
+	doc.Category = metadata.Category
 }
 
 // ... rest of file (extraction functions) ...
```

### Export Functions for Testing

To enable testing of `extractJSON` and `stripMarkdownCodeBlocks`, either:

1. **Option A**: Create `internal/llm/extract_exported_test.go` with test wrappers
2. **Option B**: Move to a separate package with exported functions

Recommended: Option A (simpler, keeps encapsulation)

```go
// internal/llm/export_test.go
package llm

// Exported for testing
func ExtractJSON(text string) string {
	return extractJSON(text)
}

func StripMarkdownCodeBlocks(text string) string {
	return stripMarkdownCodeBlocks(text)
}
```

---

## 5. Testing Strategy

### 5.1 Test Pyramid

```
        /\
       /  \     E2E Tests (1)
      /    \    - Full extraction with real LLM
     /------\
    /        \  Integration Tests (3)
   /          \ - Mock LLM responses
  /            \ - Retry logic
 /--------------\
/                \ Unit Tests (15+)
/                  \ - extractJSON edge cases
/                    \ - stripMarkdownCodeBlocks
/                      \ - Structure validation
```

### 5.2 Test Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `extractJSON` | 95% |
| `stripMarkdownCodeBlocks` | 100% |
| `tryExtractAndValidate` | 100% |
| `Enhance` (with retry) | 85% |

### 5.3 Test Commands

```bash
# Run unit tests
go test -v -short ./internal/llm/...

# Run all tests including integration
go test -v ./internal/llm/...

# Run with coverage
go test -coverprofile=coverage.out ./internal/llm/...
go tool cover -html=coverage.out
```

---

## 6. Rollout Plan

### 6.1 Phase Timeline

| Phase | Duration | Activities |
|-------|----------|------------|
| 1. Development | 2 hours | Implement prompt + extraction changes |
| 2. Unit Testing | 1 hour | Write and run unit tests |
| 3. Integration Testing | 1 hour | Test with mock LLM |
| 4. Real LLM Testing | 30 min | Test with actual Claude API |
| 5. Full Run | 1 hour | Re-run on platform.claude.com docs |
| 6. Monitoring | Ongoing | Watch for new failures |

### 6.2 Verification Checklist

- [ ] All unit tests pass
- [ ] Integration tests pass with mock LLM
- [ ] Retry logic works correctly
- [ ] `babels-broadcasts.md` processes successfully
- [ ] Other prompt library docs process successfully
- [ ] No regression in other document types
- [ ] Performance acceptable (retry adds latency)
- [ ] Error messages are clear and actionable

### 6.3 Rollback Plan

If issues arise:

1. **Immediate**: Disable LLM enhancement in config:
   ```yaml
   llm:
     enhance_metadata: false
   ```

2. **Temporary**: Increase retry attempts:
   ```go
   const maxRetries = 5
   ```

3. **Permanent**: Revert to previous `metadata.go`

### 6.4 Success Metrics

| Metric | Before Fix | Target |
|--------|------------|--------|
| Metadata enhancement success rate | ~95% | >99% |
| Avg enhancement time | ~1s | <2s (with retries) |
| Failed documents per run | 5-10 | <1 |

---

## Appendix: Alternative Solutions Considered

### A1. Structured Outputs (Claude API)

**Approach**: Use Claude API's structured output feature to force JSON schema compliance.

**Pros**:
- Guarantees correct structure
- No parsing logic needed

**Cons**:
- Requires specific API version/feature
- May not work with other LLM providers (OpenAI, Google)
- More complex API integration

**Decision**: Not implemented now, but could be added as enhancement.

### A2. Skip Problematic Document Types

**Approach**: Detect prompt library documents and skip enhancement.

**Pros**:
- Simple implementation
- No changes to core logic

**Cons**:
- Loses metadata for useful documents
- Requires maintaining skip list
- Doesn't fix root cause

**Decision**: Rejected - prefer fixing the root cause.

### A3. Two-Stage Enhancement

**Approach**: First call to detect document type, second call for metadata.

**Pros**:
- Better handling of different document types
- More targeted prompts

**Cons**:
- Doubles API calls
- Increases latency and cost
- Over-engineering for the problem

**Decision**: Rejected - retry mechanism is simpler and sufficient.

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-01 | AI | Initial plan |
