# Stripe: things that aren't bugs

Behaviors that look like mocking problems but reflect real Stripe API quirks. Worth knowing so you don't waste hours debugging your mocks.

## Idempotency keys live for 24 hours, then expire

If your code retries a failed charge using the same idempotency key the next day, Stripe will treat it as a new request, not a retry. Tests that assume keys "always dedupe" miss this.

In your code: regenerate the key on each business-level retry attempt, not on each network retry. Stripe's SDK retries (`maxNetworkRetries`) use the same key automatically; that's fine within the 24-hour window.

## `tok_visa` is a real test token. `tok_fake` is not.

If your tests build "fake" token strings, the SDK will accept them at the mock layer but fail at the integration-test layer when hitting the real sandbox. Use the documented test tokens (`tok_visa`, `tok_chargeDeclined`, `tok_chargeCustomerFail`, etc.) so the same string works in both.

Full list: https://stripe.com/docs/testing

## Webhook signature timestamps drift

If your webhook handler rejects events older than 5 minutes (good practice), tests that build webhook events with a hardcoded old timestamp will fail. Construct the timestamp inside the test using `int(time.time())`, not a constant.

## The `livemode` field matters

Test-mode responses have `"livemode": false`. If your code branches on `event.livemode` (e.g., to skip non-production webhooks), make sure fixtures match. Production fixtures shouldn't be reused in test mode.

## Amount is in the smallest unit, but currency-dependent

USD: cents. JPY: yen (no smaller unit). When mocking, fixtures need to match. Mocking a 20-dollar charge as `"amount": 20.00` will accept in tests but fail in production - Stripe expects `2000`. Fixture amounts should be integers.

## 402 vs 400 for declines

Card declines return 402. Other errors (bad parameters, missing fields) return 400. The SDK's error type depends on the HTTP code AND the `error.type` in the body. A `card_error` at 402 becomes `CardError`. A `card_error` at 400 becomes `InvalidRequestError`. Get both right in your fixture.

## Object expansions change response shape

If your code uses `expand=["customer", "payment_method"]`, the response includes inline objects instead of IDs. Mocks must match the expansion pattern the real call uses, or your code that accesses `charge.customer.email` will crash on a string ID.

## Pagination has two shapes

Older list endpoints use `{"data": [...], "has_more": true, "url": "..."}`. The newer SDK auto-paginates with `for charge in stripe.Charge.list().auto_paging_iter():` which makes multiple requests. Mock each page separately or limit your test to one page.

## Webhook event IDs don't match charge IDs

A `charge.succeeded` event has an event ID like `evt_...` and contains a `charge` object with id `ch_...`. Mocking `event.id` as `ch_...` is a common mistake that breaks event-deduplication logic.
