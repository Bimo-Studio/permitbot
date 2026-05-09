# Research: Austin Neighborhood Growth / Decline Signal Project

Date: 2026-05-09

This is Phase 1 research only. I did not implement code or create a project plan.

## User Goal

Build an application that lets an end user enter a property in Austin, view it on an OpenStreetMap-based laptop-first map, and see a side panel with quantitative signals for:

- the block or street segment containing the property
- the ZIP code containing the property
- the relevant AISD school service area

The product goal is to help a buyer or renter understand whether an area appears to be gaining momentum, stagnating, declining, or undergoing risky displacement/gentrification pressure.

The requested initial proof of concept is the City of Austin `Issued Construction Permits` dataset:

- Dataset name: Issued Construction Permits
- Socrata id: `3syk-w9eu`
- Portal page: `https://data.austintexas.gov/Building-and-Development/Issued-Construction-Permits/3syk-w9eu/about_data`
- API endpoint pattern: `https://data.austintexas.gov/resource/3syk-w9eu.json`

The preferred stack is PostgreSQL, Python, NumPy, pandas, and scikit-learn. Because the problem is inherently spatial, PostGIS should be considered part of PostgreSQL for planning purposes.

## Workspace / Codebase State

There is no application code in `/workspace` yet. The workspace currently contains workflow and staging directories plus `AGENTS.md`. Relevant implementation research is therefore external data/API research rather than codebase reading.

Only `/workspace/AGENTS.md` applies. It requires this Research -> Plan -> Annotation -> Todo -> Implement workflow. No implementation should happen until after `plan.md` is written, reviewed, annotated, and explicitly approved.

## Austin Open Data Catalog Scan

I queried the Socrata catalog endpoint for `data.austintexas.gov`:

`https://api.us.socrata.com/api/catalog/v1?domains=data.austintexas.gov&search_context=data.austintexas.gov&limit=1000&only=datasets`

As of this research run, the catalog returned 706 public datasets. Category distribution from the catalog metadata:

| Category | Dataset Count |
|---|---:|
| City Government | 126 |
| Public Safety | 113 |
| Transportation and Mobility | 80 |
| Utilities and City Services | 71 |
| Locations and Maps | 66 |
| Uncategorized | 54 |
| Recreation and Culture | 40 |
| Health and Community Services | 36 |
| City Infrastructure | 36 |
| Building and Development | 25 |
| Environment | 23 |
| Budget and Finance | 22 |
| Housing and Real Estate | 13 |
| Government | 1 |

I filtered the catalog metadata by names, descriptions, tags, and columns related to permits, building, construction, zoning, land use, code violations, crime, 311, demographics, population, ZIP, boundaries, schools, transit, traffic, sidewalks, parks, parcels, TCAD, housing, real estate, rent, affordability, fire, and EMS. 516 datasets matched at least one broad term, but many were false positives or low-value operational datasets. The most relevant datasets are grouped below.

Important caveat: this was a full metadata scan of the portal, not a manual inspection of every row in every dataset. The portal changes continuously, and several datasets are stale, test datasets, deprecated, duplicated, or operational dashboards that should not be used for inference.

## High-Value Austin Datasets

### Construction, Development, Zoning, and Land Use

These are the strongest direct signals for new investment, redevelopment, densification, demolition, and regulatory activity.

| Dataset | ID | Why It Matters |
|---|---|---|
| Issued Construction Permits | `3syk-w9eu` | Best POC source. Daily, very large, geocoded permits with issue dates, valuation, square footage, units, permit class, work class, ZIP, TCAD ID, and coordinates. |
| Issued Building Permits | `quv8-5ckq` | Point shapefile-style building permit dataset from 2006 onward, updated quarterly. Useful cross-check against `3syk-w9eu`. |
| Plan Review Cases | `n8ck-xkda` | Earlier pipeline signal before permits issue. Useful for future-looking development momentum. |
| Site Plan Cases | `mavg-96ck` | Major development signal, often useful before construction activity appears. |
| Subdivision Cases | `s7gx-9m54` | Indicates parcelization and future land development potential. |
| Zoning Cases | `edir-dcnf` | Rezoning pressure, land use change, and development entitlement signal. |
| Zoning By Address | `nbzi-qabm` | Current zoning lookup layer for a property or area. |
| Residential Demolitions dataset | `x6mf-sksh` | Strong redevelopment/displacement-risk signal, especially when paired with rising permits. |
| Issued Tree Permits | `ac2h-ha3r` | Redevelopment proxy and environmental/lot-change signal. |
| Land Use Inventory Detailed | `7vsm-dvxg` / `6qkk-xgys` | Baseline land use and parcel context. |
| Lot Line | `r8wq-g5d8` | Parcel geometry for spatial joins and denominator calculations. |
| Building Footprints Year 2013 | `3qcc-8uhz` | Baseline built-form footprint layer. |
| `UTILITIESCOMMUNICATION_building_footprints_2017` | `agk7-8sjm` | Later building footprint layer; useful for historic built-form change if comparable. |

