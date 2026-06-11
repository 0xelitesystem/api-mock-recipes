# Anthropic: Node mock recipe

The official `@anthropic-ai/sdk` Node package uses `fetch`. `nock` works.

## Install

```bash
npm install --save-dev jest nock @anthropic-ai/sdk
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => nock.disableNetConnect());
afterEach(() => nock.cleanAll());
afterAll(() => nock.enableNetConnect());
```

## Successful message

```javascript
const nock = require("nock");
const Anthropic = require("@anthropic-ai/sdk").default;
const success = require("./fixtures/messages_success.json");

test("messages.create", async () => {
  nock("https://api.anthropic.com")
    .post("/v1/messages")
    .reply(200, success);

  const client = new Anthropic({ apiKey: "sk-ant-test-fake" });
  const response = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "hello" }],
  });

  expect(response.content[0].text).toBe("Hi there.");
  expect(response.usage.input_tokens).toBe(8);
  expect(response.stop_reason).toBe("end_turn");
});
```

## Tool use

```javascript
test("tool use", async () => {
  const fixture = require("./fixtures/messages_with_tool_use.json");
  nock("https://api.anthropic.com")
    .post("/v1/messages")
    .reply(200, fixture);

  const client = new Anthropic({ apiKey: "sk-ant-test-fake" });
  const response = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    tools: [{
      name: "get_weather",
      description: "Get weather for a city",
      input_schema: {
        type: "object",
        properties: { city: { type: "string" } },
        required: ["city"],
      },
    }],
    messages: [{ role: "user", content: "weather in Paris?" }],
  });

  const toolBlock = response.content.find((b) => b.type === "tool_use");
  expect(toolBlock.name).toBe("get_weather");
  expect(toolBlock.input).toEqual({ city: "Paris" });
  expect(response.stop_reason).toBe("tool_use");
});
```

## Streaming

```javascript
test("streaming", async () => {
  const events = [
    'event: message_start',
    'data: {"type":"message_start","message":{"id":"msg_test","type":"message","role":"assistant","content":[],"model":"claude-sonnet-4-5","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":8,"output_tokens":0}}}',
    '',
    'event: content_block_start',
    'data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}',
    '',
    'event: content_block_delta',
    'data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hi"}}',
    '',
    'event: content_block_delta',
    'data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" there."}}',
    '',
    'event: content_block_stop',
    'data: {"type":"content_block_stop","index":0}',
    '',
    'event: message_delta',
    'data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":4}}',
    '',
    'event: message_stop',
    'data: {"type":"message_stop"}',
    '',
    '',
  ].join('\n');

  nock("https://api.anthropic.com")
    .post("/v1/messages")
    .reply(200, events, { "content-type": "text/event-stream" });

  const client = new Anthropic({ apiKey: "sk-ant-test-fake" });
  const stream = client.messages.stream({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "hi" }],
  });

  let text = "";
  for await (const event of stream) {
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
      text += event.delta.text;
    }
  }
  expect(text).toBe("Hi there.");
});
```

## Overloaded (529) and authentication errors

```javascript
test("overloaded", async () => {
  nock("https://api.anthropic.com")
    .post("/v1/messages")
    .reply(529, {
      type: "error",
      error: { type: "overloaded_error", message: "Overloaded" },
    });

  const client = new Anthropic({ apiKey: "sk-ant-test-fake", maxRetries: 0 });
  await expect(
    client.messages.create({
      model: "claude-sonnet-4-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "hi" }],
    })
  ).rejects.toMatchObject({ status: 529 });
});
```

## Common pitfall: `content` is always an array

Even for a simple text response, `response.content` is `[{type: "text", text: "..."}]`, not a string. Code that does `response.content.toLowerCase()` crashes on the array. Mocks must use the array form.
