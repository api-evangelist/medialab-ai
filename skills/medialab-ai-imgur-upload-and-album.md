---
name: Upload images to Imgur and collect them in an album
description: Upload one or more images to Imgur (anonymously or to an account), group them into an album, and clean up.
api: Imgur API
base_url: https://api.imgur.com/3/
endpoints:
  - POST /3/image
  - POST /3/album
  - POST /3/album/{albumHash}/add
  - GET /3/album/{albumHash}/images
  - DELETE /3/image/{deleteHash}
scopes: []
generated: '2026-08-01'
method: generated
source: postman/medialab-ai-imgur-api.postman_collection.json
---

# Upload images to Imgur and collect them in an album

Every endpoint here exists in both anonymous and authenticated form. Pick the mode
first — it changes which identifier you must keep.

## Before you start

- Register an application at <https://api.imgur.com/oauth2/addclient> to get a
  `client_id` and `client_secret`. Registration is required **even if you are not
  logging users in.**
- Every call is HTTPS on port 443.
- **Commercial use** (which includes in-app advertising, or any app belonging to a
  commercial organization) must additionally be registered with RapidAPI. The base URL
  then becomes `https://imgur-apiv3.p.rapidapi.com/` in place of
  `https://api.imgur.com/`, and you must set an `X-Mashape-Key` header.

## Authentication

- **Anonymous / public read** — `Authorization: Client-ID YOUR_CLIENT_ID`. This is
  enough to upload an image anonymously and to create an anonymous album.
- **Account-scoped** — `Authorization: Bearer YOUR_ACCESS_TOKEN`. Get the token via
  the implicit flow at `https://api.imgur.com/oauth2/authorize?client_id=...&response_type=token&state=...`
  (`token` is the only supported `response_type` — `code` and `pin` are deprecated).
  Access tokens last about a month; refresh with a `grant_type=refresh_token` POST to
  `https://api.imgur.com/oauth2/token`. Refresh tokens do not expire — store them.

There are **no OAuth scopes** on Imgur. An authorized token has full account access
for that user, so treat it as a high-consequence credential.

## Steps

1. **Upload each image** — `POST /3/image`. Anonymous uploads are not tied to an
   account; authenticated uploads land in the caller's account.
   **Keep the `deleteHash` from the response for every anonymous upload — it is the
   only credential that can ever delete that image.**
   An upload deducts **10 rate-limit credits**, not 1.
2. **Create the album** — `POST /3/album`. Anonymous album creation returns an
   `albumDeleteHash`; authenticated creation returns an `albumHash` owned by the
   account. Keep whichever you get.
3. **Add the images** — `POST /3/album/{albumHash}/add` (authenticated) or
   `POST /3/album/{albumDeleteHash}/add` (anonymous). Use
   `POST /3/album/{albumHash}` to *set* (replace) the whole image list instead of
   appending.
4. **Verify** — `GET /3/album/{albumHash}/images`. Note this endpoint is **not paged**.
5. **Clean up** — `DELETE /3/image/{deleteHash}` for an anonymous image,
   `DELETE /3/album/{albumHash}` (or `{albumDeleteHash}` anonymously) for the album,
   and `POST /3/album/{albumHash}/remove_images` to detach images without deleting them.

## Reading responses

Every response is wrapped:

```json
{ "data": { ... }, "status": 200, "success": true }
```

Response format follows the URL extension — `.json` (default), `.xml`, or `.json` plus
a `callback` parameter for JSONP.

## Rate limits — check them, they are tight

| Header | Meaning |
|---|---|
| `X-RateLimit-ClientLimit` / `-ClientRemaining` | app credits, ~12,500 requests/day |
| `X-RateLimit-UserLimit` / `-UserRemaining` / `-UserReset` | per-IP user credits |
| `X-Post-Rate-Limit-Limit` / `-Remaining` / `-Reset` | 1,250 POSTs per hour per IP |

An app is capped at roughly **1,250 uploads per day**. **If the daily limit is hit five
times in a month, the app is blocked for the rest of the month** — so back off on
`X-RateLimit-ClientRemaining` approaching zero rather than retrying into the wall.
Poll `GET /3/credits` to read current status. OAuth calls cost no credits.

## Retries

**Imgur publishes no idempotency key.** A retried `POST /3/image` after a timeout
creates a *second* image and burns another 10 credits. Before retrying, check
`GET /3/account/{username}/images` (authenticated) or rely on the `deleteHash` you
already captured; do not blind-retry uploads.
