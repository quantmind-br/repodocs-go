---
title: Delete Skill Version (Beta) (Ruby)
url: https://platform.claude.com/docs/en/api/ruby/beta/skills/versions/delete.md
source: llms
fetched_at: 2026-01-01T02:10:25.833703574-03:00
rendered_js: false
word_count: 76
summary: document describes the API endpoint for deleting a skill version via a beta client method
tags:
    - api-delete
    - skill-version
    - unix-epoch-timestamp
    - anthropic-beta
category: api
---

## Delete

`beta.skills.versions.delete(version, **kwargs) -> VersionDeleteResponse`

**delete** `/v1/skills/{skill_id}/versions/{version}`

Delete Skill Version

### Parameters

- `skill_id: String`

  Unique identifier for the skill.

  The format and length of IDs may change over time.

- `version: String`

  Version identifier for the skill.

  Each version is identified by a Unix epoch timestamp (e.g., "1759178010641129").

- `anthropic_beta: Array[AnthropicBeta]`

  Optional header to specify the beta version(s) you want to use.

  - `String`

  - `:"message-batches-2024-09-24" | :"prompt-caching-2024-07-31" | :"computer-use-2024-10-22" | 16 more`

    - `:"message-batches-2024-09-24"`

    - `:"prompt-caching-2024-07-31"`

    - `:"computer-use-2024-10-22"`

    - `:"computer-use-2025-01-24"`

    - `:"pdfs-2024-09-25"`

    - `:"token-counting-2024-11-01"`

    - `:"token-efficient-tools-2025-02-19"`

    - `:"output-128k-2025-02-19"`

    - `:"files-api-2025-04-14"`

    - `:"mcp-client-2025-04-04"`

    - `:"mcp-client-2025-11-20"`

    - `:"dev-full-thinking-2025-05-14"`

    - `:"interleaved-thinking-2025-05-14"`

    - `:"code-execution-2025-05-22"`

    - `:"extended-cache-ttl-2025-04-11"`

    - `:"context-1m-2025-08-07"`

    - `:"context-management-2025-06-27"`

    - `:"model-context-window-exceeded-2025-08-26"`

    - `:"skills-2025-10-02"`

### Returns

- `class VersionDeleteResponse`

  - `id: String`

    Version identifier for the skill.

    Each version is identified by a Unix epoch timestamp (e.g., "1759178010641129").

  - `type: String`

    Deleted object type.

    For Skill Versions, this is always `"skill_version_deleted"`.

### Example

```ruby
require "anthropic"

anthropic = Anthropic::Client.new(api_key: "my-anthropic-api-key")

version = anthropic.beta.skills.versions.delete("version", skill_id: "skill_id")

puts(version)
```