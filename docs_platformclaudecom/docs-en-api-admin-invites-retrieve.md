---
title: Get Invite
url: https://platform.claude.com/docs/en/api/admin/invites/retrieve.md
source: llms
fetched_at: 2026-01-01T02:10:50.623127902-03:00
rendered_js: false
word_count: 52
summary: document describes the API endpoint for retrieving invite details with parameters and response structure for organization invites
tags:
    - api-endpoint
    - invite-details
    - organization-invites
    - authentication-key
category: reference
---

## Retrieve

**get** `/v1/organizations/invites/{invite_id}`

Get Invite

### Path Parameters

- `invite_id: string`

  ID of the Invite.

### Returns

- `Invite = object { id, email, expires_at, 4 more }`

  - `id: string`

    ID of the Invite.

  - `email: string`

    Email of the User being invited.

  - `expires_at: string`

    RFC 3339 datetime string indicating when the Invite expires.

  - `invited_at: string`

    RFC 3339 datetime string indicating when the Invite was created.

  - `role: "user" or "developer" or "billing" or 2 more`

    Organization role of the User.

    - `"user"`

    - `"developer"`

    - `"billing"`

    - `"admin"`

    - `"claude_code_user"`

  - `status: "accepted" or "expired" or "deleted" or "pending"`

    Status of the Invite.

    - `"accepted"`

    - `"expired"`

    - `"deleted"`

    - `"pending"`

  - `type: "invite"`

    Object type.

    For Invites, this is always `"invite"`.

    - `"invite"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/invites/$INVITE_ID \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```