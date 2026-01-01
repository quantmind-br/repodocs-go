---
title: Get a Model (Beta)
url: https://platform.claude.com/docs/en/api/beta/models/retrieve.md
source: llms
fetched_at: 2026-01-01T02:10:36.760685014-03:00
rendered_js: false
word_count: 88
summary: Describes the API endpoint for retrieving details of a specific model via its ID or alias, including response structure and optional beta headers.
tags:
    - api-endpoint
    - model-retrieval
    - anthropic-beta
    - api-response
    - model-details
category: api
---

## Retrieve

**get** `/v1/models/{model_id}`

Get a specific model.

The Models API response can be used to determine information about a specific model or resolve a model alias to a model ID.

### Path Parameters

- `model_id: string`

  Model identifier or alias.

### Header Parameters

- `"anthropic-beta": optional array of AnthropicBeta`

  Optional header to specify the beta version(s) you want to use.

  - `UnionMember0 = string`

  - `UnionMember1 = "message-batches-2024-09-24" or "prompt-caching-2024-07-31" or "computer-use-2024-10-22" or 16 more`

    - `"message-batches-2024-09-24"`

    - `"prompt-caching-2024-07-31"`

    - `"computer-use-2024-10-22"`

    - `"computer-use-2025-01-24"`

    - `"pdfs-2024-09-25"`

    - `"token-counting-2024-11-01"`

    - `"token-efficient-tools-2025-02-19"`

    - `"output-128k-2025-02-19"`

    - `"files-api-2025-04-14"`

    - `"mcp-client-2025-04-04"`

    - `"mcp-client-2025-11-20"`

    - `"dev-full-thinking-2025-05-14"`

    - `"interleaved-thinking-2025-05-14"`

    - `"code-execution-2025-05-22"`

    - `"extended-cache-ttl-2025-04-11"`

    - `"context-1m-2025-08-07"`

    - `"context-management-2025-06-27"`

    - `"model-context-window-exceeded-2025-08-26"`

    - `"skills-2025-10-02"`

### Returns

- `BetaModelInfo = object { id, created_at, display_name, type }`

  - `id: string`

    Unique model identifier.

  - `created_at: string`

    RFC 3339 datetime string representing the time at which the model was released. May be set to an epoch value if the release date is unknown.

  - `display_name: string`

    A human-readable name for the model.

  - `type: "model"`

    Object type.

    For Models, this is always `"model"`.

    - `"model"`

### Example

```http
curl https://api.anthropic.com/v1/models/$MODEL_ID \
    -H 'anthropic-version: 2023-06-01' \
    -H "X-Api-Key: $ANTHROPIC_API_KEY"
```