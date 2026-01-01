---
title: Delete a Message Batch (Ruby)
url: https://platform.claude.com/docs/en/api/ruby/messages/batches/delete.md
source: llms
fetched_at: 2026-01-01T02:10:15.918873887-03:00
rendered_js: false
word_count: 60
summary: explains how to delete a completed Message Batch via API endpoint and Ruby SDK
tags:
    - api-delete
    - message-batch
    - batch-processing
    - claude-api
    - deletion-operation
category: guide
---

## Delete

`messages.batches.delete(message_batch_id) -> DeletedMessageBatch`

**delete** `/v1/messages/batches/{message_batch_id}`

Delete a Message Batch.

Message Batches can only be deleted once they've finished processing. If you'd like to delete an in-progress batch, you must first cancel it.

Learn more about the Message Batches API in our [user guide](https://docs.claude.com/en/docs/build-with-claude/batch-processing)

### Parameters

- `message_batch_id: String`

  ID of the Message Batch.

### Returns

- `class DeletedMessageBatch`

  - `id: String`

    ID of the Message Batch.

  - `type: :message_batch_deleted`

    Deleted object type.

    For Message Batches, this is always `"message_batch_deleted"`.

    - `:message_batch_deleted`

### Example

```ruby
require "anthropic"

anthropic = Anthropic::Client.new(api_key: "my-anthropic-api-key")

deleted_message_batch = anthropic.messages.batches.delete("message_batch_id")

puts(deleted_message_batch)
```