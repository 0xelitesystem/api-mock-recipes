# Anthropic: Python mock recipe

The official `anthropic` Python SDK uses `httpx`. Use `httpx.MockTransport`, same pattern as OpenAI.

## Install

```bash
pip install pytest httpx anthropic
```

## Setup

```python
# tests/conftest.py
import json
from pathlib import Path
import httpx
from anthropic import Anthropic

FIXTURES = Path(__file__).parent / "fixtures"

def load_fixture(name):
    return json.loads((FIXTURES / f"{name}.json").read_text())

def make_client(handler):
    transport = httpx.MockTransport(handler)
    http_client = httpx.Client(transport=transport)
    return Anthropic(api_key="sk-ant-test-fake", http_client=http_client)
```

## Successful message

```python
def test_messages_create():
    def handler(request):
        assert request.url.path == "/v1/messages"
        body = json.loads(request.content)
        assert body["model"].startswith("claude-")
        return httpx.Response(200, json=load_fixture("messages_success"))

    client = make_client(handler)
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": "hello"}],
    )
    assert response.content[0].text == "Hi there."
    assert response.usage.input_tokens == 8
    assert response.stop_reason == "end_turn"
```

## Tool use

```python
def test_tool_use():
    def handler(request):
        return httpx.Response(200, json=load_fixture("messages_with_tool_use"))

    client = make_client(handler)
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=[{
            "name": "get_weather",
            "description": "Get weather for a city",
            "input_schema": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"],
            },
        }],
        messages=[{"role": "user", "content": "weather in Paris?"}],
    )
    tool_block = next(b for b in response.content if b.type == "tool_use")
    assert tool_block.name == "get_weather"
    assert tool_block.input == {"city": "Paris"}
    assert response.stop_reason == "tool_use"
```

The response `content` is an array of blocks. A response with tool use can also include a text block before the tool block. Tests should iterate, not index.

## Streaming

Anthropic uses SSE with named events (`event: message_start` etc.), not just `data:` lines.

```python
def test_streaming():
    def handler(request):
        events = [
            'event: message_start\ndata: {"type":"message_start","message":{"id":"msg_test","type":"message","role":"assistant","content":[],"model":"claude-sonnet-4-5","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":8,"output_tokens":0}}}\n\n',
            'event: content_block_start\ndata: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}\n\n',
            'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hi"}}\n\n',
            'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" there."}}\n\n',
            'event: content_block_stop\ndata: {"type":"content_block_stop","index":0}\n\n',
            'event: message_delta\ndata: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":4}}\n\n',
            'event: message_stop\ndata: {"type":"message_stop"}\n\n',
        ]
        return httpx.Response(200, content="".join(events).encode(), headers={"content-type": "text/event-stream"})

    client = make_client(handler)
    with client.messages.stream(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": "hi"}],
    ) as stream:
        text = "".join(chunk for chunk in stream.text_stream)
    assert text == "Hi there."
```

## Overloaded error (529)

Anthropic returns a 529 when servers are overloaded. The SDK does NOT retry this by default.

```python
def test_overloaded():
    def handler(request):
        return httpx.Response(
            529,
            json={"type": "error", "error": {"type": "overloaded_error", "message": "Overloaded"}},
        )

    from anthropic import APIStatusError
    client = make_client(handler)
    with pytest.raises(APIStatusError) as exc_info:
        client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": "hi"}],
        )
    assert exc_info.value.status_code == 529
```

## Invalid API key

```python
def test_invalid_api_key():
    def handler(request):
        return httpx.Response(
            401,
            json={
                "type": "error",
                "error": {"type": "authentication_error", "message": "invalid x-api-key"},
            },
        )

    from anthropic import AuthenticationError
    client = make_client(handler)
    with pytest.raises(AuthenticationError):
        client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": "hi"}],
        )
```

## Common pitfall: stop_sequence vs stop_reason

`stop_reason` is the high-level reason (`end_turn`, `max_tokens`, `tool_use`, `stop_sequence`). `stop_sequence` is the specific string that triggered, or null. Confusing them produces tests that look correct but assert the wrong field.
