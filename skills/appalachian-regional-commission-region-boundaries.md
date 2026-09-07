---
name: arc-region-boundaries
description: Fetch Appalachian Regional Commission boundary geometry as GeoJSON — the ARC region, its 423 counties, the 74 Local Development Districts, subregions and congressional districts. No API key.
api: ARC Geospatial API
operations:
  - listFeatureServices
  - getFeatureService
  - getFeatureLayer
  - queryFeatureLayer
generated: '2026-09-07'
method: generated
source: >-
  Grounded in operations derived from the live ARC ArcGIS services directory and verified by calling
  them anonymously on 2026-09-07.
---

# ARC region boundaries as GeoJSON

The statutory definition of Appalachia is a boundary file, and ARC serves it. All of these are public
and need no token.

Base URL: `https://services.arcgis.com/nkunl3y8FDxPkXDl/arcgis/rest`

## The services you want (`listFeatureServices`)

```
GET /services?f=json
```

- `arc_boundary` — the ARC region outline (`arc_boundary_2008` and `arc_boundary_tiger` are variants)
- `arc_counties` — the 423 counties, keyed on `FIPS`; `arc_counties_2008` is the historic set
- `arc_local_development_districts` — the 74 LDD boundaries
- `arc_subregion_counties` — Northern, North Central, Central, South Central and Southern subregions
- `arc_congressional_districts_119th` (and `_118th`)
- `arc_rural_urban_county_types`
- `arc_us_states`, `arc_us_states_tiger`, `arc_us_states_bnd_no_coastline_20m` / `_500k` — context layers

## Pull GeoJSON (`queryFeatureLayer`)

```
GET /services/arc_counties/FeatureServer/0/query
    ?where=1%3D1
    &outFields=*
    &f=geojson
```

Returns an RFC 7946 `FeatureCollection`. `arc_counties` layer 0 exposes `OBJECTID`, `FIPS`, `STATE`,
`COUNTY`, `ARC`, `Shape__Area` and `Shape__Length`.

**Size it before you pull it.** All 423 counties fit under the 1,000 `maxRecordCount`, and the full
`f=geojson` pull with geometry succeeded in one request on 2026-09-07 — 423 features, 4.85 MB, no
`exceededTransferLimit`. Larger ARC layers will not be so kind. When a response sets
`properties.exceededTransferLimit: true`, page:

```
&resultOffset=0&resultRecordCount=100
&resultOffset=100&resultRecordCount=100
...
```

and stop when `exceededTransferLimit` is no longer set.

Confirm your total first with `?where=1=1&returnCountOnly=true&f=json` → `{"count":423}`.

## Narrowing

- One state: `where=STATE%3D%27West%20Virginia%27`
- One county: `where=FIPS%3D%2754109%27` — prefer FIPS; county names repeat across the 13 states
- Spatial: `geometry` + `geometryType=esriGeometryEnvelope` + `spatialRel=esriSpatialRelIntersects`
- Drop geometry entirely with `returnGeometry=false` when you only need the attribute table

## Rules that will bite you

- **There is no OGC API - Features and no WFS here.** `/OGCFeatureServer` and
  `/WFSServer?service=WFS&request=GetCapabilities` both miss — the first returns
  `{"error":{"code":400,"message":"Invalid URL"}}`, the second the generic services-directory HTML.
  `f=geojson` on the Esri `/query` endpoint is the standards bridge ARC actually offers.
- **Always send `f`.** No `f` means HTML.
- **Errors come back on HTTP 200** inside `body.error`.
- **Projection:** responses report `spatialReference` `wkid 102100` / `latestWkid 3857` (Web
  Mercator). Ask for WGS 84 with `outSR=4326` if that is what your consumer expects — `f=geojson`
  returns WGS 84 per RFC 7946.
- **Read only**, `capabilities: Query`. Nothing to write, nothing to undo.
- **Attribution:** `copyrightText` on `arc_counties` reads "U.S. Census Bureau, Appalachian Regional
  Commission". Carry it.
