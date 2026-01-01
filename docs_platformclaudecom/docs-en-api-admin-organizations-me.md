---
title: Get Current Organization
url: https://platform.claude.com/docs/en/api/admin/organizations/me.md
source: llms
fetched_at: 2026-01-01T02:10:46.690014967-03:00
rendered_js: false
word_count: 30
summary: describes how to retrieve authenticated organization details via API endpoint `/v1/organizations/me`
tags:
    - api-endpoint
    - organization-data
    - authentication-key
    - api-response
category: api
---

## Me

**get** `/v1/organizations/me`

Retrieve information about the organization associated with the authenticated API key.

### Returns

- `Organization = object { id, name, type }`

  - `id: string`

    ID of the Organization.

  - `name: string`

    Name of the Organization.

  - `type: "organization"`

    Object type.

    For Organizations, this is always `"organization"`.

    - `"organization"`

### Example

```http
curl https://api.anthropic.com/v1/organizations/me \
    -H "X-Api-Key: $ANTHROPIC_ADMIN_API_KEY"
```