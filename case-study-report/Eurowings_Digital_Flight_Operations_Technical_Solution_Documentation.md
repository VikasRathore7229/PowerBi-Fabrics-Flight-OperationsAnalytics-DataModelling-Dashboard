# Flight Operations Analytics
## Technical Solution Documentation

**Eurowings Digital Business Analyst Case Study**  
**Solution owner:** Vikas Rathore  
**Platform:** Microsoft Fabric, OneLake, Direct Lake and Power BI  
**Workspace:** `Eurowings Digital Case Study`  
**Lakehouse:** `EW_Flight_Operations_Lakehouse`  
**Pipeline:** `PL_EW_Flight_Operations_EndToEnd`  
**Semantic model:** `SM_EW_Flight_Operations`  
**Power BI report:** `RP_EW_Flight_Operations`  
**Documentation basis:** `Eurowings_Digital_Case_Study_Project_Handover_2026-07-15_2345_CEST(1).md`  
**Authoritative solution snapshot:** 15 July 2026, 23:45 CEST  
**Classification:** External technical documentation — presentation and submission

---

## Document purpose

This document presents the technical design, implementation, controls and analytical outputs of the Eurowings Digital Flight Operations case-study solution. It is written for external stakeholders who require a technically complete but readable description of the delivered data product.

The document covers:

- the business requirement and implemented scope;
- source data and interpretation assumptions;
- the Microsoft Fabric architecture and end-to-end data lineage;
- notebook responsibilities, transformations and validation logic;
- data-quality rules and analysis-specific eligibility;
- analytical and statistical calculations;
- the planned-TOW predictive prototype;
- the Trusted star schema and Direct Lake semantic model;
- governed DAX measures;
- the six visible Power BI report pages;
- validation evidence, limitations and operational boundaries.

The source handover contains older snapshots in appendices. This document applies the handover’s authority rule: the current authoritative section takes precedence over superseded implementation states.

---

# 1. Executive technical summary

The solution converts a flight-level Excel analysis task into a governed analytical data product in Microsoft Fabric and Power BI.

The original requirement focused on:

1. wind-speed distribution;
2. travelled-distance distribution;
3. cumulative flight coverage;
4. 50th-percentile and 85th-percentile reference points;
5. additional insights and Python or R exploration.

The delivered implementation retains these mandatory outputs as the core use case and extends them with:

- structured Raw, Standardized, Curated and Trusted data layers;
- independent ingestion and standardization of an airport reference source;
- controlled airline, aircraft and airport enrichment;
- record-level data-quality rules;
- analysis-specific eligibility instead of blanket row deletion;
- route, monthly, fleet, payload and anomaly-oriented analytical products;
- a chronological planned-TOW prediction prototype;
- a Trusted star schema with stable keys and validated foreign keys;
- a central Direct Lake semantic model;
- an action-oriented six-page Power BI report.

## 1.1 End-to-end platform chain

```text
Landing files
→ Raw ingestion
→ Standardization
→ Curated enrichment and data quality
→ Analytical aggregates
→ Predictive planned-TOW prototype
→ Trusted star schema
→ Fabric orchestration pipeline
→ Direct Lake semantic model
→ Governed DAX measure layer
→ Power BI decision-support report
```

## 1.2 Validated solution baseline

| Metric | Validated value |
|---|---:|
| Source flight records | 15,931 |
| Curated flight records | 15,931 |
| Trusted flight records | 15,931 |
| Distinct routes | 147 |
| Hard-failure rows excluded | 0 |
| Review flights retained | 349 |
| Overall review rate | 2.2% |
| DQ rule-trigger rows | 358 |
| Routes with review flights | 88 |
| Average selected distance | 728 NM |
| Wind P50 / P85 | -3 / 22 kts |
| Distance P50 / P85 | 593 / 1,105 NM |
| High TOW-utilisation flights | 44 |
| Payload-review flights | 128 |
| Model-training rows | 13,984 |
| Holdout rows per model | 1,819 |
| Model-scoring rows across both models | 3,638 |
| Selected model | Linear Regression |
| Selected-model MAE | 398.184 |
| Selected-model RMSE | 633.727 |
| Selected-model R² | 0.992408 |
| GBT MAE | 732.378 |
| GBT RMSE | 1,061.322 |
| GBT R² | 0.978707 |

## 1.3 Governing design principle

```text
Heavy, stable, governed and reusable calculations
    → Fabric notebooks and Delta tables

Dynamic calculations that depend on report filter context
    → DAX measures in the central semantic model

Presentation, explanatory narrative and visual interactions
    → Power BI report layer
```

Engineering transformations are not recreated in Power BI. The report remains live-connected to the central Direct Lake semantic model and does not use a report-local composite model for presentation-only requirements.

---

# 2. Business requirement and scope

## 2.1 Requirement-to-solution traceability

| Case-study requirement | Implemented solution |
|---|---|
| Distribution by wind speed | Flight counts grouped into governed 10-knot bands. |
| Distribution by travelled distance | Flight counts grouped into governed 100-NM bands. |
| Cumulative percentage line | Dynamic cumulative DAX measures evaluated in the active report context. |
| 50th percentile | Global P50 benchmark plus selected-scope P50 measures. |
| 85th percentile | Global P85 benchmark plus selected-scope P85 measures. |
| Clear interpretation | Record-level table and decision guidance beneath the distributions. |
| Additional insights | Route, network, fleet, payload, data-quality and predictive pages. |
| Python or R exploration | PySpark and Python notebooks for profiling, rules, aggregation and modelling. |

## 2.2 Analytical scope supported by the data

The source supports:

- descriptive operating profiles;
- wind and distance distribution analysis;
- route and aircraft-deployment analysis;
- supplied planned-weight analysis;
- data-quality review and source-validation prioritisation;
- analytical residual triage from a planned-TOW model.

The source does not support reliable conclusions about:

- delay causality;
- fuel burn;
- operational or safety risk;
- cargo weight;
- passenger load factor;
- profitability;
- market share;
- dispatch suitability;
- certified operational performance.

The report and technical model therefore distinguish supported evidence from unsupported claims.

---

# 3. Source data and interpretation controls

## 3.1 Primary flight source

**Source file:** `Data for Analysis.xlsx`  
**Landing path:**

```text
Files/landing/flight_operations/Data for Analysis.xlsx
```

**Confirmed source structure:**

```text
15,931 flight-level records
11 business columns
One source row = one technical flight record
```

## 3.2 Flight source dictionary

| Source field | Governed field | Technical interpretation |
|---|---|---|
| `Flight Date` | `flight_date` | Calendar date associated with the source flight record. |
| `Airline` | `airline_code` | Supplied operating-airline code. |
| `Pln Payload` | `planned_payload_weight` | Supplied planned payload value. |
| `Pln TOW` | `planned_tow_weight` | Supplied planned take-off weight. |
| `PAX` | `passenger_count` | Supplied passenger count. |
| `Acft MTOW` | `aircraft_mtow_weight` | Supplied aircraft maximum take-off weight. |
| `Acft Type` | `aircraft_type_code` | Supplied aircraft-type code. |
| `Destination` | `destination_iata_code` | Destination IATA code. |
| `Origin` | `origin_iata_code` | Origin IATA code. |
| `Airway Distance Travelled (Nautical miles)` | `airway_distance_nm` | Travelled airway distance in nautical miles. |
| `Wind in Kts` | `wind_component_kts` | Signed wind-component value supplied by the source. |

## 3.3 Weight-unit control

The source does not explicitly document the unit for payload, TOW and MTOW. The governed technical fields therefore use the neutral suffix `_weight`.

The solution does not relabel these values as kilograms or tonnes. Unit-specific interpretation requires confirmation from the source owner.

## 3.4 Wind-semantics control

The meaning of the sign in the supplied wind field is not confirmed. The solution therefore uses the neutral term **Wind Component** and does not relabel negative or positive values as headwind or tailwind.

Business-safe sign categories are:

```text
NEGATIVE_COMPONENT
ZERO_COMPONENT
POSITIVE_COMPONENT
```

## 3.5 External airport reference

**Source file:** `airports.csv`  
**Reference provider:** OurAirports  
**Landing path:**

```text
Files/landing/reference/ourairports/airports.csv
```

The full source snapshot contains 24 fields and approximately 85,720 rows after ingestion and standardization. The complete reference is preserved in Raw and Standardized. Curated processing selects only IATA codes used by the flight source and resolves duplicate candidates deterministically.

The retained source fields are:

