---
title: Delete Skill (Beta) (Python)
url: https://platform.claude.com/docs/en/api/python/beta/skills/delete.md
source: llms
fetched_at: 2026-01-01T02:10:22.194577144-03:00
rendered_js: false
word_count: 59
summary: Document explains how to delete a skill using the Anthropic API's beta skills deletion endpoint.
tags:
    - api-delete
    - skills-management
    - anthropic-beta
    - endpoint-specification
category: api
---

## Delete

`beta.skills.delete(strskill_id, SkillDeleteParams**kwargs)  -> SkillDeleteResponse`

**delete** `/v1/skills/{skill_id}`

Delete Skill

### Parameters

- `skill_id: str`

  Unique identifier for the skill.

  The format and length of IDs may change over time.

- `betas: Optional[List[AnthropicBetaParam]]`

  Optional header to specify the beta version(s) you want to use.

  - `UnionMember0 = str`

  - `UnionMember1 = Literal["message-batches-2024-09-24", "prompt-caching-2024-07-31", "computer-use-2024-10-22", 16 more]`

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

- `class SkillDeleteResponse: …`

  - `id: str`

    Unique identifier for the skill.

    The format and length of IDs may change over time.

  - `type: str`

    Deleted object type.

    For Skills, this is always `"skill_deleted"`.

### Example

```python
import os
from anthropic import Anthropic

client = Anthropic(
    api_key=os.environ.get("ANTHROPIC_API_KEY"),  # This is the default and can be omitted
)
skill = client.beta.skills.delete(
    skill_id="skill_id",
)
print(skill.id)
```