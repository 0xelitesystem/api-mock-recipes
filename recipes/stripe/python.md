# Stripe: Python mock recipe

Uses `responses` for `requests`-based clients or `httpx.MockTransport` for `httpx`-based clients. The official `stripe` Python SDK uses `requests`, so `responses` is the standard choice.

## Install

```bash
pip install pytest responses stripe
```

## Fixture loading

```python
# tests/conftest.py
import json
from pathlib import Path
import pytest

FIXTURES = Path(__file__).parent / "fixtures"

@pytest.fixture
def load_fixture():
    def _load(name):
        return json.loads((FIXTURES / f"{name}.json").read_text())
    return _load
```

## Successful charge

```python
import responses
import stripe

@responses.activate
def test_create_charge(load_fixture):
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json=load_fixture("charge_succeeded"),
        status=200,
    )
    stripe.api_key = "sk_test_fake"
    charge = stripe.Charge.create(amount=2000, currency="usd", source="tok_visa")
    assert charge.id.startswith("ch_")
    assert charge.status == "succeeded"
```

## Card declined

```python
@responses.activate
def test_card_declined(load_fixture):
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json=load_fixture("charge_declined"),
        status=402,
    )
    stripe.api_key = "sk_test_fake"
    with pytest.raises(stripe.error.CardError) as exc_info:
        stripe.Charge.create(amount=2000, currency="usd", source="tok_chargeDeclined")
    assert exc_info.value.code == "card_declined"
```

The official SDK translates the 402 + `error.type == "card_error"` shape into `CardError`. If you handle Stripe errors in your code by catching `CardError`, your tests must produce the same exception path - mocking just the HTTP response is enough.

## Rate limit with retry

```python
@responses.activate
def test_rate_limit_then_success(load_fixture):
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json={"error": {"type": "rate_limit_error", "message": "Too many requests"}},
        status=429,
        headers={"Retry-After": "1"},
    )
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json=load_fixture("charge_succeeded"),
        status=200,
    )
    stripe.api_key = "sk_test_fake"
    stripe.max_network_retries = 2
    charge = stripe.Charge.create(amount=2000, currency="usd", source="tok_visa")
    assert charge.status == "succeeded"
```

The SDK retries 429s automatically when `max_network_retries` is set. Test that your code path actually uses retries instead of failing on the first 429.

## Webhook signature verification (different problem)

For webhook tests, you're NOT mocking the Stripe API. You're constructing a real signed payload and verifying it. See `webhook-signature-verifier` for that pattern.

```python
import stripe
import time
import hmac
import hashlib
import json

def make_test_signature(payload, secret, timestamp=None):
    timestamp = timestamp or int(time.time())
    signed = f"{timestamp}.{payload}"
    signature = hmac.new(secret.encode(), signed.encode(), hashlib.sha256).hexdigest()
    return f"t={timestamp},v1={signature}"

def test_webhook_handler():
    payload = json.dumps({"id": "evt_test", "type": "charge.succeeded"})
    sig = make_test_signature(payload, "whsec_test")
    event = stripe.Webhook.construct_event(payload, sig, "whsec_test")
    assert event.type == "charge.succeeded"
```

## Idempotency keys

If your code uses idempotency keys, test that two calls with the same key are deduplicated. The mock won't actually dedupe, but you can check that the SDK sent the header:

```python
@responses.activate
def test_idempotency_key_sent(load_fixture):
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json=load_fixture("charge_succeeded"),
        status=200,
    )
    stripe.api_key = "sk_test_fake"
    stripe.Charge.create(
        amount=2000, currency="usd", source="tok_visa",
        idempotency_key="order_12345",
    )
    assert responses.calls[0].request.headers["Idempotency-Key"] == "order_12345"
```

## Common pitfall: testing the wrong URL

Stripe SDK calls go to `api.stripe.com/v1/...`. If you mock `stripe.com/...` (without `api.`) or `/charges` (without `/v1/`), the mock won't match and the SDK will try the real network. Set `responses.assert_all_requests_are_fired = True` so missed mocks cause test failures, not silent network requests.

```python
@responses.activate
def test_with_strict_assertions():
    responses.add(responses.POST, "https://api.stripe.com/v1/charges", json={...})
    # ... test code ...
    # After test, responses checks every registered mock was actually called.
```
