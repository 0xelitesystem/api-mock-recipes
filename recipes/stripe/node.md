# Stripe: Node mock recipe

Two solid choices: `nock` for low-level HTTP intercepts, or `msw` if you want the same mocks shared between tests and browser dev. For Node-only test suites, `nock` is the lighter pick.

## Install

```bash
npm install --save-dev jest nock stripe
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => {
  nock.disableNetConnect();
});

afterEach(() => {
  nock.cleanAll();
});

afterAll(() => {
  nock.enableNetConnect();
});
```

Add to `jest.config.js`:

```javascript
module.exports = {
  setupFilesAfterEach: ["<rootDir>/tests/setup.js"],
};
```

`disableNetConnect` is the most important line. Without it, missed mocks fall through to the real network. With it, missed mocks throw a clear error.

## Successful charge

```javascript
const nock = require("nock");
const Stripe = require("stripe");
const chargeFixture = require("./fixtures/charge_succeeded.json");

test("creates a charge", async () => {
  nock("https://api.stripe.com")
    .post("/v1/charges")
    .reply(200, chargeFixture);

  const stripe = Stripe("sk_test_fake");
  const charge = await stripe.charges.create({
    amount: 2000,
    currency: "usd",
    source: "tok_visa",
  });

  expect(charge.id).toMatch(/^ch_/);
  expect(charge.status).toBe("succeeded");
});
```

## Card declined

```javascript
test("handles card declined", async () => {
  nock("https://api.stripe.com")
    .post("/v1/charges")
    .reply(402, {
      error: {
        type: "card_error",
        code: "card_declined",
        message: "Your card was declined.",
      },
    });

  const stripe = Stripe("sk_test_fake");
  await expect(
    stripe.charges.create({ amount: 2000, currency: "usd", source: "tok_chargeDeclined" })
  ).rejects.toMatchObject({
    type: "StripeCardError",
    code: "card_declined",
  });
});
```

The Stripe Node SDK throws a `StripeCardError` for 402 + `card_error`. If your application code checks `err.type === "StripeCardError"`, you're testing the same path.

## Rate limit with retry

```javascript
test("retries on rate limit", async () => {
  nock("https://api.stripe.com")
    .post("/v1/charges")
    .reply(429, { error: { type: "rate_limit_error" } }, { "Retry-After": "1" });

  nock("https://api.stripe.com")
    .post("/v1/charges")
    .reply(200, chargeFixture);

  const stripe = Stripe("sk_test_fake", { maxNetworkRetries: 2 });
  const charge = await stripe.charges.create({ amount: 2000, currency: "usd", source: "tok_visa" });

  expect(charge.status).toBe("succeeded");
});
```

## Asserting headers

`nock` lets you assert on the request:

```javascript
test("sends idempotency key", async () => {
  let capturedHeaders;
  nock("https://api.stripe.com")
    .post("/v1/charges")
    .reply(function (uri, body) {
      capturedHeaders = this.req.headers;
      return [200, chargeFixture];
    });

  const stripe = Stripe("sk_test_fake");
  await stripe.charges.create(
    { amount: 2000, currency: "usd", source: "tok_visa" },
    { idempotencyKey: "order_12345" }
  );

  expect(capturedHeaders["idempotency-key"]).toBe("order_12345");
});
```

## Webhook construction (different problem)

For testing webhook handlers, construct a real signed payload:

```javascript
const Stripe = require("stripe");
const crypto = require("crypto");

function makeTestSignature(payload, secret, timestamp = Math.floor(Date.now() / 1000)) {
  const signedPayload = `${timestamp}.${payload}`;
  const signature = crypto.createHmac("sha256", secret).update(signedPayload).digest("hex");
  return `t=${timestamp},v1=${signature}`;
}

test("verifies a webhook", () => {
  const payload = JSON.stringify({ id: "evt_test", type: "charge.succeeded" });
  const sig = makeTestSignature(payload, "whsec_test");
  const stripe = Stripe("sk_test_fake");
  const event = stripe.webhooks.constructEvent(payload, sig, "whsec_test");
  expect(event.type).toBe("charge.succeeded");
});
```

## Common pitfall: `nock` mocks are one-shot

By default, each `nock(...).post(...).reply(...)` matches one request. Two requests to the same endpoint need two `nock` calls or a `.persist()` call. Tests that mock once but make two requests will fail confusingly.

```javascript
// matches three times
nock("https://api.stripe.com").post("/v1/charges").times(3).reply(200, fixture);

// matches forever (use sparingly)
nock("https://api.stripe.com").post("/v1/charges").persist().reply(200, fixture);
```
