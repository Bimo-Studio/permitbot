# Plan: Permitbot Thin Slice

Date: 2026-05-09

This is Phase 2 of the annotated development workflow. This document is a plan only. Do not implement until the plan has been reviewed, annotated, updated, given a todo checklist, and explicitly approved.

Canonical status: this `docs/plan.md` file is the current Permitbot implementation plan and source of truth.

User confirmations captured:

- Supabase is the hosted Postgres/PostGIS provider for the hackathon build.
- The new plan should replace the older `docs/plan.md`; git history is sufficient for prior versions.
- The first public demo should expose only Austin address candidates.

## Scope

Build the first usable Permitbot slice in `/workspace/permitbot`:

- hosted laptop-first web app on Vercel
- address search bar at the top of the app
- OpenStreetMap-based map UI
- right-side signal panel
- `Growth Signal` generated from City of Austin `Issued Construction Permits` only
- `BP` permits from 2015-present
- different filters and summaries for street segment, ZIP, AISD school area, and census tract
- ES/MS/HS school areas shown separately with boundary-year labels
- open-data-only forever
- no buy/rent recommendation
- free-tier-first deployment posture

The only first-slice signal source is:

- City of Austin `Issued Construction Permits`
- Socrata id: `3syk-w9eu`
- API: `https://data.austintexas.gov/resource/3syk-w9eu.json`

Open reference datasets are allowed for lookup, grouping, map overlays, and address search. They must not contribute independent signal values in the first slice.

## Architecture

Use a Vercel-friendly architecture with Python only in ingestion and analysis, not in request-time serverless functions.

```text
City of Austin Socrata APIs
        |
        v
Python ingestion scripts
        |
        v
Supabase Postgres + PostGIS
        |
        v
Next.js Route Handlers on Vercel
        |
        v
Next.js + Leaflet web UI
```

Why this architecture:

- Vercel is the requested hosted target and works cleanly with a Next.js app.
- Supabase is confirmed as the hosted Postgres/PostGIS provider and supports PostGIS.
- Request-time spatial work should happen in PostGIS, not in heavy Python serverless bundles.
- Python can still do the data work: Socrata ingestion, pandas transformations, NumPy-based robust summaries, and later scikit-learn experiments.
- Keeping ingestion separate from the web app avoids Vercel function duration and bundle constraints.

## Provider Defaults

Use free tiers where practical:

- Vercel Hobby for app hosting.
- Supabase Free for Postgres/PostGIS, with an explicit 500 MB database-size risk.
- GitHub Actions for CI, deployment, and optional manual ingestion.
- No paid maps, no paid geocoder, no paid property/rent/MLS APIs.

Important free-tier implication: do not store full raw JSON payloads for every permit in the database. Store a lean normalized permit table and an ingestion audit table. This improves the chance that BP permits from 2015-present, address lookup rows, street segments, ZIPs, tracts, and school boundaries fit in Supabase Free.

## Data Sources

### Signal Source

| Purpose | Dataset | Socrata ID |
|---|---|---|
| Growth Signal | Issued Construction Permits | `3syk-w9eu` |

### Reference Sources

| Purpose | Dataset | ID / Source |
|---|---|---|
| Address search | Austin Addresses | `9s7j-tygf` |
| Street segments | Street Centerline | `8hf2-pdmb` |
| ZIP overlay/grouping | Boundaries: US Zip Codes | `24mx-z6v2` |
| Census tract overlay/grouping | Boundaries: State of Texas Census Tracts | `bh9b-sg93` |
| School overlay/grouping | AISD attendance boundaries | AISD public GIS downloads |

Observed useful reference metadata:

- Austin Addresses has roughly 463,805 rows and includes `full_street_name`, point geometry, and `segment_id`.
- Street Centerline has roughly 68,482 rows and includes `segment_id`, `full_street_name`, address ranges, and line geometry.
- ZIP boundary query for `city = 'Austin'` returns roughly 44 ZIP polygons.
- Travis County census tract filter `countyfp = '453'` returns roughly 290 tracts.

## Product Model

The app should be explicit that `Growth Signal` is a permit-activity signal.

It should not imply:

- school quality
- neighborhood safety
- affordability
- displacement risk
- buy/rent advice
- future property value

Recommended labels:

- `Growth Signal`
- `Rising`
- `Steady`
- `Cooling`
- `Insufficient Data`
- `Confidence`
- `Permit Drivers`

Avoid:

- `Good`
- `Bad`
- `Dying`
- `Buy`
- `Rent`
- `Avoid`

## Database Plan

Use Supabase migrations under `supabase/migrations`.

### Extensions

Enable PostGIS and trigram search:

```sql
create extension if not exists postgis;
create extension if not exists pg_trgm;
```

### Schemas

```sql
create schema if not exists core;
create schema if not exists geo;
create schema if not exists analytics;
create schema if not exists ingest;
```

### Ingestion Audit

```sql
create table ingest.runs (
  id bigserial primary key,
  source_id text not null,
  started_at timestamptz not null default now(),
  finished_at timestamptz,
  status text not null,
  rows_read integer not null default 0,
  rows_written integer not null default 0,
  notes text
);
```

### Permits

Store normalized permit fields only.

```sql
create table core.permits (
  permit_number text primary key,
  project_id bigint,
  master_permit_number bigint,
  permit_type text not null,
  permit_type_description text,
  permit_class text,
  permit_class_mapped text,
  work_class text,
  work_class_group text not null,
  issue_date date not null,
  applied_date date,
  completed_date date,
  calendar_year_issued integer,
  status_current text,
  original_address text,
  original_zip text,
  tcad_id text,
  total_job_valuation numeric,
  total_new_add_sqft numeric,
  remodel_repair_sqft numeric,
  latitude double precision not null,
  longitude double precision not null,
  geom geometry(Point, 4326) not null,
  source_updated_at timestamptz,
  ingested_at timestamptz not null default now()
);

create index permits_geom_idx on core.permits using gist (geom);
create index permits_issue_date_idx on core.permits (issue_date);
create index permits_zip_idx on core.permits (original_zip);
create index permits_class_idx on core.permits (permit_class_mapped);
create index permits_work_group_idx on core.permits (work_class_group);
```