### Housing, Affordability, Displacement, and Demographics

These are necessary because "growth" can mean healthy investment, speculative pressure, or displacement. A buyer/renter tool should avoid calling an area "good" only because vulnerable residents are being displaced.

| Dataset | ID | Why It Matters |
|---|---|---|
| City of Austin Displacement Risk Areas 2022 | `t8nv-zcp9` | Austin-specific displacement risk typology using vulnerable populations, market appreciation, and demographic change. |
| City of Austin Displacement Risk Areas 2022 within 1 mile of Project Connect | `sbem-y2bn` | Same concept constrained to transit investment areas. |
| Project Connect Anti-Displacement Dashboard Data 2020 | `e2tx-ut3v` | Block/tract-level population and housing indicators used in displacement work. |
| Affordable Housing Inventory | `ifzc-3xz8` | Income-restricted housing inventory; important for affordability and displacement context. |
| 2026 City of Austin Demographic Profiles | `2h5e-ntwt` | Recent demographic profile source. |
| 2025 City of Austin Demographic Profiles | `k4ue-wizq` | Prior demographic profile source for comparison. |
| 2023 City of Austin Demographic Profiles | `due5-5z9i` | Earlier demographic profile source. |
| DTI Population and Employment Forecasts 2060 Dataset | `3vyq-vrp6` | Long-horizon population and employment growth expectations. |
| DTI Population and Employment Forecasts 2060 Shapefile | `fyyy-e3ut` | Spatial version of the forecast dataset. |
| COA Population by Age and Sex | `ic3t-m53x` | Coarse demographic context. |
| Demographics Stats at a Glance | `ghdg-7f7z` | High-level demographic indicators. |
| Land Database 2021 | `kk8y-6cmt` | Parcel/land data relevant to housing and land use analysis. |

### Crime, Code Enforcement, Disorder, and Safety

The literature strongly supports crime and code enforcement as local signals, especially at microplaces like blockfaces and street segments.

| Dataset | ID | Why It Matters |
|---|---|---|
| Crime Reports | `fdj4-gpfu` | Report-written APD incidents from 2003-present, weekly updates. Strong block/area safety trend signal. |
| APD Computer Aided Dispatch Incidents | `22de-7rzg` | Calls-for-service style operational signal; useful but not equivalent to reported crime. |
| NIBRS Group A Offense Crimes | `i7fg-wrk5` | More formal crime category source. |
| APD Arrests | `9tem-ywan` | Enforcement signal, but must be handled carefully due to policing bias and enforcement intensity. |
| Hate Crimes 2017-2026 | `t99n-5ib4` | Safety and social-risk signal. |
| Austin Code Complaint Cases | `6wtj-zbtb` | Direct property disorder and code-violation trend signal. |
| Austin Code Task List | `ttd7-isgm` | Code officers assigned from 311 calls; bridge between 311 and enforcement. |
| Repeat Offender Property Activity | `5yf8-fm7j` | Rental/property quality and persistent violation signal. |
| Repeat Offender Property Deficiencies | `ge82-ij4h` | Defect-level signal for distressed rental properties. |
| Repeat Offender Registrations | `86z9-i27i` | Registered repeat offender properties. |
| Repeat Offender Violation Cases With Notice of Violation Link | `cdze-ufp8` | Violation case history. |

### Mobility, Infrastructure, and Access

Mobility improvements are growth signals, but they can also produce rent pressure and displacement. These datasets help distinguish accessibility improvements from development activity alone.

