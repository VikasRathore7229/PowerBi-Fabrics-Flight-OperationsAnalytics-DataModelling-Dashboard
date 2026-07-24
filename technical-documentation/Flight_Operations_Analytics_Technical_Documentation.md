# Eurowings Digital Flight Operations Analytics
## Refined Technical Documentation and Interview Reference Guide

**Project owner:** Vikas Rathore  
**Workspace:** `Eurowings Digital Case Study`  
**Lakehouse:** `EW_Flight_Operations_Lakehouse`  
**Pipeline:** `PL_EW_Flight_Operations_EndToEnd`  
**Semantic model:** `SM_EW_Flight_Operations`  
**Power BI report:** `RP_EW_Flight_Operations`  
**Technical source used:** `Eurowings_Digital_Case_Study_Project_Handover_2026-07-15_2345_CEST(1).md`  
**Source boundary:** No other project file, notebook export, report export, dataset or external source was used to create this document.

---

## 1. How this document resolves the handover source

The handover contains a current top-level state followed by preserved older snapshots. Some older snapshots contain superseded status statements, planned designs and earlier model results. This documentation applies the source's own authority rule:

1. The top-level **Part I — Current authoritative state** controls current status, current report implementation and final numerical baselines.
2. The first preserved authoritative appendix supplies detailed notebook, schema, transformation, semantic-model and DAX information when it does not conflict with Part I.
3. Older nested snapshots are used only for technical detail that remained unchanged.
4. Conflicting older values are not carried forward. In particular, the current GBT metrics are:
   - MAE: `732.378`
   - RMSE: `1,061.322`
   - R²: `0.978707`
5. Planned but unconfirmed features are labelled as deferred or not implemented.
6. The report chapter covers only the six visible report pages. Hidden technical pages are excluded from the page-by-page walkthrough.

---

## 2. Executive technical summary

### 2.1 What was built

An end-to-end Microsoft Fabric and Power BI analytical data product was built from the supplied flight-level workbook and an airport reference file.

```text
Landing files
→ Raw source preservation
→ Standardized names and types
→ Curated enrichment, data quality and analysis eligibility
→ Analytical aggregates and statistical baselines
→ Planned-TOW predictive prototype
→ Trusted dimensional model
→ Fabric orchestration pipeline
→ Direct Lake semantic model
→ Governed DAX layer
→ Six-page Power BI decision product
```

### 2.2 Why it was built this way

The original task required wind and distance distributions, cumulative percentages and P50/P85 reference values. The solution keeps those outputs central while demonstrating how the same requirement can be implemented as a governed, reproducible data product rather than a one-off Excel-to-Power-BI report.

The architecture separates responsibilities:

| Responsibility | Implementation layer |
|---|---|
| Preserve source data and lineage | Raw Delta tables |
| Normalize names, values and data types | Standardized Delta tables |
| Apply business rules and create reusable features | Curated Delta tables |
| Create stable dimensions and facts | Trusted Delta tables |
| Recalculate values under report filters | DAX measures |
| Explain results and support interaction | Power BI report layer |

### 2.3 Current validated baseline

| Metric | Validated value |
|---|---:|
| Source flight rows | 15,931 |
| Curated flight rows | 15,931 |
| Trusted flight rows | 15,931 |
| Distinct routes | 147 |
| Hard-failure rows excluded | 0 |
| Unique review flights retained | 349 |
| Overall review rate | 2.2% |
| DQ rule-trigger rows | 358 |
| Routes containing review flights | 88 |
| Average selected distance | 728 NM |
| Global wind P50 / P85 | -3 / 22 kts |
| Global distance P50 / P85 | 593 / 1,105 NM |
| High TOW-utilisation flights | 44 |
| Payload-review flights | 128 |
| Model-training rows | 13,984 |
| Holdout rows per model | 1,819 |
| Scoring rows across two models | 3,638 |
| Selected model | Linear Regression |
| Selected-model MAE | 398.184 |
| Selected-model RMSE | 633.727 |
| Selected-model R² | 0.992408 |

### 2.4 Interview summary in one sentence

> I converted a flight-distribution task into a governed Fabric and Power BI data product that preserves source lineage, separates hard failures from review warnings, publishes reusable analytical and predictive outputs through a Direct Lake star schema, and lets users trace every aggregated signal back to the contributing flight records.

---

## 3. Business requirement, scope and interpretation controls

### 3.1 Mandatory case-study requirement

The required analysis was:

- Flight distribution by wind speed.
- Flight distribution by travelled distance.
- Clear grouping or binning.
- A cumulative percentage line for each distribution.
- Identification of the point at which 50% of flights are covered.
- Static P50 and P85 reference values.
- Additional insights and recommended additional data.
- Python or R exploration as a bonus.

### 3.2 Supported analytical scope

The available data supports:

- Descriptive operating distributions.
- Percentile-based coverage benchmarks.
- Route concentration and variability analysis.
- Aircraft deployment and planned-weight analysis.
- Data-quality review and analysis-specific eligibility.
- A planned-TOW predictive prototype for residual-based triage.

### 3.3 Unsupported conclusions

The solution does not claim:

```text
delay causality
fuel burn
operational or safety risk
cargo weight
passenger load factor
profitability
market share
certified operational performance
dispatch suitability
```

### 3.4 Mandatory wording controls

| Topic | Required wording | Wording to avoid |
|---|---|---|
| Weight fields | `planned_*_weight`, source weight unit unconfirmed | kg, tonnes, certified mass unit |
| Wind | Wind Component; negative/zero/positive component | Headwind or tailwind without source confirmation |
| Route share | Sample Share | Market Share |
| Monthly chart | Observed Monthly Pattern/Profile | Seasonality, because only part of one year is present |
| TOW >95% status | Review or validation condition | Unsafe operation |
| Predictive target | Supplied Planned TOW | Actual TOW |
| ML output | Analytical triage prototype | Operational, safety or certification model |
| Feature importance | Model association | Causal impact |

---

## 4. Source data and governed data dictionary

### 4.1 Flight source

**File:** `Data for Analysis.xlsx`  
**Landing path:** `Files/landing/flight_operations/Data for Analysis.xlsx`  
**Grain:** One supplied row per technical flight record  
**Size:** 15,931 rows and 11 business columns

| Source column | Governed column | Technical meaning |
|---|---|---|
| `Flight Date` | `flight_date` | Flight calendar date |
| `Airline` | `airline_code` | Operating-airline code |
| `Pln Payload` | `planned_payload_weight` | Supplied planned payload |
| `Pln TOW` | `planned_tow_weight` | Supplied planned take-off weight |
| `PAX` | `passenger_count` | Passenger count |
| `Acft MTOW` | `aircraft_mtow_weight` | Aircraft maximum take-off weight supplied in the file |
| `Acft Type` | `aircraft_type_code` | Aircraft type code |
| `Destination` | `destination_iata_code` | Destination IATA code |
| `Origin` | `origin_iata_code` | Origin IATA code |
| `Airway Distance Travelled (Nautical miles)` | `airway_distance_nm` | Travelled airway distance in nautical miles |
| `Wind in Kts` | `wind_component_kts` | Signed wind-component value supplied by the source |

### 4.2 Airport reference source

**File:** `airports.csv`  
**Landing path:** `Files/landing/reference/ourairports/airports.csv`  
**Raw and Standardized rows:** 85,720

The 24 received source fields are:

```text
id
ident
type
name
latitude_deg
longitude_deg
elevation_ft
continent
country_name
iso_country
region_name
iso_region
local_region
municipality
scheduled_service
gps_code
icao_code
iata_code
local_code
home_link
wikipedia_link
keywords
score
last_updated
```

### 4.3 Source assumptions

- Weight units are not documented. Technical fields deliberately use `_weight` rather than `_kg`.
- The signed wind field is not semantically confirmed as headwind or tailwind.
- `flight_record_key` is a technical source-record key, not a guaranteed operational flight-leg identifier.
- The sample covers June–December 2024. It supports observed monthly patterns, not full-year seasonality.
- Airline comparisons are operating-carrier sample comparisons, not competitor or market-share analysis.

---

## 5. Architecture — What, Why and How

### 5.1 What

The solution uses five logical Lakehouse schemas:

| Schema | Responsibility |
|---|---|
| `00control` | Optional control, validation and run artefacts |
| `01raw` | Source-preserving ingestion and technical lineage |
| `02standardized` | Governed naming, normalization, typing, parse controls and reconciliation |
| `03curated` | Business enrichment, DQ, eligibility, analytical aggregates and ML outputs |
| `04trusted` | Stable-key dimensions and facts consumed by Direct Lake |

### 5.2 Why

This separation prevents several common problems:

- Source values are not silently overwritten by cleansing logic.
- Technical parsing is separated from business validity.
- Review warnings can remain available for unaffected analyses.
- Stable calculations are created once rather than repeated in several reports.
- Power BI consumes only governed facts and dimensions instead of broad engineering tables.
- The report remains live-connected to one central semantic model.

### 5.3 How

```text
Data for Analysis.xlsx
→ NB01_Ingest_Raw
→ 01raw.flight_raw
→ NB02_Standardize_Data
→ 02standardized.flight_standardized
                         ┐
airports.csv             │
→ NB01R_Ingest_Airport_Reference
→ 01raw.airport_reference_raw
→ NB02R_Standardize_Airport_Reference
→ 02standardized.airport_reference_standardized
                         ┘
→ NB03_Curate_Quality_Features
→ NB04_Analytical_Aggregates
→ NB05_Predictive_Analytics
→ NB06_Publish_Trusted_Model
→ 04trusted dimensions and facts
→ SM_EW_Flight_Operations [Direct Lake]
→ RP_EW_Flight_Operations
```

### 5.4 Workspace items

```text
EW_Flight_Operations_Lakehouse
NB01_Ingest_Raw
NB01R_Ingest_Airport_Reference
NB02_Standardize_Data
NB02R_Standardize_Airport_Reference
NB03_Curate_Quality_Features
NB04_Analytical_Aggregates
NB05_Predictive_Analytics
NB05_Predictive_Analytics [Fabric experiment]
NB06_Publish_Trusted_Model
PL_EW_Flight_Operations_EndToEnd
SM_EW_Flight_Operations
RP_EW_Flight_Operations
```

---

## 6. Fabric pipeline orchestration

### 6.1 What

`PL_EW_Flight_Operations_EndToEnd` executes all notebook stages in dependency order.

```text
ACT_NB01_Ingest_Raw ─→ ACT_NB02_Standardize_Data ─┐
                                                   ├→ ACT_NB03_Curate_Quality_Features
ACT_NB01R_Ingest_Airport_Reference                 │
    └→ ACT_NB02R_Standardize_Airport_Reference ────┘

ACT_NB03
→ ACT_NB04_Analytical_Aggregates
→ ACT_NB05_Predictive_Analytics
→ ACT_NB06_Publish_Trusted_Model
```

### 6.2 Why

- The independent flight and airport branches can run in parallel.
- NB03 waits until both standardized inputs succeed.
- Downstream analytical and Trusted outputs are not created from incomplete upstream data.
- A notebook failure is visible at the exact failed activity and step.

### 6.3 How

| Setting | Value |
|---|---|
| Timeout | `0.01:00:00` |
| Retry | `0` |
| Retry interval | 30 seconds |
| Failure activity | `ACT_FAIL_PIPELINE` |
| Error code | `EW-PIPELINE-001` |

Retries are disabled because deterministic schema, parsing or DQ failures should not be hidden by automatic retries. A transient retry policy can be added later only if production evidence supports it.

### 6.4 Pipeline versus task flow

- **Pipeline:** executable orchestration, dependencies, timeout, monitoring and failure handling.
- **Task flow:** optional visual documentation and workspace navigation.
- A task flow does not replace the pipeline.

---

## 7. Calculation-placement rule

### 7.1 Fabric notebooks

