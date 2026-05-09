# Plan: Austin Property Area Signals

Date: 2026-05-09

This plan is based on `research.md`. It is not an implementation. No code should be written until this plan is reviewed, annotated, converted into a todo checklist, and explicitly approved.

## Goal

Build a laptop-first property intelligence app for Austin. A user enters an address or property, sees it on an OpenStreetMap-based map, and gets a side panel with transparent area signals for:

- the property's street segment / block
- the property's ZIP code
- the property's AISD elementary, middle, and high school service areas

The first proof of concept should use City of Austin `Issued Construction Permits` (`3syk-w9eu`) as the first quantitative signal source, then leave a clean path for crime, code enforcement, displacement, demographics, mobility, and school data.

## Product Language

Use neutral signal language instead of labeling a place as simply "growing" or "dying."

Recommended top-level panels:

- `Development Momentum`
- `Distress Risk`
- `Displacement Pressure`
- `Livability Context`
- `Data Confidence`

Why: "growing" can mean healthy reinvestment, speculative displacement, or commercial turnover. "Dying" can conflate disinvestment, affordability, lower permit volume, or lower reporting. The UI should expose signals, not make a buy/rent recommendation.

## Architecture

Use a conventional geospatial analytics stack:

- PostgreSQL + PostGIS for canonical storage and spatial joins
- Python for ingestion, feature engineering, and modeling
- pandas / NumPy for transformations
- scikit-learn for later supervised or clustering models
- FastAPI for serving property lookup and signal summaries
- React + TypeScript + Leaflet for the laptop-first map UI
- Docker Compose for local development

Why this approach:

- PostGIS is the right place to store permit points, ZIP polygons, school boundaries, street segments, and generated metrics.
- Python keeps the data and model pipeline close to pandas/scikit-learn.
- FastAPI avoids a second backend language and gives typed API contracts.
- Leaflet works well with OSM tiles and vector overlays.
- React/TypeScript is a practical default for a side-panel-heavy map UI.

## Data Scope

### Phase 1 POC Dataset

Start with:

- `Issued Construction Permits`, Socrata id `3syk-w9eu`

Initial permit filter:

```sql
permittype = 'BP'
AND issue_date IS NOT NULL
AND latitude IS NOT NULL
AND longitude IS NOT NULL
AND calendar_year_issued >= 2015
```

The broader table can be ingested later, but the POC should avoid naive sums across electrical, mechanical, plumbing, driveway, sidewalk, and trade permits.

### Phase 2 Dataset Additions

Add these after the permit POC is stable:

- Crime Reports (`fdj4-gpfu`)
- Austin Code Complaint Cases (`6wtj-zbtb`)
- Repeat Offender Property datasets (`5yf8-fm7j`, `ge82-ij4h`, `86z9-i27i`, `cdze-ufp8`)
- City of Austin Displacement Risk Areas 2022 (`t8nv-zcp9`)
- ZIP boundaries (`24mx-z6v2`)
- AISD attendance areas from AISD GIS downloads
- Sidewalks (`vchz-d9ng`)
- Crash data (`y2wy-tgr5`)
- Affordable Housing Inventory (`ifzc-3xz8`)

## Spatial Model

### Property Point

The user-entered address resolves to a point. That point is spatially joined to:

- nearest street segment
- ZIP polygon
- AISD ES/MS/HS attendance polygons
- optional neighborhood/council/tract polygons for context

### Street Segment / Block

Define a block as a street segment between intersections. For POC, use a line geometry table and assign permit points to the nearest segment within a threshold.

Example query shape:

```sql
SELECT s.segment_id
FROM street_segments s
ORDER BY s.geom <-> ST_SetSRID(ST_Point(:lon, :lat), 4326)::geography
LIMIT 1;
```

Implementation note: final query should use a projected geometry or geography consistently, and should apply a maximum distance threshold so bad geocodes do not attach to unrelated segments.

### ZIP and School Areas

Use polygon containment:

```sql
SELECT zipcode
FROM zip_boundaries
WHERE ST_Contains(geom, ST_SetSRID(ST_Point(:lon, :lat), 4326));
```

AISD should be stored with a school-year version:

```sql
SELECT school_level, school_name, school_year
FROM school_attendance_areas
WHERE school_year = :active_school_year
AND ST_Contains(geom, ST_SetSRID(ST_Point(:lon, :lat), 4326));
```

## Database Design

Use separate raw, normalized, spatial, and analytics tables.

### Proposed Schemas

