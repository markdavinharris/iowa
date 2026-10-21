# Iowa Early Vote 2026 dashboard — daily refresh runbook

Published artifact (update in place, never create a new one):
**https://claude.ai/artifact/E52aPcknByKLam9GVXTZMS** — title "Iowa Early Vote 2026".

The page is a single self-contained HTML file built by `build.py` from `template.html`, `app.js`, five JSON
data files and one static geometry file `data/counties.json` (Iowa county outlines as SVG paths, made once
from the Census cb_2023_us_county_20m shapefile by `geo/make_geo.py`; it never changes). All source files plus
`SPEC.md` and `counties.json` are stored in this Project under `claude/iowa-ev-dashboard/`.

## Steps (about 15 minutes; the BigQuery queries are the slow part)

1. Make a working folder in the scratchpad, e.g. `iowa/` with a `data/` subfolder. Fetch the five project docs
   `claude/iowa-ev-dashboard/{SPEC.md,build.py,template.html,app.js,counties.json,REFRESH.md}` with
   `project_read` and write each to the working folder under its own file name (strip nothing; `build.py` expects
   `template.html` and `app.js` beside it, and `counties.json` inside `data/` with the query outputs).
2. Re-run the queries in `SPEC.md`. Only 2026 data changes day to day, but `voters` is rebuilt occasionally, so
   re-run everything each time (2024/2022 files are cheap). Hand the work to parallel subagents exactly as the
   spec says, one per output file:
   - `daily.json` (Query S ×3 years + totals) and `county_daily_2026.json` (Query C, 2026; one query covers Iowa)
   - `county_daily_2024.json` and `county_daily_2022.json` (Query C, 4 chunks each)
   - `precinct_2026.json` (Query P, 2 chunks)
   Large results are saved to a file by the harness; convert those files with a script rather than retyping.
   Each subagent must report row counts and statewide sums so they can be cross-checked (county sums must
   equal the statewide Query S sums; precinct sums are within a handful of the county sums).
3. `python3 build.py` → `dashboard.html`. It prints the as-of date (last complete day) and any county/statewide
   mismatch warning. Check: as-of date should be yesterday (America/New_York) once the feed has caught up; if
   the as-of date did not move since the last run, the feed has not updated — still republish is harmless, but
   say so in the report.
4. Spot-check one number independently, e.g.
   `SELECT COUNTIF(Rejected=0 AND ReturnDate BETWEEN '2026-01-01' AND '<asof>') + COUNTIF(EarlyVoted BETWEEN '2026-01-01' AND '<asof>') FROM abev_2026 WHERE State='OH'`
   must equal the "Early votes cast" tile.
5. Screenshot once with Playwright (light, 1200 wide) and look at it; then publish with the Artifact tool:
   `action: publish`, `url: https://claude.ai/artifact/G1xAWz3szdY7puBFeuxC2F`, `file_path: <dashboard.html>`,
   `label: "Data through <as-of date>"`. Read the artifact first if the tool says it has not been read in this
   conversation. Do not pass `icon`.
6. If `template.html` or `app.js` had to change, write the changed file back to the Project at the same path.
7. Report: as-of date, statewide early votes (and change vs 2024 same point), anything odd (new rejected rows,
   a state-level change in the ABEV party flags, a changed county list, a query that hit the 3,000-row cap).

## Data notes that matter
- Iowa party = `voters.PartyRollUp` (registered party; Iowa is not a modeled-party state). The ABEV party flags
  are not used.
- As-of day = smallest `d` (days before Election Day) whose statewide returns + in-person votes exceed 500;
  later partial days are dropped from headline figures (listed in the footer).
- Election Days: 2026-11-03, 2024-11-05, 2022-11-08. Years align on days-before-Election-Day.
- Iowa's request cutoff lands at d=60 (2026-09-04). In-person early (absentee at auditor offices) began d=20.
- A few hundred `abev_2026` Iowa rows have a NULL `Jurisname` (688 requests, 56 returns, 642 early votes on
  2026-10-20); they count statewide (Query S) but belong to no county, so county sums are slightly below the
  statewide figures and `build.py` prints a WARNING about it. That is expected; do not "fix" it.
  If a Query C chunk filters on `county < 'X'`, NULL counties drop out of every chunk — fine for county files.
- Jurisname is "<NAME> COUNTY"; `voters.CountyName` is the bare name. O'Brien County is "O BRIEN" in the voter
  file and "O'BRIEN" in the Census file: `make_geo.py`/`match_precincts.py` normalise the apostrophe to a space.
- `Rejected` = 247 Iowa rows in 2026; excluded as usual.

## Precinct maps
The county view shows a precinct map. Geometry: 99 files published on the artifact (`precincts/<COUNTY>.json`,
spaces as underscores, 0.74 MB total), fetched on demand. A republish passing only `file_path` KEEPS them.
Built once by `geo/match_precincts.py` (Project: `claude/iowa-ev-dashboard/match_precincts.py`): 2024 general
precinct polygons from The New York Times precinct map repo (`IA-precincts-with-results.geojson.gz`, sourced from
the Redistricting Data Hub), matched to voter-file `PrecinctCode` by the Census 2020 blocks of each precinct's
voters (TIGER `tl_2020_19_tabblock20` interior points). Result: 1,658 of 1,659 current precinct codes placed (one
Scott County code unplaced), 1 weak match, 6 shapes holding more than one current code.

## Publishing to a coldspark.com subdomain (GitHub Pages)
Same pattern as Ohio (`claude/ohio-ev-dashboard/REFRESH.md`): the standalone page is `index.html` written by
`build.py`, plus `precincts/*.json`, a `CNAME` file, `.nojekyll` and `build/`. No Iowa repo exists yet; a session
can push only if the repo is in its sources.
