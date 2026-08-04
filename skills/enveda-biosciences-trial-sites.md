---
name: List Enveda clinical-trial sites and trial pages
description: Retrieve Enveda's published clinical-trial site directory and the trial recruitment pages that consume it, without scraping HTML.
api: openapi/enveda-biosciences-content-openapi.yml
base_url: https://enveda.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getWpslStores, getWpslStoresById, getWpslStoreCategory, getWpslStoreCategoryById, getPages, getPagesById]
generated: '2026-08-04'
method: generated
---

# List Enveda clinical-trial sites and trial pages

Enveda runs recruiting clinical trials (atopic dermatitis and asthma for ENV-294) and publishes
the participating **sites** as structured records through the WP Store Locator plugin, exposed
on the REST API as the `wpsl_stores` collection — 40 sites at harvest, including academic
centers such as the Icahn School of Medicine at Mount Sinai. This is the most substantive
non-editorial dataset the API carries, and it is reachable as JSON instead of scraping the
trial-finder widget.

Everything here is anonymous and read-only.

## 1. List the trial sites

```
GET https://enveda.com/wp-json/wp/v2/wpsl_stores?per_page=100&_fields=id,slug,title,wpsl_store_category,link
```

`getWpslStores`. `X-WP-Total` was `40` at harvest, so one page of 100 covers the whole set.
`per_page` is capped at 100.

Each record looks like:

```json
{
  "id": 1093,
  "slug": "icahn-school-of-medicine-at-mount-sinai",
  "title": { "rendered": "Icahn School of Medicine at Mount Sinai" },
  "wpsl_store_category": [21],
  "link": "https://enveda.com/?wpsl_stores=icahn-school-of-medicine-at-mount-sinai"
}
```

Note `link` is a query-string permalink — this post type has no pretty single-view route.

## 2. Resolve the site grouping

```
GET /wp/v2/wpsl_store_category?_fields=id,name,slug,count
```

`getWpslStoreCategory`. Two terms exist; map `wpsl_store_category` ids from step 1 onto them.

## 3. Fetch one site in full

```
GET /wp/v2/wpsl_stores/1093
```

`getWpslStoresById`. Address, geo and contact detail for a site live in the plugin's own
fields, which surface in `acf` / `meta` when the site populates them — on the sampled record
both were empty arrays. **Do not synthesize an address that the API did not return.** If a
record has no location payload, report it as unavailable rather than guessing.

## 4. Read the trial recruitment pages

```
GET /wp/v2/pages?_fields=id,slug,link,title&per_page=100
```

`getPages` (26 published). The trial funnel pages are identifiable by slug:

- `atopic-dermatitis-trial`, `atopic-dermatitis-follow-up`, `ad-trial-survey`
- `asthma`
- `clinical-trials-eczema-espanol` (Spanish-language variant)
- the `thank-you-qualified` / `thank-you-unqualified` screening outcomes

Fetch one with `getPagesById` to read the eligibility copy.

## Rules

- **Read-only.** Writes require an Application Password no third party holds.
- **This is recruitment marketing content, not a clinical registry.** For authoritative trial
  status, arm, and endpoint data, use ClinicalTrials.gov — not this API. Never present a
  `wpsl_stores` record as a verified enrolling site.
- Do not attempt to collect or infer participant data. Nothing in this API exposes any, and
  nothing should be added to it.
- `400 rest_invalid_param` means a parameter violated the schema (usually `per_page > 100`).
  Fix the parameter; do not retry unchanged.
