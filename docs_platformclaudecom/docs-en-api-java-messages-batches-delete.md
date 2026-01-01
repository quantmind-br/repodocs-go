---
title: Delete a Message Batch (Java)
url: https://platform.claude.com/docs/en/api/java/messages/batches/delete.md
source: llms
fetched_at: 2026-01-01T02:10:14.144798672-03:00
rendered_js: false
word_count: 62
summary: explains how to delete a completed Message Batch in the Claude API via batch processing operations
tags:
    - api-deletion
    - batch-processing
    - claude-api
    - message-batch
    - deletion-endpoint
category: guide
---

## Delete

`DeletedMessageBatch messages().batches().delete(BatchDeleteParamsparams = BatchDeleteParams.none(), RequestOptionsrequestOptions = RequestOptions.none())`

**delete** `/v1/messages/batches/{message_batch_id}`

Delete a Message Batch.

Message Batches can only be deleted once they've finished processing. If you'd like to delete an in-progress batch, you must first cancel it.

Learn more about the Message Batches API in our [user guide](https://docs.claude.com/en/docs/build-with-claude/batch-processing)

### Parameters

- `BatchDeleteParams params`

  - `Optional<String> messageBatchId`

    ID of the Message Batch.

### Returns

- `class DeletedMessageBatch:`

  - `String id`

    ID of the Message Batch.

  - `JsonValue; type "message_batch_deleted"constant`

    Deleted object type.

    For Message Batches, this is always `"message_batch_deleted"`.

    - `MESSAGE_BATCH_DELETED("message_batch_deleted")`

### Example

```java
package com.anthropic.example;

import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.batches.BatchDeleteParams;
import com.anthropic.models.messages.batches.DeletedMessageBatch;

public final class Main {
    private Main() {}

    public static void main(String[] args) {
        AnthropicClient client = AnthropicOkHttpClient.fromEnv();

        DeletedMessageBatch deletedMessageBatch = client.messages().batches().delete("message_batch_id");
    }
}
```