| Dataset | ID | Why It Matters |
|---|---|---|
| 311 Service Requests - Austin Transportation and Public Works | `38mr-dwji` | Resident-reported infrastructure pain points and maintenance demand. |
| Sidewalks | `vchz-d9ng` | Walkability and pedestrian infrastructure. |
| Completed Sidewalks 2021-09-30 | `6pkw-7frd` | Historic sidewalk completion baseline. |
| ATPW Right of Way Active Permits | `hyc6-zz9w` | Active right-of-way construction/disruption and infrastructure activity. |
| Roadway Work Zones | `qyfh-gwei` | Active construction and disruption. |
| Austin Crash Report Data - Crash Level Records | `y2wy-tgr5` | Safety and mobility risk signal. |
| Austin Crash Report Data - Crash Victim Demographic Records | `xecs-rpy9` | Crash severity and equity context. |
| Real-Time Traffic Incident Reports | `dx9v-zd7x` | Operational traffic/safety signal, not a long-term growth signal by itself. |
| Traffic Signals and Pedestrian Signals | `p53x-x73x` | Infrastructure inventory. |
| Core Transit Corridors | `fhpr-t9x2` | Transit-oriented development context. |
| DBETOD Subdistricts | `mubf-z45j` | Equitable transit-oriented development policy geography. |
| ETOD Overlay | `7tbm-t4ye` | Transit-oriented zoning/policy overlay. |
| ETOD Typologies | `hdpr-6wvx` | Transit area classification. |
| Urban Trails Network | `jdwm-wfps` | Connectivity and amenity signal. |
| Transportation bicycle facilities | `23hw-a95n` | Bike infrastructure and connectivity. |

### Boundaries, Geocoding, and Spatial Reference

These are required to make block, ZIP, neighborhood, school-area, and property-level joins reliable.

| Dataset | ID | Why It Matters |
|---|---|---|
| Addresses | `9s7j-tygf` | Local address reference for geocoding and validation. |
| Boundaries: US Zip Codes | `24mx-z6v2` | ZIP polygon assignment; should be spatially joined rather than trusting row ZIP fields alone. |
| Boundaries: State of Texas Census Tracts (2020 Census) | `bh9b-sg93` | Demographic normalization and ACS joins. |
| Boundaries: City of Austin Neighborhoods | `inrm-c3ee` | Neighborhood label context, though user requested block/ZIP/school rather than neighborhood. |
| Boundaries: City of Austin Council Districts | `w3v2-cj58` | Political geography context. |
| Boundaries: City of Austin Jurisdiction and Regulatory Boundaries | `is88-7234` | City limits and regulatory context. |
| BOUNDARIES_jurisdictions | `vnwj-xmz9` | Jurisdiction layer, likely overlapping/related to the above. |
| Census Block Groups | `dwa9-qvcr` | Public-safety/census block group reference. |

### Schools and AISD Service Areas

The Austin Open Data portal has school-related datasets, but the best source for AISD attendance/service areas appears to be AISD itself.

AISD official GIS page:

`https://www.austinisd.org/planning-asset-management/school-maps-gis`

The page says AISD GIS supports:

- school and district facility locations
- student demographic projections
- attendance area boundaries
- academic programming locations
- community partner/service locations

The same page provides GIS downloads for:

- school and facility locations, updated September 2022
- district boundary
- trustee boundaries, revised August 2022
- ES attendance areas, school year 2022-23
- MS attendance areas, school year 2022-23
- HS attendance areas, school year 2022-23

Important current-status issue: AISD boundaries are in flux. AISD announced on November 21, 2025 that trustees approved closures/consolidations of 10 schools beginning in the 2026-27 school year, while broader districtwide boundary changes and additional consolidations were moved into future work. On April 26, 2026, AISD said it was suspending additional school closures while narrowing focus to districtwide boundary work. A production app must either:

- explicitly label school-area data by school year, or
- ingest the latest AISD boundary release before launch.

There are older ArcGIS Safe Routes to School attendance boundary services from 2019/2020. They are useful for historical SRTS analysis but should not be treated as current AISD assignment boundaries without verification.

### Amenities, Parks, Food, and Community Services

These are softer growth/quality-of-life signals. They are useful when paired with crime, accessibility, and housing cost pressure.

| Dataset | ID | Why It Matters |
|---|---|---|
| Food Establishment Inspection Scores | `ecmv-9xxi` | Restaurant/business presence and turnover proxy, though only inspected establishments. |
| Number of City-supported fresh food access points | `rmsu-6rjn` | Food access. |
| Austin Public Library Locations | `tc36-hn4j` | Civic amenity. |
| Recreation Centers | `8dff-2vkt` | Community amenity. |
| BOUNDARIES_city_of_austin_parks | `v8hw-gz65` | Park proximity and access. |
| % Access to Parks | `8zqp-nigd` | Park access indicator. |
| Pool Inspections | `peux-uuwu` | Minor amenity/inspection signal. |
| Cultural asset mapping datasets | `xjsy-32ku`, `8kxv-xaqc` | Cultural amenity context, but may be stale. |

