# Research: Permitbot Thin Slice

Date: 2026-05-09

This is Phase 1 research only. I did not implement code.

Canonical status: this `docs/research.md` file is the current Permitbot research artifact and supersedes older broad-background research.

## Current User Direction

The project repo is `/workspace/permitbot`, corresponding to `https://github.com/Bimo-Studio/permitbot/tree/main`.

The first implementation should be a thin slice that:

- uses City of Austin `Issued Construction Permits` only as the first signal source
- generates basic `Growth Signal` outputs
- produces a laptop-first web app with an OpenStreetMap-based map UI and side panel
- stays open-data-only forever
- supports analysis by street segment, ZIP, school area, and census tract if practical
- includes an address search bar at the top of the UI as a first-slice requirement
- shows ES/MS/HS school areas separately with boundary-year labels
- ingests `BP` permits from 2015-present for the first pass
- avoids any buy/rent recommendation
- targets hosted deployment on Vercel
- includes GitHub Actions deployment support
- includes guidance on where to host PostgreSQL/PostGIS
- strongly prefers free-tier services where practical for hackathon viability

The relevant source dataset is:

- Name: `Issued Construction Permits`
- Socrata id: `3syk-w9eu`
- Portal page: `https://data.austintexas.gov/Building-and-Development/Issued-Construction-Permits/3syk-w9eu/about_data`
- API endpoint pattern: `https://data.austintexas.gov/resource/3syk-w9eu.json`

## Repo State

The repo is documentation-only right now.

Tracked files:

- `.DS_Store`
- `.gitignore`
- `LICENSE`
- `README.md`
- `docs/Codex_Hackathon/SUMMARY.md`
- `docs/plan.md`
- `docs/research.md`

`README.md` only contains:

```md
# permitbot
looking at austin public permit data
```

There is no app code yet:

- no Python package
- no backend
- no frontend
- no database migrations
- no Docker setup
- no tests
- no dependency manifests

Existing `docs/research.md` and `docs/plan.md` contain earlier broader research and planning. They are useful background, but the current user direction narrows the first implementation to a permit-only signal source and explicitly chooses `Growth Signal` as the product language.

## Answer: Can Filters and Signals Differ by Geography?

Yes. The app can support different filters and signal presentations by:

- street segment
- ZIP
- school area
- census tract

The clean model is:

- same permit event source
- different geography assignment layer
- different aggregation windows, confidence thresholds, and UI filters per geography

However, there is an important distinction:

- `3syk-w9eu` can remain the only signal source for the thin slice.
- Geography reference layers are still needed to assign permits and selected properties to street segments, school areas, and census tracts.

That distinction keeps the product open-data-only and permit-signal-only while still enabling useful geographies. Reference boundaries are not additional "signals"; they are lookup geometry.

If the first slice must use literally only `3syk-w9eu` and no other reference datasets, then:

- ZIP is feasible using `original_zip`, though polygon display would be unavailable.
- point-level nearby permits are feasible using latitude/longitude.
- census tract is not feasible unless a tract field exists in the permit data, which it does not appear to.
- school area is not feasible from permit data alone.
- true street segment is not feasible from permit data alone; an address-string approximation would be brittle and should be avoided.

Recommended interpretation for planning: permit data is the only signal source, while open geography/reference layers are allowed for grouping and map display.

The user confirmed the first implementation must not defer address search. The top-level property input should be an address search bar, not only a map click or coordinate input. Map click can still be useful as a secondary interaction, but it should not replace address search.

## Permit Dataset Shape

The earlier research found the permit dataset is large and current:

- all rows queried: 2,354,933
- minimum `issue_date`: 1921-09-20
- maximum `issue_date`: 2026-05-09
- records from 2015 onward: 745,728
- records from 2015 onward with latitude: 650,147
- records from 2015 onward with longitude: 650,147
- records from 2015 onward with original ZIP: 743,985
- records from 2015 onward with TCAD ID: 739,001

For `permittype = 'BP'` from 2015 onward:

- rows: 166,054
- distinct permit numbers: 166,054
- distinct project IDs: 166,054
- distinct master permit numbers: 130,966

This supports a thin slice that starts with building permits only.

## Recommended Thin-Slice Permit Filter

Use a conservative first pass:

```sql
permittype = 'BP'
AND issue_date IS NOT NULL
AND latitude IS NOT NULL
AND longitude IS NOT NULL
AND calendar_year_issued >= 2015
```

Rationale:

- `BP` avoids mixing trade permits, plumbing, electrical, mechanical, driveway, and sidewalk rows into the first signal.
- `issue_date` gives a stable time axis.
- coordinates are required for map display and spatial grouping.
- 2015 onward gives a long enough baseline while avoiding the oldest, least comparable rows.

Future planning can decide whether to include non-building permit types as separate filters.

The user confirmed this `BP` from 2015-present scope is acceptable for the first pass.

