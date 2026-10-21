# Iowa Early Vote 2026 — ColdSpark

Static dashboard tracking Iowa's 2026 mail and early in-person vote against 2024 and 2022, with partisan
split and county → precinct drill-down. Served at https://iowa.coldspark.com (root of this repo).

- `index.html` — the dashboard, a single self-contained page (data baked in at build time).
- `precincts/*.json` — precinct boundaries per county, fetched on demand by the page.
- `build/` — the build kit: `build.py` assembles `template.html` + `app.js` + `data/*.json` into the page.
  `SPEC.md` documents the BigQuery queries that produce the data files; `REFRESH.md` is the daily runbook.
  `geo/` holds the one-off scripts that made the county and precinct geometry.

Data: ColdSpark national voter file and daily ABEV feeds (BigQuery). Precinct boundaries: 2024 general
election precincts from the Redistricting Data Hub via The New York Times' precinct map repository, matched
to voter-file precinct codes by Census block. County outlines: Census cartographic boundary file 2023.
