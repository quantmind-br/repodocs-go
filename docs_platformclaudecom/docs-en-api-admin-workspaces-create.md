---
title: Create Workspace
url: https://platform.claude.com/docs/en/api/admin/workspaces/create.md
source: llms
fetched_at: 2026-01-01T02:10:08.846342433-03:00
rendered_js: false
word_count: 37
summary: describes the API endpoint for creating a workspace with required parameters and response structure
tags:
    - api-post
    - workspace-creation
    - organization-api
    - rpc-endpoint
category: api
---

## Create

**post** `/v1/organizations/workspaces`

Create Workspace

### Body Parameters

- `name: string`

  Name of the Workspace.

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
curl https://api.anthropic.com/v1/organizations/workspaces \
    -H 'Content-Type: application/json' \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY" \
    -d '{
          "name": "x"
        }'
```