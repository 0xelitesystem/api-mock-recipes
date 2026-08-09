# api-mock-recipes

Working code for mocking the API calls in your tests. Stripe, OpenAI, Anthropic, Google APIs, SendGrid, Twilio. Python and Node.

The recipes don't depend on heavy frameworks. They use the standard testing tools in each language: `pytest` + `responses` or `httpx.MockTransport` for Python, `nock` or `msw` for Node. Adapt to your setup.

## Why mock at all

The case for mocking external APIs in tests:

- Real API calls in tests are slow (network), flaky (network), and cost money (token-based APIs).
- Real API calls don't reproducibly trigger edge cases (rate limits, 5xx errors, malformed responses).
- Some APIs don't have a free test mode at all.

The case against:

- A mock that doesn't match the real API's behavior is worse than no mock - it gives false confidence.
- Mocks drift. The real API adds fields, changes error shapes, deprecates endpoints. Your mocks don't notice.

The middle ground used here: mock the API for unit tests, but keep one integration test per provider that hits the real sandbox. Run the integration tests in CI nightly, not on every PR.

## Recipes

| Provider | Recipe | Languages |
|----------|--------|-----------|
| Stripe | [`recipes/stripe/`](recipes/stripe/) | Python, Node |
| OpenAI | [`recipes/openai/`](recipes/openai/) | Python, Node |
| Anthropic | [`recipes/anthropic/`](recipes/anthropic/) | Python, Node |
| Google (Gmail, Drive) | [`recipes/google/`](recipes/google/) | Python, Node |
| SendGrid | [`recipes/sendgrid/`](recipes/sendgrid/) | Python, Node |
| Twilio | [`recipes/twilio/`](recipes/twilio/) | Python, Node |

Each provider folder contains:

- `python.md` - pytest fixtures and example tests using `responses` or `httpx.MockTransport`
- `node.md` - Jest/Vitest setup using `nock` or `msw`
- `fixtures/` - sample response payloads (success and error cases)
- `gotchas.md` - things that look like mocking issues but are actually real API quirks

## Common patterns across recipes

### Pattern: capture real responses once

The fastest way to get a faithful mock is to record real responses from the sandbox API, then replay them:

```bash
curl -X POST https://api.provider.com/v1/endpoint \
  -H "Authorization: Bearer $TEST_KEY" \
  -d '{"key":"value"}' \
  > fixtures/endpoint_success.json
```

Commit the JSON. Use it as the mock response. When the API changes, re-record.

### Pattern: test failure modes

Each recipe includes fixtures for the failure cases that actually happen in production:

- 401 / 403 (expired or wrong key)
- 429 (rate limit, with `Retry-After` header)
- 400 with provider-specific error JSON shape
- 500 / 502 / 503 (server errors, retryable)
- timeout (no response at all)
- network error (DNS, connection refused)

A test suite that only covers the happy path catches the 5% of bugs and misses the 95%.

### Pattern: don't mock your own client

Mock at the HTTP layer (`responses`, `nock`, `msw`), not at your client library layer. If you mock `my_stripe_client.create_charge()`, you've removed your own code from the test.

### Pattern: separate fixture data from assertions

Bad:

```python
def test_creates_subscription():
    responses.add(POST, "...", json={"id": "sub_123", ...50 lines of stripe payload...})
    result = create_sub("cust_456")
    assert result == {...same 50 lines...}
```

Good:

```python
def test_creates_subscription(stripe_subscription_fixture):
    responses.add(POST, "...", json=stripe_subscription_fixture)
    result = create_sub("cust_456")
    assert result["id"] == "sub_123"
    assert result["status"] == "active"
```

The fixture lives in `fixtures/subscription_created.json` and is reused.

## What this repo is not

- A mocking framework. Use `responses`, `nock`, `msw`. They already exist.
- A test runner. Use `pytest`, `vitest`, `jest`.
- A library. There's nothing to install.

These are copy-paste recipes. Open the file, take what you need.

## Related repos

- [`webhook-inspector`](https://github.com/0xelitesystem/webhook-inspector) - for testing incoming webhooks from these providers
- [`webhook-signature-verifier`](https://github.com/0xelitesystem/webhook-signature-verifier) - signature verification helpers for Stripe, GitHub, SendGrid
- [`rate-limit-tester`](https://github.com/0xelitesystem/rate-limit-tester) - test your own API's rate limit handling
- [`api-changelog-rss`](https://github.com/0xelitesystem/api-changelog-rss) - track API changes that will break your mocks
- [`jwt-inspector`](https://github.com/0xelitesystem/jwt-inspector) - decode JWTs from API responses
- [`env-var-checker`](https://github.com/0xelitesystem/env-var-checker) - check your test environment has the right keys
- [`api-security-audit-checklist`](https://github.com/0xelitesystem/api-security-audit-checklist) - audit your API client code
- [`secrets-scanner-bookmarklet`](https://github.com/0xelitesystem/secrets-scanner-bookmarklet) - find leaked keys in repos
- [`stack-trace-cleaner`](https://github.com/0xelitesystem/stack-trace-cleaner) - clean up stack traces when API mocks fail

## More

Part of a catalog of single-file browser tools and plain-language references, all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/). Built by [elitesystem.ai](https://elitesystem.ai).

## License

MIT. Use the recipes in any project, commercial or personal. No attribution required.
