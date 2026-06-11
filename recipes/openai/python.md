# OpenAI: Python mock recipe

The official `openai` Python SDK (v1+) uses `httpx`. The cleanest approach is `httpx.MockTransport`, which lets you intercept requests without monkeypatching.

## Install

```bash
pip install pytest httpx openai
```

## Setup

```python
# tests/conftest.py
import json
from pathlib import Path
import httpx
import pytest
from openai import OpenAI

FIXTURES = Path(__file__).parent / "fixtures"

def load_fixture(name):
    return json.loads((FIXTURES / f"{name}.json").read_text())

def make_client(handler):
    transport = httpx.MockTransport(handler)
    http_client = httpx.Client(transport=transport)
    return OpenAI(api_key="sk-test-fake", http_client=http_client)
```

## Successful chat completion

```python
def test_chat_completion():
    def handler(request):
        assert request.url.path == "/v1/chat/completions"
        body = json.loads(request.content)
        assert body["model"] == "gpt-4o"
        return httpx.Response(200, json=load_fixture("chat_completion_success"))

    client = make_client(handler)
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "hello"}],
    )
    assert response.choices[0].message.content == "Hi there."
    assert response.usage.total_tokens == 12
```

## Rate limit with retry-after

```python
def test_rate_limit_retry():
    call_count = {"n": 0}

    def handler(request):
        call_count["n"] += 1
        if call_count["n"] == 1:
            return httpx.Response(
                429,
                json={"error": {"type": "rate_limit_error", "message": "Rate limit"}},
                headers={"retry-after": "1"},
            )
        return httpx.Response(200, json=load_fixture("chat_completion_success"))

    client = make_client(handler)
    response = client.chat.completions.create(
        model="gpt-4o", messages=[{"role": "user", "content": "hi"}]
    )
    assert call_count["n"] == 2
    assert response.choices[0].message.content == "Hi there."
```

The SDK retries 429s by default (`max_retries=2`). Set `max_retries=0` on the client to disable for tests that need to inspect the failed path.

## Tool / function calling

```python
def test_tool_call():
    fixture = load_fixture("chat_completion_with_tool_call")

    def handler(request):
        return httpx.Response(200, json=fixture)

    client = make_client(handler)
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "what's the weather in Paris?"}],
        tools=[{
            "type": "function",
            "function": {
                "name": "get_weather",
                "parameters": {"type": "object", "properties": {"city": {"type": "string"}}},
            },
        }],
    )
    msg = response.choices[0].message
    assert msg.tool_calls[0].function.name == "get_weather"
    args = json.loads(msg.tool_calls[0].function.arguments)
    assert args["city"] == "Paris"
```

## Streaming

Streaming responses are server-sent events. Mock by streaming chunks:

```python
def test_streaming():
    def handler(request):
        chunks = [
            'data: {"choices":[{"delta":{"role":"assistant"}}]}\n\n',
            'data: {"choices":[{"delta":{"content":"Hi"}}]}\n\n',
            'data: {"choices":[{"delta":{"content":" there."}}]}\n\n',
            'data: [DONE]\n\n',
        ]
        body = "".join(chunks).encode()
        return httpx.Response(200, content=body, headers={"content-type": "text/event-stream"})

    client = make_client(handler)
    stream = client.chat.completions.create(
        model="gpt-4o", messages=[{"role": "user", "content": "hi"}], stream=True
    )
    pieces = [chunk.choices[0].delta.content or "" for chunk in stream]
    assert "".join(pieces) == "Hi there."
```

## Context length error

```python
def test_context_length_exceeded():
    def handler(request):
        return httpx.Response(
            400,
            json={
                "error": {
                    "type": "invalid_request_error",
                    "code": "context_length_exceeded",
                    "message": "This model's maximum context length is 128000 tokens.",
                }
            },
        )

    from openai import BadRequestError
    client = make_client(handler)
    with pytest.raises(BadRequestError) as exc_info:
        client.chat.completions.create(
            model="gpt-4o", messages=[{"role": "user", "content": "x" * 1000000}]
        )
    assert exc_info.value.code == "context_length_exceeded"
```

## Embeddings

```python
def test_embeddings():
    def handler(request):
        return httpx.Response(200, json=load_fixture("embedding_response"))

    client = make_client(handler)
    response = client.embeddings.create(
        model="text-embedding-3-small", input="hello world"
    )
    assert len(response.data[0].embedding) == 1536
```

## Common pitfall: token counting

The SDK does not count tokens before sending. If your test code asserts on `response.usage.prompt_tokens`, that field comes from the mock, not from real tokenization. Don't use mocks to assert real tokenizer behavior; that's what `tiktoken` is for.