```sql
CREATE SCHEMA IF NOT EXISTS raw;
CREATE SCHEMA IF NOT EXISTS core;
CREATE SCHEMA IF NOT EXISTS geo;
CREATE SCHEMA IF NOT EXISTS analytics;
```

### Raw Permit Table

```sql
CREATE TABLE raw.austin_issued_construction_permits (
  source_id text NOT NULL DEFAULT '3syk-w9eu',
  source_row_id text,
  ingested_at timestamptz NOT NULL DEFAULT now(),
  payload jsonb NOT NULL,
  PRIMARY KEY (source_id, source_row_id)
);
```

Why: keep the original Socrata row payload for reproducibility and schema drift.

### Normalized Permit Table

```sql
CREATE TABLE core.permits (
  permit_number text PRIMARY KEY,
  project_id bigint,
  master_permit_number bigint,
  permit_type text NOT NULL,
  permit_type_description text,
  permit_class text,
  permit_class_mapped text,
  work_class text,
  issue_date date,
  applied_date date,
  completed_date date,
  status_current text,
  original_address text,
  original_zip text,
  tcad_id text,
  total_job_valuation numeric,
  total_new_add_sqft numeric,
  remodel_repair_sqft numeric,
  housing_units numeric,
  latitude double precision,
  longitude double precision,
  geom geometry(Point, 4326),
  source_updated_at timestamptz,
  ingested_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX permits_geom_idx ON core.permits USING gist (geom);
CREATE INDEX permits_issue_date_idx ON core.permits (issue_date);
CREATE INDEX permits_original_zip_idx ON core.permits (original_zip);
CREATE INDEX permits_work_class_idx ON core.permits (work_class);
```

### Geography Tables

```sql
CREATE TABLE geo.zip_boundaries (
  zipcode text PRIMARY KEY,
  geom geometry(MultiPolygon, 4326),
  source_id text,
  source_updated_at timestamptz
);

CREATE TABLE geo.school_attendance_areas (
  id bigserial PRIMARY KEY,
  school_year text NOT NULL,
  school_level text NOT NULL,
  school_name text NOT NULL,
  geom geometry(MultiPolygon, 4326)
);

CREATE TABLE geo.street_segments (
  segment_id bigserial PRIMARY KEY,
  street_name text,
  from_node_id text,
  to_node_id text,
  geom geometry(LineString, 4326)
);
```

### Analytics Tables

```sql
CREATE TABLE analytics.area_period_metrics (
  area_type text NOT NULL,
  area_id text NOT NULL,
  period_start date NOT NULL,
  period_end date NOT NULL,
  metric_name text NOT NULL,
  metric_value numeric NOT NULL,
  source_count integer NOT NULL,
  generated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (area_type, area_id, period_start, period_end, metric_name)
);

CREATE TABLE analytics.area_signal_scores (
  area_type text NOT NULL,
  area_id text NOT NULL,
  signal_name text NOT NULL,
  score numeric NOT NULL,
  direction text NOT NULL,
  confidence numeric NOT NULL,
  explanation jsonb NOT NULL,
  generated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (area_type, area_id, signal_name)
);
```

Why: metrics are factual aggregates; scores are derived interpretations. Keeping them separate makes the app easier to audit.

## Permit Feature Engineering

For each area type (`street_segment`, `zipcode`, `school_area`), compute rolling features:

- permits issued in last 6 months
- permits issued in last 12 months
- permits issued per year for last 5 years
- residential new permits
- residential addition/remodel permits
- commercial new permits
- demolition permits
- total and median valuation
- total and median new square footage
- permit acceleration: recent 12 months vs prior 36-month average
- share of permits with missing valuation/square footage
- confidence based on count and geocode quality

Example feature function:

```python
def compute_growth_ratio(recent_count: float, baseline_annual_count: float) -> float:
    if baseline_annual_count <= 0:
        return 1.0 if recent_count > 0 else 0.0
    return recent_count / baseline_annual_count
```

Avoid using raw `housing_units` as a primary POC feature until outliers are investigated.

## Scoring Approach

Start with transparent deterministic scores before using machine learning.

### Development Momentum Score

Inputs:

- recent permit count percentile
- permit acceleration
- new construction count
- remodel/addition count
- valuation trend after winsorization
- new square footage trend

Example scoring sketch:

```python
def weighted_score(features: dict[str, float], weights: dict[str, float]) -> float:
    numerator = sum(features[name] * weight for name, weight in weights.items())
    denominator = sum(abs(weight) for weight in weights.values())
    return numerator / denominator if denominator else 0.0
```

