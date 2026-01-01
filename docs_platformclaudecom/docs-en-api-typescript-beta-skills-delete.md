---
title: Delete Skill (Beta) (TypeScript)
url: https://platform.claude.com/docs/en/api/typescript/beta/skills/delete.md
source: llms
fetched_at: 2026-01-01T02:10:23.185518764-03:00
rendered_js: false
word_count: 38
summary: document describes the API endpoint for deleting a skill in the Anthropic SDK, including parameters and response structure
tags:
    - api-delete
    - skill-management
    - anthropic-sdk
    - beta-features
category: api
---

## Delete

`client.beta.skills.delete(stringskillID, SkillDeleteParamsparams?, RequestOptionsoptions?): SkillDeleteResponse`

**delete** `/v1/skills/{skill_id}`

Delete Skill

### Parameters

- `skillID: string`

  Unique identifier for the skill.

  The format and length of IDs may change over time.

- `params: SkillDeleteParams`

  - `betas?: Array<AnthropicBeta>`

    Optional header to specify the beta version(s) you want to use.

    - `(string & {})`

    - `"message-batches-2024-09-24" | "prompt-caching-2024-07-31" | "computer-use-2024-10-22" | 16 more`

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

- `SkillDeleteResponse`

  - `id: string`

    Unique identifier for the skill.

    The format and length of IDs may change over time.

  - `type: string`

    Deleted object type.

    For Skills, this is always `"skill_deleted"`.

### Example

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: process.env['ANTHROPIC_API_KEY'], // This is the default and can be omitted
});

const skill = await client.beta.skills.delete('skill_id');

console.log(skill.id);
```