---
name: entergy-read-newsroom-content
description: >-
  Read Entergy's published corporate and storm-response content from entergy.com's public
  WordPress REST API, with correct pagination and a realistic picture of what is and is not
  available anonymously.
generated: '2026-09-06'
method: generated
source: openapi/entergy-wordpress-rest-openapi.yml
api: entergy-wordpress-rest
base_url: https://www.entergy.com/wp-json
operations:
  - get_wp_v2_posts
  - get_wp_v2_posts_id
  - get_wp_v2_categories
  - get_wp_v2_search
auth: none
---

# Read Entergy newsroom content

Entergy has no developer program. The one API you can call as an outsider is the WordPress
REST API behind entergy.com, which serves the same news, storm updates and corporate pages a
human reads on the site. Use it for monitoring Entergy communications; do not expect outage
data, meter data or grid data here.

## Before you start

- Base URL: `https://www.entergy.com/wp-json`
- No key, no token, no sign-up. Public read routes answer anonymously.
- Send a normal browser `User-Agent`. The HTML site sits behind a Cloudflare managed
  challenge and a bare crawler UA gets `403`. The `/wp-json/` routes are friendlier, but the
  edge is the same edge.
- There is **no published rate limit and no `Retry-After`**. Throttling, when it happens,
  arrives as an edge challenge page rather than a `429`. Pace yourself and cache.

## 1. List recent posts

`get_wp_v2_posts` — `GET /wp/v2/posts?per_page=20&page=1`

Paginate with `page` and `per_page` (max 100). Read the totals off the response headers, not
the body:

- `X-WP-Total` — total matching items (5,958 at the time of writing)
- `X-WP-TotalPages` — total pages for your `per_page`
- `Link: <...>; rel="next"` — the RFC 8288 cursor to follow

Trim the payload with `_fields=id,date,slug,title,link,categories` — the full post object
carries rendered HTML you probably do not want.

## 2. Fetch one post

`get_wp_v2_posts_id` — `GET /wp/v2/posts/{id}`

Never guess ids. `GET /wp/v2/posts/1` returns `404 rest_post_invalid_id`; resolve ids from the
collection route first.

## 3. Narrow by category

`get_wp_v2_categories` — `GET /wp/v2/categories?per_page=100`, then filter posts with
`GET /wp/v2/posts?categories=<id>`.

## 4. Search

`get_wp_v2_search` — `GET /wp/v2/search?search=<terms>&subtype=post`

## Errors you will actually see

The envelope is WordPress's, not RFC 9457:

```json
{ "code": "rest_forbidden", "message": "Sorry, you are not allowed to do that.", "data": { "status": 401 } }
```

- `rest_no_route` / 404 — wrong path **or** wrong method on a right path.
- `rest_post_invalid_id` / 404 — the id does not exist or is not public.
- `rest_forbidden` / 401 and `rest_cannot_access` / 401 — the route needs credentials.
  Entergy does not issue credentials to third parties, so treat these as **terminal, not
  retryable**. `/wp/v2/users` and `/wp/v2/settings` are both in this class.

## What is not here

- **No writes.** Every mutating route needs a WordPress Application Password on an Entergy
  CMS account. There is no idempotency mechanism on this API, so do not build a retry loop
  that could ever reach a write.
- **No outage data.** The outage map at `entergy.com/viewoutages` is a DataCapable
  application on a separate domain; nothing about it is Entergy's API.
- **No meter or usage data.** Customer interval data moves through Green Button Connect My
  Data (NAESB REQ.21 ESPI) for Entergy Texas, and that requires vendor registration at
  `myentergyadvisor.entergy.com/greenbutton/green-vendor`.
- **No GIS data.** `gis.entergy.com` runs ArcGIS Server 10.7.1, and every service requires a
  token Entergy does not hand out. `/arcgis/rest/services` answers `200` with
  `{"error":{"code":499,"message":"Token Required"}}` — branch on the body, not the status.
