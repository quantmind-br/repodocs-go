---
title: Delete Workspace Member
url: https://platform.claude.com/docs/en/api/admin/workspaces/members/delete.md
source: llms
fetched_at: 2026-01-01T02:10:27.031423445-03:00
rendered_js: false
word_count: 47
summary: document describes the API endpoint for removing a user from a workspace membership
tags:
    - api-delete
    - workspace-members
    - user-management
    - api-endpoint
category: api
---

## Delete

**delete** `/v1/organizations/workspaces/{workspace_id}/members/{user_id}`

Delete Workspace Member

### Path Parameters

- `workspace_id: string`

  ID of the Workspace.

- `user_id: string`

  ID of the User.

### Returns

- `type: "workspace_member_deleted"`

  Deleted object type.

  For Workspace Members, this is always `"workspace_member_deleted"`.

  - `"workspace_member_deleted"`

- `user_id: string`

  ID of the User.

- `workspace_id: string`

  ID of the Workspace.

### Example

```http
curl https://api.anthropic.com/v1/organizations/workspaces/$WORKSPACE_ID/members/$USER_ID \
    -X DELETE \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```