```text
id, ident, type, name, latitude_deg, longitude_deg, elevation_ft,
continent, country_name, iso_country, region_name, iso_region,
local_region, municipality, scheduled_service, gps_code, icao_code,
iata_code, local_code, home_link, wikipedia_link, keywords, score,
last_updated
```

---

# 4. Microsoft Fabric architecture

## 4.1 Workspace items

```text
Eurowings Digital Case Study
|
+-- EW_Flight_Operations_Lakehouse
|   +-- SQL analytics endpoint
|
+-- NB01_Ingest_Raw
+-- NB01R_Ingest_Airport_Reference
+-- NB02_Standardize_Data
+-- NB02R_Standardize_Airport_Reference
+-- NB03_Curate_Quality_Features
+-- NB04_Analytical_Aggregates
+-- NB05_Predictive_Analytics
+-- NB06_Publish_Trusted_Model
+-- PL_EW_Flight_Operations_EndToEnd
+-- SM_EW_Flight_Operations
+-- RP_EW_Flight_Operations
```

## 4.2 Lakehouse schema layers

| Schema | Responsibility |
|---|---|
| `00control` | Operational control and optional logging artefacts. |
| `01raw` | Source-preserving ingestion with technical lineage only. |
| `02standardized` | Technical renaming, normalization, typing, parsing controls and reconciliation. |
| `03curated` | Business enrichment, data-quality rules, eligibility logic, analytical aggregates and model outputs. |
| `04trusted` | Semantic-model-ready dimensions and facts with stable keys and validated foreign keys. |

The numeric prefixes preserve logical processing order.

## 4.3 End-to-end lineage

```text
Flight source branch
====================

Data for Analysis.xlsx
    ↓
NB01_Ingest_Raw
    ↓
01raw.flight_raw
    ↓
NB02_Standardize_Data
    ↓
02standardized.flight_standardized
    ┐
    │
    ├──────────────────────────────────────────┐
    │                                          │
Airport reference branch                      │
========================                      │
                                             │
airports.csv                                 │
    ↓                                        │
NB01R_Ingest_Airport_Reference               │
    ↓                                        │
01raw.airport_reference_raw                  │
    ↓                                        │
NB02R_Standardize_Airport_Reference          │
    ↓                                        │
02standardized.airport_reference_standardized│
    │                                        │
    └────────────────────────────────────────┘
                     ↓
NB03_Curate_Quality_Features
    ↓
03curated references, flight enrichment and DQ outputs
    ↓
NB04_Analytical_Aggregates
    ↓
03curated distributions, percentiles and analytical products
    ↓
NB05_Predictive_Analytics
    ↓
03curated planned-TOW model outputs
    ↓
NB06_Publish_Trusted_Model
    ↓
04trusted dimensions and facts
    ↓
SM_EW_Flight_Operations — Direct Lake semantic model
    ↓
RP_EW_Flight_Operations — Power BI report
```

## 4.4 Why the airport source uses a separate branch

The airport file is an independent external source snapshot. It therefore receives its own:

```text
Raw preservation
→ technical standardization
→ Curated business reference
```

It is not loaded directly into Curated. This preserves source lineage and separates technical parsing from business resolution.

Airline and aircraft mappings are different. No separate authoritative source files were supplied for them, so controlled code-defined mappings are created in Curated.

---

# 5. Pipeline orchestration

## 5.1 Pipeline topology

```text
ACT_NB01_Ingest_Raw ───────────────→ ACT_NB02_Standardize_Data ───────┐
                                                                        │
                                                                        ↓
                                                        ACT_NB03_Curate_Quality_Features
                                                                        ↑
ACT_NB01R_Ingest_Airport_Reference → ACT_NB02R_Standardize_Airport_Reference
                                                                        │
                                                                        ↓
                                                        ACT_NB04_Analytical_Aggregates
                                                                        ↓
                                                        ACT_NB05_Predictive_Analytics
                                                                        ↓
                                                        ACT_NB06_Publish_Trusted_Model
                                                                        ↓
                                                         Failed or Skipped condition
                                                                        ↓
                                                        ACT_FAIL_PIPELINE
```

The two source branches start independently and can execute in parallel. NB03 starts only after both Standardized branches succeed.

## 5.2 Pipeline activities

```text
ACT_NB01_Ingest_Raw
ACT_NB01R_Ingest_Airport_Reference
ACT_NB02_Standardize_Data
ACT_NB02R_Standardize_Airport_Reference
ACT_NB03_Curate_Quality_Features
ACT_NB04_Analytical_Aggregates
ACT_NB05_Predictive_Analytics
ACT_NB06_Publish_Trusted_Model
ACT_FAIL_PIPELINE
```

## 5.3 Dependency rules

```text
NB01 succeeds                         → NB02
NB01R succeeds                        → NB02R
NB02 and NB02R both succeed           → NB03
NB03 succeeds                         → NB04
NB04 succeeds                         → NB05
NB05 succeeds                         → NB06
NB06 fails or is skipped              → ACT_FAIL_PIPELINE
```

## 5.4 Activity policy

```text
Timeout:         0.01:00:00  [one hour]
Retry:           0
Retry interval:  30 seconds [inactive while retry = 0]
Secure input:    false
Secure output:   false
```

Retries are disabled because deterministic schema, parsing and validation failures should remain visible rather than being silently retried.

## 5.5 Shared failure activity

```text
Error code: EW-PIPELINE-001
```

The failure message directs the operator to the pipeline history and failed notebook activity. An upstream failure that causes downstream activities to be skipped still reaches an explicit failed pipeline state through the final failure dependency.

## 5.6 Successful execution evidence

A full manual pipeline run completed successfully with all eight notebook activities succeeding.

```text
Pipeline run ID: 17a617c3-ec61-4a75-9e1f-8fdf17e558fb
Pipeline status: Succeeded
```

| Activity | Observed duration |
|---|---:|
| `ACT_NB01_Ingest_Raw` | 1m 19s |
| `ACT_NB01R_Ingest_Airport_Reference` | 1m 34s |
| `ACT_NB02_Standardize_Data` | 1m 21s |
| `ACT_NB02R_Standardize_Airport_Reference` | 1m 36s |
| `ACT_NB03_Curate_Quality_Features` | 8m 39s |
| `ACT_NB04_Analytical_Aggregates` | 4m 09s |
| `ACT_NB05_Predictive_Analytics` | 4m 22s |
| `ACT_NB06_Publish_Trusted_Model` | 5m 07s |

## 5.7 Orchestration boundary

At the authoritative snapshot:

- execution is manual/on-demand;
- no recurring schedule is configured;
- notebooks generate their own run IDs when no common orchestration ID is supplied;
- the control schema exists, while notebook steps primarily emit status through notebook output;
- a deliberate controlled failure-path test is not part of the recorded successful evidence run.

---

# 6. Engineering and implementation standards

## 6.1 Naming conventions

### Notebooks

```text
NB01_...   Main-source ingestion
NB01R_...  Reference-source ingestion
NB02_...   Main-source standardization
NB02R_...  Reference-source standardization
NB03_...   Business curation, enrichment and quality
NB04_...   Statistical and analytical aggregates
NB05_...   Predictive analytics
NB06_...   Trusted publication
```

### Tables

Lowercase snake case is used:

```text
flight_raw
flight_standardized
flight_enriched
airport_reference_raw
airport_reference_standardized
route_profile
```

### Technical metadata

Lineage and processing metadata use a leading underscore:

```text
_source_file_name
_source_sheet_name
_source_row_number
_ingestion_timestamp_utc
_pipeline_run_id
_source_record_hash
_standardization_timestamp_utc
_standardization_run_id
_standardized_record_hash
_curation_run_id
_curation_timestamp_utc
_analytics_run_id
_analytics_timestamp_utc
_trusted_run_id
_trusted_timestamp_utc
```

## 6.2 Notebook step wrapper

Each notebook uses a lightweight wrapper:

```python
run_step(step_name, action)
```

The wrapper:

1. prints `STARTED`;
2. executes the step;
3. prints `SUCCEEDED`;
4. prints notebook, step and exception details on failure;
5. re-raises a `RuntimeError` so the Fabric activity fails visibly.

## 6.3 Delta write pattern

```python
dataframe.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(target_table)
```

This is appropriate for the static case-study snapshot. A production implementation would replace broad overwrite patterns with governed incremental or snapshot strategies where required.

## 6.4 Qualified identifiers

Numeric schema names are protected with qualified references:

```python
f"`{schema}`.`{table}`"
```

## 6.5 File-path conventions

```text
Spark read path:      Files/...
Local Python/Pandas:  /lakehouse/default/Files/...
```

## 6.6 Safe numerical operations

