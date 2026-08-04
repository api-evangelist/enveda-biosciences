---
name: Search enveda.com and resolve people
description: Run a cross-content-type search against enveda.com, resolve each result pointer into the full object, and read Enveda's leadership, board and advisor roster.
api: openapi/enveda-biosciences-content-openapi.yml
base_url: https://enveda.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getSearch, getPostsById, getPagesById, getNewsById, getPerson, getPersonById, getPersonCategory, getPersonCategoryById, getTypes, getTypesByType, getMedia, getMediaById]
generated: '2026-08-04'
method: generated
---

# Search enveda.com and resolve people

Two jobs that share one trick: `/wp/v2/search` returns **pointers**, not objects, so every
search result needs a second call to become useful.

Everything here is anonymous and read-only.

## Part 1 — Search

```
GET https://enveda.com/wp-json/wp/v2/search?search=PRISM&per_page=10
```

`getSearch`. A result is five fields:

```json
{
  "id": 144,
  "title": "PRISM: A foundation model for life’s chemistry",
  "url": "https://enveda.com/prism-a-foundation-model-for-lifes-chemistry/",
  "type": "post",
  "subtype": "post"
}
```

There is no body, no date and no author. To get those, branch on `subtype` and call the
matching single-resource operation with the same `id`:

| `subtype`      | operation      | path                    |
|----------------|----------------|-------------------------|
| `post`         | `getPostsById` | `/wp/v2/posts/{id}`     |
| `page`         | `getPagesById` | `/wp/v2/pages/{id}`     |
| `news`         | `getNewsById`  | `/wp/v2/news/{id}`      |
| `person`       | `getPersonById`| `/wp/v2/person/{id}`    |

The result's own `_links.self[0].href` already carries the correct URL — prefer following it
over rebuilding the path.

Narrow the search up front when you know the shape you want:

```
GET /wp/v2/search?search=ENV-294&subtype=news&per_page=20
```

Enumerate valid `type`/`subtype` values with `getTypes` (`/wp/v2/types`) rather than guessing.

## Part 2 — People

Enveda's roster is a custom post type, `person` (31 published) — not `users`. `/wp/v2/users`
returns six CMS editor accounts and is **not** the leadership team.

```
GET /wp/v2/person?per_page=100&_embed&_fields=id,slug,link,title,person-category,featured_media
```

`getPerson`. Then resolve the grouping:

```
GET /wp/v2/person-category?_fields=id,name,slug,count
```

`getPersonCategory` returned these five terms at harvest:

| id | name                  | count |
|----|-----------------------|-------|
| 6  | Leadership            | 15    |
| 8  | Advisors              | 9     |
| 7  | Board                 | 7     |
| 5  | CEO                   | 0     |
| 9  | Investors &amp; Partners  | 0     |

Term names come back HTML-escaped (`Investors &amp; Partners`) — unescape before display.
`CEO` and `Investors & Partners` are empty terms; do not report them as populated.

Headshots: either add `_embed` (image arrives under `_embedded["wp:featuredmedia"]`) or call
`getMediaById` with the `featured_media` id and read `source_url` and `alt_text`.

## Rules

- **Read-only.** All writes need an Application Password no third party holds.
- Titles are `{"rendered": "..."}` objects, HTML-escaped. Never treat them as plain strings.
- Do not attribute a role to a person beyond the `person-category` term the API returns, and do
  not infer contact details — the API exposes none.
- Empty result sets are normal: `/wp/v2/tags` and `/wp/v2/comments` both return zero records on
  this site.
- `404 rest_post_invalid_id` on a resolve means the pointer target is not publicly visible;
  drop the result rather than retrying.
