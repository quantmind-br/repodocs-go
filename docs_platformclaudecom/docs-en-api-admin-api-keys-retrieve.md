---
title: Get Api Key
url: https://platform.claude.com/docs/en/api/admin/api_keys/retrieve.md
source: llms
fetched_at: 2026-01-01T02:10:45.112090206-03:00
rendered_js: false
word_count: 51
summary: document describes the API endpoint for retrieving details of an API key with its metadata and status.
tags:
    - api-key-retrieval
    - api-key-details
    - organization-api
    - rest-api-endpoint
category: reference
---

## Retrieve

**get** `/v1/organizations/api_keys/{api_key_id}`

Get Api Key

### Path Parameters

- `api_key_id: string`

  ID of the API key.

### Returns

- `APIKey = object { id, created_at, created_by, 5 more }`

  - `id: string`

    ID of the API key.

  - `created_at: string`

    RFC 3339 datetime string indicating when the API Key was created.

  - `created_by: object { id, type }`

    The ID and type of the actor that created the API key.

    - `id: string`

      ID of the actor that created the object.

    - `type: string`

      Type of the actor that created the object.

  - `name: string`

    Name of the API key.

  - `partial_key_hint: string`

    Partially redacted hint for the API key.

  - `status: "active" or "inactive" or "archived"`

    Status of the API key.

    - `"active"`

    - `"inactive"`

    - `"archived"`

  - `type: "api_key"`

    Object type.

    For API Keys, this is always `"api_key"`.

    - `"api_key"`

  - `workspace_id: string`

    ID of the Workspace associated with the API key, or null if the API key belongs to the default Workspace.

### Example

```http
curl https://api.anthropic.com/v1/organizations/api_keys/$API_KEY_ID \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```