---
name: arc-find-research
description: Find Appalachian Regional Commission research reports, evaluations, fact sheets and maps by topic, fiscal year, state or county, using ARC's public WordPress REST API. No API key.
api: ARC Research and Data API
operations:
  - getSearch
  - getReport
  - getReportByid
  - getMap
  - getResource
  - getResearchReportEvalTopic
  - getStatesCounties
  - getTaxonomies
generated: '2026-09-07'
method: generated
source: >-
  Grounded in operations derived from the live ARC WordPress route index and verified by calling them
  anonymously on 2026-09-07.
---

# Find ARC research

ARC publishes 304 reports, 161 map records and 71 applicant/grantee resources. All of it is readable
without a key from the WordPress REST API behind arc.gov. There is no search API in the product
sense — there are collections and taxonomies, and the taxonomies are how you narrow.

Base URL: `https://www.arc.gov/wp-json`
Auth: none.

## Fast path — one search across everything (`getSearch`)

```
GET /wp/v2/search?search=broadband&per_page=20
```

Covers every public post type at once: reports, maps, resources, events, staff, pages, news. Returns
id, title, url and type. `search=grant` returned 243 hits on 2026-09-07. Use this when you do not
know which collection the answer lives in.

## Precise path — filter a collection by taxonomy

ARC's content model has no foreign keys. Shared taxonomy terms are the join, so narrowing means
resolving a term id first.

**Step 1 — resolve the term** (`getResearchReportEvalTopic`):

```
GET /wp/v2/research_report_eval_topic?per_page=100&_fields=id,name,slug,count
```

Each term carries a `count`, so you can see how much is behind it before you fetch. Verified live on
2026-09-07: Socioeconomic Analysis (id 197, 53 reports), Business and Entrepreneurship (207, 31),
Transportation (195, 30), Infrastructure (213, 26), Energy (194, 21), Public Health (199, 21).

**Step 2 — filter the collection** (`getReport`):

```
GET /wp/v2/report?research_report_eval_topic=197&per_page=20&_fields=id,title,link,date
```

Returned `X-WP-Total: 53` and `X-WP-TotalPages: 27` — matching the term's own count, which is the
check that you resolved the right vocabulary.

## Which vocabulary to use

Call `GET /wp/v2/taxonomies` and `GET /wp/v2/types` first; they describe the model and say which
taxonomies apply to which type. The ones that matter for research:

| Taxonomy | Applies to | Use it for |
|---|---|---|
| `research_report_eval_topic` | report | subject matter (19 terms) |
| `research_eval_maps_data_type` | report, map | document type |
| `fact_sheets_infographics_type`, `fact_sheets_infographics_topic` | report | fact sheets and infographics |
| `investment_priority_topic` | report, map, resource, event, success_story, page | ARC investment framing (22 terms) |
| `states_counties` | every type | geography (446 terms) |
| `tax_year` | report, map, event, post | fiscal / data year (70 terms) |
| `global_type` | report, map, resource, post, success_story | cross-site content class (31 terms) |

Beware `investment_priority_topic` (22 terms) versus the `investment_priority` post type (9 posts):
they are near-parallel vocabularies covering overlapping ground, not a foreign-key pair. Do not
merge them.

## Getting the actual document

A report record's PDF is in the media library. Request `_embed` or follow the `featured_media` id to
`/wp/v2/media/{id}`. There are 6,293 media records.

## Rules that will bite you

- **`per_page` maxes at 100**, default 10. Read `X-WP-Total` and `X-WP-TotalPages`, or follow the
  RFC 5988 `Link` header's `rel="next"`. All three are CORS-exposed.
- **Use `_fields`** to cut the payload — these records carry full rendered HTML content and Yoast SEO
  blocks you almost never want.
- **Re-read the route index** at `https://www.arc.gov/wp-json/` rather than caching a route list. ARC's
  custom post types and taxonomies are plugin configuration; nothing bumps a version when they change,
  and ARC publishes no changelog or deprecation policy.
- **Read only.** `/wp/v2/report` answers `OPTIONS` with `Allow: GET`. Writes exist in WordPress core
  but reject anonymous callers, so there is nothing to make idempotent and nothing to reverse.
- **Some routes are walls, not bugs.** `/wp/v2/users` → 401 `rest_user_cannot_view`, `/wp/v2/settings`
  → 401 `rest_forbidden`, `/wp/v2/comments` → 403 `rest_comment_disabled` (commenting is off
  site-wide). No public credential lifts any of them.
- **Errors are `{code, message, data:{status}}`** with a truthful HTTP status — unlike ARC's geospatial
  surface, which returns errors on HTTP 200.
- **The numbers are not here.** These records are metadata and links. The statistical data behind
  ARC's maps lives in the ARC Geospatial API; nothing in either surface links the two, so match by
  name or geography.
