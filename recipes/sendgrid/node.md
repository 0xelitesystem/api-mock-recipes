# SendGrid: Node mock recipe

The `@sendgrid/mail` package uses `axios`. `nock` intercepts cleanly.

## Install

```bash
npm install --save-dev jest nock @sendgrid/mail
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => nock.disableNetConnect());
afterEach(() => nock.cleanAll());
afterAll(() => nock.enableNetConnect());
```

## Successful send

```javascript
const nock = require("nock");
const sgMail = require("@sendgrid/mail");

test("send email success", async () => {
  let capturedBody;
  nock("https://api.sendgrid.com")
    .post("/v3/mail/send", (body) => {
      capturedBody = body;
      return true;
    })
    .reply(202, "", { "X-Message-Id": "abc-123" });

  sgMail.setApiKey("SG.test_fake");
  const [response] = await sgMail.send({
    to: "to@example.com",
    from: "from@example.com",
    subject: "Test",
    text: "Hello",
  });

  expect(response.statusCode).toBe(202);
  expect(capturedBody.personalizations[0].to[0].email).toBe("to@example.com");
  expect(capturedBody.from.email).toBe("from@example.com");
});
```

The `body` callback in `.post()` returns true to match. Set the captured body to a variable; assert on it after.

## Send to multiple recipients

```javascript
test("multiple recipients", async () => {
  let capturedBody;
  nock("https://api.sendgrid.com")
    .post("/v3/mail/send", (body) => { capturedBody = body; return true; })
    .reply(202);

  sgMail.setApiKey("SG.test_fake");
  await sgMail.send({
    to: ["a@example.com", "b@example.com"],
    from: "from@example.com",
    subject: "Test",
    text: "Hi",
  });

  const recipients = capturedBody.personalizations[0].to.map((t) => t.email);
  expect(recipients).toEqual(["a@example.com", "b@example.com"]);
});
```

## Error: unverified sender

```javascript
test("unverified sender 400", async () => {
  nock("https://api.sendgrid.com")
    .post("/v3/mail/send")
    .reply(400, {
      errors: [{
        message: "The from address does not match a verified Sender Identity.",
        field: "from",
      }],
    });

  sgMail.setApiKey("SG.test_fake");
  await expect(
    sgMail.send({
      to: "to@example.com",
      from: "unverified@example.com",
      subject: "Test",
      text: "Hi",
    })
  ).rejects.toMatchObject({
    code: 400,
  });
});
```

## Sandbox mode

To test that your code is using sandbox mode (no real emails sent), assert on the `mail_settings.sandbox_mode.enable` field:

```javascript
test("sandbox mode is enabled in test env", async () => {
  let capturedBody;
  nock("https://api.sendgrid.com")
    .post("/v3/mail/send", (body) => { capturedBody = body; return true; })
    .reply(202);

  sgMail.setApiKey("SG.test_fake");
  await sgMail.send({
    to: "to@example.com",
    from: "from@example.com",
    subject: "Test",
    text: "Hi",
    mailSettings: { sandboxMode: { enable: true } },
  });

  expect(capturedBody.mail_settings.sandbox_mode.enable).toBe(true);
});
```

Sandbox mode is a SendGrid feature where the API validates the request but doesn't send. Use it in staging environments, not just tests.

## Common pitfall: error shape varies

A 400 from SendGrid returns `{errors: [{message, field}]}`. A 401 returns `{errors: [{message: "Permission denied, wrong credentials"}]}` without a `field`. A 429 returns rate-limit headers and an HTML body in some cases. Don't write tests that assume the same error shape for every status code.

## Common pitfall: response body is empty on success

A 202 returns no body, just headers. Code that does `JSON.parse(response.body)` crashes on a successful send. The library handles this; raw HTTP mocks should reply with `""` not `"{}"`.