Place logic upstream when it is stable, row-level, computationally expensive, reused or governed.

Examples:

```text
route_code
airline and aircraft normalization
airport enrichment
date attributes
TOW utilisation
remaining MTOW margin
payload ratios
wind and distance bands
DQ flags
analysis eligibility
global percentiles
route medians and IQR
outlier status
correlation outputs
model predictions and metrics
```

### 7.2 DAX semantic layer

Use DAX when a result must react to current slicers and filter context.

Examples:

```text
selected flight count
selected cumulative percentage
selected-scope P50/P85
route ranking under filters
route sample share
review rate
selected-model KPI display
filter-aware titles and subtitles
```

### 7.3 Power BI report layer

Use the report for:

```text
formatting
page narrative
visual interaction
tooltips
navigation
record-level investigation
```

### 7.4 Prohibited report-local shortcuts

The report remains live-connected to the central Direct Lake model. Do not use:

```text
Add a local model
Make changes to this model
Enter Data
report-local calculated tables
report-local calculated columns
composite-model conversion for presentation-only needs
```

---

## 8. Notebook technical documentation

## 8.1 NB01 — Raw flight ingestion

### What

Ingest the supplied Excel workbook into `01raw.flight_raw` while preserving the source structure and values.

### Why

Raw must be an auditable copy of the source. If cleaning, renaming or business filtering happened here, it would be difficult to prove what was originally received or to distinguish a source defect from a transformation defect.

### How

**Source:** `Files/landing/flight_operations/Data for Analysis.xlsx`  
**Target:** `01raw.flight_raw`  
**Confirmed rows:** 15,931  
**Status:** `SUCCEEDED`

Added technical metadata:

```text
_source_file_name
_source_sheet_name
_source_row_number
_ingestion_timestamp_utc
_pipeline_run_id
_source_record_hash
```

The source record hash is deterministic over the received source values. The Raw table preserves the original business column names.

### Null, zero, negative and invalid values

- Values are preserved as received.
- Raw does not interpret `0`, negative values or null-like content as valid or invalid business values.
- No source row is deleted because of a business rule.
- Business-column values are retained as source strings; typing happens in NB02.

### Validation and failure conditions

```text
Expected row count: 15,931
Source column names preserved
Source row number unique
Source hash populated
Distinct source hashes: 15,931
```

The notebook fails visibly if the file cannot be read, required structural validation fails, row reconciliation fails or a required helper stage raises an exception.

### Lineage and reconciliation

- `_source_row_number` preserves source position.
- `_source_record_hash` supports record-level comparison.
- Row count is checked before the Raw write is accepted.
- No business transformation is hidden inside the ingestion step.

### What this notebook does not do

```text
No governed column renaming
No date or numeric parsing
No null correction
No DQ classification
No record exclusion
No feature engineering
```

---

## 8.2 NB01R — Raw airport-reference ingestion

### What

Preserve the complete received OurAirports snapshot in `01raw.airport_reference_raw`.

### Why

The airport file is an independent external source. It requires its own lineage and should not be loaded directly into a business reference table. Preserving all rows allows duplicate mappings and schema changes to be audited later.

### How

**Source:** `Files/landing/reference/ourairports/airports.csv`  
**Local Python path:** `/lakehouse/default/Files/landing/reference/ourairports/airports.csv`  
**Target:** `01raw.airport_reference_raw`  
**Confirmed rows:** 85,720  
**Status:** `SUCCEEDED`

Added metadata:

```text
_source_file_name
_reference_source_name
_source_row_number
_ingestion_timestamp_utc
_pipeline_run_id
_source_record_hash
```

### Schema-drift handling

The notebook originally expected an older 19-column snapshot, while the received file contained 24 columns. The refined logic separates:

- `REQUIRED_SOURCE_COLUMNS`: missing fields that must fail the notebook because downstream processing depends on them.
- `CURRENT_SNAPSHOT_COLUMNS`: used to report additive or missing snapshot fields.

Additive source columns are retained and reported rather than causing a failure.

### Null, duplicate and invalid handling

- All received rows are retained.
- Exact duplicate source rows are reported, not silently removed.
- Duplicate IATA mappings are not resolved in Raw.
- Missing required columns are a hard notebook failure.
- Additional columns are not a hard failure.

### Validation

```text
File exists
File is not empty
No duplicate column names
Required columns exist
Row count preserved
Column order preserved
Source-row number unique
Source hash populated
```

### What this notebook does not do

```text
No filtering to airports used by the flight sample
No type conversion
No duplicate-IATA resolution
No business match selection
```

---

## 8.3 NB02 — Standardize flight data

### What

Convert the Raw flight table into a technically consistent, typed and governed table: `02standardized.flight_standardized`.

### Why

Source labels, whitespace, null tokens and string-form numeric values are unsuitable as a reusable contract. Standardization provides one predictable schema for all downstream notebooks while preserving lineage back to Raw.

### How

**Source:** `01raw.flight_raw`  
**Target:** `02standardized.flight_standardized`  
**Rows:** 15,931  
**Parse-error rows:** 0  
**Status:** `SUCCEEDED`

Key transformations:

1. Rename the 11 business fields to governed snake_case names.
2. Trim strings and collapse repeated whitespace.
3. Convert technical null tokens to actual nulls.
4. Uppercase airline, aircraft and airport codes.
5. Parse `flight_date` as a date.
6. Parse numeric and whole-number fields.
7. Create parse and standardization controls.
8. Preserve Raw lineage.
9. Reconcile source and target row counts.

Standardization metadata:

```text
_standardization_timestamp_utc
_standardization_run_id
_standardized_record_hash
standardization_error_count
standardization_error_reasons
standardization_status
```

### Typed output contract

| Field | Type intent |
|---|---|
| `flight_date` | Date |
| `airline_code` | String |
| `planned_payload_weight` | Numeric / whole number |
| `planned_tow_weight` | Numeric / whole number |
| `passenger_count` | Numeric / whole number |
| `aircraft_mtow_weight` | Numeric / whole number |
| `aircraft_type_code` | String |
| `destination_iata_code` | String |
| `origin_iata_code` | String |
| `airway_distance_nm` | Numeric / whole number |
| `wind_component_kts` | Numeric / whole number |

### Null, zero, negative and invalid handling

- Null-like strings become null; they are not invented or imputed.
- A successful parse does not mean a value is operationally valid.
- `0` and negative numerics can parse successfully and remain in the Standardized row.
- Business validity is evaluated later in NB03.
- Parse failures are represented through status/error fields rather than silent coercion.
- The current dataset produced zero parse-error rows.

### Failure conditions

The notebook is expected to fail for structural defects such as missing required source columns, duplicate target mappings, an invalid input contract, failed row reconciliation or an unhandled parse/write error. Business issues such as zero payload are not removed here.

### Lineage and reconciliation

- Raw lineage fields remain available.
- `_standardized_record_hash` supports comparison at the standardized contract.
- Source and target rows reconcile at 15,931.

### What this notebook does not do

```text
No airline or aircraft business mapping
No airport enrichment
No hard-failure exclusion
No review-warning classification
No analytical feature calculation
```

---

## 8.4 NB02R — Standardize airport reference

### What

Create a typed and normalized airport master in `02standardized.airport_reference_standardized` while retaining all source records.

### Why

Airport enrichment requires consistent IATA codes, coordinates, source scores, service flags and timestamps. Duplicate IATA candidates must remain visible until the Curated layer applies a deterministic business selection rule.

### How

**Source:** `01raw.airport_reference_raw`  
**Target:** `02standardized.airport_reference_standardized`  
**Rows:** 85,720  
**Parse-error rows:** 0  
**Distinct IATA codes:** 9,056  
**Status:** `SUCCEEDED`

Governed fields:

```text
airport_source_id
airport_ident
airport_type
airport_name
latitude_deg
longitude_deg
elevation_ft
continent_code
country_name
country_code
region_name
region_code
local_region_name
municipality_name
scheduled_service_flag
gps_code
icao_code
iata_code
local_code
home_link
wikipedia_link
keywords
source_score
source_last_updated_timestamp
```

Parse-success controls:

```text
airport_source_id_parse_success
latitude_parse_success
longitude_parse_success
elevation_parse_success
scheduled_service_parse_success
source_score_parse_success
source_last_updated_parse_success
```

### Null, invalid and duplicate handling

- All records remain in Standardized.
- Code casing is normalized.
- Coordinates, score and timestamp are parsed.
- Scheduled-service values such as `0` and `1` are accepted, along with common text equivalents.
- Duplicate IATA mappings are reported, not resolved.
- Invalid parsed values are surfaced through parse-success fields and standardization status.

### What this notebook does not do

```text
No filter to flight-used airports
No deterministic selection of one airport per IATA code
No flight join
No row deletion for duplicate IATA candidates
```

---

## 8.5 NB03 — Curate quality and flight features

### What

Build the analysis-ready flight table, controlled master references, DQ framework, review classifications and use-case-specific eligibility.

### Why

A single blanket valid/invalid flag is insufficient. A record with zero payload may still be useful for wind or distance analysis, while being inappropriate for payload analysis or model training. NB03 preserves this distinction.

### How

**Inputs:**

```text
02standardized.flight_standardized
02standardized.airport_reference_standardized
```

**Core outputs:**

```text
03curated.flight_enriched
03curated.ref_airline
03curated.ref_aircraft_type
03curated.ref_airport
03curated.dq_rule_catalog
03curated.dq_rule_results
03curated.dq_record_summary
03curated.dq_batch_summary
03curated.column_profile_summary
```

**Cell structure recorded in the handover:**

```text
00 Documentation
01 Parameters
02 Setup and helpers
03 Read and validate Standardized sources
04 Create airline reference
05 Create aircraft reference
06 Build Curated airport reference
07 Validate reference coverage
08 Build flight enrichment and features
09 Create DQ rule catalog
10 Evaluate DQ rules
11 Build DQ record summary and final flight table
12 Write core Curated outputs
13 Build and write DQ batch summary
14 Build and write column profiles
15 Final Curated validation
16 Completion summary
```

### Controlled airline mapping

| Code | Name | Entity | Status |
|---|---|---|---|
| `EW` | Eurowings | Eurowings GmbH | `CURRENT` |
| `E6` | Eurowings Europe | Eurowings Europe Limited | `CURRENT` |
| `4U` | Germanwings | Germanwings GmbH | `LEGACY_REVIEW` |

The single `4U` record is a weak sample and is not treated as a meaningful carrier comparison.

### Controlled aircraft mapping

| Code | Aircraft | Family | Generation |
|---|---|---|---|
| `A319` | Airbus A319ceo | Airbus A320 Family | CEO |
| `A320` | Airbus A320ceo | Airbus A320 Family | CEO |
| `A20N` | Airbus A320neo | Airbus A320 Family | NEO |
| `A321` | Airbus A321ceo | Airbus A320 Family | CEO |
| `A21N` | Airbus A321neo | Airbus A320 Family | NEO |

Aircraft configuration is created as:

```text
aircraft_configuration_code
= aircraft_type_code + "-" + aircraft_mtow_weight
```

Examples include `A319-69500`, `A320-71500`, `A320-81000`, `A20N-81000`, `A321-91000` and `A21N-91000`.

Different MTOW variants are not automatically errors. Registration-level or fleet-master data would be required to validate them.

### Airport candidate resolution

Curated filters the full airport master to IATA codes used by the flight data and chooses one row per IATA code using this order:

1. Scheduled-service airport.
2. Large airport.
3. Medium airport.
4. Small airport.
5. Higher source score.
6. Lower source ID.

The selected record also stores whether the IATA match was unique or resolved from several candidates.

### Main enriched field families

#### Identity and lineage

```text
flight_record_key
_source_*
_standardization_*
_curation_*
```

#### Airline and aircraft

