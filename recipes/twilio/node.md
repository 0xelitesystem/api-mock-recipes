# Twilio: Node mock recipe

The `twilio` Node package uses `axios`. `nock` works cleanly.

## Install

```bash
npm install --save-dev jest nock twilio
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => nock.disableNetConnect());
afterEach(() => nock.cleanAll());
afterAll(() => nock.enableNetConnect());
```

## Send an SMS

```javascript
const nock = require("nock");
const Twilio = require("twilio");

const ACCOUNT_SID = "ACtest1234567890abcdef1234567890abcdef";
const AUTH_TOKEN = "test_auth_token";

test("send SMS", async () => {
  nock("https://api.twilio.com")
    .post(`/2010-04-01/Accounts/${ACCOUNT_SID}/Messages.json`)
    .reply(201, {
      sid: "SMtest1234567890",
      account_sid: ACCOUNT_SID,
      from: "+15551234567",
      to: "+15559999999",
      body: "Hello from a test",
      status: "queued",
      direction: "outbound-api",
      num_segments: "1",
      price: null,
      price_unit: "USD",
      error_code: null,
      error_message: null,
      uri: `/2010-04-01/Accounts/${ACCOUNT_SID}/Messages/SMtest1234567890.json`,
    });

  const client = Twilio(ACCOUNT_SID, AUTH_TOKEN);
  const message = await client.messages.create({
    body: "Hello from a test",
    from: "+15551234567",
    to: "+15559999999",
  });

  expect(message.sid).toBe("SMtest1234567890");
  expect(message.status).toBe("queued");
});
```

## Inspect the request body

Twilio expects form-encoded bodies. With `nock`, intercept the body in the callback:

```javascript
const querystring = require("querystring");

test("request is form-encoded", async () => {
  let capturedBody;
  nock("https://api.twilio.com")
    .post(`/2010-04-01/Accounts/${ACCOUNT_SID}/Messages.json`, (body) => {
      capturedBody = body;
      return true;
    })
    .reply(201, { sid: "SMtest", status: "queued" });

  const client = Twilio(ACCOUNT_SID, AUTH_TOKEN);
  await client.messages.create({
    body: "Hi",
    from: "+15551234567",
    to: "+15559999999",
  });

  const parsed = querystring.parse(capturedBody);
  expect(parsed.Body).toBe("Hi");
  expect(parsed.From).toBe("+15551234567");
  expect(parsed.To).toBe("+15559999999");
});
```

`nock`'s body callback receives the form-encoded string. Field names are PascalCase (`Body`, `From`, `To`).

## Error: invalid phone number

```javascript
test("invalid phone number", async () => {
  nock("https://api.twilio.com")
    .post(`/2010-04-01/Accounts/${ACCOUNT_SID}/Messages.json`)
    .reply(400, {
      code: 21211,
      message: "The 'To' number +invalid is not a valid phone number.",
      more_info: "https://www.twilio.com/docs/errors/21211",
      status: 400,
    });

  const client = Twilio(ACCOUNT_SID, AUTH_TOKEN);
  await expect(
    client.messages.create({
      body: "Hi",
      from: "+15551234567",
      to: "+invalid",
    })
  ).rejects.toMatchObject({
    code: 21211,
    status: 400,
  });
});
```

## Make a voice call

```javascript
test("make voice call", async () => {
  nock("https://api.twilio.com")
    .post(`/2010-04-01/Accounts/${ACCOUNT_SID}/Calls.json`)
    .reply(201, {
      sid: "CAtest1234567890",
      account_sid: ACCOUNT_SID,
      from: "+15551234567",
      to: "+15559999999",
      status: "queued",
      direction: "outbound-api",
      duration: null,
      price: null,
      uri: `/2010-04-01/Accounts/${ACCOUNT_SID}/Calls/CAtest1234567890.json`,
    });

  const client = Twilio(ACCOUNT_SID, AUTH_TOKEN);
  const call = await client.calls.create({
    to: "+15559999999",
    from: "+15551234567",
    url: "http://demo.twilio.com/docs/voice.xml",
  });
  expect(call.sid).toBe("CAtest1234567890");
});
```

## Verify webhook signatures

For testing inbound webhooks, construct a valid signature using Twilio's RequestValidator:

```javascript
const Twilio = require("twilio");

test("validates incoming webhook signature", () => {
  const validator = new Twilio.RequestValidator(AUTH_TOKEN);
  const url = "https://your-app.example.com/twilio/webhook";
  const params = { From: "+15551234567", To: "+15559999999", Body: "Hi" };

  const signature = validator.getExpectedTwilioSignature(url, params);
  const isValid = validator.validate(url, params, signature);

  expect(isValid).toBe(true);
});
```

## Common pitfall: tracking the account SID across mocks

Every Twilio resource URL contains the account SID. Tests that set up multiple mocks must use the same SID throughout. Easy mistake: copy-pasting fixtures from real responses that use the real account SID.
