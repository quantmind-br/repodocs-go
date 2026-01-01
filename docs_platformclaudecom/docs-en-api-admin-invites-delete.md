---
title: Delete Invite
url: https://platform.claude.com/docs/en/api/admin/invites/delete.md
source: llms
fetched_at: 2026-01-01T02:10:19.608279089-03:00
rendered_js: false
word_count: 33
summary: document describes how to delete an invite via API endpoint for Anthropic organizations
tags:
    - api-delete
    - invite-management
    - organization-api
category: api
---

## Delete

**delete** `/v1/organizations/invites/{invite_id}`

Delete Invite

### Path Parameters

- `invite_id: string`

  ID of the Invite.

### Returns

- `id: string`

  ID of the Invite.

- `type: "invite_deleted"`

  Deleted object type.

  For Invites, this is always `"invite_deleted"`.

  - `"invite_deleted"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/invites/$INVITE_ID \
    -X DELETE \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```