```text
airline_name
legal_entity_name
carrier_status
airline_master_match_status
aircraft_model_name
aircraft_family
aircraft_generation
manufacturer_name
aircraft_configuration_code
aircraft_master_match_status
aircraft_type_mtow_variant_count
```

#### Airport, route and market

```text
origin_airport_reference_key
origin_airport_name
origin_city
origin_country_name
origin_country_code
origin_region_name
origin_region_code
origin_continent_code
origin_latitude_deg
origin_longitude_deg
origin_airport_type
origin_scheduled_service_flag
origin_airport_match_resolution
origin_airport_match_status

destination_airport_reference_key
destination_airport_name
destination_city
destination_country_name
destination_country_code
destination_region_name
destination_region_code
destination_continent_code
destination_latitude_deg
destination_longitude_deg
destination_airport_type
destination_scheduled_service_flag
destination_airport_match_resolution
destination_airport_match_status

route_code
country_pair_code
market_scope
```

`market_scope` is `DOMESTIC`, `INTERNATIONAL` or `UNKNOWN`.

#### Date attributes

```text
flight_year
flight_quarter_number
flight_quarter
flight_month_number
flight_month_name
flight_year_month
flight_year_month_sort
flight_week_of_year
flight_day_of_week_number
flight_day_of_week_name
```

#### Weight and payload features

| Field | Formula | Why it exists |
|---|---|---|
| `tow_utilisation_pct` | `planned_tow_weight / aircraft_mtow_weight` | Shows how much of supplied MTOW is used by planned TOW |
| `remaining_mtow_margin_weight` | `aircraft_mtow_weight - planned_tow_weight` | Shows remaining supplied margin before MTOW |
| `payload_to_tow_pct` | `planned_payload_weight / planned_tow_weight` | Compares planned payload with planned TOW |
| `payload_to_mtow_pct` | `planned_payload_weight / aircraft_mtow_weight` | Compares planned payload with supplied aircraft MTOW |
| `non_payload_planned_mass` | `planned_tow_weight - planned_payload_weight` | Derives the planned TOW component not represented by supplied payload |
| `planned_payload_per_pax_proxy` | `planned_payload_weight / passenger_count` | A proxy for comparing payload and passenger count; not passenger weight |

All ratios use denominator protection. A zero or null denominator produces a protected null/invalid analytical result rather than a divide-by-zero failure.

#### Wind features

```text
absolute_wind_kts
wind_sign_category
wind_magnitude_category
```

Magnitude categories:

| Absolute wind | Category |
|---:|---|
| 0–40 | `NORMAL` |
| 41–60 | `HIGH` |
| 61–80 | `EXTREME` |
| >80 | `CRITICAL_REVIEW` |

#### Distribution bands

Wind uses 10-knot bands:

```text
wind_band_10kt_key
wind_band_lower_kts
wind_band_upper_kts
wind_band_label
wind_band_sort
```

Distance uses 100-NM bands:

```text
distance_band_100nm_key
distance_band_lower_nm
distance_band_upper_nm
distance_band_label
distance_band_sort
```

#### Governed Fleet and Payload statuses

TOW status fields:

```text
tow_utilisation_status_code
tow_utilisation_status_label
tow_utilisation_status_sort
```

Payload status fields:

```text
payload_review_status_code
payload_review_status_label
payload_review_status_sort
```

TOW logic:

```text
NOT_ELIGIBLE
→ is_weight_analysis_eligible = false

HIGH_UTILISATION
→ eligible and tow_utilisation_pct > 0.95

STANDARD_UTILISATION
→ all other eligible rows
```

Payload logic:

```text
NOT_ELIGIBLE
→ is_payload_analysis_eligible = false

REQUIRES_REVIEW
→ payload-review rule triggered

STANDARD
→ all other eligible rows
```

### Data-quality table grains

| Table | Grain |
|---|---|
| `dq_rule_catalog` | One row per DQ rule |
| `dq_rule_results` | One row per triggered rule per flight |
| `dq_record_summary` | One row per flight |
| `dq_batch_summary` | One row per curation execution |
| `column_profile_summary` | One row per profiled column per run |

`column_profile_summary` includes data type, row count, null count, distinct count, minimum, maximum, mean, P50 and P85.

### Hard-failure rules

A hard failure means the row is not allowed into Trusted.

| Code | Rule |
|---|---|
| `DQ001` | Standardization status is not PASS |
| `DQ002` | Required business field missing |
| `DQ003` | Distance <= 0 |
| `DQ004` | Passenger count <= 0 |
| `DQ005` | Planned TOW <= 0 |
| `DQ006` | MTOW <= 0 |
| `DQ007` | Planned TOW > MTOW |
| `DQ008` | Planned payload < 0 |
| `DQ009` | Planned payload > planned TOW |
| `DQ010` | Invalid airport-code format |
| `DQ011` | Airport reference match missing |
| `DQ012` | Aircraft reference match missing |
| `DQ013` | Airline reference match missing |
| `DQ014` | Exact standardized duplicate |

Current hard-failure count: `0`.

### Review-warning rules

A review warning is retained and handled by analytical eligibility.

| Code | Rule | Trigger count |
|---|---|---:|
| `DQ101` | Planned payload = 0 with passenger count > 0 | 128 |
| `DQ102` | Origin equals destination | 5 |
| `DQ103` | Absolute wind > 60 kts | 175 |
| `DQ104` | Absolute wind > 80 kts | 5 |
| `DQ105` | TOW utilisation > 95% | 44 |
| `DQ106` | Legacy airline code | 1 |

`DQ104` is a subset of `DQ103`.

### Analysis-specific eligibility

```text
is_wind_distribution_eligible
is_distance_distribution_eligible
is_route_analysis_eligible
is_weight_analysis_eligible
is_payload_analysis_eligible
is_carrier_comparison_eligible
is_model_training_eligible
```

A row can be retained in Trusted and still be excluded from a particular calculation.

### Null, zero, negative and invalid handling

| Condition | Treatment |
|---|---|
| Missing required business field | Hard failure |
| Distance <= 0 | Hard failure |
| Passenger count <= 0 | Hard failure |
| Planned TOW <= 0 | Hard failure |
| MTOW <= 0 | Hard failure |
| Planned TOW > MTOW | Hard failure |
| Planned payload < 0 | Hard failure |
| Planned payload > planned TOW | Hard failure |
| Payload = 0 and PAX > 0 | Review warning; retained; not payload-analysis eligible |
| Origin = destination | Review warning; retained; use governed route eligibility |
| High absolute wind | Review warning; retained |
| TOW utilisation >95% | Review warning; retained |
| Invalid master-data match | Hard failure according to the corresponding rule |

### Confirmed output

```text
Curated rows: 15,931
Hard failures: 0
Unique review flights: 349
DQ trigger rows: 358
Records without review warning: 15,582
Validation: PASS
```

The trigger count exceeds the distinct review-flight count because one flight can trigger more than one review rule.

### What this notebook does not do

```text
It does not delete review-warning rows.
It does not treat high utilisation as a safety conclusion.
It does not treat payload per passenger as passenger weight.
It does not infer market share or carrier performance.
It does not calculate dynamic report-filter percentiles.
```

---

## 8.6 NB04 — Analytical aggregates

### What

Create reusable statistical and analytical outputs from the Curated flight population.

### Why

The report needs stable global baselines, route profiles, outlier outputs and population summaries. Computing these once in Fabric makes the logic reusable and testable. Dynamic calculations that must respond to slicers remain in DAX.

### How

**Inputs:**

```text
03curated.flight_enriched
03curated.dq_record_summary
03curated.ref_airline
03curated.ref_aircraft_type
03curated.ref_airport
```

**Cell structure:**

```text
00 Documentation
01 Parameters
02 Setup and helpers
03 Read and validate Curated inputs
04 Build global wind distribution
05 Build global distance distribution
06 Build global percentile reference
07 Build route profile
08 Build route-month profile
09 Build carrier profile
10 Build aircraft-route profile
11 Build route-distance outliers
12 Build transparent anomaly features
13 Build metric correlation matrix
14 Build analysis-population summary
15 Write analytical outputs
16 Validate analytical outputs
17 Completion summary
```

**Outputs:**

```text
03curated.wind_distribution_global
03curated.distance_distribution_global
03curated.percentile_reference
03curated.route_profile
03curated.route_month_profile
03curated.carrier_profile
03curated.aircraft_route_profile
03curated.flight_route_outlier
03curated.flight_anomaly_features
03curated.metric_correlation
03curated.analysis_population_summary
```

### Global distributions

#### Wind

- Grain: one row per 10-knot band.
- Includes flight count, share, cumulative count, cumulative percentage, population count, minimum, maximum and mean.

#### Distance

- Grain: one row per 100-NM band.
- Uses the same count, share and cumulative structure.

The cumulative formulas are conceptually:

```text
flight_share
= band_flight_count / total_eligible_flights

cumulative_flight_count at band b
= sum of eligible flights from the first ordered band through band b

cumulative_flight_pct
= cumulative_flight_count / total_eligible_flights
```

**Why:** The bar chart shows concentration in each band. The cumulative line shows how quickly the selected population is covered as the band boundary increases.

### Percentile reference

Published percentiles:

```text
P05 P10 P25 P50 P75 P85 P90 P95 P99
```

Current mandatory global values:

| Metric | P50 | P85 |
|---|---:|---:|
| Wind Component | -3 kts | 22 kts |
| Airway Distance | 593 NM | 1,105 NM |

Interpretation:

- P50 is the median: 50% of eligible flights are at or below the value.
- P85 is an upper coverage benchmark: 85% are at or below the value.
- Neither value is a risk threshold.

### Route profile

**Grain:** one row per route.

Key outputs:

```text
flight count
sample share
operating days
first and last date
mean/min/max/population-standard-deviation distance
distance P25/P50/P75/P85
distance IQR
wind P50/P85
median passengers
median planned payload
median planned TOW
median TOW utilisation
payload-zero count
review count
sample reliability
```

**Why median distance:** Travelled airway distance can vary by routing. The median is a robust route baseline and is less affected by unusually high or low observations than the mean.

### Route-month profile

**Grain:** one row per route, year and month.

Supports:

```text
observed monthly volume
route-level wind changes
passenger and payload patterns
monthly TOW utilisation
review rates
```

It is not presented as full seasonality.

### Carrier profile

**Grain:** one row per airline code.

Includes flight count, sample share, route and airport counts, aircraft configurations, median distance/PAX/payload/TOW/utilisation, CEO/NEO counts and review rates.

Use `sample share`, not market share.

### Aircraft-route profile

**Grain:** one route and aircraft configuration.

Supports aircraft deployment, configuration mix, TOW utilisation, remaining margin and review concentration. It does not measure fuel efficiency.

### Route-distance outlier calculation

For each route:

```text
IQR = Route P75 - Route P25
Lower bound = Route P25 - 1.5 × IQR
Upper bound = Route P75 + 1.5 × IQR
```

A flight is evaluated only when:

```text
route_sample_size >= 10
```

Statuses:

```text
EVALUATED
INSUFFICIENT_SAMPLE
ROUTE_NOT_ELIGIBLE
ROUTE_BASELINE_MISSING
```

Directions:

```text
LOW
HIGH
NOT_OUTLIER
NOT_EVALUATED
```

**Why:** A route-specific baseline is more meaningful than one global distance threshold because expected distance differs strongly by route.

### Transparent anomaly features

The notebook also creates rule-based features at flight grain:

```text
route distance deviation
route-distance outlier status
TOW-utilisation z-score by aircraft configuration
payload z-score by aircraft configuration
passenger z-score by route
absolute-wind global percentile
distance global percentile
TOW-utilisation global percentile
rule-based anomaly score
rule-based anomaly category
```

Score:

```text
Route-distance outlier      +1
Absolute wind > 60          +1
Absolute wind > 80          +1
TOW utilisation > 95%       +1
Payload zero with PAX       +1
Same origin/destination     +1
```

