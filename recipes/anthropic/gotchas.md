# Anthropic: things that aren't bugs

## 529 does NOT auto-retry by default in the SDK

OpenAI's SDK retries 429s/5xx automatically. Anthropic's SDK has retry logic too, but a 529 `overloaded_error` is treated differently from a 429 `rate_limit_error`. When mocking, decide which you're testing. If your code wraps Anthropic calls in your own retry logic for 529s, the test needs to allow multiple HTTP attempts.

## `content` is always an array, even for a single text response

`response.content` is always a list of blocks: `[{"type": "text", "text": "..."}]`. Treating it like a string crashes. This is true for streaming AND non-streaming.

Tool-use responses can contain a text block AND a tool_use block in the same array. Iterate, don't index.

## `stop_reason` values

The valid stop_reasons are `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, and `refusal`. Mocking `stop_reason: "stop"` (which is OpenAI's value) breaks any conditional that checks for the Anthropic-specific value.

## `stop_sequence` is a string or null, not a boolean

`stop_sequence` contains the literal stop string that triggered, or `null`. Code that does `if response.stop_sequence:` works fine. Code that compares it to `true` doesn't.

## Streaming events have named event types

SSE events from Anthropic use named events (`event: message_start\ndata: {...}`), not just `data:` lines. Mock streams that only send `data:` lines won't parse correctly in the SDK.

The event sequence is: `message_start`, then `content_block_start`/`delta`/`stop` for each block, then `message_delta`, then `message_stop`. `message_delta` is where the final `stop_reason` and `usage.output_tokens` arrive. Don't put them in `message_start`.

## System prompt is a separate parameter, not a message

`system` is its own top-level parameter:

```python
client.messages.create(
    model="claude-sonnet-4-5",
    system="You are a helpful assistant.",
    messages=[{"role": "user", "content": "hi"}],
    max_tokens=1024,
)
```

Putting it as `{"role": "system", "content": "..."}` in `messages` is the OpenAI shape and won't work with Anthropic's API. Your mocks should expect `system` at the top level of the request body, not in the messages array.

## `max_tokens` is required

Unlike OpenAI where `max_tokens` is optional, Anthropic requires it. If your tests omit it, the real API returns 400. Mocks that accept missing `max_tokens` hide this bug.

## Vision input uses `source`, not `image_url`

Image blocks look like:

```json
{"type": "image", "source": {"type": "base64", "media_type": "image/jpeg", "data": "..."}}
```

Not OpenAI's `{"type": "image_url", "image_url": {"url": "..."}}`. Different field names. Different nesting.

## Cache control on system prompts and tools

If your code uses prompt caching (`cache_control: {"type": "ephemeral"}` on system blocks or tool definitions), the response `usage` field includes `cache_creation_input_tokens` and `cache_read_input_tokens`. Mocks that omit these fields cause AttributeErrors in code that reads them.

## Tool result blocks go in user messages, not assistant

To send a tool result back, the message looks like:

```json
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_...", "content": "result text"}]}
```

Role is `user`, not `tool` (which is OpenAI's convention). Mocks that accept the wrong role hide schema bugs.
