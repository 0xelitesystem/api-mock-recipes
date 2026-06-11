# Contributing

New recipes and corrections welcome. Here's how to add one well.

## Adding a new provider

Create `recipes/{provider}/` with:

- `python.md`: Python mocking recipe
- `node.md`: Node mocking recipe
- `gotchas.md`: provider-specific quirks that aren't really mocking problems
- `fixtures/`: sample JSON response payloads

Each recipe should cover at minimum:

1. The successful happy-path call
2. One authentication error (401)
3. One rate-limit case (429) with retry semantics
4. One validation error (400)
5. The "inspect what we actually sent" assertion pattern
6. A common pitfall section at the bottom

Use real fixture data captured from sandbox APIs. Strip any real account IDs, then commit. A fixture made up from scratch usually misses fields the SDK depends on.

## Adding to an existing provider

Open a PR adding a new test case. Match the existing style: code block, brief explanation, no marketing voice.

If you find a gotcha that's not documented, add it to `gotchas.md`. Keep entries to 2-5 sentences. The point is to surface the surprise, not to write a tutorial.

## Style

- Direct, technical voice. "Use X" not "we recommend X"
- Working code. Every code block should run with copy-paste plus the listed install command
- Pinned to current SDK versions where the API has changed (e.g., `openai` v1+, `anthropic` post-2024)
- Real-world failure modes only. Skip theoretical errors that don't happen in production
- No marketing about Anthropic, OpenAI, Stripe, etc. Just the technical content

## Testing your changes

Before opening a PR, run the code in your example against the relevant real sandbox to confirm the request/response shapes are still accurate. SDK versions change. APIs add fields. If a fixture is more than a year old, it might already be wrong.

## What doesn't belong

- Mock libraries that aren't already widely used (`responses`, `nock`, `msw`, `httpx.MockTransport`, `pytest-httpx`)
- Recipes for SaaS that doesn't have a public sandbox to verify against
- Code that depends on a specific test framework's matchers (use plain assertions where possible)
- Generic test setup guides not specific to mocking external APIs
