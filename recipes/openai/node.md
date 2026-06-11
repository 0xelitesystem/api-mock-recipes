# OpenAI: Node mock recipe

The official `openai` Node SDK uses `fetch`. Both `nock` and `msw` work; `nock` is the lighter setup.

## Install

```bash
npm install --save-dev jest nock openai
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => nock.disableNetConnect());
afterEach(() => nock.cleanAll());
afterAll(() => nock.enableNetConnect());
```

## Successful chat completion

```javascript
const nock = require("nock");
const OpenAI = require("openai").default;
const success = require("./fixtures/chat_completion_success.json");

test("chat completion", async () => {
  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(200, success);

  const client = new OpenAI({ apiKey: "sk-test-fake" });
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "hello" }],
  });

  expect(response.choices[0].message.content).toBe("Hi there.");
  expect(response.usage.total_tokens).toBe(12);
});
```

## Rate limit with retry

```javascript
test("retries on rate limit", async () => {
  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(429, { error: { type: "rate_limit_error" } }, { "retry-after": "1" });

  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(200, success);

  const client = new OpenAI({ apiKey: "sk-test-fake", maxRetries: 2 });
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "hi" }],
  });
  expect(response.choices[0].message.content).toBe("Hi there.");
});
```

## Streaming

```javascript
test("streaming", async () => {
  const sseBody = [
    'data: {"choices":[{"delta":{"role":"assistant"}}]}',
    "",
    'data: {"choices":[{"delta":{"content":"Hi"}}]}',
    "",
    'data: {"choices":[{"delta":{"content":" there."}}]}',
    "",
    "data: [DONE]",
    "",
    "",
  ].join("\n");

  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(200, sseBody, { "content-type": "text/event-stream" });

  const client = new OpenAI({ apiKey: "sk-test-fake" });
  const stream = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "hi" }],
    stream: true,
  });

  let text = "";
  for await (const chunk of stream) {
    text += chunk.choices[0]?.delta?.content || "";
  }
  expect(text).toBe("Hi there.");
});
```

## Tool calling

```javascript
test("tool call", async () => {
  const fixture = require("./fixtures/chat_completion_with_tool_call.json");
  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(200, fixture);

  const client = new OpenAI({ apiKey: "sk-test-fake" });
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "weather in Paris?" }],
    tools: [{
      type: "function",
      function: {
        name: "get_weather",
        parameters: { type: "object", properties: { city: { type: "string" } } },
      },
    }],
  });

  const toolCall = response.choices[0].message.tool_calls[0];
  expect(toolCall.function.name).toBe("get_weather");
  const args = JSON.parse(toolCall.function.arguments);
  expect(args.city).toBe("Paris");
});
```

## Context length error

```javascript
test("context length exceeded", async () => {
  nock("https://api.openai.com")
    .post("/v1/chat/completions")
    .reply(400, {
      error: {
        type: "invalid_request_error",
        code: "context_length_exceeded",
        message: "This model's maximum context length is 128000 tokens.",
      },
    });

  const client = new OpenAI({ apiKey: "sk-test-fake", maxRetries: 0 });
  await expect(
    client.chat.completions.create({
      model: "gpt-4o",
      messages: [{ role: "user", content: "x".repeat(1000000) }],
    })
  ).rejects.toMatchObject({
    status: 400,
    code: "context_length_exceeded",
  });
});
```

## Common pitfall: maxRetries default is 2

If your test expects exactly one HTTP call and the mocked response is a 5xx, the SDK will retry. You'll get a confusing "no more mocks matched" error from `nock`. Set `maxRetries: 0` on the client for tests where you want to inspect a single failure.
