---
name: Look up a song and read its Genius annotations
description: Find a song on Genius by search term, then read the crowd-sourced annotations attached to it.
api: Genius API
base_url: https://api.genius.com/
endpoints:
  - GET /search
  - GET /songs/:id
  - GET /referents
  - GET /annotations/:id
scopes: []
generated: '2026-08-01'
method: generated
source: https://docs.genius.com/
---

# Look up a song and read its Genius annotations

This is a read-only flow. It needs **no user authorization** — a client access token
generated on the [Genius API Client management page](https://genius.com/api-clients)
is enough, because none of these endpoints is restricted by a required scope.

## Before you start

- Register an application at <https://genius.com/api-clients> to get a `client_id`,
  `client_secret` and a client access token.
- **Commercial use of the Genius API is not allowed without a license.** Contact
  `api-sales@genius.com` before shipping anything commercial (this includes anything
  with in-app advertising).
- All interaction must be over HTTPS.

## Authentication

Send the token in the `Authorization` header on every request. This is the preferred
form; a query parameter is supported but do not use it.

```
GET /search?q=... HTTP/1.1
Host: api.genius.com
Accept: application/json
Authorization: Bearer YOUR_CLIENT_ACCESS_TOKEN
```

## Steps

1. **Search for the song** — `GET /search` with `q` set to the search term. The
   search capability covers all content hosted on Genius.
2. **Read the song** — take the song `id` from the search hits and call
   `GET /songs/:id`. The response includes details about the document itself and
   information about all the referents attached to it, including the text they refer to.
3. **List the referents** — `GET /referents?song_id=<id>` returns the sections of the
   song that annotations are attached to. Page it with `per_page` and `page`
   (`per_page=5&page=3` returns items 11–15). Pass **only one** of `song_id` and
   `web_page_id` — never both.
4. **Read a specific annotation** — `GET /annotations/:id` returns both the substance
   of the annotation and the information needed to display it in its original context.

## Text formatting

Every one of these endpoints accepts `text_format`, a comma-separated list of one or
more of `dom`, `plain`, `html`. It defaults to `dom`. The response is an object keyed
by the formats you asked for:

- `plain` — plain text, no markup
- `html` — a string of unescaped HTML suitable for rendering by a browser
- `dom` — a nested object representing an HTML DOM hierarchy

Ask for `plain` when you are feeding the text to a model and do not need markup.

## Reading the response

Every Genius response is JSON shaped as:

```json
{ "meta": { "status": 200 }, "response": { ... } }
```

`meta.status` is an integer copy of the HTTP status code. The payload is under
`response`.

## Errors

On a 4xx or 5xx the `meta` object also carries a `message` string:

```json
{ "meta": { "status": 404, "message": "Not found" } }
```

Genius publishes no error-code registry and no `Retry-After` or rate-limit headers —
check `meta.status` rather than assuming a body shape. See
`errors/medialab-ai-problem-types.yml`.

## What this flow will not do

There is no idempotency contract on the Genius API, so do not build retry logic that
assumes replay safety. This flow is read-only, so retries are safe by nature.
