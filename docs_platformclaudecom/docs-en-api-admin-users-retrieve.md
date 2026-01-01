---
title: Get User
url: https://platform.claude.com/docs/en/api/admin/users/retrieve.md
source: llms
fetched_at: 2026-01-01T02:10:59.472905269-03:00
rendered_js: false
word_count: 44
summary: document describes the API endpoint for retrieving user details from an organization's API.
tags:
    - api-endpoint
    - user-retrieval
    - organization-api
    - authentication-key
category: api
---

## Retrieve

**get** `/v1/organizations/users/{user_id}`

Get User

### Path Parameters

- `user_id: string`

  ID of the User.

### Returns

- `User = object { id, added_at, email, 3 more }`

  - `id: string`

    ID of the User.

  - `added_at: string`

    RFC 3339 datetime string indicating when the User joined the Organization.

  - `email: string`

    Email of the User.

  - `name: string`

    Name of the User.

  - `role: "user" or "developer" or "billing" or 2 more`

    Organization role of the User.

    - `"user"`

    - `"developer"`

    - `"billing"`

    - `"admin"`

    - `"claude_code_user"`

  - `type: "user"`

    Object type.

    For Users, this is always `"user"`.

    - `"user"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/users/$USER_ID \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```