Category:

```text
0   NORMAL
1   OBSERVE
2   REVIEW
3+  HIGH_REVIEW
```

This is transparent rule-based scoring, not a trained model.

### Metric correlation

Metrics:

```text
planned_tow_weight
planned_payload_weight
airway_distance_nm
wind_component_kts
passenger_count
aircraft_mtow_weight
tow_utilisation_pct
```

The output has one row per ordered metric pair, producing 49 rows.

Pearson correlation is calculated using pairwise-complete observations. It describes linear association; it does not prove causality.

### Analysis population summary

Published populations:

```text
ALL_FLIGHTS
WIND_DISTRIBUTION
DISTANCE_DISTRIBUTION
ROUTE_ANALYSIS
WEIGHT_ANALYSIS
PAYLOAD_ANALYSIS
CARRIER_COMPARISON
MODEL_TRAINING
```

Each row records total, eligible, excluded and eligibility percentage, along with the eligibility field and exclusion explanation.

### Null, zero and invalid handling

- Each output uses the governed eligibility flag for its analytical purpose.
- Noneligible records are excluded from that calculation, not deleted from Curated.
- Route outliers are not evaluated when the route has fewer than 10 eligible observations.
- Correlations use only pairs where both values are present.
- Global distribution and percentile logic uses eligible, non-null values.

### Confirmed result

```text
Source rows: 15,931
Analytical output tables: 11
Route-distance outlier rows: 2,041
Wind P50/P85: -3 / 22 kts
Distance P50/P85: 593 / 1,105 NM
Validation: PASS
```

### What this notebook does not do

```text
It does not replace dynamic DAX cumulative percentages.
It does not claim correlation is causation.
It does not treat the rule-based anomaly score as machine learning.
It does not evaluate low-sample routes as if their outlier bounds were reliable.
```

---

## 8.7 NB05 — Predictive analytics

### What

Train and compare two models that predict the supplied `planned_tow_weight`, then publish holdout predictions, metrics and explainability outputs.

### Why

The case-study source already contains planned TOW, payload, passenger, distance, wind, aircraft and route context. Predicting planned TOW is therefore a defensible demonstration of data-science capability. A fuel or safety model would require outcomes and operational inputs that are not present.

### How

**Input:** `03curated.flight_enriched`  
**Target:** `planned_tow_weight`  
**Eligibility:** `is_model_training_eligible = true`

Model-eligible population:

```text
15,803 rows
2024-06-02 to 2024-12-31
```

Numeric features:

```text
planned_payload_weight
airway_distance_nm
wind_component_kts
passenger_count
aircraft_mtow_weight
flight_month_number
```

Categorical features:

```text
aircraft_type_code
route_code
airline_code
```

Categorical processing:

```text
StringIndexer(handleInvalid="keep")
→ OneHotEncoder(handleInvalid="keep", dropLast=True)
```

Feature assembly:

```text
VectorAssembler(handleInvalid="error")
```

### Chronological split

```text
Strategy: LAST_CALENDAR_MONTH_HOLDOUT
Training: 2024-06-02 to 2024-11-30, 13,984 rows
Test:     2024-12-01 to 2024-12-31,  1,819 rows
Seed: 42
```

**Why:** December remains unseen during training. This is more defensible than randomly mixing dates because it tests whether the model generalizes to a later period and reduces time leakage.

### Models

#### Linear Regression

```text
maxIter = 100
regParam = 0.0
elasticNetParam = 0.0
standardization = true
```

#### Gradient-Boosted Tree Regression

```text
maxIter = 60
maxDepth = 5
stepSize = 0.05
subsamplingRate = 0.8
seed = 42
```

### Evaluation metrics

| Model | MAE | RMSE | R² |
|---|---:|---:|---:|
| Linear Regression | 398.184 | 633.727 | 0.992408 |
| Gradient-Boosted Tree Regression | 732.378 | 1,061.322 | 0.978707 |

Formulas:

```text
error_i = predicted_i - supplied_i
residual_i = supplied_i - predicted_i
absolute_error_i = |predicted_i - supplied_i|

MAE = mean(absolute_error_i)
RMSE = sqrt(mean(error_i²))
R² = 1 - [sum((supplied_i - predicted_i)²) / sum((supplied_i - mean(supplied))²)]
```

Interpretation:

- **MAE** is the average absolute difference in the source weight unit.
- **RMSE** penalizes large differences more strongly than MAE.
- **R²** describes the proportion of target variation explained by the fitted model on the holdout; it does not prove operational suitability.

Linear Regression is selected because it has lower MAE and RMSE and higher R² on the chronological holdout.

### Prediction fields

```text
flight_record_key
flight_date
model_code
model_name
algorithm
data_split
actual_planned_tow_weight
predicted_planned_tow_weight
prediction_error_weight
residual_weight
absolute_error_weight
squared_error_weight
absolute_percentage_error
prediction_direction
all modelling features
model-run metadata
```

Sign convention:

```text
prediction_error_weight = predicted - supplied
residual_weight         = supplied - predicted
absolute_error_weight   = ABS(predicted - supplied)
```

```text
OVER_PREDICTED
→ predicted planned TOW is above supplied planned TOW

UNDER_PREDICTED
→ predicted planned TOW is below supplied planned TOW

EXACT
→ predicted and supplied values are equal
```

Do not confuse prediction direction with residual sign.

### Outputs and grains

| Output | Grain |
|---|---|
| `03curated.tow_model_predictions` | One test flight per model |
| `03curated.model_metrics` | One model and metric |
| `03curated.model_feature_importance` | One model and feature representation |
| `03curated.model_run_summary` | One model run/model |

Confirmed sizes:

```text
Prediction rows: 3,638
Metric rows: 6
Feature-importance rows: 162
Run-summary rows: 2
```

Feature importance is published for GBT only at:

```text
ENCODED_FEATURE
LOGICAL_FEATURE
```

Logical importance is normalized to sum to 1.0. Linear Regression coefficient importance is not published.

### Null and invalid handling

- Model training requires non-null target and feature values.
- Positive planned payload is required by model eligibility.
- Categorical unseen/invalid values are retained through `handleInvalid="keep"`.
- Numeric invalid values reaching the assembler cause a visible failure through `handleInvalid="error"`.
- Train and test populations must both be non-empty.
- Predictions and metrics are validated for null and NaN values.

### Validation

```text
Required columns present
Unique source flight_record_key
No null target/features in eligible population
Positive target
Non-empty chronological train/test split
Prediction count = test count for each model
No duplicate (flight_record_key, model_code)
No null/NaN predictions or metrics
Exactly 6 metric rows
Exactly 2 model-summary rows
Logical GBT importance sums to 1.0
```

### What this notebook does not do

```text
No operational anomaly threshold
No governed residual band
No dispatch or safety decision
No certification claim
No fuel prediction
No LR coefficient-importance publication
No causal interpretation of feature importance
```

---

## 8.8 NB06 — Publish Trusted model

### What

Publish the semantic-model-ready `04trusted` layer with stable keys, validated relationships and one governed grain per table.

### Why

Power BI should consume stable facts and dimensions rather than broad Curated tables. Trusted publication creates clear business grains, conformed dimensions, deterministic keys and testable foreign-key relationships.

### How

**Trusted inclusion rule:**

```text
Include flights where has_hard_failure = false
```

Review-warning rows remain in Trusted with DQ fields and analytical eligibility.

Reconciliation:

```text
Curated flight rows: 15,931
Hard-failure rows excluded: 0
Trusted fact_flight rows: 15,931
```

Key strategy:

- Preserve `flight_record_key` for source-record lineage.
- Use deterministic `xxhash64` keys for dimensions and aggregate facts.
- Normalize nulls to `<NULL>` inside stable-key expressions.
- Add `_trusted_run_id` and `_trusted_timestamp_utc` to every Trusted table.

### Date dimension

```text
Start: 2024-06-01
End: 2024-12-31
Rows: 214
```

The dimension begins on the first day of the earliest observed month so route-month facts using month-start keys have valid relationships.

### Trusted dimensions

```text
dim_date
dim_airline
dim_aircraft
dim_origin_airport
dim_destination_airport
dim_route
dim_wind_band
dim_distance_band
dim_dq_rule
dim_analysis_population
dim_model
```

### Trusted facts

```text
fact_flight
fact_dq_rule_trigger
fact_route_month
fact_aircraft_route
fact_model_scoring
fact_model_metric
fact_model_feature_importance
fact_percentile_reference
fact_analysis_population
fact_wind_distribution_global
fact_distance_distribution_global
```

### Fact grains

| Fact | Grain |
|---|---|
| `fact_flight` | One Trusted technical source flight record |
| `fact_dq_rule_trigger` | One triggered DQ rule per Trusted flight |
| `fact_route_month` | One route and month |
| `fact_aircraft_route` | One route and aircraft configuration |
| `fact_model_scoring` | One Trusted holdout flight and model |
| `fact_model_metric` | One model and metric |
| `fact_model_feature_importance` | One model and feature |
| `fact_percentile_reference` | One metric and percentile benchmark |
| `fact_analysis_population` | One analysis population |
| `fact_wind_distribution_global` | One global wind band |
| `fact_distance_distribution_global` | One global distance band |

### Role-playing airport dimensions

Origin and destination are published as separate dimensions. This allows both relationships to be active and avoids inactive-relationship DAX in the interview model.

### Model dimension

`dim_model` ranks models by chronological-test RMSE and marks exactly one `is_selected_model = true`.

Current selected model:

```text
LR — Linear Regression
Selection metric: CHRONOLOGICAL_TEST_RMSE
RMSE: 633.7265365563417
```

### Fleet status validation

The Trusted validation step confirms:

```text
six status fields present in fact_flight
no null status values
valid code/label/sort mappings
status counts reconcile to fact_flight
high-utilisation baseline = 44
payload-review baseline = 128
```

### Trusted validation

```text
Core fact reconciles to no-hard-failure Curated rows
Unique primary keys
No prohibited null dimension keys
Foreign-key validation for all facts
No hard failures in fact_flight
Exactly one selected model
Model scoring reconciles to Curated predictions
Global cumulative percentages end at 1.0 or 100.0
```

### Confirmed result

```text
Status: SUCCEEDED
Dimensions: 11
Facts: 11
Trusted flight rows: 15,931
DQ trigger rows: 358
Model-scoring rows: 3,638
Selected model: LR
Validation: PASS
```

### What this notebook does not do

```text
It does not remove warning-level rows.
It does not load Raw or broad Curated tables into the semantic model.
It does not create report-local logic.
It does not publish an operational anomaly threshold.
```

---

## 9. Cross-notebook handling of null, zero, negative and invalid values

| Stage | Treatment |
|---|---|
| Raw | Preserve source values and rows exactly; no business judgment |
| Standardized | Normalize null tokens, parse values, create parse/status controls; do not delete business-invalid values |
| Curated | Evaluate hard failures, review warnings and use-case eligibility |
| Analytical aggregates | Include only rows eligible for the specific output; mark low-sample cases as not evaluated |
| Predictive | Train/score only model-eligible rows with complete required features |
| Trusted | Exclude hard failures only; retain review warnings and their eligibility flags |
| Semantic model/report | Measures use governed eligibility and DQ fields rather than reimplementing rules |

### 9.1 Example: zero planned payload with positive passengers

```text
Raw: retained
Standardized: parsed as zero
Curated: DQ101 review warning
Payload analysis: not eligible because planned_payload_weight > 0 is required
Model training: not eligible because positive planned payload is required
Trusted: retained because it is not a hard failure
Wind/distance/other unaffected analyses: governed by their own eligibility flags
```

### 9.2 Example: planned TOW greater than MTOW

```text
DQ007 hard failure
→ excluded from Trusted
```

### 9.3 Example: origin equals destination

```text
DQ102 review warning
→ retained in Trusted
→ route use controlled through is_route_analysis_eligible
→ not silently deleted
```

