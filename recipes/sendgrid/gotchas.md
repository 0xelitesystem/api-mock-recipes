# SendGrid: things that aren't bugs

## 202 means accepted, not delivered

The API returns 202 the moment your request is queued. The actual send happens asynchronously. A 202 response is no guarantee the email was delivered, accepted by the recipient's mail server, or even survived spam filters. Real delivery status arrives via webhooks (`delivered`, `bounce`, `blocked`, `dropped` events). Tests against the send API alone cannot verify delivery.

## Empty response body on success

A successful 202 returns an empty body with headers (`X-Message-Id`, etc.). Code that always parses the response as JSON crashes on a successful send. Both Python and Node SDKs handle this; if you wrote your own HTTP client, you must handle it.

## Bouncing emails don't fail the send

If you send to `bounce-test@example.com` (or any address you know will bounce), SendGrid still returns 202. The bounce reaches you through:

- Activity log in the dashboard
- Webhook event (if configured)
- The bounces API endpoint (`/v3/suppression/bounces`)

There is no synchronous "this won't deliver" response. Tests should not assume bounce detection happens on send.

## Sandbox mode looks like success

When `mail_settings.sandbox_mode.enable` is true, SendGrid validates the request and returns 202 without sending. The response looks identical to a real send. If your test environment uses sandbox mode and your production doesn't, the only difference visible to your code is whether the email actually arrives. Make sure tests assert on the `sandbox_mode` flag in the request, not just the status code.

## API key permissions matter

SendGrid API keys can be scoped (full access, restricted access with specific permissions, or billing-only). A key with only "Mail Send" permission gets 401 on stats endpoints. Tests that mix endpoints need to mock different responses based on key permissions, or test each endpoint separately.

## Rate limits are global, not per-endpoint

The 600/sec limit is across all your API keys. If multiple services in your codebase share a SendGrid account, one rogue service can rate-limit the others. Tests can't simulate this; it's a production observability concern.

## Templates have their own IDs

Dynamic template emails use `template_id` (starts with `d-`) and `dynamic_template_data`. The Mail helpers behave differently when a template_id is set - the `subject` and content fields are ignored in favor of the template. Tests should mock both paths if your code branches on template usage.

## Personalizations array vs flat to/from

The full request body uses a `personalizations` array where each entry has its own `to`, `cc`, `bcc`, `dynamic_template_data`, and `subject`. The simple SDK helpers flatten this for the common case. If your code constructs the body manually (instead of using the helpers), you must structure it as `personalizations[]`. Tests should match the level your code uses.

## Substitutions vs dynamic template data

Older SendGrid templates used `substitutions` (e.g., `-name-`). Newer dynamic templates use `dynamic_template_data` (Handlebars-style `{{name}}`). They are different fields with different shapes. Tests must mock the one your template actually uses.

## Webhook signatures use ECDSA, not HMAC

Unlike Stripe and most other providers, SendGrid signs webhooks with ECDSA over a specific payload format. Verification requires the public key from the dashboard. Don't try to verify SendGrid webhooks with HMAC patterns from other providers; you'll always fail validation.
