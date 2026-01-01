---
title: Get Workspace Member
url: https://platform.claude.com/docs/en/api/admin/workspaces/members/retrieve.md
source: llms
fetched_at: 2026-01-01T02:11:00.751467359-03:00
rendered_js: false
word_count: 45
summary: describes how to retrieve a workspace member's details via API using workspace and user IDs
tags:
    - api-endpoint
    - workspace-membership
    - role-access
    - authentication
category: api
---

## Retrieve

**get** `/v1/organizations/workspaces/{workspace_id}/members/{user_id}`

Get Workspace Member

### Path Parameters

- `workspace_id: string`

  ID of the Workspace.

- `user_id: string`

  ID of the User.

### Returns

- `WorkspaceMember = object { type, user_id, workspace_id, workspace_role }`

  - `type: "workspace_member"`

    Object type.

    For Workspace Members, this is always `"workspace_member"`.

    - `"workspace_member"`

  - `user_id: string`

    ID of the User.

  - `workspace_id: string`

    ID of the Workspace.

  - `workspace_role: "workspace_user" or "workspace_developer" or "workspace_admin" or "workspace_billing"`

    Role of the Workspace Member.

    - `"workspace_user"`

    - `"workspace_developer"`

    - `"workspace_admin"`

    - `"workspace_billing"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/workspaces/$WORKSPACE_ID/members/$USER_ID \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```