## High-Value Permit Columns

For the thin slice, use:

- identity: `permit_number`, `project_id`, `masterpermitnum`
- type: `permittype`, `permit_type_desc`, `permit_class`, `permit_class_mapped`, `work_class`
- dates: `issue_date`, `applieddate`, `completed_date`, `calendar_year_issued`
- location: `latitude`, `longitude`, `original_address1`, `original_zip`, `tcad_id`, `location`
- activity magnitude: `total_job_valuation`, `building_valuation`, `total_new_add_sqft`, `remodel_repair_sqft`, `total_existing_bldg_sqft`
- descriptive context: `description`, `status_current`

Avoid using `housing_units` as a primary first-slice signal because earlier aggregate checks showed implausible outliers.

## Basic Growth Signal Definition

For the first implementation, `Growth Signal` should be a transparent permit-activity score, not a full neighborhood-health score.

Recommended first-slice signals:

- `Recent Activity`: permit count in the last 12 months
- `Permit Momentum`: last 12 months compared with prior 36-month annual average
- `New Build Activity`: count of `work_class = 'New'`
- `Renovation Activity`: count of remodel/addition-style work classes
- `Investment Magnitude`: median and winsorized sum of valuation
- `Project Scale`: median and sum of new/add/remodel square footage where available
- `Data Confidence`: source count, missing valuation rate, missing square-footage rate, and geography assignment quality

Recommended first-slice direction labels:

- `rising`
- `steady`
- `cooling`
- `insufficient data`

Avoid:

- "good/bad"
- "buy/rent"
- "dying"
- claims about school quality, safety, affordability, or displacement from permit data alone

## Geography-Specific Filters and Signals

### Street Segment

Use street segment for the most local signal.

Good filters:

- work class
- permit class mapped: residential/commercial
- recent window: 6, 12, 36 months
- radius/segment assignment confidence
- minimum event count

Good signals:

- recent permits on or near the segment
- segment permit momentum
- new-vs-remodel mix
- valuation range
- confidence warning for sparse event counts

Notes:

- This requires a street centerline or OSM road reference layer.
- Assign permits by nearest segment within a maximum distance.
- Show low confidence when few permits exist.

### ZIP

Use ZIP for a broad market-context signal.

Good filters:

- residential vs commercial
- work class
- year range
- valuation percentile

Good signals:

- permit count trend
- permit count compared with citywide ZIP percentiles
- valuation trend
- new/remodel mix

Notes:

- `original_zip` is available in the permit data and can support the first aggregation.
- ZIP polygon display requires an open ZIP boundary layer.

### School Area

Use school area as a household-oriented geography, not a claim about school quality.

Good filters:

- school level: elementary, middle, high
- school-year boundary version
- residential-only permits
- new vs remodel

Good signals:

- residential permit momentum within the school attendance area
- new/residential activity count
- remodel/addition activity count
- confidence based on boundary vintage

Notes:

- `3syk-w9eu` does not include school attendance area.
- This requires open AISD attendance boundary data.
- The app must display the school-year vintage because AISD boundaries are changing.
- The UI should show elementary, middle, and high school areas separately.

### Census Tract

Use census tract for future demographic/displacement compatibility and a more stable statistical area than street segment.

Good filters:

- residential/commercial
- work class
- year range
- valuation band

Good signals:

- permit momentum by tract
- tract percentile against other Austin tracts
- recent activity compared with historical tract baseline
- data confidence based on event count

Notes:

- `3syk-w9eu` does not appear to include census tract directly.
- Tract assignment requires a census tract boundary layer.
- Census tract is especially useful later when demographic datasets are added, but in the thin slice it can still aggregate permit activity.

## Minimal Open Geography Reference Layers

To support the requested geographies while keeping `3syk-w9eu` as the only signal source, planning should allow these open reference layers:

- street centerlines or OSM road geometries for street segments
- ZIP boundaries for ZIP polygon display
- AISD attendance boundaries for school areas
- Census tract boundaries for tract assignment

These should be treated as lookup tables:

- they should not contribute to `Growth Signal` score values
- they should identify the user's selected area
- they should define aggregation groups
- they should provide map overlays

## Map UI Thin Slice

The first UI should be laptop-first:

- map on the left
- side panel on the right
- address search bar at the top
- selected property marker
- permit points visible on map
- selected geography overlay
- area tabs for street segment, ZIP, school area, and census tract
- side-panel cards for `Growth Signal`, drivers, raw metrics, and confidence

Address search is a required part of the first implementation. It should not be deferred. The plan should choose a compliant geocoding approach:

- local open address lookup from Austin address data if practical
- or a geocoder abstraction backed by an open-data-compatible service
- no public Nominatim autocomplete
- cache geocoding results
- preserve OpenStreetMap attribution and policy compliance

Map click can be a secondary fallback, but not the primary input.

## Open-Data-Only Constraint

Open-data-only forever is compatible with this project.