### Utilities, Environmental Risk, and Physical Constraints

These can explain why some blocks are not growing, even if nearby areas are. They can also identify hidden risks for property decisions.

| Dataset | ID | Why It Matters |
|---|---|---|
| Austin Water - Residential Water Consumption | `sxk7-7k6z` | ZIP-level utility usage and occupancy proxy. |
| Austin Water - Commercial Water Consumption | `5h9c-wmds` | Commercial activity proxy. |
| Austin Water - Water Service Connection Count By Zip Code | `uizf-mcbc` | Growth in utility connections by ZIP. |
| Austin Water - Wastewater Service Connection Count By Zip Code | `6v99-vnq3` | Growth in wastewater connections by ZIP. |
| Residential Water Meter Inspections/Installs | `6zcs-yxb3` | Point/activity signal for new or changed service. |
| 2025 Single Family Green Building Map | `mvmv-cuj9` | Green/residential investment signal. |
| 2024 Single Family Green Building Map | `s2pw-6c89` | Prior green-building signal. |
| Green Building Energy Code Compliance | `i7vh-fpaj` | Building quality/compliance signal. |
| Impervious Cover 2023 | `scq8-2cei` | Built-surface intensity and environmental runoff context. |
| Impervious Cover 2021 | `3ya2-4qw9` | Earlier comparison layer. |
| Greater Austin Fully Developed Floodplain | `pjz8-kff2` | Development constraint and property risk. |
| Watershed Boundaries | `2829-xbvw` | Environmental and drainage context. |
| Brownfield Site List | `22wq-47zy` | Environmental contamination risk. |
| Pollutant Report Socrata | `7i3t-mxrc` | Reported pollutant spill locations. |

## Issued Construction Permits POC Findings

### Dataset Description

The permit dataset includes building, electrical, mechanical, plumbing, driveway, and sidewalk permits issued by the City of Austin. It includes issue date, location, council district, expiration date, work description, square footage, valuation, and unit fields. It is described as BLDS-compliant.

Austin Development Services warns that the database is continuously updated, may differ from official department data, and can produce different totals depending on when reports are run.

### Size and Date Coverage

Direct Socrata aggregate query results during this research run:

- all rows: 2,354,933
- minimum `issue_date`: 1921-09-20
- maximum `issue_date`: 2026-05-09
- minimum `applieddate`: 1921-09-20
- maximum `applieddate`: 2026-05-08

For records from 2015 onward across all permit types:

- total rows: 745,728
- rows with latitude: 650,147
- rows with longitude: 650,147
- rows with original ZIP: 743,985
- rows with TCAD ID: 739,001

For `permittype = 'BP'` from 2015 onward:

- rows: 166,054
- distinct permit numbers: 166,054
- distinct project IDs: 166,054
- distinct master permit numbers: 130,966

This indicates that using `permittype = 'BP'` gives cleaner building-permit records than treating every permit row equally. The broader table still matters because trade permits, driveway/sidewalk permits, and utility-related permits can signal activity, but they should not be summed naively with building permits.

### Useful Columns

High-value fields for the POC:

- identity: `permit_number`, `project_id`, `masterpermitnum`
- location: `location`, `latitude`, `longitude`, `original_address1`, `original_city`, `original_state`, `original_zip`, `tcad_id`
- date: `applieddate`, `issue_date`, `calendar_year_issued`, `fiscal_year_issued`, `completed_date`, `statusdate`
- type: `permittype`, `permit_type_desc`, `permit_class`, `permit_class_mapped`, `work_class`, `certificate_of_occupancy`
- intensity: `total_job_valuation`, `building_valuation`, `total_valuation_remodel`, `total_new_add_sqft`, `remodel_repair_sqft`, `total_existing_bldg_sqft`, `total_lot_sq_ft`, `housing_units`, `number_of_floors`
- context: `council_district`, computed region columns, `description`, `property_legal_description`

### Data Quality Concerns

The raw table should not be used as a simple "sum everything" dataset.

Observed issues:

- The table includes multiple permit types, not only building permits.
- Trade permits and related permits can create duplicate signals around a single project.
- `housing_units` appears unreliable for naive annual summation; 2024 building-permit-only raw sums produced implausibly large unit totals.
- Some records are missing latitude/longitude.
- Some records have broad/range addresses, such as `300-417 HACKBERRY LN`, which may not map to a single parcel.
- Valuation fields can have extreme outliers and should be winsorized or modeled robustly.
- Permit issuance is not the same as completion. `completed_date` and `status_current` need separate handling.
- Daily updates mean reproducibility requires storing ingestion time and source revision time.