### 9.4 Example: null required field

```text
Standardized parsing/status exposes the null
DQ002 marks a hard failure
NB06 excludes the row from Trusted
```

---

## 10. Semantic model — What, Why and How

### 10.1 What

`SM_EW_Flight_Operations` is the central Power BI semantic model.

```text
Storage mode: Direct Lake on OneLake
Source: EW_Flight_Operations_Lakehouse / 04trusted
Dimensions: 11
Facts: 11
Tables: 22
Report connection: Live connection
```

No Raw, Standardized or broad Curated tables are loaded.

### 10.2 Why

The semantic model provides:

- One governed KPI definition across all pages.
- Consistent slicer behavior.
- Clear fact grains and dimension context.
- Reusable DAX instead of page-specific calculations.
- Direct Lake access without importing the dataset into a report-local model.
- Separation between fixed global benchmarks and dynamic selected-scope calculations.
- A model that can support more than one report without duplicating logic.

### 10.3 How

Relationship principles:

```text
1-to-many
single-direction filtering
dimension → fact
active relationships
no many-to-many
no bidirectional filtering
no fact-to-fact relationships
stable surrogate keys
marked date table
separate origin and destination airport dimensions
```

The handover records 26 base active relationships and a later five-relationship enrichment of `fact_dq_rule_trigger`, producing an expected current total of 31 active relationships.

### 10.4 Main relationship inventory

#### `fact_flight`

```text
dim_date → fact_flight
dim_airline → fact_flight
dim_aircraft → fact_flight
dim_origin_airport → fact_flight
dim_destination_airport → fact_flight
dim_route → fact_flight
dim_wind_band → fact_flight
dim_distance_band → fact_flight
```

#### `fact_dq_rule_trigger`

Base relationships:

```text
dim_dq_rule → fact_dq_rule_trigger
dim_date → fact_dq_rule_trigger
```

Later conformed-dimension enrichment:

```text
dim_airline → fact_dq_rule_trigger
dim_aircraft → fact_dq_rule_trigger
dim_origin_airport → fact_dq_rule_trigger
dim_destination_airport → fact_dq_rule_trigger
dim_route → fact_dq_rule_trigger
```

#### Aggregate facts

```text
dim_date → fact_route_month
dim_route → fact_route_month

dim_aircraft → fact_aircraft_route
dim_route → fact_aircraft_route
```

#### Predictive facts

```text
dim_model → fact_model_scoring
dim_date → fact_model_scoring
dim_airline → fact_model_scoring
dim_aircraft → fact_model_scoring
dim_origin_airport → fact_model_scoring
dim_destination_airport → fact_model_scoring
dim_route → fact_model_scoring

dim_model → fact_model_metric
dim_model → fact_model_feature_importance
```

#### Population and global distribution facts

```text
dim_analysis_population → fact_analysis_population
dim_wind_band → fact_wind_distribution_global
dim_distance_band → fact_distance_distribution_global
```

#### Disconnected benchmark fact

`fact_percentile_reference` is intentionally disconnected. Measures select a metric and percentile explicitly, so the published global values are not changed by ordinary report slicers.

### 10.5 Dedicated measure table

```DAX
_Measures =
ROW (
    "Placeholder",
    BLANK ()
)
```

The placeholder column is hidden. Measures are organized by domain:

```text
01 Core
02 Distributions
03 Percentiles
04 Route and Network
05 Aircraft and Weight
06 Payload and Passenger
07 Data Quality
08 Predictive Analytics
09 Executive Overview
```

### 10.6 Field hygiene

Technical fields are hidden from ordinary report authors unless they are needed for grain, validation or a specific visual.

Hidden categories include:

```text
surrogate keys
foreign keys
run IDs
source hashes
ingestion, standardization, curation, analytics and trusted timestamps
raw JSON metadata
sort-helper fields after sort configuration
helper count fields replaced by measures
```

Business fields remain visible, including measures, route/aircraft labels, DQ status and analysis-eligibility flags.

### 10.7 Date and sort configuration

`dim_date[calendar_date]` is marked as the date column.

Sort metadata:

```text
flight_month_name → flight_month_number
flight_day_of_week_name → flight_day_of_week_number
flight_quarter → flight_quarter_number
flight_year_month → flight_year_month_sort
wind_band_label → wind_band_sort
distance_band_label → distance_band_sort
severity → severity_rank
```

### 10.8 Geography configuration

Origin and destination dimensions contain latitude, longitude, city, country, region, continent and airport-name metadata. Separate role-playing dimensions keep both origin and destination relationships active.

### 10.9 Direct Lake schema synchronization

A physical Delta schema change is not automatically exposed by refreshing the browser or report.

Required sequence:

```text
Run NB06
→ refresh SQL analytics endpoint metadata
→ verify the columns
→ open the semantic model
→ Edit tables / synchronize table schema
→ save the model
→ reopen or refresh the report field list
```

---

## 11. Core DAX and calculation reference

## 11.1 Core measures

```DAX
Flight Count =
SUM ( fact_flight[flight_count] )
```

`COUNTROWS(fact_flight)` is an equivalent earlier implementation, but the current governed fact includes `flight_count = 1`.

```DAX
Distinct Routes =
DISTINCTCOUNT ( fact_flight[route_key] )
```

```DAX
Review Flights =
COALESCE (
    CALCULATE (
        [Flight Count],
        KEEPFILTERS (
            fact_flight[review_status] = "Warning-Level Review"
        )
    ),
    0
)
```

```DAX
Review Rate =
DIVIDE (
    [Review Flights],
    [Flight Count],
    0
)
```

```DAX
Average TOW Utilisation =
AVERAGE ( fact_flight[tow_utilisation_pct] )
```

```DAX
Average Planned Payload =
AVERAGE ( fact_flight[planned_payload_weight] )
```

### Why these patterns are used

- `COALESCE(..., 0)` displays zero rather than blank when no review rows are selected.
- `DIVIDE()` safely handles a zero denominator.
- `KEEPFILTERS()` adds the governed condition without replacing the user's other filters.

---

## 11.2 Dynamic distribution measures

### Wind

```DAX
Wind Eligible Flights =
CALCULATE (
    [Flight Count],
    fact_flight[is_wind_distribution_eligible] = TRUE ()
)
```

```DAX
Wind Cumulative Flights =
VAR CurrentBand =
    MAX ( dim_wind_band[wind_band_sort] )
RETURN
    CALCULATE (
        [Wind Eligible Flights],
        FILTER (
            ALLSELECTED ( dim_wind_band ),
            dim_wind_band[wind_band_sort] <= CurrentBand
        )
    )
```

```DAX
Wind Selected Population =
CALCULATE (
    [Wind Eligible Flights],
    ALLSELECTED ( dim_wind_band )
)
```

```DAX
Wind Cumulative % =
DIVIDE (
    [Wind Cumulative Flights],
    [Wind Selected Population]
)
```

### Distance

The same pattern uses `dim_distance_band[distance_band_sort]` and `is_distance_distribution_eligible`.

### Why `ALLSELECTED` is used

`ALLSELECTED` removes the current individual band while preserving the user's broader slicer selection. This allows each band to compare itself with the selected distribution population rather than the whole unfiltered dataset.

---

## 11.3 Global and selected percentiles

### Fixed global benchmark

```DAX
Global Wind P50 =
CALCULATE (
    MAX ( fact_percentile_reference[percentile_value] ),
    fact_percentile_reference[metric_code] = "WIND",
    fact_percentile_reference[percentile_code] = "P50"
)
```

The P85 and distance measures use the same pattern with the corresponding metric and percentile codes.

### Dynamic selected scope

```DAX
Selected Wind P50 =
PERCENTILEX.INC (
    FILTER (
        ALLSELECTED ( fact_flight ),
        fact_flight[is_wind_distribution_eligible] = TRUE ()
            && NOT ISBLANK ( fact_flight[wind_component_kts] )
    ),
    fact_flight[wind_component_kts],
    0.50
)
```

```DAX
Selected Wind P85 =
PERCENTILEX.INC (
    FILTER (
        ALLSELECTED ( fact_flight ),
        fact_flight[is_wind_distribution_eligible] = TRUE ()
            && NOT ISBLANK ( fact_flight[wind_component_kts] )
    ),
    fact_flight[wind_component_kts],
    0.85
)
```

Distance P50/P85 use `airway_distance_nm` and `is_distance_distribution_eligible`.

### Global versus selected

- **Global P50/P85:** fixed published benchmark from the full governed dataset.
- **Selected P50/P85:** recalculated for the current report selection.
- With no slicers, selected values equal global values.

---

## 11.4 Route and Network measures

```DAX
Average Distance =
CALCULATE (
    AVERAGE ( fact_flight[airway_distance_nm] ),
    fact_flight[is_route_analysis_eligible] = TRUE ()
)
```

```DAX
Routes with Data Quality Review =
CALCULATE (
    DISTINCTCOUNT ( fact_flight[route_key] ),
    fact_flight[has_review_flag] = TRUE ()
)
```

```DAX
Route Sample Share % =
DIVIDE (
    [Flight Count],
    CALCULATE (
        [Flight Count],
        ALLSELECTED ( dim_route[route_code] )
    ),
    0
)
```

```DAX
Route Review Rate =
DIVIDE (
    [Review Flights],
    [Flight Count],
    0
)
```

```DAX
Route Distance Variability =
CALCULATE (
    STDEV.P ( fact_flight[airway_distance_nm] ),
    KEEPFILTERS (
        fact_flight[is_route_analysis_eligible] = TRUE ()
    )
)
```

```DAX
Aircraft Configuration Count =
CALCULATE (
    DISTINCTCOUNT ( fact_flight[aircraft_key] ),
    KEEPFILTERS (
        fact_flight[is_route_analysis_eligible] = TRUE ()
    )
)
```

`STDEV.P` is used because the selected route records are treated as the complete selected population, not as a sample from which another population standard deviation is estimated.

---

## 11.5 Fleet and Payload measures

```DAX
Average Remaining MTOW Margin =
CALCULATE (
    AVERAGE ( fact_flight[remaining_mtow_margin_weight] ),
    KEEPFILTERS (
        fact_flight[is_weight_analysis_eligible] = TRUE ()
    )
)
```

```DAX
High TOW Utilisation Flights =
COALESCE (
    CALCULATE (
        [Flight Count],
        KEEPFILTERS (
            fact_flight[tow_utilisation_status_code]
                = "HIGH_UTILISATION"
        )
    ),
    0
)
```

```DAX
High TOW Utilisation Rate =
DIVIDE (
    [High TOW Utilisation Flights],
    [Flight Count],
    0
)
```

```DAX
Median TOW Utilisation =
CALCULATE (
    MEDIAN ( fact_flight[tow_utilisation_pct] ),
    KEEPFILTERS (
        fact_flight[is_weight_analysis_eligible] = TRUE ()
    )
)
```

```DAX
Aircraft Share Within Route % =
VAR RouteFlightCount =
    CALCULATE (
        [Flight Count],
        ALLSELECTED (
            dim_aircraft[aircraft_configuration_code]
        )
    )
RETURN
    DIVIDE (
        [Flight Count],
        RouteFlightCount,
        0
    )
```

```DAX
Payload Review Flights =
COALESCE (
    CALCULATE (
        [Flight Count],
        KEEPFILTERS (
            fact_flight[payload_review_status_code]
                = "REQUIRES_REVIEW"
        )
    ),
    0
)
```

---

## 11.6 Predictive measures

```DAX
Model Scored Flights =
COALESCE (
    DISTINCTCOUNT (
        fact_model_scoring[flight_record_key]
    ),
    0
)
```

```DAX
Model MAE =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    KEEPFILTERS (
        fact_model_metric[metric_name] = "MAE"
    )
)
```

```DAX
Model RMSE =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    KEEPFILTERS (
        fact_model_metric[metric_name] = "RMSE"
    )
)
```

