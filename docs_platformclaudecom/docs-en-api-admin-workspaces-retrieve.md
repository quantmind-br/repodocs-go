---
title: Get Workspace
url: https://platform.claude.com/docs/en/api/admin/workspaces/retrieve.md
source: llms
fetched_at: 2026-01-01T02:11:00.168239239-03:00
rendered_js: false
word_count: 37
summary: document describes the API endpoint for retrieving workspace details with its parameters, return structure, and authentication requirements
tags:
    - api-endpoint
    - workspace-retrieval
    - rpc-method
    - authentication-key
    - api-reference
category: api
---

## Retrieve

**get** `/v1/organizations/workspaces/{workspace_id}`

Get Workspace

### Path Parameters

- `workspace_id: string`

  ID of the Workspace.

### Returns

- `Workspace = object { id, archived_at, created_at, 3 more }`

  - `id: string`

    ID of the Workspace.

  - `archived_at: string`

    RFC 3339 datetime string indicating when the Workspace was archived, or null if the Workspace is not archived.

  - `created_at: string`

    RFC 3339 datetime string indicating when the Workspace was created.

  - `display_color: string`

    Hex color code representing the Workspace in the Anthropic Console.

  - `name: string`

    Name of the Workspace.

  - `type: "workspace"`

    Object type.

    For Workspaces, this is always `"workspace"`.

    - `"workspace"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/workspaces/$WORKSPACE_ID \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```