### POC Signal Potential

`3syk-w9eu` is strong enough for a first "development momentum" signal, especially if the first version focuses on:

- number of new building permits by year
- number of remodel/addition permits by year
- demolition permits by year
- total and median project valuation
- total and median new square footage
- permit activity acceleration over 1, 3, and 5 year windows
- recent activity within 6 or 12 months
- spatial concentration around the property block/street segment

It is not strong enough by itself to classify a neighborhood as "growing" or "dying." It measures investment activity, not affordability, safety, school quality, business health, vacancy, resident displacement, or resident satisfaction.

## Geographic Units

### Property

A user-entered property should be geocoded to a point and then joined to polygons and nearby street segments. Austin's `Addresses` dataset can be used as a local address reference. Nominatim can support early demos, but production autocomplete/search should not rely on the public OSM-hosted Nominatim service.

### Block / Street Segment

The strongest scientific literature on permits, crime, and code enforcement uses microplaces such as blockfaces or street segments. For this app, "block along a street" should be defined as a street segment between intersections, optionally with side-of-street handling if parcel geometry permits it.

Practical spatial approach for later planning:

- source street centerlines from Austin mobility data, OSM, or another official road network
- split streets at intersections
- assign permits/crime/code cases by point-in-buffer or nearest segment
- compute rates per parcel, address, lot area, or housing unit where denominators exist
- show confidence lower when the segment has too few parcels/events

### ZIP Code

ZIP is useful to users and easy to explain, but ZIP codes are coarse and can hide block-level changes. Use City ZIP polygons (`24mx-z6v2`) for spatial assignment instead of relying only on row ZIP fields.

### AISD School Service Area

"School service area" is not one geography. A property can map to elementary, middle, and high school attendance areas, and boundary school years matter. The app should show the service-area vintage and probably display ES/MS/HS separately unless the user intentionally wants only one level.

AISD's own GIS downloads are the authoritative source found during research, but the available page currently references 2022-23 attendance areas. Current 2026-27 transition/boundary status needs to be verified before final implementation.

## Evidence Base: What Predicts Growth, Decline, and Risk?

There is no single scientific proof that a neighborhood is "growing" or "dying." The credible approach is to model several signal families, normalize them locally, track them over time, and show confidence/limitations. Strong evidence supports the following signal families.

### 1. Private Investment and Building Permits

Building permits are a defensible proxy for private investment and physical improvement. The strongest fit to this project is the literature that uses building permits at blockface/street-segment scale.

Evidence:

- Lacoe, Bostic, and Acolin, "Crime and private investment in urban neighborhoods," Journal of Urban Economics, 2018. The study uses blockface-level crime and building permit data in Chicago and Los Angeles and finds that private investment decreases on blocks where crime increases in the prior year.
- Tillyer, Acolin, and Walter, "Place-Based Improvements for Public Safety: Private Investment, Public Code Enforcement, and Changes in Crime at Microplaces across Six U.S. Cities," Justice Quarterly, 2023. The study finds building permits and code enforcement significantly associated with reductions in crime on street segments, with spatial diffusion to nearby segments.
- Acolin et al., "Spatial spillover effects of crime on private investment at nearby micro-places," Urban Studies, 2022. Uses crime and building permits at blockface level across six cities and finds higher crime associated with lower subsequent investment.

Implication: Permit trends should be a core score component, especially at street-segment scale, but should be paired with safety and code enforcement signals.

### 2. Crime and Safety

Crime is a strong disamenity in housing markets and affects investment decisions. Crime effects are local, vary by type, and can be mediated by reporting and enforcement patterns.

Evidence:

- Gibbons and Machin, "Valuing school quality, better transport, and lower crime: evidence from house prices," Oxford Review of Economic Policy, 2008, reviews evidence that housing prices reflect localized amenities and disamenities including crime.
- Autor, Palmer, and Pathak, "Gentrification and the Amenity Value of Crime Reductions," NBER, 2017, finds public safety improvements can be capitalized into property values.
- Crime/private investment papers above connect crime trends to permit activity at microplaces.

Implication: Use reported crime trends by type and severity, not just raw incident counts. Show separate safety direction rather than hiding it inside a single opaque score.