A reusable `safe_ratio()` pattern prevents divide-by-zero and null errors. Ratios are created only where the denominator is valid.

## 6.7 Statistical methods

| Use case | Method |
|---|---|
| Exact global percentiles | `approxQuantile(..., relativeError=0.0)` |
| Grouped percentiles | `percentile_approx(..., accuracy=100000)` |
| Cumulative distributions | Spark window functions |
| Correlation | Pairwise-complete Pearson correlation |
| Route-distance outliers | Route-specific 1.5 × IQR |
| Outlier sample control | Route sample size must be at least 10 |

---

# 7. Notebook implementation

# 7.1 NB01 — Raw flight ingestion

## What

Ingests the supplied Excel flight file into the Raw Lakehouse layer.

## Why

Raw ingestion creates an immutable-style technical representation of the source before business renaming, typing or quality treatment. This supports traceability and prevents source logic from being mixed with downstream interpretation.

## Input and output

```text
Input:  Files/landing/flight_operations/Data for Analysis.xlsx
Output: 01raw.flight_raw
```

## How

- preserves all source business-column names exactly;
- preserves source values without business transformation;
- stores business values as strings;
- adds lineage metadata only;
- creates a deterministic source-record hash;
- does not filter records;
- does not rename business columns;
- does not apply data-quality rules.

## Added metadata

```text
_source_file_name
_source_sheet_name
_source_row_number
_ingestion_timestamp_utc
_pipeline_run_id
_source_record_hash
```

## Validation

```text
Expected rows:                15,931
Source column names:          preserved exactly
Source-row number:            unique
Source hash:                  populated
Distinct source hashes:       15,931
```

Nulls, zeros and invalid business values are preserved in Raw. Their interpretation occurs only in later layers.

---

# 7.2 NB01R — Raw airport-reference ingestion

## What

Preserves the complete OurAirports CSV snapshot in Raw.

## Why

The reference file is an independent external source and requires its own lineage. Raw preservation allows downstream parsing and candidate-resolution rules to be audited against the received file.

## Input and output

```text
Input:  Files/landing/reference/ourairports/airports.csv
Output: 01raw.airport_reference_raw
```

## How

- preserves all received columns and their order;
- validates required downstream columns;
- tolerates additive source columns;
- hashes all received source fields;
- reports exact duplicate rows without silently removing them.

The downloaded source contained 24 columns rather than the older expected 19-column structure. The final implementation distinguishes:

```text
REQUIRED_SOURCE_COLUMNS
CURRENT_SNAPSHOT_COLUMNS
```

Missing required columns cause failure. Additive columns are reported and retained.

## Validation

- file exists and is non-empty;
- no duplicate column names;
- required columns exist;
- row count and column order are preserved;
- source-row numbers are unique;
- hashes are populated.

---

# 7.3 NB02 — Flight standardization

## What

Converts the Raw flight table into a technically consistent typed table.

## Why

Standardization separates technical parsing from business curation. It establishes governed names, data types and parse outcomes while preserving every source row.

## Input and output

```text
Input:  01raw.flight_raw
Output: 02standardized.flight_standardized
```

## How

### Column renaming

```text
Flight Date                                  → flight_date
Airline                                      → airline_code
Pln Payload                                  → planned_payload_weight
Pln TOW                                      → planned_tow_weight
PAX                                          → passenger_count
Acft MTOW                                    → aircraft_mtow_weight
Acft Type                                    → aircraft_type_code
Destination                                  → destination_iata_code
Origin                                       → origin_iata_code
Airway Distance Travelled (Nautical miles)   → airway_distance_nm
Wind in Kts                                  → wind_component_kts
```

### Technical normalization

- text is trimmed;
- repeated whitespace is collapsed;
- technical null tokens are converted to null;
- controlled codes are uppercased;
- dates and whole-number fields are parsed;
- Raw lineage is retained;
- Standardized hashes and run metadata are added.

### Standardization controls

```text
standardization_error_count
standardization_error_reasons
standardization_status
_standardization_timestamp_utc
_standardization_run_id
_standardized_record_hash
```

Rows are not deleted for parse issues. Parsing failures are recorded so downstream quality rules can classify them.

## Validated result

```text
Rows:             15,931
Parse-error rows:      0
Validation:         PASS
```

---

# 7.4 NB02R — Airport-reference standardization

## What

Creates a normalized, typed airport master while retaining all received airport records.

## Why

Technical parsing must be completed before Curated duplicate resolution. Standardized retains all candidate airport rows so business selection remains transparent.

## Input and output

```text
Input:  01raw.airport_reference_raw
Output: 02standardized.airport_reference_standardized
```

## Key standardized fields

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

## Parsing-success fields

```text
airport_source_id_parse_success
latitude_parse_success
longitude_parse_success
elevation_parse_success
scheduled_service_parse_success
source_score_parse_success
source_last_updated_parse_success
```

## Behaviour

- all source records are retained;
- code fields are normalized;
- numeric and timestamp fields are parsed;
- duplicate IATA mappings are reported but not resolved;
- the source snapshot’s `0`/`1` scheduled-service representation is accepted together with common boolean equivalents.

## Validated result

```text
Rows:                         85,720
Parse-error rows:                  0
Distinct IATA codes:           9,056
Validation:                      PASS
```

---

# 7.5 NB03 — Curated quality and flight features

## What

Creates the business-enriched, quality-controlled, analysis-ready flight layer.

## Why

This notebook converts technically valid fields into reusable business context. It centralizes enrichment, row-level formulas, DQ rules and analytical eligibility so these rules are defined once and reused consistently.

## Inputs

```text
02standardized.flight_standardized
02standardized.airport_reference_standardized
```

## Outputs

```text
03curated.ref_airline
03curated.ref_aircraft_type
03curated.ref_airport
03curated.flight_enriched
03curated.dq_rule_catalog
03curated.dq_rule_results
03curated.dq_record_summary
03curated.dq_batch_summary
03curated.column_profile_summary
```

## Reference enrichment

### Airlines

| Code | Name | Entity | Status |
|---|---|---|---|
| `EW` | Eurowings | Eurowings GmbH | `CURRENT` |
| `E6` | Eurowings Europe | Eurowings Europe Limited | `CURRENT` |
| `4U` | Germanwings | Germanwings GmbH | `LEGACY_REVIEW` |

The airline comparison represents the supplied operating-carrier sample. It is not competitor or market-share analysis. The single `4U` row is treated as a weak sample.

### Aircraft

| Code | Model | Family | Generation |
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

The presence of two A320 MTOW variants is not classified as an error without aircraft-registration or fleet-master evidence.

### Airports

Curated airport logic:

1. filters the Standardized airport master to IATA codes used by the flight source;
2. ranks duplicate candidates by scheduled service, airport size, source score and source ID;
3. publishes one deterministic Curated row per IATA code;
4. records whether the selected match was unique or resolved from multiple candidates.

## Flight enrichment

### Identity and lineage

```text
flight_record_key
source, standardization and curation metadata
```

`flight_record_key` is a technical source-record key, not an operational flight-leg identifier.

### Route and market fields

```text
route_code
country_pair_code
market_scope
```

`market_scope` values:

```text
DOMESTIC
INTERNATIONAL
UNKNOWN
```

### Date features

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

The source covers June to December 2024. The report therefore describes **observed monthly patterns**, not full seasonality.

## Weight and payload formulas

| Field | Formula | Purpose |
|---|---|---|
| `tow_utilisation_pct` | `planned_tow_weight / aircraft_mtow_weight` | Shows the share of supplied MTOW used by supplied planned TOW. |
| `remaining_mtow_margin_weight` | `aircraft_mtow_weight - planned_tow_weight` | Shows remaining supplied MTOW margin. |
| `payload_to_tow_pct` | `planned_payload_weight / planned_tow_weight` | Compares supplied planned payload with supplied planned TOW. |
| `payload_to_mtow_pct` | `planned_payload_weight / aircraft_mtow_weight` | Compares supplied planned payload with supplied MTOW. |
| `non_payload_planned_mass` | `planned_tow_weight - planned_payload_weight` | Derives the planned non-payload component. |
| `planned_payload_per_pax_proxy` | `planned_payload_weight / passenger_count` | Provides a payload-per-passenger proxy for validation and pattern analysis. |

The last metric is a proxy. It is not average passenger weight and must not be presented as such.

Safe-ratio logic prevents division by null or zero denominators. Invalid rows are flagged or made ineligible rather than producing unsafe ratios.

## Wind features

```text
absolute_wind_kts
wind_sign_category
wind_magnitude_category
```

Magnitude categories:

```text
0–40 kts     NORMAL
41–60 kts    HIGH
61–80 kts    EXTREME
>80 kts      CRITICAL_REVIEW
```

## Distribution bands

### Wind

```text
10-knot bands
wind_band_key
wind_band_lower_kts
wind_band_upper_kts
wind_band_label
wind_band_sort
```

### Distance

```text
100-NM bands
distance_band_key
distance_band_lower_nm
distance_band_upper_nm
distance_band_label
distance_band_sort
```

## Data-quality framework

The notebook publishes:

- a governed rule catalogue;
- one record per triggered rule and flight;
- one summary record per flight;
- one batch-level summary per run;
- one profile record per field and run.

Review warnings are retained. Only hard failures are eligible for exclusion from Trusted.

## Validated NB03 baseline

```text
Curated flight records:                 15,931
Hard-failure records:                        0
Unique review records:                     349
Records without review flags:           15,582
Payload zero with positive PAX:             128
Same origin and destination:                   5
Absolute wind above 60 kts:                 175
Absolute wind above 80 kts:                   5
TOW utilisation above 95%:                   44
Legacy 4U records:                            1
```

---

# 7.6 NB04 — Analytical aggregates

## What

Creates reusable statistical and analytical products from the Curated flight layer.

## Why

Stable global benchmarks and reusable aggregate facts are calculated once upstream. Dynamic selected-scope values remain in DAX so they respond correctly to report filters.

## Inputs

```text
03curated.flight_enriched
03curated.dq_record_summary
03curated.ref_airline
03curated.ref_aircraft_type
03curated.ref_airport
```

## Outputs

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

## Global distributions

### Wind distribution

- grain: one row per 10-knot wind band;
- calculates count, share, cumulative count and cumulative percentage;
- stores min, max and mean values within each band;
- provides an independently validated global benchmark.

### Distance distribution

- grain: one row per 100-NM band;
- applies the same grouping and cumulative pattern as wind.

These static global facts validate the mandatory case-study output. They do not replace the dynamic DAX cumulative lines.

## Percentiles

Published percentile codes:

```text
P05, P10, P25, P50, P75, P85, P90, P95, P99
```

Mandatory global values:

```text
Wind P50:       -3 kts
Wind P85:       22 kts
Distance P50:   593 NM
Distance P85:   1,105 NM
```

P50 is the value at or below which 50% of the eligible population falls. P85 is the value at or below which 85% falls.

## Route profile

Grain:

```text
One row per route
```

The profile includes flight count, selected-sample share, operating days, date range, distance statistics, wind percentiles, median PAX, median payload, median TOW, median TOW utilisation, review counts and sample reliability.

Median distance is used as the route baseline because travelled airway distance can vary within the same route.

## Route-month profile

Grain:

```text
One row per route, year and month
```

It supports observed monthly volume, wind, passenger, payload, TOW-utilisation and review-rate analysis.

## Carrier profile

Grain:

```text
One row per airline code
```

It includes sample share, routes, airports, configurations, medians, generation mix and review rates. The correct term is **sample share**, not market share.

## Aircraft-route profile

Grain:

```text
One row per route and aircraft configuration
```

It supports fleet deployment, configuration mix, TOW-utilisation, remaining margin and review-rate analysis. It does not measure fuel efficiency.

## Route-distance outliers

For each route with at least 10 eligible flights:

```text
IQR = P75 - P25
Lower bound = P25 - 1.5 × IQR
Upper bound = P75 + 1.5 × IQR
```

Status values:

```text
EVALUATED
INSUFFICIENT_SAMPLE
ROUTE_NOT_ELIGIBLE
ROUTE_BASELINE_MISSING
```

Direction values:

```text
LOW
HIGH
NOT_OUTLIER
NOT_EVALUATED
```

## Transparent anomaly features

The rule-based anomaly score is not a trained model.

```text
Route-distance outlier           +1
Absolute wind > 60 kts           +1
Absolute wind > 80 kts           +1
TOW utilisation > 95%            +1
Payload = 0 with positive PAX     +1
Origin = destination              +1
```

Categories:

```text
0       NORMAL
1       OBSERVE
2       REVIEW
3 or more  HIGH_REVIEW
```

## Correlation matrix

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

The ordered matrix contains 49 rows. Correlation is descriptive and does not establish causality.

## Analysis-population summary

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

Each summary stores total, eligible and excluded records, eligibility percentage, governing field and exclusion explanation.

## Validated result

```text
Source flight rows:                15,931
Analytical output tables:              11
Route-distance outlier rows:         2,041
Validation:                            PASS
```

---

# 7.7 NB05 — Predictive analytics

## What

Builds a planned-take-off-weight prediction prototype and compares Linear Regression with Gradient-Boosted Trees.

## Why

The source contains a defensible target, `planned_tow_weight`. A planned-TOW model demonstrates data-science capability without inventing an unsupported fuel, delay or safety target.

## Input and target

```text
Input:        03curated.flight_enriched
Target:       planned_tow_weight
Eligibility:  is_model_training_eligible = true
```

## Features

Numeric:

```text
planned_payload_weight
airway_distance_nm
wind_component_kts
passenger_count
aircraft_mtow_weight
flight_month_number
```

Categorical:

```text
aircraft_type_code
route_code
airline_code
```

Categorical values are indexed with `handleInvalid="keep"`, one-hot encoded and assembled with numeric features. Invalid unseen categories are therefore retained in a fallback category rather than causing scoring failure.

## Chronological holdout

```text
Strategy: LAST_CALENDAR_MONTH_HOLDOUT
Training: 2 Jun 2024 to 30 Nov 2024
Test:     1 Dec 2024 to 31 Dec 2024
Training rows: 13,984
Test rows:      1,819
```

A chronological holdout is used instead of a random split to avoid leakage across time and to test the model on a later unseen period.

## Algorithms

### Linear Regression

```text
maxIter = 100
regParam = 0.0
elasticNetParam = 0.0
standardization = true
```

### Gradient-Boosted Tree Regression

```text
maxIter = 60
maxDepth = 5
stepSize = 0.05
subsamplingRate = 0.8
seed = 42
```

## Evaluation metrics

### MAE

```text
MAE = mean(|actual - predicted|)
```

It expresses the average absolute prediction difference.

### RMSE

```text
RMSE = sqrt(mean((actual - predicted)²))
```

It gives greater weight to large prediction differences.

### R²

```text
R² = 1 - residual variation / total target variation
```

It describes how much target variation is explained within the evaluated sample. It does not prove causality or operational suitability.

## Authoritative model comparison

| Model | Holdout flights | MAE | RMSE | R² |
|---|---:|---:|---:|---:|
| Linear Regression | 1,819 | 398.184 | 633.727 | 0.992408 |
| Gradient-Boosted Tree Regression | 1,819 | 732.378 | 1,061.322 | 0.978707 |

Linear Regression is selected because it has the lower chronological-holdout RMSE.

## Outputs

```text
03curated.tow_model_predictions
03curated.model_metrics
03curated.model_feature_importance
03curated.model_run_summary
```

### Model predictions

Grain:

```text
One row per test flight and model
```

Confirmed rows:

```text
1,819 flights × 2 models = 3,638 rows
```

Important fields:

```text
actual_planned_tow_weight
predicted_planned_tow_weight
prediction_error_weight
residual_weight
absolute_error_weight
squared_error_weight
absolute_percentage_error
prediction_direction
```

### Feature importance

GBT importance is published at:

```text
ENCODED_FEATURE
LOGICAL_FEATURE
```

Logical-feature importance is normalized to sum to 1.0. Importance reflects association within the fitted GBT model, not causality.

### Run summary

The interpretation scope is explicitly stored as:

```text
ANALYTICAL_PROTOTYPE_NOT_OPERATIONAL_CERTIFICATION
```

## Validation

- required columns exist;
- eligible source keys are unique;
- eligible target and features contain no nulls;
- target values are positive;
- train and test populations are non-empty;
- each model produces one prediction per holdout flight;
- `(flight_record_key, model_code)` is unique;
- predictions and metrics contain no null or NaN values;
- exactly one metric set is published per model;
- logical GBT feature importance sums to 1.0.

---

# 7.8 NB06 — Trusted publication

## What

Publishes the semantic-model-ready star schema in `04trusted`.

## Why

The Trusted layer isolates report consumers from broad Curated implementation tables. It provides stable dimensions, facts, keys, lineage and relationship-ready grains.

## Inclusion rule

```text
Include records where has_hard_failure = false
```

Review-warning records remain available. They are excluded only from analyses affected by the relevant rule.

