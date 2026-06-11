# Twilio: things that aren't bugs

## Account SID lives in the URL path

Every Twilio resource URL embeds the account SID: `/2010-04-01/Accounts/{AccountSid}/Messages.json`. The auth header alone is not enough; the SID must match what's in the URL. Mocking a different SID than the client is configured with means the mock never matches.

## Form-encoded request bodies, not JSON

The Twilio REST API uses `application/x-www-form-urlencoded` for request bodies. Field names are PascalCase (`From`, `To`, `Body`, `Url`). Code that constructs JSON bodies bypasses the SDK and hits the API as form-encoded anyway through `requests` / `axios` form encoding. If you mock expecting JSON, the body matcher fails.

## Initial status is `queued`, never `delivered`

The response to a message create is always `status: "queued"` (or `"accepted"` for messaging services). Actual delivery status (`sent`, `delivered`, `failed`, `undelivered`) arrives via the `statusCallback` webhook after Twilio attempts delivery. There is no way to make the create endpoint return `"delivered"` directly. Tests that check for `delivered` immediately after create are testing the mock, not real behavior.

## Error codes split between 2xxxx and 3xxxx

Twilio error codes follow a pattern:

- `1xxxx`: warnings (some)
- `2xxxx`: client errors returned synchronously in the API response (`21211` invalid number, `21610` blacklisted, etc.)
- `3xxxx`: carrier and delivery errors that arrive via webhooks AFTER an initial success (`30003` unreachable, `30005` unknown destination, etc.)

You cannot test 3xxxx errors against the create endpoint. They arrive on `statusCallback` POSTs to your webhook handler. Reference: https://www.twilio.com/docs/api/errors

## Trial accounts have prefix restrictions

Trial Twilio accounts can only send SMS to verified numbers. Production accounts cannot. If your test fixtures use a trial account's restriction error (`21608`), they won't match production behavior. Use production-style fixtures.

## Magic test phone numbers

Twilio reserves specific numbers for testing:

- `+15005550006`: success
- `+15005550001`: invalid number
- `+15005550009`: blacklisted
- `+15005550008`: queue full

These are intended for use against the real Twilio sandbox, not mocks. They produce deterministic error codes against the real API, which is more useful for integration tests than full mocking.

## Status callback URLs trigger separate webhooks

When you create a message with `statusCallback: "https://you.example.com/twilio"`, Twilio makes POST requests to that URL as the message progresses. These are not part of the create response. To test the callback handler, write a separate test that simulates the inbound POST with the expected form-encoded body.

## Webhook signatures use SHA1-HMAC over a specific concatenation

Twilio signs webhooks with SHA1-HMAC over `URL + sorted(params)`. The signature is in the `X-Twilio-Signature` header. Use Twilio's `RequestValidator` rather than rolling your own; the param sorting and URL-encoding rules have edge cases.

## Messaging services vs phone numbers as `From`

You can send from a `phone_number` (E.164 format) OR a `messaging_service_sid` (starts with `MG`). They are mutually exclusive. Tests that fix `from` to a phone number in one place and `messaging_service_sid` elsewhere conflict.

## Outbound-API direction in the response

The `direction` field is `outbound-api` for messages your code sent, `outbound-call` for calls initiated by your code, and `inbound` for incoming messages received at your numbers. Mocking the wrong direction confuses downstream code that filters by it.

## Subaccounts duplicate everything

If your code uses Twilio subaccounts, each subaccount has its own SID, auth token, phone numbers, and messages. Mocks need to match the subaccount SID, not the master account SID. This is a frequent source of mock-mismatch errors in multi-tenant apps.
