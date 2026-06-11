# Google APIs: things that aren't bugs

## Different APIs use different hosts

There is no single `googleapis.com` host you can mock once and forget. Each API uses its own subdomain:

- Gmail: `gmail.googleapis.com`
- Drive: `www.googleapis.com/drive/v3` (most endpoints), `www.googleapis.com/upload/drive/v3` (uploads)
- Calendar: `www.googleapis.com/calendar/v3`
- Sheets: `sheets.googleapis.com`
- YouTube Data: `www.googleapis.com/youtube/v3`
- YouTube Analytics: `youtubeanalytics.googleapis.com`

Mocks targeting the wrong host fail silently if `disableNetConnect()` isn't set, then hit the real API.

## Pagination tokens are opaque strings

Don't construct test page tokens that look like base64 or JSON. Use a simple string like `"tok123"` so it's obvious in test output. Production code should never parse or inspect page tokens; treat them as opaque.

## Drive file lists default to 100 items, max 1000

If your code does `drive.files.list({})` without `pageSize`, the response includes up to 100 files. Mocking a single response with 500 items will make tests pass but production will paginate differently. Set `pageSize` explicitly and mock matching page sizes.

## Gmail message bodies are base64url, not base64

Gmail returns email body content as base64url encoding (`-` and `_` instead of `+` and `/`). Decoding with standard base64 fails on certain payloads. Use `base64.urlsafe_b64decode` in Python or replace characters in Node before decoding.

## OAuth tokens expire mid-session

If your code holds a long-running operation (e.g., processing 1000 files), the OAuth token can expire mid-loop. The SDK auto-refreshes with a refresh token but adds a request to the OAuth endpoint. Mocks for long flows need either:
- A future `expiry_date` so refresh isn't triggered
- A mock for `https://oauth2.googleapis.com/token` returning a new access token

## 403 has many meanings

A 403 from Google APIs can mean:
- Quota exceeded (`userRateLimitExceeded`, `quotaExceeded`)
- Insufficient OAuth scopes
- Permission denied on the specific resource
- API not enabled in the GCP project

Each requires a different retry strategy. The `error.errors[0].reason` field distinguishes them. Mocks that return generic 403s without the `reason` field hide bugs in retry logic.

## Batch endpoint is multipart, not JSON

`https://www.googleapis.com/batch/{api}/{version}` accepts a multipart body and returns multipart. Mocking JSON for batch endpoints fails to parse. Skip batch in unit tests; cover it in integration tests against the real API.

## Service account vs OAuth tokens

Service account auth (server-to-server) uses a JWT exchange at the token endpoint. OAuth (user) uses a different flow. They produce different access token shapes and have different scope behavior. Mocks should match what your code uses.

## `mimeType` for Google-native files

Google Docs, Sheets, and Slides have mimeTypes like `application/vnd.google-apps.document`. They cannot be downloaded directly; you must export them with `drive.files.export()`. Tests that assume `.download()` works on any file will pass with mocks (since the mock returns whatever bytes you give it) and fail in production.

## Field masks affect response shape

Most Google APIs accept a `fields` parameter that restricts which fields the response includes. If your code uses `fields: "id,name"`, the response only has those two fields. Mocking a full response means code that doesn't handle missing fields passes tests but breaks in production. Match your mock to the field mask.