```DAX
Model R² =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    KEEPFILTERS (
        fact_model_metric[metric_name] = "R_SQUARED"
    )
)
```

Filter boundary:

- `Model Scored Flights` responds to scoring-fact filters such as route, airline, aircraft and prediction direction.
- MAE, RMSE and R² are published model-level holdout metrics. They respond to model selection, but they must not be described as route-specific recalculations.

---

## 11.7 Executive Overview display measures

```DAX
Overview Wind Envelope =
FORMAT ( [Selected Wind P50], "0;-0;0" )
    & " / "
    & FORMAT ( [Selected Wind P85], "0;-0;0" )
    & " kts"
```

The explicit negative format prevents `-3` from appearing as `3`.

```DAX
Overview Distance Envelope =
FORMAT ( [Selected Distance P50], "#,0" )
    & " / "
    & FORMAT ( [Selected Distance P85], "#,0" )
    & " NM"
```

```DAX
Overview Average Distance =
FORMAT ( [Average Distance], "#,0" ) & " NM"
```

```DAX
Overview Selected Model Name =
CALCULATE (
    SELECTEDVALUE ( dim_model[model_name] ),
    REMOVEFILTERS ( dim_model ),
    dim_model[is_selected_model] = TRUE ()
)
```

```DAX
Overview Selected Model RMSE =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    REMOVEFILTERS ( dim_model ),
    dim_model[is_selected_model] = TRUE (),
    fact_model_metric[metric_name] = "RMSE"
)
```

```DAX
Overview Selected Model R² =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    REMOVEFILTERS ( dim_model ),
    dim_model[is_selected_model] = TRUE (),
    fact_model_metric[metric_name] = "R_SQUARED"
)
```

```DAX
Overview Predictive Metrics =
FORMAT ( [Overview Selected Model RMSE], "#,0.0" )
    & " | "
    & FORMAT ( [Overview Selected Model R²], "0.000" )
```

`REMOVEFILTERS(dim_model)` is deliberate here: the Overview always displays the governed selected model, not whichever comparison model happens to be selected elsewhere.

---

## 12. Power BI report — visible pages only

The report contains six visible pages. Each page follows the same decision-product principle:

```text
Signal
→ diagnosis
→ underlying records or priority queue
→ defensible interpretation
→ next action or required validation
```

Common design:

```text
1280 × 720 / 16:9 canvas
white header
Eurowings Digital logo
cyan and burgundy accent rule
page-aware navigator
fixed right filter rail
footer with source, model and refresh context
```

## 12.1 Page 00 — Executive Overview

### What

A one-page summary of scale, operating envelope, route review concentration, Fleet/Payload review conditions, predictive model performance and trusted-data status.

### Why

An interviewer or operational stakeholder should understand the analytical product in under one minute before opening the detailed pages.

### How

#### KPI row

| KPI | Baseline |
|---|---:|
| Flight Count | 15,931 |
| Distinct Routes | 147 |
| Data Quality Review Flights | 349 |
| Data Quality Review Rate | 2.2% |

#### Panel 1 — Operating Envelope

```text
Wind P50 / P85: -3 / 22 kts
Distance P50 / P85: 593 / 1,105 NM
```

**What:** Selected-scope median and upper coverage benchmark.  
**Why:** Summarizes where the central 50% and 85% boundaries of the current population lie.  
**How:** Dynamic percentile measures over eligible selected flights, formatted with units.

#### Panel 2 — Network & Quality

```text
Average Distance: 728 NM
Routes with Review Records: 88
```

**What:** Network scale and route-level spread of DQ review.  
**Why:** Shows that review attention is distributed across many routes rather than limited to a single route.  
**How:** Filter-aware average distance and distinct route count under `has_review_flag = true`.

#### Panel 3 — Fleet & Payload

```text
Above 95% TOW Utilisation: 44
Payload Review Flights: 128
```

**What:** Counts of governed validation conditions.  
**Why:** Directs attention to records requiring source or fleet-context validation.  
**How:** Physical status fields published in `fact_flight` and counted through governed measures.

#### Panel 4 — Predictive Review

```text
Selected Model: Linear Regression
Holdout RMSE | R²: 633.7 | 0.992
```

**What:** Current governed model and its chronological-holdout performance.  
**Why:** Demonstrates that the model is selected from tested alternatives, not chosen arbitrarily.  
**How:** Measures deliberately remove ordinary model-selection context and read the `is_selected_model` flag.

#### Trusted Data Foundation strip

```text
15,931 reconciled flight records | 0 hard failures | 349 review flights retained
```

**What:** End-to-end trust baseline.  
**Why:** Separates source reconciliation, hard exclusions and retained warnings.  
**How:** Uses distinct flight counts, not DQ trigger-row counts.

### Page analysis

- The selected population is dominated by flights at or below 593 NM, while 85% are at or below 1,105 NM.
- Review flags affect 349 flights but do not reduce the Trusted row count because no hard failure was found.
- The predictive prototype performs strongly on the December holdout, but the Overview explicitly labels it as analytical triage rather than operational certification.

### Interaction

Common slicers include date, airline, aircraft, origin, destination and route. All dynamic KPI and operating-envelope values respond to the relevant filters, while the selected-model cards remain tied to the governed selected model.

### Limitation

The Overview is a summary. It does not provide the detailed records needed for diagnosis; users must open the corresponding analytical page.

---

## 12.2 Page 01 — Wind & Distance Distributions

### What

The mandatory case-study page showing flight distributions by wind and distance, cumulative coverage and global P50/P85 benchmarks.

### Why

A frequency distribution answers where flights are concentrated. The cumulative line answers how much of the selected flight population is covered as the band value increases. P50 and P85 provide transparent reference points.

### How

#### KPI row

```text
Flight Count
Wind-Eligible Flights
Distance-Eligible Flights
Data Quality Review Rate
```

The unfiltered eligible populations are 15,931 for both wind and distance.

#### Wind visual

```text
Columns: Wind Eligible Flights
Band size: 10 kts
Line: Wind Cumulative %
Global reference: P50 = -3 kts
Global reference: P85 = 22 kts
Dynamic selected P50/P85 display
```

#### Distance visual

```text
Columns: Distance Eligible Flights
Band size: 100 NM
Line: Distance Cumulative %
Global reference: P50 = 593 NM
Global reference: P85 = 1,105 NM
Dynamic selected P50/P85 display
```

#### Record list

Title:

```text
Flights in Selected Distribution Band
```

Fields:

```text
Flight Date
Airline
Route
Aircraft Configuration
Wind Component
Airway Distance
Planned TOW
Planned Payload
Review Status
```

A technical `flight_record_key` is available when grain must be enforced, but it is hidden from ordinary presentation unless needed.

### Page analysis

- Half of the governed wind population is at or below `-3 kts`.
- Eighty-five percent is at or below `22 kts`.
- Half of the governed distance population is at or below `593 NM`.
- Eighty-five percent is at or below `1,105 NM`.
- Values beyond P85 form the upper 15% tail and are candidates for deeper route, aircraft and source-context analysis.

### P50 versus cumulative 50% crossing

They represent the same conceptual boundary but are displayed differently:

- P50 is calculated from individual eligible flight values.
- The cumulative line is grouped into bands.
- The line crosses 50% in the band containing the median; the band label may not equal the exact P50 value because of binning.

### Interaction

```text
Click a wind band → filter the record list to flights in that band
Click a distance band → filter the record list to flights in that band
```

The two distribution charts are configured not to create confusing mutual distortion by default.

### Decision guide

```text
Use P50 to describe the typical selected population.
Use P85 as an upper coverage benchmark for scenario design.
Inspect tail-band records before drawing operational conclusions.
Do not infer delay, fuel burn or operational risk from wind or distance alone.
```

---

## 12.3 Page 02 — Network & Route Priorities

### What

A route-level prioritization page combining volume, distance variability, monthly pattern and DQ review concentration.

### Why

Volume alone does not determine analytical priority. A route can be important because it has many flights, unusually variable travelled distance, concentrated review warnings or several aircraft configurations. The page combines these signals and provides an investigation queue.

### How

#### KPI row

| KPI | Unfiltered value |
|---|---:|
| Flight Count | 15,931 |
| Distinct Routes | 147 |
| Average Distance | 728 NM |
| Routes with Data Quality Review | 88 |

#### Top Routes by Flight Volume

```text
Visual: Clustered horizontal bar chart
Category: dim_route[route_code]
Value: [Flight Count]
Filter: Top 10 by Flight Count
Sort: Descending
```

Tooltip measures:

```text
Flight Count
Route Sample Share %
Average Distance
Review Flights
Route Review Rate
```

Documented unfiltered sanity checks:

```text
PMI-DUS  1,332 flights
BER-DUS    606 flights
VIE-DUS    459 flights
MUC-DUS    447 flights
```

These are sample counts and sample shares, not market share.

#### Routes with Highest Distance Variability

```text
Visual: Clustered horizontal bar chart
Category: dim_route[route_code]
Value: [Route Distance Variability]
Minimum selected route sample: 10 flights
Filter: Top 10 by variability
Sort: Descending
```

A scatter-based priority matrix was rejected because one high-volume route compressed the other points and reduced readability.

Documented sanity checks:

```text
BER-DXB  120 NM
CGN-DWC   87 NM
MAH-DUS   76 NM
STR-DWC   44 NM
```

Variability is the population standard deviation of eligible travelled distance for a route. It is not a performance score.

#### Observed Monthly Flight Profile

```text
Visual: Line chart
X-axis: dim_date[flight_year_month]
Y-axis: [Flight Count]
Sort: flight_year_month_sort
```

The chart shows the selected route or network volume by month. It is labelled observed monthly profile because the source covers only June–December 2024.

#### Route Investigation Queue

Fields:

```text
Route
Flights
Sample Share
Average Distance
Distance Variability
Review Flights
Review Rate
Aircraft Configuration Count
```

Filter:

```text
Flight Count >= 10
```

Sort:

```text
Review Flights descending
```

Totals are turned off because summing route-level percentages and variability is not meaningful.

### Page analysis

- Route concentration is visible through the top-volume chart.
- 88 routes contain at least one review flight, indicating a distributed validation requirement.
- Routes with high distance variability require trajectory, routing or source-definition investigation rather than an immediate performance conclusion.
- Low-sample routes are protected by a minimum sample rule before variability ranking.

### Interaction

```text
Top Route selection
→ filters Monthly Profile and Route Investigation Queue
→ does not change KPI cards

Distance Variability selection
→ filters Monthly Profile and Route Investigation Queue
→ does not change KPI cards

Month selection
→ filters Route Investigation Queue
→ does not change route-ranking charts or KPIs
```

### Decision guide

```text
Prioritise routes combining high volume, distance variability or review concentration.
Validate low-sample routes before comparison.
Sample share is not market share or route performance.
```

---

## 12.4 Page 03 — Fleet & Payload

### What

A flight and aircraft-configuration page for planned TOW utilisation, remaining MTOW margin, payload/passenger relationships and route deployment.

### Why

The page identifies records and configurations that require validation while avoiding unsupported safety or efficiency conclusions.

### How

#### KPI row

| KPI | Unfiltered value |
|---|---:|
| Average TOW Utilisation | 80% |
| Average Remaining MTOW Margin | 15,806 |
| Average Planned Payload | 12,061 |
| High TOW Utilisation Flights | 44 |

No weight unit is displayed because the source unit is unconfirmed.

#### TOW Utilisation by Aircraft Configuration

```text
Visual: Clustered horizontal bar chart
Y-axis: dim_aircraft[aircraft_configuration_code]
X-axis: Average TOW Utilisation, Median TOW Utilisation
Sort: Average utilisation descending
Axis: Starts at zero
```

Tooltip context includes flight count, remaining margin, high-utilisation count and rate.

