---
name: Search and read the Imgur public gallery
description: Search Imgur's public gallery, page through a section, and read an item's comments and tags without any user login.
api: Imgur API
base_url: https://api.imgur.com/3/
endpoints:
  - GET /3/gallery/search/{sort}/{window}/{page}
  - GET /3/gallery/{section}/{sort}/{window}/{page}
  - GET /3/gallery/image/{galleryImageHash}
  - GET /3/gallery/album/{galleryHash}
  - GET /3/gallery/{galleryHash}/comments/{commentSort}
  - GET /3/gallery/{galleryHash}/tags
  - GET /3/tags
  - GET /3/credits
scopes: []
generated: '2026-08-01'
method: generated
source: postman/medialab-ai-imgur-api.postman_collection.json
---

# Search and read the Imgur public gallery

Read-only. No user login is needed — a registered application's Client-ID is enough
for every endpoint in this flow.

## Authentication

```
GET /3/gallery/search/top/week/0?q=cats HTTP/1.1
Host: api.imgur.com
Authorization: Client-ID YOUR_CLIENT_ID
```

Register at <https://api.imgur.com/oauth2/addclient>. Registration is required even
for anonymous read access — the Client-ID is how Imgur knows which application is
calling, and it is what your rate-limit credits are charged against.

## Steps

1. **Search** — `GET /3/gallery/search/{sort}/{window}/{page}` with a `q` query
   parameter. `{window}` is the time window; `{page}` starts at 0.
2. **Or browse a section** — `GET /3/gallery/{section}/{sort}/{window}/{page}`. It
   also accepts `showViral`, `mature` and `album_previews` query parameters.
   Subreddit-sourced galleries are at `GET /3/gallery/r/{subreddit}/{sort}/{window}/{page}`
   and tag galleries at `GET /3/gallery/t/{tagName}/{sort}/{window}/{page}`.
3. **Resolve an item** — gallery items are either images or albums:
   `GET /3/gallery/image/{galleryImageHash}` or `GET /3/gallery/album/{galleryHash}`.
4. **Read the discussion** — `GET /3/gallery/{galleryHash}/comments/{commentSort}`,
   a single comment with `GET /3/gallery/{galleryHash}/comment/{commentId}`, and reply
   threads with `GET /3/comment/{commentId}/replies`.
5. **Read the tags** — `GET /3/gallery/{galleryHash}/tags` for one item,
   `GET /3/tags` for the default tag set, and
   `GET /3/gallery/tag_info/{tagName}` for a tag's metadata.

## Paging

Plural actions page via query string:

- `page` — page number of the result set (default `0`)
- `perPage` — results per page (default `50`, max `100`)

**Exception: `/gallery` endpoints do not support `perPage`** — the path `{page}`
segment is your only control there. `/album/{id}/images` is not paged at all.

## Cache with ETags — it is free and it is the point

Imgur returns an `ETag` on responses. Save it, and send it back on the next request
to the same route as `If-None-Match: "a695f4e9672bf7fc7a779ac12ead684d72292506"` —
**the quotation marks are part of the value**. If nothing changed you get `304 Not
Modified` with no body.

Note the trap: conditional requests that return 304 **still count towards your rate
limits.** ETags save bandwidth and parse time, not credits.

## Budget your credits

Read calls cost 1 credit; an app gets roughly 12,500 requests per day. Watch
`X-RateLimit-ClientRemaining` on every response and poll `GET /3/credits` for a direct
reading. Hitting the daily cap five times in one month blocks the app for the rest of
that month, so treat the ceiling as hard.

Per-IP user limits (`X-RateLimit-UserLimit` / `-UserRemaining` / `-UserReset`) cut a
single user off for an hour rather than blocking the app.

## Response shape

```json
{ "data": [ ... ], "status": 200, "success": true }
```

Append `.json` (default), `.xml`, or `.json?callback=fn` to choose the format. Errors
use the same envelope with `success: false`. Imgur publishes no error-code registry —
branch on HTTP status and `success`, not on a body message.

## Status

API status is published at <https://status.imgur.com/>. Check it before assuming a
5xx is your fault.
