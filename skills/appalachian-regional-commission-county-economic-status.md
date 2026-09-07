---
name: arc-county-economic-status
description: Look up the Appalachian Regional Commission's economic data for a county — economic status designation, unemployment, per-capita income and poverty — from ARC's public ArcGIS feature services. No API key.
api: ARC Geospatial API
operations:
  - listFeatureServices
  - getFeatureLayer
  - queryFeatureLayer
generated: '2026-09-07'
method: generated
source: >-
  Grounded in operations derived from the live ARC ArcGIS services directory and verified by calling
  them anonymously on 2026-09-07.
---

# ARC county economic status

ARC classifies every one of the 423 Appalachian counties each fiscal year — distressed, at-risk,
transitional, competitive or attainment — from unemployment, per-capita market income and poverty.
Those designations, and the raw measures behind them, are public ArcGIS feature layers.

Base URL: `https://services.arcgis.com/nkunl3y8FDxPkXDl/arcgis/rest`
Auth: none.

## 1. Find the right service for the year you want (`listFeatureServices`)

```
GET /services?f=json
```

108 services come back. The naming carries the year, and ARC publishes a new service per year rather
than versioning one:

- `arc_county_economic_status_fy2027_tiger` — current economic-status designations
- `arc_distressed_areas_fy2027_tiger` — distressed areas
- `arcfy2014` … `arcfy2025` — the earlier annual designations
- `arc_acs_2023_poverty_rates_tiger`, `arc_acs_2023_median_hh_income_tiger` — ACS measures

There is no manifest saying which service is current. Read the names.

## 2. Read the schema before you query (`getFeatureLayer`)

```
GET /services/arc_county_economic_status_fy2027_tiger/FeatureServer/0?f=json
```

Never guess a column. The measure fields are year-suffixed and change between services — the FY2027
layer carries `CLF2024`, `EMP2024`, `UNRATE2024`, `PCI2024`, `PCMI2024`, `PCIUS2024`, `INVPCMIUS2024`
and three-year variants, alongside the constant keys `FIPS`, `STATE`, `COUNTY` and `ARC`.

Every ARC service observed exposes exactly one layer, at index `0`.

## 3. Query the county (`queryFeatureLayer`)

```
GET /services/arc_county_economic_status_fy2027_tiger/FeatureServer/0/query
    ?where=COUNTY%3D%27Perry%27%20AND%20STATE%3D%27Kentucky%27
    &outFields=*
    &returnGeometry=false
    &f=json
```

`where` is SQL-92. Match on `FIPS` when you have it — county names repeat across states (there is a
Perry County in Kentucky, Ohio, Pennsylvania, Mississippi and Alabama), so a name-only filter will
silently return several counties.

The 13 states in the region, confirmed live from
`?where=1=1&outFields=STATE&returnDistinctValues=true`: Alabama, Georgia, Kentucky, Maryland,
Mississippi, New York, North Carolina, Ohio, Pennsylvania, South Carolina, Tennessee, Virginia,
West Virginia.

## Rules that will bite you

- **Always send `f`.** Omit it and you get the HTML services directory, not JSON. Use `f=geojson`
  when you want RFC 7946 output.
- **Errors arrive with HTTP 200.** A failure looks like
  `{"error":{"code":400,"message":"","details":["The requested layer (layerId: 99) was not found."]}}`
  on a 200 response. Check `body.error`, never `response.ok`.
- **Page past 1,000.** `maxRecordCount` is 1000 on every ARC layer. Use `resultOffset` and
  `resultRecordCount`, and stop when `properties.exceededTransferLimit` is absent or false.
  `?where=1=1&returnCountOnly=true` on `arc_counties` returns 423 — that is the whole region.
- **Read only.** These services advertise `capabilities: Query`. There is no write operation, and so
  nothing to reverse. No idempotency key is needed or accepted.
- **No rate limits are published** and no rate-limit headers are returned. Responses carry
  `cache-control: public, max-age=30` and an ETag; use conditional requests rather than polling.
- **Join across services on `FIPS`.** It is the only shared key. There is no identifier linking these
  layers back to the reports and maps on arc.gov.
