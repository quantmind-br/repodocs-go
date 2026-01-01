---
title: Delete a Message Batch (Python)
url: https://platform.claude.com/docs/en/api/python/messages/batches/delete.md
source: llms
fetched_at: 2026-01-01T02:10:15.533506394-03:00
rendered_js: false
word_count: 61
summary: explains how to delete a finished Message Batch via API endpoint and Python client
tags:
    - api-delete
    - message-batch
    - batch-processing
    - claude-api
    - endpoint-methods
category: api
---

## Delete

`messages.batches.delete(strmessage_batch_id)  -> DeletedMessageBatch`

**delete** `/v1/messages/batches/{message_batch_id}`

Delete a Message Batch.

Message Batches can only be deleted once they've finished processing. If you'd like to delete an in-progress batch, you must first cancel it.

Learn more about the Message Batches API in our [user guide](https://docs.claude.com/en/docs/build-with-claude/batch-processing)

### Parameters

- `message_batch_id: str`

  ID of the Message Batch.

### Returns

- `class DeletedMessageBatch: …`

  - `id: str`

    ID of the Message Batch.

  - `type: Literal["message_batch_deleted"]`

    Deleted object type.

    For Message Batches, this is always `"message_batch_deleted"`.

    - `"message_batch_deleted"`

### Example

```python
import os
from anthropic import Anthropic

client = Anthropic(
    api_key=os.environ.get("ANTHROPIC_API_KEY"),  # This is the default and can be omitted
)
deleted_message_batch = client.messages.batches.delete(
    "message_batch_id",
)
print(deleted_message_batch.id)
```