### 3. Code Enforcement, Physical Disorder, Vacancy, Foreclosure

Physical disorder and unresolved property issues are classic decline/disinvestment signals. Austin has strong code datasets; foreclosure/vacancy may require outside sources.

Evidence:

- Tillyer et al. finds code enforcement and private investment associated with lower crime at street segments.
- Williams, Galster, and Verma, "Home Foreclosures as Early Warning Indicator of Neighborhood Decline," Journal of the American Planning Association, 2014, evaluates foreclosure as a frequently updated early-warning indicator for future neighborhood market and quality-of-life changes.
- HUD's Neighborhood Change Indicators project explicitly includes blight, vacancy, abandonment, assisted housing, and community development framing in its machine-learning neighborhood change work.

Implication: Austin Code Complaint Cases, Repeat Offender Property datasets, 311 task lists, and future foreclosure/vacancy data should become a major "decline risk" pillar.

### 4. Housing Market Pressure and Displacement Risk

Rising property values can mean healthy demand, speculative pressure, or displacement risk. A buyer/renter product should distinguish "momentum" from "affordability stress."

Evidence:

- Austin's Uprooted project and City displacement risk datasets use a three-part analysis: vulnerable populations, residential market appreciation, and demographic change.
- City of Austin Displacement Risk Areas 2022 carries the same conceptual categories: Vulnerable, Active Displacement Risk, Chronic Displacement Risk, and Historic Displacement.
- Bunten, Preis, and Aron-Dine, "Re-measuring gentrification," Urban Studies, 2024, proposes an expectations-based measure using the gap between house value percentile and income percentile and finds that gap predicts future income growth.

Implication: A "growing" label should be separated from "displacement pressure" and "affordability risk." Without that distinction, the product could mislead users or encode harmful assumptions.

### 5. Demographic Change

Demographics are slower-moving than permits or 311 calls, but they are central to gentrification/displacement definitions.

Evidence:

- Uprooted compares tracts using vulnerability, demographic change, and housing market change.
- HUD Neighborhood Change Indicators trained machine-learning models on administrative and other data to predict future binary neighborhood outcomes.
- Glaeser, Kim, and Luca, "Nowcasting Gentrification: Using Yelp Data to Quantify Neighborhood Change," NBER/AEA Papers and Proceedings, finds gentrification associated with changes in education, age, and racial composition and with changes in local business categories.

Implication: Use ACS/census demographic features at tract/block group level, but don't over-interpret annual movement because ACS estimates lag and have uncertainty.

### 6. Business and Amenity Change

Amenities matter, but they are context-dependent. New restaurants and cafes can be a leading signal of neighborhood change, while parks and greenways can be positive or negative depending on safety.

Evidence:

- Glaeser, Kim, and Luca finds local business changes, especially groceries, cafes, restaurants, bars, and coffee shops, can help nowcast or forecast gentrification and housing price changes.
- Troy and Grove, "Property values, parks, and crime: A hedonic analysis in Baltimore," Landscape and Urban Planning, 2008, finds parks are positively valued where crime is below a threshold and negatively valued where crime is above it.
- Recent greenline/greenway property-value research similarly finds green amenities are moderated by crime conditions.

Implication: Amenities should be features, not standalone proof. Combine food/retail/parks/transit with crime and affordability context.

### 7. Schools, Transit, and Accessibility

School quality, transit accessibility, and public safety are repeatedly found to be capitalized into housing values, but causal estimates are difficult because these amenities correlate with income, zoning, and other neighborhood traits.

Evidence:

- Gibbons and Machin reviews evidence on school quality, transport, and crime in housing values.
- Debrezion, Pels, and Rietveld, "The Impact of Railway Stations on Residential and Commercial Property Value: A Meta-analysis," 2007, finds rail station effects vary from negative to insignificant to positive depending on accessibility and local context.
- Rennert's 2022 meta-analysis finds transit access value is real but varies based on network, policy, and context.
- Austin's Project Connect anti-displacement work explicitly recognizes that transit investment can increase cost of living and displacement risk near transit lines.

Implication: Transit and school boundaries are important side-panel context, but the app should be cautious about school "quality" claims and should rely on official current boundaries and official performance data if used.

## Recommended Signal Families for Later Planning

This is not an implementation plan; it is a research-derived feature taxonomy for the next phase.

### Growth / Investment Momentum

- building permits per parcel/address/housing unit
- new construction permits
- remodel/addition permits
- total/median valuation
- new square footage
- plan review, site plan, subdivision, zoning cases
- utility connection growth
- green building projects