Allowed sources under this constraint:

- City of Austin Open Data
- AISD public GIS downloads
- Census TIGER/Line and ACS data
- OpenStreetMap data and tiles, subject to usage policies
- other public-domain or openly licensed public datasets

Not allowed unless the user later changes the rule:

- paid property valuation APIs
- proprietary rent estimates
- MLS data
- private school rating APIs
- commercial POI datasets

Because open-data-only excludes many direct price/rent signals, the app should be explicit that `Growth Signal` is based on permit activity, not market price prediction.

The user confirmed the app must not make buy/rent recommendations.

## Deployment Research Notes

The user wants hosted deployment on Vercel and a GitHub Action to handle deployment.

Planning implication:

- The frontend is a natural fit for Vercel.
- If the backend is small enough, API routes/serverless functions can live in the Vercel app, but Python geospatial dependencies may be awkward in Vercel serverless.
- A split deployment may be cleaner: Vercel for frontend/API facade, hosted PostgreSQL/PostGIS for data, and either Vercel functions or a separately hosted Python API for heavier geospatial work.
- GitHub Actions can run lint/tests/build and trigger Vercel deployment through Vercel's GitHub integration or the Vercel CLI with project/org tokens.

PostgreSQL/PostGIS hosting needs a plan decision. Candidate directions:

- Supabase Postgres with PostGIS extension: strong developer experience, hosted Postgres, easy connection strings, good for hackathon/demo and early production.
- Neon Postgres: strong serverless Postgres option; PostGIS support must be verified during planning for the selected plan/region.
- Crunchy Bridge, Render Postgres, Railway, or Fly Postgres: viable hosted Postgres options; PostGIS availability and operational maturity vary.
- Self-managed Postgres/PostGIS on a VM: maximum control, more ops burden.

Recommended default for planning unless changed: use Supabase Postgres with PostGIS for the hosted database, Vercel for the web app, and a GitHub Action that validates and deploys the app.

The user confirmed the defaults are acceptable and added that free-tier services should be preferred as much as possible to improve hackathon viability.

Free-tier planning implications:

- Prefer Vercel Hobby/free tier for the web app if project limits allow it.
- Prefer Supabase free tier for hosted Postgres/PostGIS if data volume and compute fit.
- Keep ingestion batchable from a local machine or GitHub Actions rather than requiring paid background workers.
- Keep permit ingestion scoped and resumable so the initial demo can work even if the free database cannot hold every historical row immediately.
- Avoid paid map, geocoder, property, rent, or POI APIs.
- Consider precomputed summaries to reduce database load during judging/demo.

## Implementation Implications for the Next Plan

The next `plan.md` should be rewritten around a hackathon-scale thin slice:

- scaffold the project from scratch inside the existing repo
- create a small backend API
- create a basic map UI
- ingest or fetch a manageable subset of permit data
- compute basic permit-derived `Growth Signal`
- support geography-specific tabs/filters
- use reference geographies only where needed
- document open-data constraints
- implement address search in the first slice
- prepare Vercel deployment and GitHub Actions
- recommend a hosted Postgres/PostGIS provider
- bias toward free-tier deployment and database choices

The plan should not include broad second-phase datasets as implementation work. Those can remain future work.

## Risks and Constraints

- The repo has no existing code, so implementation will be a greenfield scaffold.
- Full historical permit ingestion is large and may be too slow for a first hackathon-style slice.
- Socrata API paging and rate limits need careful handling.
- Public OSM geocoding should not be used for autocomplete or production traffic.
- School boundaries are time-sensitive and should be versioned.
- Street segment assignment can be wrong near intersections unless distance thresholds and confidence warnings are used.
- Permit valuation and housing-unit fields contain outliers; first-slice scoring should be robust and conservative.
- A permit-only signal cannot measure crime, affordability, displacement, school quality, or livability.
- Vercel serverless deployment may constrain Python geospatial dependencies and long-running ingestion jobs.
- Hosted Postgres/PostGIS choice affects connection pooling, migrations, background ingestion, and deployment architecture.
- Free-tier services may impose database size, compute, cold-start, request, bandwidth, and job-duration constraints.

## Direct Scope Decisions Captured

- Repo: `/workspace/permitbot`
- First signal source: `3syk-w9eu` only
- Data policy: open-data-only forever
- Product label: `Growth Signal`
- First UI: web app with map and side panel
- Geography support: yes, use different filters and signals by street segment, ZIP, school area, and census tract, with open reference geometries where required
- Property input: address search bar at top, required in first implementation
- School areas: ES/MS/HS separately with boundary-year labels
- Permit scope: `BP` permits from 2015-present
- Recommendation policy: no buy/rent recommendation
- Deployment target: Vercel with GitHub Actions
- Database hosting: plan should recommend Postgres/PostGIS hosting, likely Supabase by default
- Cost posture: use free-tier services as much as practical for hackathon deployment