## Reconciliation

```text
Curated flight rows:          15,931
Hard-failure rows excluded:        0
Trusted fact_flight rows:     15,931
```

## Stable-key strategy

- the technical flight key remains `flight_record_key`;
- surrogate keys use deterministic `xxhash64` expressions;
- null natural-key components are normalized to `<NULL>`;
- every Trusted table receives `_trusted_run_id` and `_trusted_timestamp_utc`.

## Date dimension

```text
Start: 1 Jun 2024
End:   31 Dec 2024
Rows:  214
```

The dimension is continuous and starts on the first day of the earliest observed month so month-level facts using first-of-month keys have valid relationships.

## Trusted dimensions

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

## Trusted facts

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

## Role-playing airports

Separate origin and destination dimensions are published so both relationships remain active and business use remains clear.

## Model selection

`dim_model` ranks models by chronological-test RMSE and marks exactly one selected model.

```text
Selected model: Linear Regression
Selection metric: CHRONOLOGICAL_TEST_RMSE
Selected RMSE: 633.7265365563417
```

## Validation

- Trusted fact reconciles to the no-hard-failure Curated population;
- dimension and fact primary keys are unique;
- prohibited null keys are absent;
- supporting-fact foreign keys are valid;
- no hard-failure row appears in `fact_flight`;
- exactly one model is selected;
- model-scoring rows reconcile to Curated predictions;
- global cumulative percentages end at 100%.

## Validated result

```text
Status:                    SUCCEEDED
Trusted fact-flight rows:  15,931
DQ-trigger rows:              358
Model-scoring rows:         3,638
Published dimensions:          11
Published facts:               11
Validation:                  PASS
```

---

# 8. Data-quality and analytical eligibility framework

## 8.1 Treatment principle

```text
Hard failure
→ excluded from Trusted

Review warning
→ retained in Trusted
→ excluded only from affected analyses where required
```

A review warning is not an automatic deletion and is not an operational failure.

## 8.2 Hard-failure rules

| Code | Hard-failure condition |
|---|---|
| `DQ001` | Standardization status is not `PASS`. |
| `DQ002` | Required business field is missing. |
| `DQ003` | Distance is less than or equal to zero. |
| `DQ004` | Passenger count is less than or equal to zero. |
| `DQ005` | Planned TOW is less than or equal to zero. |
| `DQ006` | MTOW is less than or equal to zero. |
| `DQ007` | Planned TOW is greater than MTOW. |
| `DQ008` | Planned payload is negative. |
| `DQ009` | Planned payload is greater than planned TOW. |
| `DQ010` | Airport-code format is invalid. |
| `DQ011` | Airport reference match is missing. |
| `DQ012` | Aircraft reference match is missing. |
| `DQ013` | Airline reference match is missing. |
| `DQ014` | Exact Standardized duplicate. |

The validated source population triggered none of these hard-failure conditions.

## 8.3 Review-warning rules

| Code | Review condition | Confirmed triggers |
|---|---|---:|
| `DQ101` | Planned payload equals zero while PAX is positive. | 128 |
| `DQ102` | Origin equals destination. | 5 |
| `DQ103` | Absolute wind exceeds 60 kts. | 175 |
| `DQ104` | Absolute wind exceeds 80 kts. | 5 |
| `DQ105` | TOW utilisation exceeds 95%. | 44 |
| `DQ106` | Legacy airline code. | 1 |

`DQ104` is a subset of `DQ103`; one flight can therefore trigger more than one rule. This explains why 358 rule triggers correspond to 349 distinct review flights.

## 8.4 Analysis-specific eligibility

```text
is_wind_distribution_eligible
is_distance_distribution_eligible
is_route_analysis_eligible
is_weight_analysis_eligible
is_payload_analysis_eligible
is_carrier_comparison_eligible
is_model_training_eligible
```

This avoids a single blanket valid/invalid flag. For example, a flight may remain eligible for wind analysis while being excluded from payload analysis.

## 8.5 Handling of null, zero, negative and inconsistent values

| Condition | Technical treatment |
|---|---|
| Source null token | Converted to technical null during Standardization and recorded by parse/status controls. |
| Parsing failure | Row retained in Standardized with error count, reason and status; downstream hard-failure rule applies. |
| Zero or negative distance | Hard failure. |
| Zero or negative passenger count | Hard failure. |
| Zero or negative planned TOW | Hard failure. |
| Zero or negative MTOW | Hard failure. |
| Planned TOW greater than MTOW | Hard failure. |
| Negative planned payload | Hard failure. |
| Planned payload greater than planned TOW | Hard failure. |
| Zero payload with positive PAX | Review warning; row retained but excluded from payload/model analysis where governed. |
| Origin equal to destination | Review warning; row retained for unaffected analyses. |
| Extreme wind | Review warning; row retained and identified for review. |
| High TOW utilisation | Review warning; row retained and presented for validation. |

---

# 9. Analytical calculations and statistical interpretation

## 9.1 Distribution grouping

### Wind

```text
Band width: 10 kts
```

### Distance

```text
Band width: 100 NM
```

Grouping makes the flight population interpretable without displaying thousands of individual values.

## 9.2 Cumulative count and cumulative percentage

For an ordered band `b`:

```text
Cumulative count(b)
= sum of flight counts from the first band through band b

Cumulative percentage(b)
= cumulative count(b) / total eligible selected population
```

The line shows how quickly the selected population is covered as the band values increase.

## 9.3 Percentiles

```text
P50 = value at or below which 50% of eligible flights fall
P85 = value at or below which 85% of eligible flights fall
```

Two forms are maintained:

- **Global percentiles:** stable benchmarks calculated upstream and intentionally unaffected by ordinary slicers.
- **Selected-scope percentiles:** DAX calculations that respond to report filters.

## 9.4 Standard deviation

Route distance variability uses population standard deviation:

```text
STDEV.P(distance)
```

It measures how spread out travelled distances are around the route’s average within the selected population. It is not a route-performance score.

## 9.5 Median

The median is used for several route and fleet profile measures because it is less sensitive to extreme values than the mean.

## 9.6 IQR outlier method

```text
IQR = P75 - P25
Lower bound = P25 - 1.5 × IQR
Upper bound = P75 + 1.5 × IQR
```

The method is evaluated only for routes with at least 10 eligible observations.

## 9.7 Correlation

Pearson correlation is used to describe linear association between selected metrics. Correlation does not prove that one metric causes another.

---

# 10. Trusted data model

## 10.1 Table-grain rule

A field must describe the grain of its table.

```text
Flight-level classifications       → fact_flight
Rule trigger per flight            → fact_dq_rule_trigger
Route-month measures               → fact_route_month
Aircraft-route measures            → fact_aircraft_route
Model score per flight and model   → fact_model_scoring
Model-level metrics                → fact_model_metric
Feature-level explainability       → fact_model_feature_importance
```

## 10.2 Fact grains

| Fact | Grain |
|---|---|
| `fact_flight` | One Trusted technical flight record. |
| `fact_dq_rule_trigger` | One triggered DQ rule per Trusted flight. |
| `fact_route_month` | One route and calendar month. |
| `fact_aircraft_route` | One route and aircraft configuration. |
| `fact_model_scoring` | One holdout flight and model. |
| `fact_model_metric` | One model and metric. |
| `fact_model_feature_importance` | One model and encoded/logical feature. |
| `fact_percentile_reference` | One metric and percentile. |
| `fact_analysis_population` | One governed analytical population. |
| `fact_wind_distribution_global` | One global wind band. |
| `fact_distance_distribution_global` | One global distance band. |

## 10.3 Trusted `fact_flight`

The main fact includes:

- dimension foreign keys;
- source flight measures;
- derived wind, route, payload and weight fields;
- DQ review indicators and reason fields;
- analysis-specific eligibility;
- carrier sample reliability;
- selected lineage metadata.

It adds:

```text
flight_count = 1
trusted_inclusion_status = NO_HARD_FAILURE
```

## 10.4 Lineage

The model preserves traceability across:

```text
_source_record_hash
_standardized_record_hash
_source_row_number
pipeline and notebook run IDs
processing timestamps
flight_record_key
```

Technical fields are hidden from ordinary report authors but remain available for audit and model logic.

---

# 11. Direct Lake semantic model

## 11.1 Identity and storage mode

```text
Semantic model: SM_EW_Flight_Operations
Source: EW_Flight_Operations_Lakehouse / 04trusted
Storage mode: Direct Lake on OneLake
Tables loaded: 22
```

Only Trusted tables are loaded. Raw, Standardized and broad Curated tables are excluded from the reporting model.

