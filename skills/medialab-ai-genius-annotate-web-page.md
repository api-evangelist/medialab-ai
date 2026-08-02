---
name: Annotate a public web page with Genius
description: Create, update and delete a Genius annotation on a public web page, using user-authorized OAuth scopes.
api: Genius API
base_url: https://api.genius.com/
endpoints:
  - GET /web_pages/lookup
  - POST /annotations
  - PUT /annotations/:id
  - DELETE /annotations/:id
scopes:
  - create_annotation
  - manage_annotation
generated: '2026-08-01'
method: generated
source: https://docs.genius.com/
---

# Annotate a public web page with Genius

This flow **writes**, so it requires a user-authorized OAuth 2.0 access token — a
client access token will not work. `POST /annotations` requires the
`create_annotation` scope; `PUT` and `DELETE` require `manage_annotation`.

## Get a user token

Use the authorization code flow. Do **not** use the `token` response type unless you
have no server or native platform — Genius warns it is "much less secure than the full
code exchange process".

1. Send the user to:

   ```
   https://api.genius.com/oauth/authorize?
     client_id=YOUR_CLIENT_ID&
     redirect_uri=YOUR_REDIRECT_URI&
     scope=create_annotation%20manage_annotation&
     state=SOME_STATE_VALUE&
     response_type=code
   ```

   Scopes are space-separated. Always send a hard-to-guess `state` — it is round-tripped
   back to you and is what prevents forged redirects.

2. Exchange the returned `code` by POSTing to `https://api.genius.com/oauth/token`
   with `code`, `client_id`, `client_secret`, `redirect_uri`, `response_type=code`
   and `grant_type=authorization_code`. The response is `{"access_token": "..."}`.

3. Send it on every call as `Authorization: Bearer ACCESS_TOKEN`.

## Steps

1. **Check whether Genius already knows the page** — `GET /web_pages/lookup`, passing
   as many of `raw_annotatable_url`, `canonical_url` and `og_url` as you have. Data is
   only available for pages that already have at least one annotation, so a miss here
   is normal for a first annotation.
2. **Create the annotation** — `POST /annotations` (scope `create_annotation`). The
   payload has three parts:
   - `annotation.body.markdown` — **required**, the text of the note, in markdown.
   - `referent.raw_annotatable_url` — **required**, the original URL of the page.
   - `referent.fragment` — **required**, the highlighted fragment.
   - `referent.context_for_display.before_html` / `.after_html` — the HTML either side
     of the fragment; prefer up to 200 characters each.
   - `web_page` — at least one of `canonical_url`, `og_url`, `title`. Supplying the
     canonical and og URLs is what makes the new annotation appear on the correct page.

   The return value is the new annotation object, in the same form
   `GET /annotations/:id` would return.
3. **Update it** — `PUT /annotations/:id` (scope `manage_annotation`) accepts the same
   parameters as `POST /annotations`. It only works on annotations created by the
   authenticated user.
4. **Delete it** — `DELETE /annotations/:id` (scope `manage_annotation`), again only
   for annotations created by the authenticated user.

## Voting is a separate scope

`PUT /annotations/:id/upvote`, `/downvote` and `/unvote` require the `vote` scope, not
`manage_annotation`. Request `vote` in the authorization URL if you need them.

## Displaying annotations back on the page

Genius publishes a companion embed. Adding

```html
<script src="https://genius.codes"></script>
```

to the page makes it annotatable and renders the annotations you created on it. See
`components/medialab-ai-components.yml`.

## Errors and retries

The response envelope is `{"meta": {"status": ..., "message": ...}}`. On a failure
`meta.message` carries the detail string.

**There is no idempotency key on the Genius API.** A retried `POST /annotations` after
a timeout can create a duplicate annotation. Before retrying, call `GET /referents`
with the `web_page_id` (or `created_by_id`) to check whether the first attempt landed.
