# Twilio: Python mock recipe

The `twilio` Python SDK uses `requests`. `responses` works cleanly.

## Install

```bash
pip install pytest responses twilio
```

## Send an SMS

```python
import json
import responses
from twilio.rest import Client


ACCOUNT_SID = "ACtest1234567890abcdef1234567890abcdef"
AUTH_TOKEN = "test_auth_token"


@responses.activate
def test_send_sms():
    responses.add(
        responses.POST,
        f"https://api.twilio.com/2010-04-01/Accounts/{ACCOUNT_SID}/Messages.json",
        json={
            "sid": "SMtest1234567890",
            "account_sid": ACCOUNT_SID,
            "from": "+15551234567",
            "to": "+15559999999",
            "body": "Hello from a test",
            "status": "queued",
            "date_created": "Wed, 15 Nov 2023 10:00:00 +0000",
            "date_sent": None,
            "direction": "outbound-api",
            "num_segments": "1",
            "price": None,
            "price_unit": "USD",
            "error_code": None,
            "error_message": None,
            "uri": f"/2010-04-01/Accounts/{ACCOUNT_SID}/Messages/SMtest1234567890.json",
        },
        status=201,
    )

    client = Client(ACCOUNT_SID, AUTH_TOKEN)
    message = client.messages.create(
        body="Hello from a test",
        from_="+15551234567",
        to="+15559999999",
    )
    assert message.sid == "SMtest1234567890"
    assert message.status == "queued"
```

## Inspect the request body

Twilio uses `application/x-www-form-urlencoded` POSTs, not JSON. The request body contains form-encoded fields.

```python
from urllib.parse import parse_qs

@responses.activate
def test_request_body_form_encoded():
    responses.add(
        responses.POST,
        f"https://api.twilio.com/2010-04-01/Accounts/{ACCOUNT_SID}/Messages.json",
        json={"sid": "SMtest", "status": "queued"},
        status=201,
    )
    client = Client(ACCOUNT_SID, AUTH_TOKEN)
    client.messages.create(body="Hi", from_="+15551234567", to="+15559999999")

    body = responses.calls[0].request.body
    parsed = parse_qs(body)
    assert parsed["Body"] == ["Hi"]
    assert parsed["From"] == ["+15551234567"]
    assert parsed["To"] == ["+15559999999"]
```

Field names in the form body are PascalCase (`Body`, `From`, `To`), not snake_case. This is the API's actual wire format.

## Error: invalid phone number (400)

```python
@responses.activate
def test_invalid_phone():
    responses.add(
        responses.POST,
        f"https://api.twilio.com/2010-04-01/Accounts/{ACCOUNT_SID}/Messages.json",
        json={
            "code": 21211,
            "message": "The 'To' number +invalid is not a valid phone number.",
            "more_info": "https://www.twilio.com/docs/errors/21211",
            "status": 400,
        },
        status=400,
    )

    from twilio.base.exceptions import TwilioRestException
    client = Client(ACCOUNT_SID, AUTH_TOKEN)
    with pytest.raises(TwilioRestException) as exc_info:
        client.messages.create(body="Hi", from_="+15551234567", to="+invalid")
    assert exc_info.value.code == 21211
    assert exc_info.value.status == 400
```

Twilio error codes are 5-digit integers documented at https://www.twilio.com/docs/api/errors. `2xxxx` codes are client errors; `3xxxx` are carrier/delivery problems that arrive via status callbacks, not in the original response.

## Make a voice call

```python
@responses.activate
def test_make_call():
    responses.add(
        responses.POST,
        f"https://api.twilio.com/2010-04-01/Accounts/{ACCOUNT_SID}/Calls.json",
        json={
            "sid": "CAtest1234567890",
            "account_sid": ACCOUNT_SID,
            "from": "+15551234567",
            "to": "+15559999999",
            "status": "queued",
            "direction": "outbound-api",
            "duration": None,
            "price": None,
            "uri": f"/2010-04-01/Accounts/{ACCOUNT_SID}/Calls/CAtest1234567890.json",
        },
        status=201,
    )
    client = Client(ACCOUNT_SID, AUTH_TOKEN)
    call = client.calls.create(
        to="+15559999999",
        from_="+15551234567",
        url="http://demo.twilio.com/docs/voice.xml",
    )
    assert call.sid == "CAtest1234567890"
```

## Common pitfall: the account SID is in the URL path

Every Twilio resource URL starts with `/2010-04-01/Accounts/{AccountSid}/...`. If you use a different account SID in the mock URL than in the client, the mock won't match. Both must be the same string.

## Common pitfall: status callbacks are not in the response

When you create a message, the immediate response shows `status: "queued"`. The actual delivery status (`sent`, `delivered`, `failed`, `undelivered`) arrives later via webhooks if you configured a `statusCallback` URL. You cannot test "the SMS delivered" against the create API mock alone.