## 11.2 Relationship design

All relationships are:

```text
Active
Dimension 1 → Fact many
Single-direction filtering
No many-to-many relationships
No bidirectional relationships
```

### Main-flight relationships

```text
dim_date              → fact_flight
dim_airline           → fact_flight
dim_aircraft          → fact_flight
dim_origin_airport    → fact_flight
dim_destination_airport → fact_flight
dim_route             → fact_flight
dim_wind_band         → fact_flight
dim_distance_band     → fact_flight
```

### Supporting relationships

```text
dim_dq_rule                → fact_dq_rule_trigger
dim_date                   → fact_dq_rule_trigger
dim_date                   → fact_route_month
dim_route                  → fact_route_month
dim_aircraft               → fact_aircraft_route
dim_route                  → fact_aircraft_route
dim_model                  → fact_model_scoring
dim_date                   → fact_model_scoring
dim_airline                → fact_model_scoring
dim_aircraft               → fact_model_scoring
dim_origin_airport         → fact_model_scoring
dim_destination_airport    → fact_model_scoring
dim_route                  → fact_model_scoring
dim_model                  → fact_model_metric
dim_model                  → fact_model_feature_importance
dim_analysis_population    → fact_analysis_population
dim_wind_band              → fact_wind_distribution_global
dim_distance_band          → fact_distance_distribution_global
```

`fact_percentile_reference` is intentionally disconnected and queried through explicit DAX measures.

## 11.3 Role-playing dimensions

Origin and destination use separate airport dimensions. This allows both relationships to remain active and avoids inactive relationship switching in common report interactions.

## 11.4 Date metadata

```text
Date table: dim_date
Date column: calendar_date
Format: dd mmm yyyy
Summarize by: None
```

Sort metadata is configured for month, day, quarter, year-month and distribution-band labels.

## 11.5 Geography metadata

Origin and destination dimensions contain categorized:

```text
latitude
longitude
city
country/region
continent
state or province
place
```

## 11.6 Hierarchies

```text
Date Hierarchy
    Year → Quarter → Month → Date

Origin Geography
    Continent → Country → Region → City → Airport → IATA

Destination Geography
    Continent → Country → Region → City → Airport → IATA
```

## 11.7 Field hygiene

Hidden fields include:

- surrogate and foreign keys;
- technical run IDs;
- source and processing hashes;
- timestamps;
- raw metadata JSON;
- helper counts replaced by explicit measures;
- sort-helper columns after metadata configuration.

Business fields remain visible with controlled formatting and summarisation.

## 11.8 Dedicated measure table

```DAX
_Measures =
ROW (
    "Placeholder",
    BLANK ()
)
```

The placeholder field is hidden and the table is used only to hold explicit measures.

Current logical folders include:

```text
01 Core
02 Distributions
03 Percentiles
04 Route and Network
05 Aircraft and Weight
06 Payload and Passenger
07 Data Quality
08 Predictive Analytics
```

---

# 12. Governed DAX measure layer

## 12.1 Core measures

```DAX
Flight Count =
SUM ( fact_flight[flight_count] )
```

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

```DAX
Last Trusted Publication UTC =
CALCULATE (
    MAX ( fact_flight[_trusted_timestamp_utc] ),
    REMOVEFILTERS ()
)
```

## 12.2 Dynamic wind distribution

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

## 12.3 Dynamic distance distribution

```DAX
Distance Eligible Flights =
CALCULATE (
    [Flight Count],
    fact_flight[is_distance_distribution_eligible] = TRUE ()
)
```

```DAX
Distance Cumulative Flights =
VAR CurrentBand =
    MAX ( dim_distance_band[distance_band_sort] )
RETURN
    CALCULATE (
        [Distance Eligible Flights],
        FILTER (
            ALLSELECTED ( dim_distance_band ),
            dim_distance_band[distance_band_sort] <= CurrentBand
        )
    )
```

```DAX
Distance Selected Population =
CALCULATE (
    [Distance Eligible Flights],
    ALLSELECTED ( dim_distance_band )
)
```

```DAX
Distance Cumulative % =
DIVIDE (
    [Distance Cumulative Flights],
    [Distance Selected Population]
)
```

## 12.4 Fixed global percentile measures

The disconnected percentile fact keeps these benchmarks independent of ordinary slicers.

```DAX
Global Wind P50 =
CALCULATE (
    MAX ( fact_percentile_reference[percentile_value] ),
    fact_percentile_reference[metric_code] = "WIND",
    fact_percentile_reference[percentile_code] = "P50"
)
```

The same pattern is used for:

```text
Global Wind P85
Global Distance P50
Global Distance P85
```

## 12.5 Dynamic selected-scope percentiles

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

The same pattern is applied for selected wind P85, selected distance P50 and selected distance P85.

## 12.6 Route and network measures

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