### Redevelopment / Displacement Pressure

- demolition permits
- high-value new construction in vulnerable tracts
- permit acceleration compared with citywide baseline
- displacement risk category
- affordability inventory loss/gain
- transit-oriented overlay/proximity
- property value/income gap if a property value source is added

### Decline / Distress Risk

- rising code complaints
- repeat offender property activity
- unresolved violation trend
- 311 service request intensity
- falling permits/investment relative to nearby areas
- crime increase by severity
- vacancy/foreclosure/tax delinquency if sourced later

### Livability / Amenity Context

- sidewalk and bike infrastructure
- crash rates
- park access and recreation centers
- library/civic access
- food access and restaurant/business presence
- transit corridor or Project Connect proximity
- school service area and school-year vintage

### Confidence / Data Quality

- event count sufficiency
- geocode quality
- currentness of source
- boundary vintage
- missing field rate
- volatility/outlier flags
- whether an area is too small for stable inference

## OpenStreetMap and Geocoding Research

OpenStreetMap is appropriate as a map data source, but public OSM infrastructure has limits.

Findings:

- Leaflet or MapLibre can render an OSM-based map in a laptop-first app.
- The public `nominatim.openstreetmap.org` service has an absolute maximum of 1 request per second, requires a valid User-Agent or Referer, requires attribution, requires caching of repeated queries, and forbids autocomplete and systematic bulk queries.
- The public OSM tile servers have capacity limitations, require clear attribution, and provide no SLA.

Implication for planning:

- For a proof of concept, public OSM tiles and carefully throttled Nominatim may be acceptable.
- For production, use a commercial OSM-derived tile/geocoder provider, self-hosted Nominatim/tiles, or a local Austin address index.
- Since users are entering property addresses, avoid sending unnecessary personal or confidential data to third-party geocoders.

## Modeling Research Implications

The scientifically defensible model should not be a black-box "buy/rent here" verdict. A better structure is a transparent scorecard:

- block segment: near-real-time, high spatial precision, noisier
- ZIP: stable and explainable, but too coarse for property decisions
- school service area: meaningful to families, but boundary-vintage-sensitive

Potential modeling levels:

- descriptive trend scores for POC
- standardized z-scores or percentiles against citywide and nearby-area baselines
- trend slope over rolling 1/3/5-year windows
- anomaly and acceleration detection
- supervised learning only after defining labels such as future permit growth, future crime reduction, future displacement risk, or future price/rent movement

scikit-learn is useful for:

- clustering similar areas
- dimensionality reduction for correlated signals
- gradient boosting or random forests for predicting explicitly defined future outcomes
- calibration and feature importance

But for the first product version, interpretable engineered indicators are likely more valuable than a complex model because users need to understand why a side panel says an area is improving, declining, or risky.

## Major Risks

- Label risk: "growing" and "dying" are loaded terms. Consider "Momentum," "Distress," "Displacement Pressure," and "Confidence" instead.
- Causal risk: permits and amenities are not proof of better resident outcomes.
- Bias risk: crime, arrests, 311, and code enforcement reflect reporting/enforcement behavior, not only underlying conditions.
- Boundary risk: AISD service areas are changing; stale school boundaries would be harmful.
- Spatial risk: block-level event counts can be sparse and volatile.
- Data-quality risk: permits have outliers, missing coordinates, broad addresses, duplicate project semantics, and continuously changing source records.
- Legal/product risk: this could be interpreted as housing advice. The UI should present signals and caveats, not a recommendation to buy/rent.

## Open Questions Before Planning

- Should the product use labels like "growing/dying," or use more neutral signal names?
- Is the geography limited to City of Austin, AISD, Travis County, or the Austin MSA?
- Should school service area mean elementary only, or ES/MS/HS separately?
- Should the first version use only free/open data, or can it use paid property/rent/sales data?
- Should users be able to compare multiple addresses side-by-side?
- Is the desired first deliverable a data pipeline, a model notebook, a database schema, a backend API, or a full map UI prototype?

## Source Index

Austin / Socrata / Data Sources:

- Austin Open Data Portal: `https://data.austintexas.gov/`
- Socrata catalog API query used: `https://api.us.socrata.com/api/catalog/v1?domains=data.austintexas.gov&search_context=data.austintexas.gov&limit=1000&only=datasets`
- Issued Construction Permits: `https://data.austintexas.gov/Building-and-Development/Issued-Construction-Permits/3syk-w9eu/about_data`
- Issued Construction Permits API: `https://data.austintexas.gov/resource/3syk-w9eu.json`
- Data.gov Issued Construction Permits metadata: `https://catalog.data.gov/dataset/issued-construction-permits`
- Socrata API endpoints docs: `https://dev.socrata.com/docs/endpoints`
- Socrata consumer API docs: `https://dev.socrata.com/consumers/getting-started`
- Socrata output formats: `https://dev.socrata.com/docs/formats/`

AISD / Schools:

- AISD School Maps / GIS: `https://www.austinisd.org/planning-asset-management/school-maps-gis`
- AISD Board approves consolidations, November 21, 2025: `https://www.austinisd.org/press-releases/2025/11/21/austin-isd-board-approves-24-state-mandated-turnaround-plans-school`
- AISD update on closures and boundary work, April 26, 2026: `https://www.austinisd.org/announcements/2026/04/26/update-school-closures-and-district-boundary-work-en-espanol`
- NCES School Attendance Boundary Survey: `https://nces.ed.gov/programs/edge/SABS`

Austin displacement / local research:

- City of Austin Displacement Risk Areas 2022 metadata: `https://catalog.data.gov/dataset/city-of-austin-displacement-risk-areas-2022`
- Project Connect Anti-Displacement Dashboard Data 2020 metadata: `https://catalog.data.gov/dataset/project-connect-anti-displacement-dashboard-data-2020`
- Uprooted report page: `https://sites.utexas.edu/gentrificationproject/austin-uprooted-report-maps/`
- Uprooted methodology: `https://sites.utexas.edu/gentrificationproject/gentrification-mapping-methodology/`
- UT Law announcement for Uprooted: `https://law.utexas.edu/clinics/2018/09/18/uprooted-residential-displacement/`
- Austin Project Connect anti-displacement initiatives: `https://www.austintexas.gov/housing/project-connect-anti-displacement-initiatives`

Scientific and policy literature:

- HUD Neighborhood Change Indicators, 2024: `https://www.huduser.gov/portal/publications/Neighborhood-Change-Indicators.html`
- HUD Mapping Gentrification methodology article: `https://www.huduser.gov/portal/periodicals/cityscape/vol26num1/article20.html`
- Lacoe, Bostic, Acolin, "Crime and private investment in urban neighborhoods": `https://www.sciencedirect.com/science/article/pii/S0094119018300883`
- University of Washington publication page for the same paper: `https://research.be.uw.edu/publications/title-269/`
- Tillyer, Acolin, Walter, "Place-Based Improvements for Public Safety": `https://pmc.ncbi.nlm.nih.gov/articles/PMC11576047/`
- Acolin et al., "Spatial spillover effects of crime on private investment": `https://journals.sagepub.com/doi/abs/10.1177/00420980211029761`
- Gibbons and Machin, "Valuing school quality, better transport, and lower crime": `https://researchonline.lse.ac.uk/20471/`
- Autor, Palmer, Pathak, "Gentrification and the Amenity Value of Crime Reductions": `https://www.nber.org/papers/w23914`
- Glaeser, Kim, Luca, "Measuring Gentrification / Nowcasting Gentrification": `https://www.nber.org/papers/w24952`
- Harvard Kennedy School page for "Nowcasting Gentrification": `https://www.hks.harvard.edu/publications/nowcasting-gentrification-using-yelp-data-quantify-neighborhood-change`
- Troy and Grove, "Property values, parks, and crime": `https://research.fs.usda.gov/treesearch/18507`
- Debrezion, Pels, Rietveld, rail station property value meta-analysis: `https://link.springer.com/article/10.1007/s11146-007-9032-z`
- Rennert, rail access meta-analysis: `https://www.sciencedirect.com/science/article/pii/S0965856422001677`
- Williams, Galster, Verma, foreclosure early warning indicator: `https://www.tandfonline.com/doi/abs/10.1080/01944363.2013.888306`
- Bunten, Preis, Aron-Dine, "Re-measuring gentrification": `https://journals.sagepub.com/doi/full/10.1177/00420980231173846`
- Freemark, "Upzoning Chicago": `https://journals.sagepub.com/doi/10.1177/1078087418824672`

OpenStreetMap:

- Nominatim usage policy: `https://operations.osmfoundation.org/policies/nominatim/`
- OSM tile usage policy: `https://operations.osmfoundation.org/policies/tiles/`
- OSM geocoding overview: `https://wiki.openstreetmap.org/wiki/Geocoding`