The real implementation should use typed dataclasses or Pydantic models rather than unstructured dictionaries if this becomes production code.

### Confidence Score

Inputs:

- source event count
- percent geocoded
- boundary vintage
- missing field rate
- area size and denominator availability

Example:

```python
def confidence_from_counts(source_count: int, minimum_count: int) -> float:
    if minimum_count <= 0:
        return 0.0
    return min(source_count / minimum_count, 1.0)
```

### Later Machine Learning

Use scikit-learn only after defining a target label. Candidate labels:

- future permit acceleration
- future code complaint increase
- future reported crime decrease/increase
- future displacement-risk change
- future assessed value / rent movement if a legal data source is added

Do not train a vague "growing/dying" classifier without a measurable target.

## API Design

FastAPI should expose a small set of read-only endpoints first.

### Address Lookup

```http
GET /api/properties/lookup?query=1400%20Congress%20Ave%20Austin%20TX
```

Returns geocoded candidates. For POC, this can wrap a local address table or carefully throttled geocoder. Production should not rely on public Nominatim autocomplete.

### Property Summary

```http
GET /api/properties/summary?lat=30.2672&lon=-97.7431
```

Response shape:

```json
{
  "property": {
    "latitude": 30.2672,
    "longitude": -97.7431
  },
  "areas": {
    "street_segment": {
      "id": "segment-123",
      "label": "Example St between A St and B St"
    },
    "zipcode": {
      "id": "78701",
      "label": "78701"
    },
    "schools": [
      {
        "level": "ES",
        "name": "Example Elementary",
        "school_year": "2026-27"
      }
    ]
  },
  "signals": [
    {
      "area_type": "zipcode",
      "area_id": "78701",
      "signal_name": "Development Momentum",
      "score": 0.72,
      "direction": "rising",
      "confidence": 0.84,
      "drivers": [
        "permit count above citywide median",
        "recent permits above three-year baseline"
      ]
    }
  ]
}
```

## UI Plan

Laptop-first layout:

- full-height map on the left
- fixed side panel on the right, 380-460px wide
- search input at top of side panel
- selected property marker on map
- highlighted street segment, ZIP, and school polygons
- signal cards grouped by area
- explicit data confidence and source vintage labels

Suggested component structure:

```text
src/
  App.tsx
  api/client.ts
  components/AddressSearch.tsx
  components/MapView.tsx
  components/SignalPanel.tsx
  components/SignalCard.tsx
  components/AreaSummary.tsx
  types/api.ts
```

Important UI rules:

- Do not tell users "buy" or "do not buy."
- Show "why" for every score.
- Show source dates.
- Warn when a block-level signal has low event counts.
- Display ES/MS/HS service areas separately.

## OpenStreetMap Policy Plan

POC:

- Leaflet with OSM tiles may be acceptable for local development.
- Show `© OpenStreetMap contributors` attribution.
- Do not implement public Nominatim autocomplete.
- Cache geocoding results.
- Throttle requests if Nominatim is used manually.

Production:

- Use a commercial OSM-derived tile/geocoding provider, self-hosted tiles/Nominatim, or a local address locator.
- Do not depend on public OSM services for regular end-user traffic.

## Proposed File Paths for Implementation

These are the files that should be created or modified during the eventual implementation phase. They are listed now for plan review only.

### Project Root

- `README.md` - project overview, local setup, data-source caveats
- `.env.example` - required environment variables
- `docker-compose.yml` - PostgreSQL/PostGIS plus backend/frontend services
- `pyproject.toml` - Python dependencies and tooling
- `package.json` - frontend scripts and dependencies if using a single repo root

### Database

- `db/migrations/001_enable_postgis.sql`
- `db/migrations/002_create_raw_tables.sql`
- `db/migrations/003_create_core_tables.sql`
- `db/migrations/004_create_geo_tables.sql`
- `db/migrations/005_create_analytics_tables.sql`

### Python Backend and Pipeline

- `backend/app/main.py` - FastAPI app bootstrap
- `backend/app/api/properties.py` - property lookup and summary endpoints
- `backend/app/core/config.py` - settings
- `backend/app/db/session.py` - database connection/session management
- `backend/app/models/schemas.py` - Pydantic response models
- `backend/app/services/geocoder.py` - local or provider geocoding abstraction
- `backend/app/services/signals.py` - signal lookup and explanation assembly
- `backend/app/services/spatial.py` - point-to-area and nearest-segment queries
- `backend/pipeline/socrata_client.py` - Socrata paging and query helper
- `backend/pipeline/ingest_permits.py` - raw and normalized permit ingestion
- `backend/pipeline/ingest_boundaries.py` - ZIP/school/street boundary loaders
- `backend/pipeline/features/permits.py` - permit feature generation
- `backend/pipeline/features/scores.py` - deterministic scoring
- `backend/tests/test_permit_features.py` - feature logic tests
- `backend/tests/test_signal_scoring.py` - scoring tests