## 12.7 Fleet and payload measures

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
            fact_flight[is_weight_analysis_eligible] = TRUE ()
        ),
        KEEPFILTERS (
            fact_flight[tow_utilisation_pct] > 0.95
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
Payload Review Flights =
COALESCE (
    CALCULATE (
        [Flight Count],
        KEEPFILTERS ( fact_flight[planned_payload_weight] = 0 ),
        KEEPFILTERS ( fact_flight[passenger_count] > 0 )
    ),
    0
)
```

```DAX
Aircraft Share Within Route % =
DIVIDE (
    [Flight Count],
    CALCULATE (
        [Flight Count],
        REMOVEFILTERS (
            dim_aircraft[aircraft_configuration_code]
        )
    ),
    0
)
```

## 12.8 Predictive measures

```DAX
Model Scored Flights =
COALESCE (
    DISTINCTCOUNT ( fact_model_scoring[flight_record_key] ),
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

Model-level metrics respond to Model selection. They are published holdout metrics and are not recalculated as route-specific or airline-specific performance metrics.

---

# 13. Power BI report product

## 13.1 Product principle

Each analytical page follows:

```text
Signal
→ diagnosis
→ underlying records or priority queue
→ defensible interpretation
→ next action or required validation
```

Aggregated visuals are paired with a record-level table or investigation queue.

## 13.2 Visible page inventory

```text
00 Executive Overview
01 Distributions
02 Network & Route Priorities
03 Fleet & Payload
04 Predictive Review
05 Data Quality & Trust
```

Technical validation and drill-through pages are not part of the external navigation.

## 13.3 Common report shell

```text
Canvas: 1280 × 720
White header
Eurowings Digital logo
Cyan and burgundy accent rule
Page-aware navigation
Right-side filter rail
Consistent footer and card language
```

Common filters:

```text
Date Range
Airline
Aircraft Model
Origin
Destination
Route
```

The report includes a page-level reset action. Shared slicers and interactions are designed to preserve a consistent user experience.

---

# 14. Page 00 — Executive Overview

## What the page shows

- total selected flights;
- distinct routes;
- data-quality review flights and review rate;
- selected wind and distance operating envelope;
- network review concentration;
- fleet and payload review indicators;
- selected predictive model and holdout metrics;
- Trusted Data Foundation reconciliation.

## Why it is included

The page allows a stakeholder to understand the scope, trusted population, main analytical signals and limitations in under one minute.

## How it is implemented

### KPI row

```text
Flight Count
Distinct Routes
Data Quality Review Flights
Data Quality Review Rate
```

### Operating Envelope

```text
Wind P50 / P85:       -3 / 22 kts
Distance P50 / P85:   593 / 1,105 NM
```

The selected-scope values remain dynamic under page filters.

### Network and Quality

```text
Average Distance:              728 NM
Routes with Review Records:     88
```

### Fleet and Payload

```text
Above 95% TOW Utilisation:      44
Payload Review Flights:         128
```

### Predictive Review

```text
Selected Model:                 Linear Regression
Holdout RMSE | R²:              633.7 | 0.992
```

### Trusted Data Foundation

```text
15,931 reconciled flight records
0 hard failures
349 review flights retained
```

## Interpretation boundary

The page makes clear that review indicators require validation and are not safety, performance or certification conclusions.

---

# 15. Page 01 — Distributions

## What the page shows

- 10-knot wind distribution;
- 100-NM distance distribution;
- dynamic cumulative-percentage lines;
- global P50 and P85 benchmark references;
- selected-scope P50 and P85 values;
- exact flight records behind the selected band.

## Why it is included

This page directly answers the mandatory case-study requirement and converts the distributions into an inspectable operating-envelope view.

## How it is implemented

### Wind visual

```text
Columns: eligible flights per 10-knot band
Line: dynamic cumulative percentage
Global P50: -3 kts
Global P85: 22 kts
```

### Distance visual

```text
Columns: eligible flights per 100-NM band
Line: dynamic cumulative percentage
Global P50: 593 NM
Global P85: 1,105 NM
```

### Record list

Selecting a wind or distance band filters a flight-level table containing business context such as date, airline, route, aircraft configuration, wind, distance, planned TOW, planned payload and review status.

### Interaction rule

A selection from either distribution filters the record list without unintentionally distorting the other distribution.

## Analysis

- P50 describes the central selected population.
- P85 defines an upper coverage boundary.
- Flights beyond P85 form a transparent tail population for further route, aircraft and weather investigation.
- Wind or distance alone does not establish delay, fuel burn or operational risk.

---

# 16. Page 02 — Network & Route Priorities

## What the page shows

- flight volume by route;
- routes with the highest distance variability;
- observed monthly flight profile;
- a route investigation queue;
- review concentration by route.

## Why it is included

The page converts network-level signals into a prioritised list of routes requiring deeper validation or operational context.

## How it is implemented

### KPI row

```text
Flight Count:                        15,931
Distinct Routes:                     147
Average Distance:                    728 NM
Routes with Data Quality Review:      88
```

### Top Routes by Flight Volume

Top 10 routes are ranked by selected flight count. Example unfiltered checks from the authoritative handover include:

```text
PMI-DUS   1,332
BER-DUS     606
VIE-DUS     459
MUC-DUS     447
```

These values represent sample concentration, not market share.

### Routes with Highest Distance Variability

A ranked bar chart uses population standard deviation for routes with at least 10 selected flights. Distance variability is dispersion, not route performance.

### Observed Monthly Flight Profile

A line chart displays selected flight volume by `flight_year_month`. The wording is **observed monthly profile**, not seasonality.

### Route Investigation Queue

The queue combines:

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

It is sorted to support investigation rather than visual ranking alone.

## Analysis

Routes combining high volume, high distance variability or review concentration receive higher investigation priority. Low-sample routes require validation before comparison.

---

# 17. Page 03 — Fleet & Payload

## What the page shows

- TOW utilisation by aircraft configuration;
- planned payload versus passenger count;
- aircraft deployment by route;
- high-TOW-utilisation records;
- payload-review records.

## Why it is included

The page identifies supplied weight and payload patterns requiring validation without converting them into safety, fuel or commercial conclusions.

## How it is implemented

### KPI logic

The page uses measures including:

```text
Average TOW Utilisation
Average Remaining MTOW Margin
Average Planned Payload
High TOW Utilisation Flights
Payload Review Flights
```

Validated review populations:

```text
High TOW Utilisation Flights: 44
Payload Review Flights:       128
```

### TOW utilisation by aircraft configuration

The visual compares average and median supplied planned-TOW utilisation and exposes remaining margin and review counts through tooltips.

### Planned payload versus passenger count

A flight-level scatter examines the selected payload/PAX pattern. Zero-payload records with positive passenger counts are explicitly treated as source-semantic review records.

### Aircraft deployment by route

A route/configuration matrix or equivalent profile displays observed configuration mix. Deployment share is not interpreted as fleet productivity or optimisation.

### Record-level review

The page provides exact records for:

```text
TOW utilisation > 95%
Planned payload = 0 with positive PAX
```

## Analysis

- Higher utilisation means less remaining supplied MTOW margin.
- It does not establish unsafe operation.
- Payload-per-PAX remains a proxy.
- Aircraft MTOW variants require fleet-master confirmation before classification.

---

# 18. Page 04 — Predictive Review

## What the page shows

- selected model holdout metrics;
- supplied versus predicted planned TOW;
- prediction direction;
- GBT logical feature importance;
- flights ranked by absolute residual.

## Why it is included

The page demonstrates a transparent analytical prototype and provides a record-level triage queue. It does not turn the model into an operational decision system.

## How it is implemented

### KPI row

```text
Model Scored Flights
Model MAE
Model RMSE
Model R²
```

Validated model values:

| Model | Scored flights | MAE | RMSE | R² |
|---|---:|---:|---:|---:|
| Linear Regression | 1,819 | 398.2 | 633.7 | 0.992 |
| Gradient-Boosted Tree Regression | 1,819 | 732.4 | 1,061.3 | 0.979 |

### Supplied versus predicted planned TOW

A scatter chart compares:

```text
X-axis: supplied planned TOW
Y-axis: predicted planned TOW
Legend: OVER_PREDICTED / UNDER_PREDICTED
```

Equal axis ranges prevent visual exaggeration of residuals.

### GBT logical feature importance

A fixed GBT explainability visual displays normalized logical-feature association. The Model slicer does not blank this visual when Linear Regression is selected.

### Residual investigation queue

Flights are sorted by absolute residual descending. The queue includes date, airline, route, aircraft configuration, supplied TOW, predicted TOW, residual and direction.

## Analysis

Absolute residuals are used to prioritise validation. Large residuals should first be checked against data quality, aircraft context and route context.

## Limitations

- no governed anomaly threshold is defined;
- model metrics are global holdout metrics, not dynamically recalculated route metrics;
- the source weight unit is unconfirmed;
- GBT feature importance does not explain Linear Regression coefficients;
- no dispatch, safety or certification decision is supported.

---

# 19. Page 05 — Data Quality & Trust

## What the page shows

- total selected flight population;
- hard failures;
- review flights and review rate;
- triggered DQ rules;
- analysis exclusions by use case;
- record-level evidence behind the selected quality finding.

## Why it is included

The page makes the trust model visible. It explains why a record is reviewed, which analysis is affected and why a warning is retained rather than automatically deleted.

## How it is implemented

### KPI logic

```text
Flight Count
Hard Failure Count
Review Flights
Review Rate
```

Validated baseline:

```text
Trusted flights:       15,931
Unique review flights:    349
DQ rule triggers:         358
Hard failures:              0
```

### Rule-trigger analysis

The page ranks rule triggers by rule, severity and quality dimension.

### Analysis exclusions

Eligibility is shown separately for:

```text
Wind Distribution
Distance Distribution
Route Analysis
Weight Analysis
Payload Analysis
Carrier Comparison
Model Training
```

### Record-level list

The list links the selected rule or use-case exclusion to flight date, airline, route, aircraft, rule, severity, reason, affected analysis and eligibility status.

## Analysis

- review flags are not automatic deletions;
- records remain available for unaffected analyses;
- hard failures are the Trusted exclusion criterion;
- recurring patterns should be escalated at source, route or aircraft-context level after record review.

---

# 20. Report interactions and user experience

## 20.1 Common interaction contract

```text
Hover
→ compact contextual tooltip

Click a chart element
→ filter the page’s investigation table or priority queue

Record selection
→ preserve exact row-level context

Reset
→ restore the page’s governed default filter state
```

## 20.2 Page-specific interaction examples

```text
Wind band
→ filters selected-band flight records

Route bar
→ filters monthly profile and route queue

Aircraft configuration
→ filters payload/deployment context and active record queue

Prediction direction
→ filters scatter and residual table

DQ rule
→ filters records that triggered the rule
```

## 20.3 Local-model prohibition

The report does not use:

```text
Add a local model
Make changes to this model
Enter Data for technical logic
report-local calculated tables
report-local calculated columns
composite-model conversion for presentation-only requirements
```

Static narrative belongs in text boxes. Stable classifications belong upstream. Dynamic selected-context logic belongs in DAX.

---

# 21. Validation and reconciliation

## 21.1 Row reconciliation

```text
Raw flight rows          15,931
Standardized rows        15,931
Curated rows             15,931
Trusted fact rows        15,931
Hard failures excluded        0
```

## 21.2 Quality reconciliation

```text
Unique review flights    349
DQ trigger rows           358
```

The difference is expected because a flight can trigger multiple review rules.

## 21.3 Distribution validation

```text
Wind eligible flights         15,931
Distance eligible flights     15,931
Global cumulative percentage  ends at 100%
Wind P50 / P85                 -3 / 22 kts
Distance P50 / P85             593 / 1,105 NM
```

## 21.4 Fleet and payload validation

```text
TOW high-utilisation flights  44
TOW standard flights          15,887
Payload review flights        128
Payload-analysis eligible     15,803
```

## 21.5 Predictive validation

```text
Training rows                 13,984
Holdout rows per model         1,819
Scoring rows across models     3,638
Selected model                 Linear Regression
```

## 21.6 Semantic-model validation

- 22 Trusted tables are loaded;
- dimensions filter facts through active one-to-many relationships;
- no many-to-many or bidirectional relationships are used;
- origin and destination relationships are both active through role-playing dimensions;
- technical fields are hidden without removing lineage;
- date and band sort metadata are configured;
- global percentile fact remains intentionally disconnected.

---

# 22. Interpretation, governance and limitations

## 22.1 Mandatory interpretation controls

```text
Weight units are unconfirmed.
Signed wind semantics are unconfirmed.
Sample share is not market share.
High TOW utilisation is a review condition, not evidence of unsafe operation.
The predictive model is an analytical triage prototype.
The source covers June–December 2024 only.
Feature importance is association, not causality.
The model target is supplied planned TOW, not actual TOW.
```

## 22.2 Claims deliberately rejected

### Fuel analysis

The source does not contain actual fuel quantities, aircraft operating-empty weight, route fuel plan, reserves or actual burn. A fuel-efficiency conclusion is therefore unsupported.

### Cargo analysis

Planned payload cannot be decomposed into cargo, baggage, mail and passenger-related components with the supplied data.

### Passenger load factor

Seat capacity is not supplied. Passenger count alone cannot produce load factor.

### Profitability and market share

Revenue, cost, capacity and market-total data are not available. Sample share must not be relabelled as market share.

### Safety or dispatch suitability

The case-study source and analytical prototype do not support safety, certification or dispatch decisions.

---

# 23. Additional data recommended

The source handover identifies data that would materially improve future analysis.

## 23.1 Operations

- scheduled and actual departure/arrival times;
- delay codes and disruption reasons;
- flight-plan and actual trajectory data;
- airport and airspace constraints;
- weather observations and forecasts;
- aircraft registration and configuration master;
- operational status and cancellation/diversion indicators.

## 23.2 Fuel and weight

- operating empty weight;
- zero-fuel weight;
- maximum zero-fuel weight;
- maximum landing weight;
- planned block, taxi, trip, contingency and reserve fuel;
- actual fuel uplift and burn;
- actual take-off and landing weight.

## 23.3 Capacity and payload

- seat capacity by aircraft configuration;
- passenger, baggage, cargo and mail decomposition;
- booked and boarded passenger counts;
- cabin configuration;
- aircraft registration.

## 23.4 Commercial context

- fare and revenue data;
- route cost data;
- capacity offered;
- competitor and market-total schedules;
- customer and booking-channel context.

These additions would enable stronger operational, commercial and causal analysis while preserving the same layered architecture.

---

# 24. Delivery status at the authoritative snapshot

| Component | Status |
|---|---|
| Fabric Lakehouse architecture | Implemented and validated |
| Eight notebooks | Implemented |
| End-to-end pipeline | Successful full execution recorded |
| Trusted star schema | Published and validated |
| Direct Lake semantic model | Implemented |
| Distribution page | Completed and frozen |
| Network page | Analytical implementation complete; final presentation QA noted in source |
| Fleet & Payload page | Main analytical implementation complete; final interaction/presentation QA noted in source |
| Predictive Review page | Analytical implementation complete; final micro-QA noted in source |
| Data Quality page | Implemented; final presentation cleanup noted in source |
| Executive Overview | Dynamic content implemented; micro-QA noted in source |
| Hidden flight-detail page | Placeholder/shell at snapshot |
| Technical validation page | Exists |

The source handover freezes the engineering and Trusted architecture as version 1.0. Reopening is limited to demonstrated defects, required governed fields, approved new sources or documented production-hardening changes.

---

# 25. External presentation summary

The solution can be described in five technical statements:

1. **The mandatory requirement remains central.** Wind and distance distributions use governed bands, cumulative coverage and global/selected percentiles.
2. **The data is governed before visualization.** Raw preservation, technical standardization, business curation and Trusted publication are separated.
3. **Data quality is analysis-specific.** Hard failures are excluded; review warnings are retained and excluded only where the affected analysis requires it.
4. **The semantic model is reusable.** Stable logic sits upstream, dynamic filter logic sits in DAX, and the report stays live-connected through Direct Lake.
5. **The predictive work is deliberately bounded.** It predicts supplied planned TOW on a chronological holdout and supports residual triage only.

---

# Appendix A — Trusted table inventory

## Dimensions

| Table | Purpose |
|---|---|
| `dim_date` | Continuous date context and date hierarchy. |
| `dim_airline` | Airline code, name, entity and status. |
| `dim_aircraft` | Aircraft type, model, family, generation and configuration. |
| `dim_origin_airport` | Origin role-playing airport dimension. |
| `dim_destination_airport` | Destination role-playing airport dimension. |
| `dim_route` | Origin-destination route context. |
| `dim_wind_band` | Governed 10-knot band labels and sort order. |
| `dim_distance_band` | Governed 100-NM band labels and sort order. |
| `dim_dq_rule` | DQ rule metadata, severity and affected analysis. |
| `dim_analysis_population` | Governed analytical-population definitions. |
| `dim_model` | Model identity, parameters, selection ranking and selected-model flag. |

## Facts

| Table | Grain |
|---|---|
| `fact_flight` | One Trusted technical flight record. |
| `fact_dq_rule_trigger` | One triggered DQ rule per Trusted flight. |
| `fact_route_month` | One route and month. |
| `fact_aircraft_route` | One route and aircraft configuration. |
| `fact_model_scoring` | One holdout flight and model. |
| `fact_model_metric` | One model and metric. |
| `fact_model_feature_importance` | One model and feature representation. |
| `fact_percentile_reference` | One metric and percentile. |
| `fact_analysis_population` | One governed analysis population. |
| `fact_wind_distribution_global` | One global 10-knot wind band. |
| `fact_distance_distribution_global` | One global 100-NM distance band. |

---

# Appendix B — Curated analytical table inventory

```text
ref_airline
ref_aircraft_type
ref_airport
flight_enriched
dq_rule_catalog
dq_rule_results
dq_record_summary
dq_batch_summary
column_profile_summary
wind_distribution_global
distance_distribution_global
percentile_reference
route_profile
route_month_profile
carrier_profile
aircraft_route_profile
flight_route_outlier
flight_anomaly_features
metric_correlation
analysis_population_summary
tow_model_predictions
model_metrics
model_feature_importance
model_run_summary
```

---

# Appendix C — Key formula reference

```text
TOW utilisation
= planned TOW / aircraft MTOW

Remaining MTOW margin
= aircraft MTOW - planned TOW

Payload-to-TOW ratio
= planned payload / planned TOW

Payload-to-MTOW ratio
= planned payload / aircraft MTOW

Non-payload planned mass
= planned TOW - planned payload

Payload-per-PAX proxy
= planned payload / passenger count

Cumulative percentage
= cumulative eligible flights / selected eligible population

IQR
= P75 - P25

Route outlier lower bound
= P25 - 1.5 × IQR

Route outlier upper bound
= P75 + 1.5 × IQR

MAE
= mean absolute prediction error

RMSE
= square root of mean squared prediction error

R²
= proportion of target variance explained within the evaluation sample
```

---

# Appendix D — Technical glossary

| Term | Definition in this solution |
|---|---|
| Raw | Source-preserving layer with lineage only. |
| Standardized | Technically typed and normalized layer with parse controls. |
| Curated | Business-enriched layer with rules, features and analytical products. |
| Trusted | Star-schema-ready reporting layer with stable keys and validated relationships. |
| Direct Lake | Power BI storage mode reading governed OneLake data without importing a duplicate report-local model. |
| P50 | Median; 50% of the eligible population is at or below the value. |
| P85 | Value at or below which 85% of the eligible population falls. |
| Review warning | A retained record requiring validation or use-case-specific exclusion. |
| Hard failure | A rule condition that excludes a record from Trusted. |
| Eligibility | A governed flag defining whether a row can participate in a specific analysis. |
| Residual | Difference between supplied and predicted planned TOW. |
| Feature importance | GBT model association measure; not proof of causality. |
| Sample share | Share within the supplied selected dataset; not market share. |

---

# Appendix E — Authoritative validation snapshot

```text
Source / Curated / Trusted flight rows:  15,931
Distinct routes:                            147
Hard failures:                                0
Review flights:                             349
Review rate:                               2.2%
DQ triggers:                                358
Routes with reviews:                         88
Average distance:                        728 NM
Wind P50 / P85:                     -3 / 22 kts
Distance P50 / P85:             593 / 1,105 NM
High TOW utilisation:                       44
Payload reviews:                           128
Training rows:                          13,984
Holdout rows per model:                  1,819
Scoring rows:                            3,638
Selected model:              Linear Regression
Selected MAE:                         398.184
Selected RMSE:                        633.727
Selected R²:                         0.992408
```
