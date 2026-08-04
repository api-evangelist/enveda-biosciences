---
name: Read the Enveda newsroom
description: Page through Enveda's published news releases and In-Veda blog posts, resolving categories, authors and featured images in a single request.
api: openapi/enveda-biosciences-content-openapi.yml
base_url: https://enveda.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getNews, getNewsById, getNewsCategory, getNewsCategoryById, getPosts, getPostsById, getIssue, getUsers]
generated: '2026-08-04'
method: generated
---

# Read the Enveda newsroom

Enveda keeps company announcements in a WordPress **custom post type** called `news` (34
published), and its editorial writing in the standard `posts` collection under the name
**In-Veda** (8 published). They are two separate collections — reading only `/wp/v2/posts`
misses every press release.

Everything here is anonymous. Do not send credentials.

## 1. List news releases, newest first

```
GET https://enveda.com/wp-json/wp/v2/news?per_page=20&page=1&orderby=date&order=desc
```

`getNews`. Read the pagination from the response headers, never from the body:

- `X-WP-Total` — total releases (34 at harvest)
- `X-WP-TotalPages` — pages at the current `per_page`
- `Link: <...>; rel="next"` — the RFC 8288 next page

`per_page` is capped at **100**; exceeding it returns `400 rest_invalid_param`.

## 2. Trim the payload

News bodies are full HTML and large. Ask for only what you need:

```
GET /wp/v2/news?per_page=20&_fields=id,date,slug,link,title,excerpt,news-category
```

Titles and excerpts come back as `{"rendered": "<html string>"}` and are HTML-escaped
(`Enveda&#8217;s`). Unescape before displaying.

## 3. Resolve categories, author and image in one call

```
GET /wp/v2/news?per_page=10&_embed
```

`_embed` inlines `author`, `wp:featuredmedia` and `wp:term` into an `_embedded` object, so you
do not have to follow `news-category` term ids yourself. If you prefer explicit calls, use
`getNewsCategory` (`/wp/v2/news-category`, 2 terms) and `getUsers`.

## 4. Fetch one release

```
GET /wp/v2/news/1098
```

`getNewsById`. An unknown or unpublished id returns `404 rest_post_invalid_id`.

## 5. Read In-Veda separately

```
GET https://enveda.com/wp-json/wp/v2/posts?per_page=20&_embed
```

`getPosts`. In-Veda posts are classified by the `issue` taxonomy (`getIssue`,
`/wp/v2/issue`) rather than by tags — `/wp/v2/tags` is empty on this site.

## Rules

- **Read-only.** Every `create*`/`update*`/`delete*` operation in the spec requires a WordPress
  Application Password that no third party holds. Do not attempt writes.
- **No retries on 4xx.** Errors use the WordPress envelope
  `{"code": "...", "message": "...", "data": {"status": n}}`, not RFC 9457. `400
  rest_invalid_param` means fix the parameter, not retry. See
  `errors/enveda-biosciences-problem-types.yml`.
- **No idempotency keys exist** on this API — see `conventions/enveda-biosciences-conventions.yml`.
- Responses carry `cache-control: max-age=600`. Respect it; do not poll faster than every
  10 minutes.
- Enveda publishes an `llms.txt` at <https://enveda.com/llms.txt> that indexes the same news and
  blog items as human URLs — use it when you want the page, not the record.
