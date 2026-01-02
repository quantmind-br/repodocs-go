---
title: Token caching
url: https://github.com/QwenLM/qwen-code/blob/main/docs/users/features/token-caching.md
source: git
fetched_at: 2026-01-02T12:26:14.486882299-03:00
rendered_js: false
word_count: 193
summary: Explains how Qwen Code optimizes API costs via token caching for API key users, reducing token processing and costs automatically.
tags:
    - token-caching
    - api-cost-reduction
    - qwen-api
    - automatic-optimization
    - cache-monitoring
category: guide
---

# Token Caching and Cost Optimization

Qwen Code automatically optimizes API costs through token caching when using API key authentication. This feature stores frequently used content like system instructions and conversation history to reduce the number of tokens processed in subsequent requests.

## How It Benefits You

- **Cost reduction**: Less tokens mean lower API costs
- **Faster responses**: Cached content is retrieved more quickly
- **Automatic optimization**: No configuration needed - it works behind the scenes

## Token caching is available for

- API key users (Qwen API key, OpenAI-compatible providers)

## Monitoring Your Savings

Use the `/stats` command to see your cached token savings:

- When active, the stats display shows how many tokens were served from cache
- You'll see both the absolute number and percentage of cached tokens
- Example: "10,500 (90.4%) of input tokens were served from the cache, reducing costs."

This information is only displayed when cached tokens are being used, which occurs with API key authentication but not with OAuth authentication.

## Example Stats Display

![Qwen Code Stats Display](https://img.alicdn.com/imgextra/i3/O1CN01F1yzRs1juyZu63jdS_!!6000000004609-2-tps-1038-738.png)

The above image shows an example of the `/stats` command output, highlighting the cached token savings information.
