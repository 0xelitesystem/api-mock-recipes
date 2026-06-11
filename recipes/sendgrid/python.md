# SendGrid: Python mock recipe

The `sendgrid` Python SDK uses `python-http-client` under the hood, which uses `urllib`. The `responses` library does not intercept `urllib` directly. Use `httpretty` instead, or build at the HTTP layer with `requests-mock` after monkeypatching the transport.

The cleaner path: patch `sendgrid.SendGridAPIClient.send` directly for unit tests, and reserve real HTTP mocking for integration tests against the sandbox endpoint.

## Install

```bash
pip install pytest sendgrid responses httpretty
```

## Option A: patch the client method (recommended for unit tests)

```python
from unittest.mock import MagicMock, patch
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail


def test_send_email_success():
    sg = SendGridAPIClient(api_key="SG.test_fake")

    mock_response = MagicMock()
    mock_response.status_code = 202
    mock_response.headers = {"X-Message-Id": "abc-123"}
    mock_response.body = b""

    with patch.object(sg, "send", return_value=mock_response) as mock_send:
        message = Mail(
            from_email="from@example.com",
            to_emails="to@example.com",
            subject="Test",
            plain_text_content="Hello",
        )
        response = sg.send(message)
        assert response.status_code == 202
        mock_send.assert_called_once()
        # Inspect the Mail object that was sent
        sent_mail = mock_send.call_args[0][0]
        assert sent_mail.get()["from"]["email"] == "from@example.com"
```

## Option B: HTTP-layer mock with httpretty

```python
import httpretty
import json
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail


@httpretty.activate
def test_send_email_http_layer():
    httpretty.register_uri(
        httpretty.POST,
        "https://api.sendgrid.com/v3/mail/send",
        status=202,
        adding_headers={"X-Message-Id": "abc-123"},
        body="",
    )

    sg = SendGridAPIClient(api_key="SG.test_fake")
    message = Mail(
        from_email="from@example.com",
        to_emails="to@example.com",
        subject="Test",
        plain_text_content="Hello",
    )
    response = sg.send(message)
    assert response.status_code == 202
    assert response.headers["X-Message-Id"] == "abc-123"

    # Inspect the request body that was sent
    last_request = httpretty.last_request()
    body = json.loads(last_request.body)
    assert body["from"]["email"] == "from@example.com"
    assert body["personalizations"][0]["to"][0]["email"] == "to@example.com"
```

## Error: invalid from address (400)

```python
@httpretty.activate
def test_send_email_bad_from():
    httpretty.register_uri(
        httpretty.POST,
        "https://api.sendgrid.com/v3/mail/send",
        status=400,
        body=json.dumps({
            "errors": [{
                "message": "The from address does not match a verified Sender Identity.",
                "field": "from",
                "help": "https://sendgrid.com/docs/..."
            }]
        }),
    )

    from python_http_client.exceptions import HTTPError
    sg = SendGridAPIClient(api_key="SG.test_fake")
    message = Mail(
        from_email="unverified@example.com",
        to_emails="to@example.com",
        subject="Test",
        plain_text_content="Hello",
    )
    with pytest.raises(HTTPError) as exc_info:
        sg.send(message)
    assert exc_info.value.status_code == 400
```

## Inspecting the request payload

The most useful assertion in SendGrid tests is "what did we actually send?" SendGrid silently accepts a lot of malformed mail and surfaces issues asynchronously. Verifying the request shape catches client-side bugs:

```python
@httpretty.activate
def test_personalizations_correct():
    httpretty.register_uri(
        httpretty.POST,
        "https://api.sendgrid.com/v3/mail/send",
        status=202,
    )
    sg = SendGridAPIClient(api_key="SG.test_fake")
    message = Mail(
        from_email="from@example.com",
        to_emails=["a@example.com", "b@example.com"],
        subject="Test",
        plain_text_content="Hi",
    )
    sg.send(message)

    body = json.loads(httpretty.last_request().body)
    recipients = [t["email"] for t in body["personalizations"][0]["to"]]
    assert recipients == ["a@example.com", "b@example.com"]
```

## Common pitfall: 202 is success

A successful send returns HTTP 202, not 200. Tests that check `status_code == 200` will fail. 202 means "accepted for delivery" - SendGrid hasn't actually delivered yet, just queued.

## Common pitfall: bouncing emails don't fail the send

If you send to a known-bad address, the SendGrid API still returns 202. The bounce is reported via webhooks later. There is no way to test "this email actually delivered" via the send API mock - that's a webhook test.