**Interpretation:** Higher planned TOW utilisation means less remaining supplied MTOW margin. It does not establish unsafe operation, fuel efficiency or operational performance.

#### Planned Payload versus Passenger Count

```text
Visual: Flight-level scatter
Grain: flight_record_key
X-axis: Maximum passenger_count
Y-axis: Maximum planned_payload_weight
Legend: aircraft model
Filter: is_payload_analysis_eligible = true
Eligible rows: 15,803
```

`Maximum` is used because each scatter point is already at one-flight grain; summing the values would be wrong.

Power BI may use high-density sampling because 15,803 points exceed normal individually rendered limits. This affects display density, not the underlying dataset.

**Interpretation:** Planned payload varies substantially at similar passenger counts. Passenger count alone does not explain planned payload. The data does not isolate baggage, cargo or other payload components.

#### Aircraft Deployment by Route

```text
Visual: Matrix
Rows: dim_route[route_code]
Columns: dim_aircraft[aircraft_configuration_code]
Values: Flight Count
Filter: Top 10 routes by Flight Count
Background colour: Aircraft Share Within Route %
```

The heat map shows deployment concentration, not good or bad performance.

#### High TOW Utilisation Review table

```text
Flight Date
Route
Aircraft Configuration
Planned TOW
Aircraft MTOW
TOW Utilisation
Remaining MTOW Margin
optional DQ status / technical key
```

Filter:

```text
tow_utilisation_status_code = HIGH_UTILISATION
```

Sort:

```text
TOW Utilisation descending
```

The aircraft configuration field uses the actual code with `Don't summarize`. A previous count aggregation produced misleading values and was corrected.

### Page analysis

- Forty-four flights exceed the governed 95% utilisation review threshold.
- One hundred twenty-eight records require payload review because planned payload is zero while passengers are positive.
- Payload variation at similar passenger counts demonstrates why passenger count cannot be treated as a complete payload explanation.
- Multiple MTOW variants for an aircraft type require fleet-master or registration data before being classified as erroneous.

### Interaction

```text
Slicers → filter all KPIs and visuals
TOW chart → filters diagnostic visuals and review table, not KPIs
Payload scatter → may filter review table, not KPI cards
Deployment matrix → does not filter the High TOW table by default
Review table → does not filter other visuals
```

The matrix-to-review-table interaction is disabled because most route/configuration intersections contain none of the 44 high-utilisation flights and would make the table appear incorrectly empty.

### Decision guide

```text
Prioritise high-utilisation flights and concentrated route–configuration patterns for validation.
These are review signals, not safety or performance conclusions.
```

---

## 12.5 Page 04 — Predictive Review

### What

A model-comparison and residual-review page for supplied versus predicted planned TOW.

### Why

Aggregate model metrics show overall holdout performance, but they do not show which records create the largest errors. The page combines overall metrics, point-level comparison, explainability and a residual ranking table.

### How

#### KPI row

| Model | Scored flights | MAE | RMSE | R² |
|---|---:|---:|---:|---:|
| Linear Regression | 1,819 | 398.2 | 633.7 | 0.992 |
| Gradient-Boosted Tree | 1,819 | 732.4 | 1,061.3 | 0.979 |

The model slicer is single-select. The intended default is Linear Regression.

#### Supplied versus Predicted Planned TOW

```text
Visual: Scatter
Values/grain: fact_model_scoring[flight_record_key]
X-axis: Maximum actual_planned_tow_weight
Y-axis: Maximum predicted_planned_tow_weight
Legend: prediction_direction
Axis ranges: approximately 40,000 to 90,000 on both axes
```

Equal axis ranges prevent visual exaggeration of error.

#### GBT Logical Feature Importance

```text
Visual: Clustered bar chart
Category: logical_feature_name
Value: Sum importance_value
Visual filter: model_code = GBT
Visual filter: importance_level = LOGICAL_FEATURE
```

The Model slicer does not filter this chart. Otherwise selecting Linear Regression would blank the only published feature-importance output.

Feature importance shows normalized association within the GBT model, not causality and not Linear Regression coefficients.

#### Flights Ranked by Absolute Residual

Fields:

```text
Flight Date
Airline
Route
Aircraft Configuration
Supplied Planned TOW
Predicted Planned TOW
Absolute Residual
Prediction Direction
```

The table uses `flight_record_key` to preserve row grain and is sorted by absolute residual descending.

### Page analysis

- Linear Regression is the governed selected model because it performs better on all three holdout metrics.
- The high R² indicates a strong fit for this supplied dataset and feature set, but does not validate operational deployment.
- The residual ranking is the actionable output: it identifies the flights where supplied and predicted planned TOW differ most.
- Feature importance is GBT-only and cannot be used to explain the selected Linear Regression model.

### Interaction

```text
Model slicer
→ filters KPIs, scatter and residual table
→ does not filter GBT feature importance

Prediction Direction
→ filters scatter and residual table

Scatter selection
→ filters residual table

Feature Importance chart
→ does not filter KPIs, scatter or residual table
```

### Decision guide

```text
Use absolute residuals to prioritise flights for investigation.
Validate data quality and aircraft/route context first.
GBT importance shows model association, not causality.
The prototype does not support dispatch, safety or certification decisions.
```

### Deferred features

```text
No governed anomaly threshold
No residual-band slicer
No model-review status
No report-local binning
No LR coefficient importance
```

---

## 12.6 Page 05 — Data Quality & Trust

### What

A rule-level and record-level page showing what was flagged, how many records are affected, which analyses exclude those records and why Trusted still contains review-warning rows.

### Why

A DQ summary without underlying records is not actionable. The page demonstrates that data quality is governed at both rule and flight level and that warnings are not automatically deleted.

### How

#### KPI row

```text
Flight Count
Hard Failure Count
Review Flights
Review Rate
```

Baseline:

```text
15,931 Trusted flights
0 hard failures
349 unique review flights
2.2% review rate
358 rule triggers
```

#### Data-Quality Rule Triggers chart

Shows trigger counts by DQ rule. The principal validated counts are:

```text
DQ101  128  zero payload with positive passengers
DQ102    5  origin equals destination
DQ103  175  absolute wind above 60 kts
DQ104    5  absolute wind above 80 kts
DQ105   44  TOW utilisation above 95%
DQ106    1  legacy airline code
```

The counts total 358. Because some flights trigger several rules, the distinct review-flight count is 349.

#### Global Analysis Exclusions by Use Case

Uses `fact_analysis_population` to show total, eligible and excluded populations for wind, distance, route, weight, payload, carrier comparison and model training.

**Why:** A record can be acceptable for one analysis and excluded from another. The visual prevents a warning from being interpreted as a universal deletion.

#### Selected rule-context banner

Displays the selected rule context so the user understands why the record table changed.

#### Record-level DQ table

Shows the flight records behind the selected rule or analysis exclusion, with business context and review reason. The enriched DQ trigger fact supports conformed date, airline, aircraft, airport and route filtering.

### Page analysis

- There are no hard failures, so all 15,931 source flight rows reach Trusted.
- The largest review category is absolute wind above 60 kts, but five of those records also exceed 80 kts.
- Payload-zero-with-passengers affects 128 records and explains the difference between total flights and payload/model eligible populations.
- Warning flags represent validation needs, not operational failures.

### Interaction

```text
Select a DQ rule
→ update the rule banner
→ filter the contributing record table

Select an analysis population/exclusion
→ show records excluded from that analytical use case
```

### Decision guide

```text
Review flags are not automatic deletions.
Exclude a record only from analyses affected by the rule.
Hard failures are the only Trusted exclusion criterion.
Validate record context before correction or source escalation.
```

---

## 13. Data-quality treatment and analytical eligibility reference

### 13.1 Hard failure versus review warning

| Classification | Trusted treatment | Analytical treatment |
|---|---|---|
| Hard failure | Excluded from `fact_flight` | Not available to report analysis |
| Review warning | Retained in `fact_flight` | Included or excluded through use-case eligibility |

### 13.2 Why this matters

Deleting every warning-level record would reduce transparency and unnecessarily remove valid information. For example, zero planned payload with positive passengers is unsuitable for payload analysis, but the row can still carry valid wind, distance, date, route and aircraft information.

### 13.3 Governed eligibility conditions explicitly documented

Weight analysis:

```text
planned_tow_weight > 0
aircraft_mtow_weight > 0
planned_tow_weight <= aircraft_mtow_weight
```

Payload analysis:

```text
planned_payload_weight > 0
planned_payload_weight <= planned_tow_weight
passenger_count > 0
```

Model training:

```text
no hard failure
non-null target and features
positive planned payload
is_model_training_eligible = true
```

Wind, distance and route calculations use their own governed eligibility fields rather than a generic valid-record flag.

---

## 14. Row counts, hashes and lineage controls

### 14.1 Reconciliation chain

```text
Flight source workbook:        15,931
01raw.flight_raw:              15,931
02standardized.flight_standardized: 15,931
03curated.flight_enriched:     15,931
04trusted.fact_flight:         15,931
```

No flight row was lost because the current source contains zero hard failures.

### 14.2 Distinct DQ grains

```text
Unique review flights: 349
DQ trigger rows:        358
```

The difference is expected because `fact_dq_rule_trigger` has one row per rule per flight.

### 14.3 Predictive reconciliation

```text
Training rows: 13,984
Holdout flights: 1,819
Models: 2
Scoring rows: 1,819 × 2 = 3,638
Metric rows: 2 models × 3 metrics = 6
```

### 14.4 Technical lineage prefixes

```text
_source_*
_ingestion_*
_pipeline_*
_standardization_*
_curation_*
_analytics_*
_model_*
_trusted_*
```

### 14.5 Hash and key strategy

- Raw and Standardized rows receive record hashes for deterministic comparison.
- Trusted dimensions and aggregate facts use `xxhash64` stable keys.
- Null components are normalized to `<NULL>` before key hashing.
- `flight_record_key` is retained as the technical source-record key.

---

## 15. Code and modelling conventions

```text
Notebook names: NB01, NB01R, NB02, NB02R, NB03, NB04, NB05, NB06
Pipeline activities: ACT_<NotebookName>
Physical tables and columns: snake_case
Schema-qualified Spark references: `schema`.`table`
UTC timestamps
Deterministic stable keys
Explicit required-column checks
Explicit row-count, uniqueness and foreign-key checks
Delta overwrite with overwriteSchema=true for the static case-study snapshot
No silent exception swallowing
No repeated business rules in report-local calculated columns
```

### 15.1 Step error wrapper

Each executable stage is wrapped by a lightweight helper such as:

```python
run_step(step_name, action)
```

Behavior:

```text
Print STARTED
Execute the stage
Print SUCCEEDED
On error: print notebook, step, exception type and message
Re-raise RuntimeError so the Fabric activity fails visibly
```

### 15.2 Ratio handling

A reusable safe-ratio pattern prevents null or zero denominators from causing notebook failure.

### 15.3 Statistical functions

```text
Global percentiles: approxQuantile(relativeError=0.0)
Grouped percentiles: percentile_approx(accuracy=100000)
Cumulative distributions: ordered Spark window functions
Correlation: pairwise-complete Pearson correlation
Outliers: route-specific 1.5 × IQR with minimum route sample
```

### 15.4 DAX conventions

```text
Measures in _Measures
Display folders by analytical domain
COALESCE for count measures where zero is meaningful
DIVIDE for ratios
KEEPFILTERS for governed conditions
REMOVEFILTERS only for intentionally fixed global/selected-model context
Descriptions written in business language
Numeric base measures remain numeric
Unit-bearing display measures remain separate text measures
```

---

## 16. Resolved technical issues and design decisions