`work_class_group` should normalize raw values into a small set:

```text
new
remodel
addition
demolition
repair
other
```

### Addresses

```sql
create table core.addresses (
  object_id bigint primary key,
  place_id text,
  segment_id text,
  full_street_name text not null,
  street_name text,
  street_type text,
  address_number text,
  geom geometry(Point, 4326) not null,
  modified_date timestamptz,
  ingested_at timestamptz not null default now()
);

create index addresses_geom_idx on core.addresses using gist (geom);
create index addresses_full_street_trgm_idx
  on core.addresses using gin (full_street_name gin_trgm_ops);
create index addresses_segment_idx on core.addresses (segment_id);
```

Address search should query this table. Do not use public Nominatim autocomplete. The first public demo should return only Austin address candidates from this local open-data table.

### Street Segments

```sql
create table geo.street_segments (
  segment_id text primary key,
  full_street_name text,
  left_from_address text,
  left_to_address text,
  right_from_address text,
  right_to_address text,
  road_class text,
  speed_limit numeric,
  geom geometry(MultiLineString, 4326) not null,
  modified_date timestamptz,
  ingested_at timestamptz not null default now()
);

create index street_segments_geom_idx on geo.street_segments using gist (geom);
create index street_segments_name_idx on geo.street_segments (full_street_name);
```

### ZIP Boundaries

```sql
create table geo.zip_boundaries (
  zipcode text primary key,
  city text,
  state text,
  geom geometry(MultiPolygon, 4326) not null,
  ingested_at timestamptz not null default now()
);

create index zip_boundaries_geom_idx on geo.zip_boundaries using gist (geom);
```

### Census Tracts

```sql
create table geo.census_tracts (
  geoid text primary key,
  name text,
  namelsad text,
  countyfp text,
  geom geometry(MultiPolygon, 4326) not null,
  ingested_at timestamptz not null default now()
);

create index census_tracts_geom_idx on geo.census_tracts using gist (geom);
```

### School Areas

```sql
create table geo.school_areas (
  id text primary key,
  school_year text not null,
  school_level text not null check (school_level in ('ES', 'MS', 'HS')),
  school_name text not null,
  source_name text not null,
  geom geometry(MultiPolygon, 4326) not null,
  ingested_at timestamptz not null default now()
);

create index school_areas_geom_idx on geo.school_areas using gist (geom);
create index school_areas_lookup_idx on geo.school_areas (school_year, school_level);
```

### Area Assignments

Precompute permit-to-area assignments to keep API requests fast.

```sql
create table analytics.permit_area_assignments (
  permit_number text not null references core.permits (permit_number) on delete cascade,
  area_type text not null,
  area_id text not null,
  area_label text not null,
  assignment_method text not null,
  distance_meters numeric,
  confidence numeric not null,
  primary key (permit_number, area_type, area_id)
);

create index permit_area_assignments_area_idx
  on analytics.permit_area_assignments (area_type, area_id);
```

Area types:

```text
street_segment
zipcode
census_tract
school_es
school_ms
school_hs
```

### Growth Signals

Precompute metrics and scores by area and filter.

```sql
create table analytics.growth_signals (
  area_type text not null,
  area_id text not null,
  area_label text not null,
  filter_key text not null,
  filter_label text not null,
  recent_permits_12m integer not null,
  baseline_annual_permits_36m numeric not null,
  momentum_ratio numeric not null,
  new_permits_12m integer not null,
  remodel_permits_12m integer not null,
  median_valuation_12m numeric,
  total_valuation_12m numeric,
  median_sqft_12m numeric,
  total_sqft_12m numeric,
  missing_valuation_rate numeric not null,
  missing_sqft_rate numeric not null,
  growth_score numeric not null,
  direction text not null,
  confidence numeric not null,
  drivers jsonb not null,
  generated_at timestamptz not null default now(),
  primary key (area_type, area_id, filter_key)
);

create index growth_signals_area_idx
  on analytics.growth_signals (area_type, area_id);
```

## Ingestion Plan

Use Python scripts under `pipeline/`.

### Python Dependencies

Use `pyproject.toml` with:

```toml
[project]
dependencies = [
  "requests>=2.32",
  "pandas>=2.2",
  "numpy>=2.0",
  "psycopg[binary]>=3.2",
  "python-dotenv>=1.0",
  "pydantic>=2.8",
]

[project.optional-dependencies]
ml = ["scikit-learn>=1.5"]
```

Keep `scikit-learn` optional in the first slice. The first score is deterministic. Add ML only when there is a defined supervised target.

### Socrata Client

Use a small client that pages with `$limit` and `$offset`.

```python
class SocrataClient:
    def __init__(self, base_url: str) -> None:
        self.base_url = base_url

    def fetch_page(
        self,
        resource_id: str,
        *,
        select: str,
        where: str,
        limit: int,
        offset: int,
    ) -> list[dict[str, object]]:
        response = requests.get(
            f"{self.base_url}/resource/{resource_id}.json",
            params={
                "$select": select,
                "$where": where,
                "$limit": limit,
                "$offset": offset,
            },
            timeout=60,
        )
        response.raise_for_status()
        return response.json()
```

### Permit Ingestion

Ingest permits with:

```sql
permittype = 'BP'
AND issue_date IS NOT NULL
AND latitude IS NOT NULL
AND longitude IS NOT NULL
AND calendar_year_issued >= 2015
```

Normalize each row into `core.permits`.

```python
def classify_work_class(work_class: str | None) -> str:
    value = (work_class or "").strip().lower()
    if value == "new":
        return "new"
    if "demo" in value:
        return "demolition"
    if "addition" in value:
        return "addition"
    if "remodel" in value:
        return "remodel"
    if "repair" in value:
        return "repair"
    return "other"
```

Use upserts so ingestion is repeatable:

```sql
insert into core.permits (...)
values (...)
on conflict (permit_number) do update set
  status_current = excluded.status_current,
  total_job_valuation = excluded.total_job_valuation,
  total_new_add_sqft = excluded.total_new_add_sqft,
  remodel_repair_sqft = excluded.remodel_repair_sqft,
  source_updated_at = excluded.source_updated_at,
  ingested_at = now();
```

### Reference Ingestion

Ingest address, street, ZIP, census tract, and school boundary data separately:

- `pipeline/ingest_addresses.py`
- `pipeline/ingest_street_segments.py`
- `pipeline/ingest_zip_boundaries.py`
- `pipeline/ingest_census_tracts.py`
- `pipeline/ingest_school_areas.py`

The Socrata reference ingestors should use the same client and insert geometries with `ST_GeomFromGeoJSON`.

Example geometry insert shape:

```sql
insert into geo.street_segments (segment_id, full_street_name, geom)
values (%s, %s, ST_SetSRID(ST_GeomFromGeoJSON(%s), 4326))
on conflict (segment_id) do update set
  full_street_name = excluded.full_street_name,
  geom = excluded.geom,
  ingested_at = now();
```

AISD school areas need special handling because the official GIS source may be a downloadable GIS file rather than a Socrata API. The implementation should support one of these paths:

- download official AISD GeoJSON directly if available
- download official AISD zipped shapefiles and convert during local ingestion
- store small official GeoJSON boundary files under `data/reference/aisd/` if the file size and license are acceptable

The selected boundary year must be stored and displayed.

## Area Assignment Plan

Create a pipeline step:

```text
pipeline/assign_areas.py
```

### Street Segment Assignment

Use the nearest street segment within a threshold, such as 40 meters.

```sql
insert into analytics.permit_area_assignments (...)
select
  p.permit_number,
  'street_segment',
  s.segment_id,
  coalesce(s.full_street_name, 'Unknown street segment'),
  'nearest_segment',
  ST_Distance(p.geom::geography, s.geom::geography),
  case
    when ST_Distance(p.geom::geography, s.geom::geography) <= 20 then 1.0
    when ST_Distance(p.geom::geography, s.geom::geography) <= 40 then 0.7
    else 0.0
  end
from core.permits p
join lateral (
  select segment_id, full_street_name, geom
  from geo.street_segments
  where ST_DWithin(p.geom::geography, geom::geography, 40)
  order by p.geom <-> geom
  limit 1
) s on true
on conflict do nothing;
```

### ZIP Assignment

Prefer polygon containment where available. Use `original_zip` as fallback only if polygon assignment fails.

```sql
select z.zipcode
from geo.zip_boundaries z
where ST_Contains(z.geom, p.geom);
```

### Census Tract Assignment

Use polygon containment:

```sql
select c.geoid
from geo.census_tracts c
where ST_Contains(c.geom, p.geom);
```

### School Assignment

Assign separately for ES, MS, and HS:

```sql
select s.id, s.school_level, s.school_name, s.school_year
from geo.school_areas s
where s.school_year = :active_school_year
and ST_Contains(s.geom, p.geom);
```

## Growth Signal Plan

Create:

```text
pipeline/compute_growth_signals.py
```

### Filters

Precompute these filter keys for each area:

| Filter Key | Meaning |
|---|---|
| `all_bp` | all building permits |
| `residential` | `permit_class_mapped = 'Residential'` |
| `commercial` | `permit_class_mapped = 'Commercial'` |
| `new` | `work_class_group = 'new'` |
| `remodel_addition` | `work_class_group in ('remodel', 'addition')` |
| `demolition` | `work_class_group = 'demolition'` |

Some filters should be hidden or marked low-confidence when counts are too sparse.

### Formula

For each `area_type`, `area_id`, and `filter_key`:

```python
recent_count = permits_issued_in_last_12_months
baseline_annual_count = permits_issued_in_months_13_to_48 / 3
momentum_ratio = recent_count / baseline_annual_count if baseline_annual_count > 0 else recent_count
```

Use robust valuation summaries:

```python
def winsorized_sum(values: np.ndarray, lower: float = 0.02, upper: float = 0.98) -> float:
    clean_values = values[~np.isnan(values)]
    if clean_values.size == 0:
        return 0.0
    low, high = np.quantile(clean_values, [lower, upper])
    return float(np.clip(clean_values, low, high).sum())
```

Score sketch:

```python
def growth_score(
    recent_percentile: float,
    momentum_ratio: float,
    valuation_percentile: float,
    confidence: float,
) -> float:
    capped_momentum = min(momentum_ratio / 2.0, 1.0)
    raw_score = (
        0.45 * recent_percentile
        + 0.35 * capped_momentum
        + 0.20 * valuation_percentile
    )
    return round(raw_score * confidence, 3)
```

Direction sketch:

```python
def direction(score: float, confidence: float) -> str:
    if confidence < 0.35:
        return "insufficient_data"
    if score >= 0.66:
        return "rising"
    if score >= 0.4:
        return "steady"
    return "cooling"
```

Confidence sketch:

```python
def confidence(source_count: int, missing_rate: float, assignment_confidence: float) -> float:
    count_component = min(source_count / 20.0, 1.0)
    completeness_component = max(1.0 - missing_rate, 0.0)
    return round(0.5 * count_component + 0.3 * completeness_component + 0.2 * assignment_confidence, 3)
```

Street segments should use a lower count threshold than ZIPs and tracts, but should display low-confidence warnings more often.

## API Plan

Use Next.js App Router route handlers.

### Address Search

Path:

```text
app/api/search/addresses/route.ts
```

Request:

```http
GET /api/search/addresses?q=108%20walnut
```

Query:

```sql
select
  object_id,
  full_street_name,
  segment_id,
  ST_X(geom) as longitude,
  ST_Y(geom) as latitude,
  similarity(full_street_name, $1) as score
from core.addresses
where full_street_name % $1
order by score desc, full_street_name asc
limit 8;
```

Route handler sketch:

```ts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const query = searchParams.get("q")?.trim();

  if (!query || query.length < 3) {
    return Response.json({ results: [] });
  }

  const results = await searchAddresses(query);
  return Response.json({ results });
}
```

The API should not fall back to non-Austin geocoders in the first public demo. If a query has no local Austin match, return an empty result set with UI copy such as `No Austin address match found`.

### Property Summary

Path:

```text
app/api/properties/summary/route.ts
```

Request:

```http
GET /api/properties/summary?addressId=4653
```

Response shape:

```json
{
  "property": {
    "addressId": "4653",
    "label": "108 W WALNUT DR",
    "latitude": 30.351522539332,
    "longitude": -97.701647035633
  },
  "areas": {
    "streetSegment": { "id": "2031817", "label": "W WALNUT DR" },
    "zipcode": { "id": "78757", "label": "78757" },
    "censusTract": { "id": "48453001813", "label": "Census Tract 18.13" },
    "schools": [
      { "level": "ES", "id": "es-...", "name": "...", "schoolYear": "2022-23" },
      { "level": "MS", "id": "ms-...", "name": "...", "schoolYear": "2022-23" },
      { "level": "HS", "id": "hs-...", "name": "...", "schoolYear": "2022-23" }
    ]
  },
  "signals": [
    {
      "areaType": "zipcode",
      "areaId": "78757",
      "filterKey": "all_bp",
      "filterLabel": "All building permits",
      "growthScore": 0.72,
      "direction": "rising",
      "confidence": 0.84,
      "metrics": {
        "recentPermits12m": 140,
        "baselineAnnualPermits36m": 105,
        "momentumRatio": 1.33
      },
      "drivers": [
        "Recent permits are above the prior three-year annual baseline",
        "Permit volume is above the citywide median for ZIP areas"
      ]
    }
  ],
  "overlays": {
    "streetSegment": { "type": "Feature", "geometry": {} },
    "zipcode": { "type": "Feature", "geometry": {} },
    "censusTract": { "type": "Feature", "geometry": {} },
    "schools": []
  }
}
```

### Nearby Permits

Path:

```text
app/api/permits/nearby/route.ts
```

Use this for map permit dots around the selected address. Limit results aggressively for Vercel and browser performance.

```sql
select
  permit_number,
  permit_class_mapped,
  work_class_group,
  issue_date,
  total_job_valuation,
  ST_X(geom) as longitude,
  ST_Y(geom) as latitude
from core.permits
where ST_DWithin(
  geom::geography,
  ST_SetSRID(ST_Point($1, $2), 4326)::geography,
  $3
)
order by issue_date desc
limit 500;
```

## Frontend Plan

Use Next.js App Router, TypeScript, and Leaflet.

Layout:

```text
┌─────────────────────────────────────────────────────────────┐
│ Address search bar                                           │
├───────────────────────────────────────┬─────────────────────┤
│                                       │ Growth Signal panel │
│ Map                                   │                     │
│ - selected address marker             │ Area tabs           │
│ - permit points                       │ Filter controls     │
│ - selected overlays                   │ Metric cards        │
│                                       │ Confidence notes    │
└───────────────────────────────────────┴─────────────────────┘
```

Area tabs:

- `Street Segment`
- `ZIP`
- `Census Tract`
- `Schools`

For `Schools`, show ES/MS/HS cards separately.

Filter controls:

- all building permits
- residential
- commercial
- new
- remodel/addition
- demolition

### Component Structure

```text
app/
  layout.tsx
  page.tsx
  globals.css
  api/
    search/addresses/route.ts
    properties/summary/route.ts
    permits/nearby/route.ts
components/
  AddressSearch.tsx
  MapView.tsx
  SignalPanel.tsx
  AreaTabs.tsx
  FilterBar.tsx
  SignalCard.tsx
  MetricGrid.tsx
  ConfidenceNote.tsx
lib/
  db.ts
  queries/
    addresses.ts
    properties.ts
    permits.ts
  types.ts
```

### Address Search Component

```tsx
"use client";

export function AddressSearch({ onSelect }: AddressSearchProps) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<AddressCandidate[]>([]);

  useEffect(() => {
    if (query.trim().length < 3) {
      setResults([]);
      return;
    }

    const timeoutId = window.setTimeout(async () => {
      const response = await fetch(`/api/search/addresses?q=${encodeURIComponent(query)}`);
      const payload: AddressSearchResponse = await response.json();
      setResults(payload.results);
    }, 250);

    return () => window.clearTimeout(timeoutId);
  }, [query]);

  return (
    <div className="address-search">
      <input value={query} onChange={(event) => setQuery(event.target.value)} />
      <div className="address-search__results">
        {results.map((result) => (
          <button key={result.addressId} onClick={() => onSelect(result)}>
            {result.label}
          </button>
        ))}
      </div>
    </div>
  );
}
```

