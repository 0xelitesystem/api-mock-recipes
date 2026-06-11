# Google APIs: Python mock recipe

Google's `google-api-python-client` builds clients dynamically from discovery documents. The cleanest way to mock is `HttpMock` from `googleapiclient.http`, which is built into the library.

## Install

```bash
pip install pytest google-api-python-client google-auth
```

## Mocking pattern

`HttpMock` plays back recorded responses. You pass a fixture file and an expected response code. The first call to `.execute()` returns the fixture content.

```python
# tests/conftest.py
import json
from pathlib import Path
from unittest.mock import MagicMock
import pytest
from googleapiclient.discovery import build
from googleapiclient.http import HttpMock

FIXTURES = Path(__file__).parent / "fixtures"

@pytest.fixture
def gmail_service_factory():
    def _factory(fixture_name, status="200"):
        http = HttpMock(str(FIXTURES / f"{fixture_name}.json"), {"status": status})
        # cache_discovery=False avoids hitting the real discovery doc
        return build("gmail", "v1", http=http, cache_discovery=False, developerKey="fake")
    return _factory
```

Note: this still calls the discovery document fetch over the network on first `build()`. To avoid that completely, use `build_from_document` with a cached discovery JSON:

```python
@pytest.fixture
def gmail_service_factory():
    discovery = (FIXTURES / "gmail_discovery_v1.json").read_text()
    def _factory(fixture_name, status="200"):
        http = HttpMock(str(FIXTURES / f"{fixture_name}.json"), {"status": status})
        from googleapiclient.discovery import build_from_document
        return build_from_document(discovery, http=http, developerKey="fake")
    return _factory
```

Cache the discovery doc once by running `curl https://gmail.googleapis.com/$discovery/rest?version=v1 > gmail_discovery_v1.json`.

## Get a Gmail message

```python
def test_get_message(gmail_service_factory):
    service = gmail_service_factory("gmail_message")
    msg = service.users().messages().get(userId="me", id="abc123").execute()
    assert msg["id"] == "18b1f5e1a2b3c4d5"
    assert msg["payload"]["headers"][0]["name"] == "From"
```

## List with pagination

```python
def test_list_messages_paginated(gmail_service_factory):
    # HttpMock returns the same response for every call. For pagination, use HttpMockSequence.
    from googleapiclient.http import HttpMockSequence
    http = HttpMockSequence([
        ({"status": "200"}, json.dumps({
            "messages": [{"id": "m1"}, {"id": "m2"}],
            "nextPageToken": "tok123",
        })),
        ({"status": "200"}, json.dumps({
            "messages": [{"id": "m3"}],
        })),
    ])
    from googleapiclient.discovery import build_from_document
    discovery = (FIXTURES / "gmail_discovery_v1.json").read_text()
    service = build_from_document(discovery, http=http, developerKey="fake")

    all_ids = []
    page_token = None
    while True:
        result = service.users().messages().list(userId="me", pageToken=page_token).execute()
        all_ids.extend([m["id"] for m in result.get("messages", [])])
        page_token = result.get("nextPageToken")
        if not page_token:
            break
    assert all_ids == ["m1", "m2", "m3"]
```

## Drive file metadata

```python
def test_get_drive_file(monkeypatch):
    discovery = (FIXTURES / "drive_discovery_v3.json").read_text()
    http = HttpMock(str(FIXTURES / "drive_file_metadata.json"), {"status": "200"})
    from googleapiclient.discovery import build_from_document
    service = build_from_document(discovery, http=http, developerKey="fake")
    metadata = service.files().get(fileId="1abc...").execute()
    assert metadata["name"] == "Quarterly Report.pdf"
    assert metadata["mimeType"] == "application/pdf"
```

## Auth errors (401)

```python
def test_invalid_credentials():
    http = HttpMock(
        str(FIXTURES / "auth_error.json"),
        {"status": "401"},
    )
    from googleapiclient.discovery import build_from_document
    from googleapiclient.errors import HttpError
    discovery = (FIXTURES / "gmail_discovery_v1.json").read_text()
    service = build_from_document(discovery, http=http, developerKey="fake")
    with pytest.raises(HttpError) as exc_info:
        service.users().messages().get(userId="me", id="abc").execute()
    assert exc_info.value.resp.status == 401
```

`auth_error.json` contains:

```json
{"error": {"code": 401, "message": "Request had invalid authentication credentials.", "status": "UNAUTHENTICATED"}}
```

## Alternative: mock the HTTP transport directly with `httpx` or `responses`

If you don't want to deal with `HttpMock` / discovery docs, build your own thin wrapper that calls Google APIs via `httpx` and mock that. Faster for simple cases but loses the SDK's request building.

## Common pitfall: discovery document fetched on every `build()`

`build("gmail", "v1")` makes an HTTPS request to fetch the discovery document UNLESS you cache it or use `build_from_document`. Tests that "don't make network calls" actually do, slowly. Cache the discovery doc once and reuse it.

## Common pitfall: `execute()` is what triggers the request

`service.users().messages().get(...)` builds the request object but doesn't send it. `.execute()` sends. Asserting "no requests were made" after building a request without executing is misleading.