| Issue | Resolution | Why it matters |
|---|---|---|
| Status measures could not populate slicers | Published physical status code/label/sort fields in NB03 and NB06 | Slicers require columns, not measures |
| New Direct Lake columns did not appear | Synchronized semantic-model table schema after SQL endpoint refresh | Browser refresh alone does not update table metadata |
| Scatter rejected unsummarized numeric axes | Used `flight_record_key` as point grain and `Maximum` for one-row numeric values | Prevents invalid aggregation and enables flight-level points |
| Predictive KPIs mixed LR and GBT | Model slicer set to single-select; model-aware measures used | Avoids combining model-level metrics |
| Scatter contained excessive empty space | Applied comparable custom X/Y ranges | Prevents visual compression and exaggeration |
| Copied visuals retained wrong labels/tooltips | Replaced titles, axes and tooltip fields | Avoids presenting Fleet fields as predictive fields |
| Absolute residual slicer lacked meaning | Removed it | No approved threshold existed |
| Residual bins required local model | Deferred bins rather than converting the report | Preserved central Direct Lake governance |
| GBT feature chart blanked under LR | Disabled model-slicer interaction for the GBT-only chart | Keeps published explainability visible |
| Feature importance wording implied causality | Reworded as normalized model association | Prevents unsupported causal claim |
| Matrix selection blanked High TOW table | Disabled matrix-to-table interaction | Most intersections contain none of 44 high-utilisation rows |
| Negative wind P50 displayed without sign | Used explicit `0;-0;0` format | Preserves `-3` correctly |
| Overview predictive card was unreadable | Split model name and metric summary into separate cards | One measure per card improves clarity |

---

## 17. Interview question and answer reference

### 17.1 Why did you build a Fabric pipeline instead of loading Excel directly into Power BI?

The task could have been completed directly in Power BI, but that would mix ingestion, cleansing, DQ, analysis and presentation. The Fabric design preserves source lineage, makes transformations testable and reusable, publishes a central semantic model and supports end-to-end reruns through one pipeline trigger.

### 17.2 What happens to a row with zero payload and positive passengers?

It remains in Raw and Standardized. NB03 triggers `DQ101`, classifies it as a review warning, makes it ineligible for payload analysis and model training, and retains it in Trusted because it is not a hard failure. It can still contribute to analyses for which its eligibility flag remains true.

### 17.3 What happens when origin equals destination?

The row triggers `DQ102` and remains in Trusted as a review-warning record. Its use in route analysis is controlled by `is_route_analysis_eligible`; it is not silently deleted.

### 17.4 What would be a hard failure?

Examples include a missing required business field, non-positive distance/PAX/TOW/MTOW, planned TOW greater than MTOW, negative payload, payload greater than planned TOW, invalid IATA format, missing master-data matches or an exact standardized duplicate.

### 17.5 Did the source contain any hard failures?

No. All 15,931 flight rows reconciled through Trusted. The solution still implements and validates the rules so future runs would exclude hard-failure rows.

### 17.6 Why are there 358 rule triggers but 349 review flights?

The rule-trigger fact has one row per triggered rule per flight. Some flights trigger more than one rule, so trigger rows are greater than distinct affected flights.

### 17.7 Why do you use P50 and P85?

P50 describes the median or typical boundary: half of eligible flights are at or below it. P85 gives an upper coverage benchmark: 85% are at or below it. They are transparent planning and comparison benchmarks, not operational risk limits.

### 17.8 Is the point where the cumulative line reaches 50% the same as P50?

Conceptually yes. P50 is calculated from individual values, while the cumulative chart is grouped into bands. The cumulative line reaches 50% in the band that contains the median, so the band boundary may not exactly equal the unbinned P50 value.

### 17.9 Why did you calculate both global and selected percentiles?

The global values provide a fixed baseline for comparison. Selected values recalculate under date, airline, aircraft, route or airport filters. With no selection, both match.

### 17.10 Why was route distance variability calculated?

The same route can have different travelled airway distances because of operational routing. Population standard deviation quantifies how dispersed the selected distances are. It helps prioritize investigation, but it is not a route performance score.

### 17.11 Why use route-specific IQR outliers?

A global distance threshold would incorrectly compare short and long routes. Route-specific P25/P75 and 1.5×IQR bounds evaluate a flight against its route's own historical distribution. Routes with fewer than 10 eligible observations are not evaluated.

### 17.12 Why use median as a route baseline?

Median is less affected by unusually high or low distance values than the mean. It therefore provides a robust route baseline for travelled distance.

### 17.13 Why was `planned_payload_per_pax_proxy` created?

It compares supplied planned payload with passenger count and helps identify unusual relationships. It is explicitly a proxy because payload can include components other than passengers and the weight unit is unconfirmed.

### 17.14 Why did you not label weight fields in kilograms?

The source does not document the unit. Adding `kg` would create false certainty. Technical names use `_weight`, and visuals avoid a specific unit.

### 17.15 Why do you call the field Wind Component rather than headwind/tailwind?

The sign convention is not documented. Using headwind or tailwind would assume semantics that the source does not confirm.

### 17.16 Why is Linear Regression selected instead of GBT?

On the chronological December holdout, Linear Regression has lower MAE and RMSE and higher R². The selection is based on governed holdout metrics, specifically RMSE ranking in `dim_model`.

### 17.17 Why use a chronological holdout?

It keeps the latest month unseen during training and tests generalization to a later period. A random split could mix earlier and later operating patterns and create time leakage.

### 17.18 Does R² = 0.992 mean the model is production-ready?

No. It indicates strong holdout fit for the supplied target and features. The source still lacks confirmed weight units, actual TOW, registration, weather detail, flight-plan context and operational outcomes. The model is a triage prototype, not operational certification.

### 17.19 Why is GBT feature importance shown when Linear Regression is selected?

The pipeline publishes normalized feature importance only for GBT. The chart is explicitly labelled GBT Logical Feature Importance and is isolated from the LR model slicer. It demonstrates one model's internal association structure; it does not explain LR coefficients.

### 17.20 Why not create residual bands in Power BI?

A governed threshold was not approved, and report-local bins would require a local/composite model. The design preserved Direct Lake governance and used a residual ranking table instead.

### 17.21 Why Direct Lake?

Direct Lake allows the semantic model to query the Trusted Lakehouse tables through OneLake while keeping one central model and avoiding report-local imports. It supports governed reusable measures and consistent relationships.

### 17.22 Why are origin and destination separate dimensions?

Separate role-playing dimensions allow both relationships to be active and keep report filtering simple. A single airport dimension would require inactive relationships and additional DAX logic.

### 17.23 Why is `fact_percentile_reference` disconnected?

The table stores fixed global benchmarks. Disconnecting it prevents route, airline or aircraft slicers from changing the published values. Explicit measures select the required metric and percentile.

### 17.24 Why not use bidirectional relationships?

Single-direction dimension-to-fact filtering provides predictable behavior and avoids ambiguous filter paths. The model has no fact-to-fact or many-to-many relationships.

### 17.25 How do you ensure that filters do not change published model metrics incorrectly?

MAE, RMSE and R² come from the model-metric fact and respond to model selection. They are not recalculated by route or aircraft. The Overview uses the governed selected-model flag and removes ordinary model context intentionally.

### 17.26 What does the Data Quality page prove?

It proves that source records are reconciled, hard failures are separated from warnings, warnings remain traceable to exact rules and records, and eligibility is determined per analysis rather than through one blanket valid/invalid flag.

### 17.27 What additional data would improve the solution?

The handover identifies the need for confirmed weight units and signed-wind semantics. It also notes that aircraft registration/fleet master, actual TOW, richer weather, flight-plan/trajectory context and operational outcomes would be required before stronger operational or predictive conclusions.

---

## 18. Compact formula cheat sheet

| Calculation | Formula | Purpose |
|---|---|---|
| TOW utilisation | `planned_tow / aircraft_mtow` | Share of supplied MTOW used by planned TOW |
| Remaining MTOW margin | `aircraft_mtow - planned_tow` | Remaining supplied margin |
| Payload/TOW | `planned_payload / planned_tow` | Payload relative to planned TOW |
| Payload/MTOW | `planned_payload / aircraft_mtow` | Payload relative to supplied MTOW |
| Non-payload planned mass | `planned_tow - planned_payload` | Planned TOW not represented by supplied payload |
| Payload/PAX proxy | `planned_payload / passenger_count` | Compare payload and passenger count |
| Flight share | `band_count / eligible_population` | Share of eligible flights in a band |
| Cumulative percentage | `cumulative_count / eligible_population` | Coverage through the current ordered band |
| IQR | `P75 - P25` | Middle-50% spread |
| Outlier lower bound | `P25 - 1.5×IQR` | Route-specific low-distance threshold |
| Outlier upper bound | `P75 + 1.5×IQR` | Route-specific high-distance threshold |
| Prediction error | `predicted - supplied` | Signed model difference |
| Residual | `supplied - predicted` | Opposite sign convention |
| Absolute residual | `ABS(predicted - supplied)` | Error magnitude for ranking |
| MAE | `mean(abs(error))` | Average error magnitude |
| RMSE | `sqrt(mean(error²))` | Error metric emphasizing large differences |
| Review rate | `review_flights / flight_count` | Share of selected flights with warnings |
| Route sample share | `route_flights / selected_all_route_flights` | Route share within the selected sample |

---

## 19. Technical glossary

| Term | Compact meaning |
|---|---|
| Grain | What one row represents in a table |
| Raw | Source-preserving layer with lineage only |
| Standardized | Typed, normalized technical contract |
| Curated | Business-enriched, DQ-controlled analytical layer |
| Trusted | Stable facts and dimensions approved for semantic consumption |
| Direct Lake | Power BI storage mode reading governed OneLake tables through a semantic model |
| Dimension | Descriptive table used to filter facts, such as route or aircraft |
| Fact | Table containing measurable events or aggregates at a defined grain |
| P50 | Median; 50% of observations are at or below it |
| P85 | 85th percentile; 85% are at or below it |
| Cumulative count | Running total through the current ordered band |
| Cumulative percentage | Running total divided by the selected population |
| Mean | Arithmetic average |
| Median | Middle value after ordering observations |
| Standard deviation | Typical dispersion around the mean |
| IQR | P75 minus P25; spread of the middle 50% |
| Outlier | Observation outside the defined route-specific IQR bounds |
| Correlation | Strength and direction of linear association, not causality |
| Holdout | Data not used during model training, reserved for evaluation |
| Residual | Difference between supplied and predicted target using the documented sign convention |
| MAE | Mean absolute error |
| RMSE | Root mean squared error |
| R² | Coefficient of determination |
| Review warning | Record retained for validation and controlled by analytical eligibility |
| Hard failure | Record excluded from Trusted |
| Surrogate key | Technical stable key used for dimensional relationships |
| Role-playing dimension | Separate dimension copies representing different roles, such as origin and destination |
| `KEEPFILTERS` | Adds a measure condition while preserving existing filter context |
| `ALLSELECTED` | Removes the current category filter while preserving broader user selections |
| `REMOVEFILTERS` | Deliberately clears a filter when a fixed benchmark or governed selection is required |

---

## 20. Final validated implementation status

### Engineering

```text
Raw, Standardized, Curated and Trusted layers implemented
Eight notebook activities validated
15,931 rows reconciled end to end
0 hard failures
349 review flights retained
11 analytical aggregate outputs
LR and GBT holdout models evaluated
11 Trusted dimensions and 11 Trusted facts published
End-to-end Fabric pipeline succeeded
```

### Semantic model

```text
Direct Lake central model
22 Trusted tables
single-direction star-schema relationships
role-playing origin/destination dimensions
governed measure folders
hidden technical fields
marked date dimension
static global and dynamic selected percentile logic
```

### Report

```text
00 Executive Overview
01 Distributions
02 Network & Route Priorities
03 Fleet & Payload
04 Predictive Review
05 Data Quality & Trust
```

All six visible pages contain substantive analytical content. The report preserves the mandatory distribution task as the primary requirement and extends it with route, fleet, payload, predictive and DQ decision support without exceeding the source's analytical limits.