Implementation should add accessible labels and keyboard behavior, not just mouse selection.

### Map Component

Use Leaflet in a client component. If SSR causes issues, load `MapView` dynamically from the page with `ssr: false`.

```tsx
const MapView = dynamic(() => import("@/components/MapView"), {
  ssr: false,
});
```

Map responsibilities:

- render OSM tiles with attribution
- center on selected address
- render selected property marker
- render selected area overlay
- render up to 500 nearby permit points

## Deployment Plan

### Vercel

Use Vercel for the Next.js app. Store these environment variables in Vercel:

```text
DATABASE_URL
NEXT_PUBLIC_APP_ENV
```

Use a pooled Supabase connection string for Vercel request handlers if available. Keep any service-role or direct database credentials server-only.

### GitHub Actions

Use two workflows:

```text
.github/workflows/ci.yml
.github/workflows/vercel-deploy.yml
```

`ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run build
```

`vercel-deploy.yml`:

```yaml
name: Deploy to Vercel

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      - run: npx vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      - run: npx vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

Required GitHub secrets:

```text
VERCEL_TOKEN
VERCEL_ORG_ID
VERCEL_PROJECT_ID
```

### Database Migrations

Use Supabase CLI locally first:

```text
supabase db push
```

Add a manual migration workflow only if needed:

```text
.github/workflows/db-migrate.yml
```

Keep database migration separate from Vercel deployment so a UI deploy cannot accidentally mutate the database during judging.

### Ingestion

Use local ingestion for the first hackathon pass:

```text
npm run db:migrate
npm run ingest:references
npm run ingest:permits
npm run compute:signals
```

Optionally add a manual GitHub Action:

```text
.github/workflows/ingest.yml
```

Do not schedule ingestion until after the demo is stable.

## File Plan

These are the files expected to be created or modified during implementation.

### Root

- `README.md` - update from placeholder to setup, data, deployment, and caveats.
- `.gitignore` - add local env files, build output, Python caches, Vercel files.
- `.env.example` - document required local variables.
- `package.json` - Next.js scripts and JS dependencies.
- `package-lock.json` - lock JS dependencies.
- `tsconfig.json` - TypeScript config.
- `next.config.ts` - Next.js config.
- `eslint.config.mjs` - lint config.
- `postcss.config.mjs` - if CSS tooling requires it.
- `pyproject.toml` - Python pipeline dependencies and tool config.
- `plan.md` - implementation progress source of truth later.

### App

- `app/layout.tsx` - root layout and metadata.
- `app/page.tsx` - main page shell.
- `app/globals.css` - laptop-first layout and visual styling.
- `app/api/search/addresses/route.ts` - address search endpoint.
- `app/api/properties/summary/route.ts` - selected property summary endpoint.
- `app/api/permits/nearby/route.ts` - nearby permit point endpoint.

### Components

- `components/AddressSearch.tsx` - top address search bar.
- `components/MapView.tsx` - Leaflet map and overlays.
- `components/SignalPanel.tsx` - right-side panel.
- `components/AreaTabs.tsx` - street/ZIP/tract/school tabs.
- `components/FilterBar.tsx` - permit filter controls.
- `components/SignalCard.tsx` - growth signal display.
- `components/MetricGrid.tsx` - raw metric display.
- `components/ConfidenceNote.tsx` - confidence and caveat display.

### Library

- `lib/db.ts` - server-only Postgres client.
- `lib/types.ts` - shared TypeScript types.
- `lib/queries/addresses.ts` - address SQL.
- `lib/queries/properties.ts` - area and signal SQL.
- `lib/queries/permits.ts` - nearby permit SQL.
- `lib/format.ts` - formatting helpers.

### Supabase / SQL

- `supabase/config.toml` - local Supabase config if using CLI.
- `supabase/migrations/0001_extensions.sql`
- `supabase/migrations/0002_core_tables.sql`
- `supabase/migrations/0003_geo_tables.sql`
- `supabase/migrations/0004_analytics_tables.sql`
- `supabase/migrations/0005_views.sql`

### Python Pipeline

- `pipeline/__init__.py`
- `pipeline/config.py`
- `pipeline/db.py`
- `pipeline/socrata.py`
- `pipeline/ingest_permits.py`
- `pipeline/ingest_addresses.py`
- `pipeline/ingest_street_segments.py`
- `pipeline/ingest_zip_boundaries.py`
- `pipeline/ingest_census_tracts.py`
- `pipeline/ingest_school_areas.py`
- `pipeline/assign_areas.py`
- `pipeline/compute_growth_signals.py`
- `pipeline/run_all.py`

### Data References

- `data/reference/aisd/README.md` - document AISD boundary source and school year.
- `data/reference/aisd/.gitkeep` - placeholder unless small GeoJSON files are committed.

### GitHub Actions

- `.github/workflows/ci.yml`
- `.github/workflows/vercel-deploy.yml`
- `.github/workflows/ingest.yml` - optional manual ingestion workflow.

### Docs

- `docs/data_sources.md` - exact open datasets, IDs, purpose, and caveats.
- `docs/scoring_methodology.md` - permit-only Growth Signal formula.
- `docs/deployment.md` - Vercel, Supabase, GitHub secrets, free-tier notes.
- `docs/open_data_policy.md` - open-data-only rule and excluded data types.

The existing `docs/plan.md` should be replaced with this current plan during the planning update. Older planning context can be recovered from git history.

## Testing and Validation Plan

### Python

Add focused tests for pure logic:

- `classify_work_class`
- valuation winsorization
- score direction thresholds
- confidence calculation

Use `pytest` if adding Python tests.

### TypeScript

Run:

```text
npm run lint
npm run typecheck
npm run build
```

### Database

Validate counts after ingestion:

```sql
select count(*) from core.permits;
select count(*) from core.addresses;
select count(*) from geo.street_segments;
select area_type, count(*) from analytics.growth_signals group by area_type;
```

Validate one known address:

- address search returns a candidate
- property summary resolves street segment, ZIP, tract, and school areas
- map overlay returns valid GeoJSON
- side panel renders Growth Signal cards

## Trade-Offs

### Next.js Route Handlers Instead of FastAPI

Chosen for the first slice because the app must deploy to Vercel on a free-tier-friendly path. FastAPI remains attractive for a later dedicated backend, but it adds hosting complexity for the hackathon slice.

### Python Offline Pipeline Instead of Python API

Chosen because pandas/NumPy/scikit-learn are useful for ingestion and analysis but are not ideal for Vercel request-time functions. This keeps Python where it is strongest and keeps the web app lightweight.

### Supabase Free Tier Instead of Self-Hosted Postgres

Chosen because it minimizes ops burden and supports PostGIS. The trade-off is the 500 MB free database-size limit and project pausing after inactivity.

### Lean Normalized Tables Instead of Full Raw JSON

Chosen because free-tier database size matters. The trade-off is less perfect reproducibility. Mitigate by documenting Socrata source URLs, storing ingestion run metadata, and making ingestion repeatable.

### Local Address Dataset Instead of Nominatim Autocomplete

Chosen because address search is required, but public Nominatim is not appropriate for autocomplete. Austin Addresses is open data and gives a controlled local search index.

### Street Segment From City Centerline

Chosen because Austin Addresses already has `segment_id`, and Street Centerline has matching segment geometries. The trade-off is that permit-to-segment assignment still requires nearest-line matching and confidence warnings.

### Permit-Only Growth Signal

Chosen because the user narrowed the first implementation to `3syk-w9eu`. The trade-off is that the app cannot claim overall neighborhood health. Keep the UI honest by exposing raw permit drivers and confidence.

## Implementation Phases After Approval

Do not execute these yet.

### Phase 1: Scaffold

Create Next.js app structure, TypeScript config, Python pipeline package, Supabase migration folder, and documentation skeleton.

### Phase 2: Database

Add PostGIS migrations for core, geo, analytics, and ingest schemas.

### Phase 3: Ingestion

Implement Socrata client and ingestion scripts for permits, addresses, street centerlines, ZIP boundaries, census tracts, and school areas.

### Phase 4: Signals

Implement area assignment and precomputed Growth Signal generation.

### Phase 5: API

Implement address search, property summary, and nearby permit endpoints.

### Phase 6: UI

Implement map, search, side panel, area tabs, filters, signal cards, and confidence display.

### Phase 7: Deployment

Add GitHub Actions CI and Vercel deployment workflow, document Supabase setup and secrets.

### Phase 8: Validation

Run ingestion on a demo database, verify known addresses, run type checks/build/tests, and document caveats.

## Open Review Points

- Confirm AISD boundary source/vintage if you already have a preferred file.

## Todo

Implementation rules for this checklist:

- Complete tasks in the exact order listed.
- Do not skip items.
- Do not mark an item complete until it is actually implemented and verified.
- Do not create partial implementations.
- Do not create mock implementations.
- Do not hard-code fake data in place of real open-data-backed behavior.
- Use TDD for every code todo: write or update the smallest relevant correctness test first, watch it fail for the expected reason when feasible, implement the real code, then make the test pass.
- Use BDD for every user-visible or plan-facing todo: add or update an acceptance test that proves the behavior adheres to this plan, then make that test pass with real data-backed behavior.
- If a todo is configuration-only or documentation-only, its verification must still be explicit: CI validation, build validation, migration validation, or a BDD acceptance check must prove the plan requirement is satisfied.
- CI must pass all tests before implementation is considered complete.
- Code coverage must be at least 50% for both TypeScript and Python test suites and must not decrease once a higher threshold is established.
- If a task is blocked, write the blocker to `blockers.md`, leave the todo unchecked, and continue only if the next item does not depend on the blocker.
- After completing each checked item, update this checklist in `plan.md` and `docs/plan.md`.

### Phase 1: Project Scaffold

- [ ] `package.json` — create Next.js app scripts and dependencies for the Vercel-hosted web app.
- [ ] `package-lock.json` — lock JavaScript dependencies after installing the app dependencies.
- [ ] `tsconfig.json` — configure strict TypeScript for the app and route handlers.
- [ ] `next.config.ts` — add minimal Next.js configuration required for the app.
- [ ] `eslint.config.mjs` — add lint configuration compatible with the chosen Next.js version.
- [ ] `app/layout.tsx` — create the root layout and metadata for Permitbot.
- [ ] `app/page.tsx` — create the main page shell with address search, map, and side panel regions.
- [ ] `app/globals.css` — create the laptop-first base layout and visual styling.
- [ ] `components/.gitkeep` — create the components directory before adding component files.
- [ ] `lib/.gitkeep` — create the library directory before adding shared app code.
- [ ] `pyproject.toml` — define Python pipeline dependencies and optional ML dependencies.
- [ ] `pyproject.toml` — configure pytest and coverage settings with a minimum 50% Python coverage threshold.
- [ ] `vitest.config.ts` — configure TypeScript unit/component tests and minimum 50% coverage threshold.
- [ ] `playwright.config.ts` — configure BDD-style acceptance tests for plan adherence.
- [ ] `pipeline/__init__.py` — create the Python pipeline package.
- [ ] `tests/bdd/.gitkeep` — create the acceptance-test directory for plan-level BDD scenarios.
- [ ] `tests/unit/.gitkeep` — create the TypeScript unit-test directory for TDD tests.
- [ ] `.env.example` — document required Supabase, database, and deployment environment variables.
- [ ] `.gitignore` — add local env files, build output, Python caches, Supabase temp files, and Vercel files.

### Phase 2: Database Migrations

- [ ] `supabase/config.toml` — initialize local Supabase CLI configuration for migrations.
- [ ] `supabase/migrations/0001_extensions.sql` — enable PostGIS and trigram extensions.
- [ ] `supabase/migrations/0002_ingest_schema.sql` — create `ingest.runs` for ingestion audit tracking.
- [ ] `supabase/migrations/0003_core_tables.sql` — create `core.permits` and `core.addresses`.
- [ ] `supabase/migrations/0004_geo_tables.sql` — create `geo.street_segments`, `geo.zip_boundaries`, `geo.census_tracts`, and `geo.school_areas`.
- [ ] `supabase/migrations/0005_analytics_tables.sql` — create `analytics.permit_area_assignments` and `analytics.growth_signals`.
- [ ] `supabase/migrations/0006_api_views.sql` — add read-focused views or helper SQL needed by the API without duplicating business logic.

### Phase 3: Python Pipeline Foundation

- [ ] `pipeline/config.py` — load and validate environment variables with no implicit production defaults.
- [ ] `pipeline/db.py` — implement a psycopg connection helper and transaction wrapper.
- [ ] `pipeline/socrata.py` — implement paginated Socrata API fetching with timeout and error handling.
- [ ] `pipeline/geometry.py` — add GeoJSON serialization helpers for PostGIS inserts.
- [ ] `pipeline/work_classes.py` — implement real permit work-class normalization.
- [ ] `pipeline/tests/test_work_classes.py` — test work-class normalization with real expected categories.
- [ ] `tests/bdd/pipeline-readiness.spec.ts` — add a BDD acceptance check that the pipeline commands and required environment variables match the plan.

### Phase 4: Reference Data Ingestion

- [ ] `pipeline/ingest_addresses.py` — ingest Austin Addresses from `9s7j-tygf` into `core.addresses`.
- [ ] `pipeline/ingest_street_segments.py` — ingest Street Centerline from `8hf2-pdmb` into `geo.street_segments`.
- [ ] `pipeline/ingest_zip_boundaries.py` — ingest Austin ZIP polygons from `24mx-z6v2` into `geo.zip_boundaries`.
- [ ] `pipeline/ingest_census_tracts.py` — ingest Travis County census tracts from `bh9b-sg93` into `geo.census_tracts`.
- [ ] `data/reference/aisd/README.md` — document the exact AISD attendance boundary source and required school-year labeling.
- [ ] `pipeline/ingest_school_areas.py` — ingest ES/MS/HS AISD school areas from the documented open source into `geo.school_areas`.
- [ ] `pipeline/run_reference_ingest.py` — run all reference ingestion steps in dependency order.

### Phase 5: Permit Ingestion

- [ ] `pipeline/ingest_permits.py` — ingest only `BP` permits from 2015-present with non-null issue date and coordinates.
- [ ] `pipeline/ingest_permits.py` — upsert normalized permit fields into `core.permits` without storing full raw JSON payloads.
- [ ] `pipeline/ingest_permits.py` — write ingestion run status and row counts to `ingest.runs`.
- [ ] `pipeline/tests/test_permit_transform.py` — test permit row transformation using real-shaped sample rows.
- [ ] `tests/bdd/permit-ingestion.spec.ts` — add a BDD acceptance check that permit ingestion targets only real `BP` permits from 2015-present and rejects out-of-scope rows.

### Phase 6: Area Assignment

- [ ] `pipeline/assign_street_segments.py` — assign permits to nearest street segment within the configured distance threshold.
- [ ] `pipeline/assign_zipcodes.py` — assign permits to ZIP polygons and use permit ZIP only as documented fallback.
- [ ] `pipeline/assign_census_tracts.py` — assign permits to census tracts by polygon containment.
- [ ] `pipeline/assign_school_areas.py` — assign permits to ES/MS/HS school areas by polygon containment and active school year.
- [ ] `pipeline/assign_areas.py` — run all assignment steps in order and record assignment confidence.
- [ ] `pipeline/tests/test_area_assignment_sql.py` — test generated assignment SQL fragments or query builders for street, ZIP, tract, and school assignment behavior.
- [ ] `tests/bdd/area-assignment.spec.ts` — add a BDD acceptance check that the plan-required geographies are represented: street segment, ZIP, census tract, ES, MS, and HS.

### Phase 7: Growth Signal Computation

- [ ] `pipeline/scoring.py` — implement deterministic Growth Signal scoring, direction labels, and confidence calculations.
- [ ] `pipeline/tests/test_scoring.py` — test scoring thresholds, confidence behavior, and insufficient-data cases.
- [ ] `pipeline/compute_growth_signals.py` — compute area/filter metrics for street segments, ZIPs, census tracts, ES areas, MS areas, and HS areas.
- [ ] `pipeline/compute_growth_signals.py` — write real precomputed rows to `analytics.growth_signals`.
- [ ] `pipeline/run_all.py` — run reference ingestion, permit ingestion, area assignment, and signal computation in order.
- [ ] `tests/bdd/growth-signal.spec.ts` — add a BDD acceptance check that Growth Signal output is permit-only, includes confidence, and never contains buy/rent recommendations.

### Phase 8: Server-Side App Data Access

- [ ] `lib/types.ts` — define TypeScript types for address candidates, property summaries, areas, signals, metrics, drivers, and overlays.
- [ ] `lib/db.ts` — implement server-only Postgres access using `DATABASE_URL`.
- [ ] `lib/queries/addresses.ts` — implement Austin-only address search using `core.addresses` trigram search.
- [ ] `lib/queries/properties.ts` — implement property area lookup and signal lookup from real database tables.
- [ ] `lib/queries/permits.ts` — implement nearby permit query from `core.permits`.
- [ ] `lib/geojson.ts` — implement GeoJSON conversion/parsing helpers for API responses.
- [ ] `tests/unit/api-query-contracts.test.ts` — test query helper behavior for Austin-only search, property summary, permit lookup, and GeoJSON conversion.

### Phase 9: API Routes

- [ ] `app/api/search/addresses/route.ts` — return only Austin address candidates from the local open-data address table.
- [ ] `app/api/properties/summary/route.ts` — return selected address, assigned areas, Growth Signals, and overlays from real database records.
- [ ] `app/api/permits/nearby/route.ts` — return bounded nearby permit points from real permit records.
- [ ] `app/api/health/route.ts` — expose a simple database-backed health check for deployment validation.
- [ ] `tests/unit/api-routes.test.ts` — test route validation, missing-query behavior, empty-result behavior, and response shape without mocked business outcomes.
- [ ] `tests/bdd/api-contract.spec.ts` — add a BDD acceptance check that API responses expose only plan-approved data and real database-backed fields.

### Phase 10: UI Components

- [ ] `components/AddressSearch.tsx` — implement the required top address search bar with keyboard-accessible Austin-only results.
- [ ] `components/MapView.tsx` — implement Leaflet map with OSM attribution, selected marker, permit points, and real overlays.
- [ ] `components/SignalPanel.tsx` — implement the right-side panel using real property summary data.
- [ ] `components/AreaTabs.tsx` — implement Street Segment, ZIP, Census Tract, and Schools tabs.
- [ ] `components/FilterBar.tsx` — implement filter controls backed by available precomputed signal filter keys.
- [ ] `components/SignalCard.tsx` — display Growth Signal, direction, confidence, and drivers.
- [ ] `components/MetricGrid.tsx` — display raw permit metrics from API responses.
- [ ] `components/ConfidenceNote.tsx` — display low-confidence and permit-only caveats.
- [ ] `tests/unit/components.test.tsx` — test component rendering for search, area tabs, filters, signal cards, metrics, and confidence notes.
- [ ] `tests/bdd/ui-plan.spec.ts` — add a BDD acceptance check for the required laptop layout, top address search bar, map, side panel, area tabs, ES/MS/HS labels, and no buy/rent recommendation.

### Phase 11: Page Integration

- [ ] `app/page.tsx` — wire address selection to property summary fetching.
- [ ] `app/page.tsx` — wire selected property to map center, overlays, and nearby permit loading.
- [ ] `app/page.tsx` — wire area tab and filter selection to the side panel.
- [ ] `app/page.tsx` — handle no Austin address match, loading, empty-signal, and API error states.
- [ ] `app/globals.css` — polish the laptop-first map/sidebar layout without adding mobile-specific scope.
- [ ] `tests/bdd/end-to-end-demo.spec.ts` — add a BDD acceptance check that a real Austin address search can drive the map and side panel using database-backed API responses.

### Phase 12: Documentation

- [ ] `README.md` — document what Permitbot does, the permit-only Growth Signal caveat, and local setup.
- [ ] `docs/data_sources.md` — document all signal and reference open datasets, IDs, purpose, and refresh caveats.
- [ ] `docs/scoring_methodology.md` — document Growth Signal formulas, filters, confidence, and limitations.
- [ ] `docs/deployment.md` — document Vercel, Supabase, GitHub secrets, migrations, and free-tier constraints.
- [ ] `docs/open_data_policy.md` — document the open-data-only rule and excluded paid/private data sources.
- [ ] `docs/plan.md` — keep this file synchronized with root `plan.md`.

### Phase 13: GitHub Actions and Deployment

- [ ] `.github/workflows/ci.yml` — add CI that installs dependencies and runs lint, typecheck, tests, and build.
- [ ] `.github/workflows/ci.yml` — enforce TypeScript and Python coverage at 50% minimum and fail if tests fail.
- [ ] `.github/workflows/vercel-deploy.yml` — add production Vercel deployment using GitHub secrets and Vercel CLI.
- [ ] `.github/workflows/ingest.yml` — add a manual-only ingestion workflow that never runs on every push.
- [ ] `docs/deployment.md` — document required GitHub secrets: `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`, and `DATABASE_URL`.

### Phase 14: Verification

- [ ] Local command — run JavaScript dependency installation and verify the lockfile is current.
- [ ] Local command — run Python test suite for pipeline pure logic.
- [ ] Local command — run Python coverage and verify it is at least 50%.
- [ ] Local command — run TypeScript unit/component tests.
- [ ] Local command — run TypeScript coverage and verify it is at least 50%.
- [ ] Local command — run BDD acceptance tests.
- [ ] Local command — run `npm run lint`.
- [ ] Local command — run `npm run typecheck`.
- [ ] Local command — run `npm run build`.
- [ ] Database command — run migrations against the configured Supabase database.
- [ ] Pipeline command — run reference ingestion against the configured Supabase database.
- [ ] Pipeline command — run permit ingestion against the configured Supabase database.
- [ ] Pipeline command — run area assignment and Growth Signal computation.
- [ ] Database command — verify nonzero row counts for addresses, street segments, ZIPs, tracts, permits, assignments, and Growth Signals.
- [ ] App check — verify address search returns only Austin candidates.
- [ ] App check — verify a selected Austin address displays street segment, ZIP, census tract, and ES/MS/HS areas.
- [ ] App check — verify map renders OSM tiles, selected marker, real permit points, and real overlays.
- [ ] App check — verify side panel displays real Growth Signal values and no buy/rent recommendation.

Todo list added to plan.md — say 'implement it all' to start.