### Frontend

- `frontend/package.json` - frontend dependencies and scripts
- `frontend/index.html`
- `frontend/src/App.tsx`
- `frontend/src/api/client.ts`
- `frontend/src/components/AddressSearch.tsx`
- `frontend/src/components/MapView.tsx`
- `frontend/src/components/SignalPanel.tsx`
- `frontend/src/components/SignalCard.tsx`
- `frontend/src/components/AreaSummary.tsx`
- `frontend/src/types/api.ts`
- `frontend/src/styles.css`

### Documentation

- `docs/data_sources.md` - dataset ids, refresh cadence, caveats
- `docs/scoring_methodology.md` - formulas and interpretation
- `docs/osm_policy.md` - map/geocoder usage constraints

## Implementation Phases

### Phase A: Project Skeleton

Create the repo structure, Docker Compose, backend app, frontend app, and PostGIS database.

Output:

- local backend health endpoint
- local frontend shell
- database with PostGIS enabled

### Phase B: Permit Ingestion

Ingest `3syk-w9eu` with Socrata paging into `raw.austin_issued_construction_permits`, normalize building permits into `core.permits`, and track ingestion time.

Output:

- repeatable permit ingestion command
- normalized permit table with spatial index
- row counts and data-quality checks

### Phase C: Boundary Ingestion

Load ZIP boundaries, street segments, and AISD school areas.

Output:

- property point can resolve to ZIP
- property point can resolve to school areas
- property point can resolve to nearest street segment

### Phase D: Permit Metrics and Signals

Generate area-period permit metrics and deterministic development momentum scores.

Output:

- metrics for street segments, ZIPs, and school service areas
- development momentum score with explanation
- confidence score

### Phase E: Backend API

Expose property summary endpoint that returns geography assignments and signal summaries.

Output:

- `/api/properties/summary`
- response includes area labels, signals, drivers, confidence, and source dates

### Phase F: Laptop Map UI

Build the laptop-first UI with map, address search, selected property marker, area overlays, and side panel signal cards.

Output:

- user can enter/select a property
- map highlights relevant areas
- side panel shows block, ZIP, and school signals

### Phase G: Validation

Run type checks, backend tests, frontend build, and spot-check known Austin properties.

Output:

- documented validation commands
- basic test coverage for feature calculations and scoring
- known caveats documented

## Trade-Offs and Alternatives

### FastAPI vs Django

FastAPI is lighter and better suited for typed read APIs and data-science-adjacent services. Django is stronger if the app needs authentication, admin workflows, and complex relational CRUD. For the POC, FastAPI is the better fit.

### Leaflet vs MapLibre

Leaflet is simpler for a 2D property map with polygons and markers. MapLibre is stronger for vector tiles and high-performance styling. Start with Leaflet unless the plan shifts toward large vector-tile overlays.

### Deterministic Scores vs ML

Deterministic scores are easier to explain and audit. ML may be useful later, but only after defining a measurable target. Start deterministic.

### Public Nominatim vs Local Address Index

Public Nominatim is convenient but unsuitable for autocomplete or production traffic. A local Austin address index is safer and more controllable. For the POC, use a minimal local address lookup or a provider abstraction.

### ZIP vs Census Tract

Users understand ZIP codes, but tracts are better for demographic analysis. Support ZIP for user-facing summary and keep tracts/block groups as internal feature geographies.

### Street Segment Assignment

Nearest-segment assignment is practical but can be wrong near intersections, highways, or bad geocodes. Parcel-based assignment would be better but requires reliable parcel geometry. Start with nearest segment plus distance thresholds and confidence flags.

## Questions for Review

- Should the first implementation deliver the full map UI, or should it first deliver the database and permit scoring pipeline?
- Should school service areas be ES/MS/HS separately in the first UI?
- Should the project include paid/private property value or rent data later, or remain open-data-only?
- Should "Development Momentum" be the first score name, or do you prefer "Growth Signal"?
- Should the first geography be street segment, ZIP, and school area exactly, or should census tract be included because displacement/demographic datasets are tract-based?

Plan written to plan.md — please open it, add inline notes at any problem spots, and say 'address the notes' when ready. Do not implement yet.
