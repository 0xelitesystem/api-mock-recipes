# OpenAI: things that aren't bugs

## SDK retries by default

Both Python and Node SDKs retry idempotent failures (429, 5xx) twice by default. If you mock once and the first response is a 429, the test will fail with "out of mocks" rather than the expected rate-limit error. Set `max_retries=0` to inspect a single failure path.

## Model names include dated suffixes

`gpt-4o` in your request gets routed to a specific version like `gpt-4o-2024-08-06` in the response's `model` field. Fixtures should include the dated string. Tests that assert `response.model == "gpt-4o"` will fail. Use `response.model.startswith("gpt-4o")` or just don't assert on it.

## Streaming chunks have `delta`, not `message`

The non-streaming response shape is `choices[0].message.content`. The streaming chunk shape is `choices[0].delta.content`. Code that mixes them up fails on the first chunk. Mocks for streaming must produce delta-shaped chunks.

## `[DONE]` is not JSON

The final SSE line is the literal string `data: [DONE]`, not `data: {"done": true}`. Tests that try to JSON-parse every line crash on the last one. The SDK handles this; raw HTTP mocks need to send the literal.

## Token counts come back, but don't trust them in tests

`response.usage.total_tokens` reflects what the API counted. In mocks, this is whatever number you put in the fixture. Don't use mocked usage numbers to test billing logic - your billing tests will pass with garbage data. Use real `tiktoken` counts for those tests.

## Refusal field

Newer responses include `message.refusal` which is null on normal responses but a string when the model refuses. Code that does `message.content.lower()` crashes when `content` is null and `refusal` is set. Mocks should include both refusal and normal cases.

## Tool call arguments are stringified JSON

`tool_calls[0].function.arguments` is a JSON string, not a parsed object. You must `JSON.parse` / `json.loads` it before using. Tests should mirror this: the fixture's `arguments` field is a string.

## Function calling (legacy) vs tools

`function_call` (old) and `tool_calls` (new) are different fields. Some old SDK code still uses `function_call`. New SDKs emit `tool_calls` with a `tools` array. Don't mix them in one mock; the SDK will pick the wrong path.

## Embeddings are normalized but you can ask for raw

The `text-embedding-3-*` models support `dimensions` parameter to truncate. If your code passes `dimensions=512`, the response embedding has length 512, not 1536. Fixture length must match.

## Vision requests have a different content shape

For image inputs, `content` is an array of `{"type": "text"|"image_url", ...}` objects, not a string. Mocks for vision endpoints need to accept the array form in requests and return text-only content in responses.
