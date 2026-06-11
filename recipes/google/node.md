# Google APIs: Node mock recipe

The `googleapis` Node package uses `gaxios` (Google's `axios` fork) underneath. `nock` intercepts at the HTTP layer and works for any Google API endpoint.

## Install

```bash
npm install --save-dev jest nock googleapis
```

## Setup

```javascript
// tests/setup.js
const nock = require("nock");

beforeAll(() => nock.disableNetConnect());
afterEach(() => nock.cleanAll());
afterAll(() => nock.enableNetConnect());
```

## Auth without real OAuth

Pass a pre-built auth object so the SDK doesn't try to fetch tokens:

```javascript
const { google } = require("googleapis");

function makeAuth() {
  const auth = new google.auth.OAuth2();
  auth.setCredentials({
    access_token: "fake-token",
    refresh_token: "fake-refresh",
    expiry_date: Date.now() + 3600_000,
  });
  return auth;
}
```

`expiry_date` in the future prevents the SDK from trying to refresh the token mid-test.

## Get a Gmail message

```javascript
const nock = require("nock");
const { google } = require("googleapis");
const messageFixture = require("./fixtures/gmail_message.json");

test("get gmail message", async () => {
  nock("https://gmail.googleapis.com")
    .get("/gmail/v1/users/me/messages/abc123")
    .reply(200, messageFixture);

  const gmail = google.gmail({ version: "v1", auth: makeAuth() });
  const response = await gmail.users.messages.get({ userId: "me", id: "abc123" });

  expect(response.data.id).toBe("18b1f5e1a2b3c4d5");
});
```

## List with pagination

```javascript
test("list with pagination", async () => {
  nock("https://gmail.googleapis.com")
    .get("/gmail/v1/users/me/messages")
    .query(true)  // any query string
    .reply(200, { messages: [{ id: "m1" }, { id: "m2" }], nextPageToken: "tok123" });

  nock("https://gmail.googleapis.com")
    .get("/gmail/v1/users/me/messages")
    .query({ pageToken: "tok123" })
    .reply(200, { messages: [{ id: "m3" }] });

  const gmail = google.gmail({ version: "v1", auth: makeAuth() });
  const allIds = [];
  let pageToken;
  do {
    const res = await gmail.users.messages.list({ userId: "me", pageToken });
    allIds.push(...(res.data.messages || []).map((m) => m.id));
    pageToken = res.data.nextPageToken;
  } while (pageToken);

  expect(allIds).toEqual(["m1", "m2", "m3"]);
});
```

## Drive file metadata

```javascript
test("get drive file metadata", async () => {
  const fileFixture = require("./fixtures/drive_file_metadata.json");
  nock("https://www.googleapis.com")
    .get("/drive/v3/files/1abc")
    .query(true)
    .reply(200, fileFixture);

  const drive = google.drive({ version: "v3", auth: makeAuth() });
  const response = await drive.files.get({ fileId: "1abc" });
  expect(response.data.name).toBe("Quarterly Report.pdf");
});
```

## Auth errors (401)

```javascript
test("invalid credentials", async () => {
  nock("https://gmail.googleapis.com")
    .get("/gmail/v1/users/me/messages/abc")
    .reply(401, {
      error: {
        code: 401,
        message: "Request had invalid authentication credentials.",
        status: "UNAUTHENTICATED",
      },
    });

  const gmail = google.gmail({ version: "v1", auth: makeAuth() });
  await expect(
    gmail.users.messages.get({ userId: "me", id: "abc" })
  ).rejects.toMatchObject({ code: 401 });
});
```

## Batch endpoint

Google batch requests POST a multipart body to `/batch/{api}/v1`. The response is also multipart. Most code doesn't use batch; if yours does, mock with `.post("/batch/...")` and a multipart-encoded string response, not JSON.

## Common pitfall: API host varies by service

- Gmail: `https://gmail.googleapis.com`
- Drive (most endpoints): `https://www.googleapis.com`
- Drive (upload endpoint): `https://www.googleapis.com/upload/drive/v3/files`
- Calendar: `https://www.googleapis.com/calendar/v3`
- YouTube: `https://www.googleapis.com/youtube/v3`

Mocking the wrong host means the request falls through to the real network. Check the request URL in the SDK source if you're unsure.

## Common pitfall: OAuth refresh during the test

If `expiry_date` is in the past, the SDK tries to refresh the token at the OAuth endpoint. If you haven't mocked that, the test fails with a confusing token error. Either pre-set a future `expiry_date` or mock `https://oauth2.googleapis.com/token` to return a refreshed token.
