# Flight Operations Analytics — Technical Documentation

**Project:** Flight Operations Analytics — Data Modelling and Dashboard  
**Platform:** Microsoft Fabric / Power BI  
**Owner:** Vikas Rathore  
**Status:** v1.0 — Engineering layer frozen  
**Date:** July 2026

---

## 1. Executive Summary

This project converts a flight operations dataset into an end-to-end analytical data product in Microsoft Fabric and Power BI. The design intentionally goes beyond a direct Excel-to-Power-BI import to demonstrate business translation, data-product design, data engineering collaboration, semantic modelling, Power BI development, advanced analytics, and machine learning familiarity.

The implemented solution contains:

- A Fabric workspace and Lakehouse.
- Layered schemas: `00control`, `01raw`, `02standardized`, `03curated`, `04trusted`.
- Independent Raw and Standardized branches for flight data and the external airport reference source.
- Source-preserving Raw Delta tables with lineage and hashes.
- Typed Standardized tables with parse-success flags, reconciliation, and schema controls.
- Curated airline, aircraft, and airport references.
- A flight-level enriched table with date, route, market, weight, wind, and distribution features.
- A configurable data-quality rule catalogue, rule-trigger fact, record summary, batch summary, and column profiling.
- Analysis-specific eligibility flags rather than one blanket valid/invalid flag.
- Global wind and distance distributions with cumulative percentages.
- Fixed global P50/P85 benchmarks and a broader percentile reference table.
- Route, route-month, carrier, and aircraft-route analytical profiles.
- Route-specific distance outliers, transparent anomaly features, and a metric-correlation matrix.
- A PySpark planned-TOW prediction prototype comparing Linear Regression and Gradient-Boosted Trees.
- A Trusted star-schema-ready layer containing 11 dimensions and 11 facts.
- A Fabric Data Factory pipeline that orchestrates all eight notebooks with parallel source branches, success dependencies, fail-fast behaviour, and a shared pipeline failure activity.
- A confirmed successful end-to-end pipeline execution with all eight notebook activities succeeding.

### Calculation Placement Principle

```text
Heavy, stable, governed, and reusable logic
    → Fabric notebooks and Delta tables

Dynamic calculations dependent on report filter context
    → Power BI semantic-model measures

Formatting, narrative, and visual interaction
    → Power BI report layer
```

---

## 2. Source Data

### 2.1 Flight Source

**File:** `Data for Analysis.xlsx`  
**Landing path:** `Files/landing/flight_operations/Data for Analysis.xlsx`  
**Size:** 15,931 rows × 11 business columns

| Source Column | Standardized Column | Meaning |
|---|---|---|
| `Flight Date` | `flight_date` | Calendar date of the flight record |
| `Airline` | `airline_code` | Operating-airline code |
| `Pln Payload` | `planned_payload_weight` | Planned payload amount |
| `Pln TOW` | `planned_tow_weight` | Planned take-off weight |
| `PAX` | `passenger_count` | Passenger count |
| `Acft MTOW` | `aircraft_mtow_weight` | Certified maximum take-off weight |
| `Acft Type` | `aircraft_type_code` | Aircraft type code |
| `Destination` | `destination_iata_code` | Destination IATA code |
| `Origin` | `origin_iata_code` | Origin IATA code |
| `Airway Distance Travelled (Nautical miles)` | `airway_distance_nm` | Travelled airway distance in nautical miles |
| `Wind in Kts` | `wind_component_kts` | Signed wind-component measure supplied by the source |

### 2.2 Unit and Wind Caveats

The dataset does not explicitly document the unit for payload, TOW, and MTOW. Technical field names use the suffix `_weight`, not `_kg`. These fields must not be renamed to kilograms unless the source owner confirms the unit.

The signed wind field must not be labelled "headwind" or "tailwind" until the source definition is confirmed. Business-safe categories:

```text
NEGATIVE_COMPONENT
ZERO_COMPONENT
POSITIVE_COMPONENT
```

### 2.3 Airport Reference Source

**File:** `airports.csv`  
**Source:** OurAirports  
**Landing path:** `Files/landing/reference/ourairports/airports.csv`  
**Size:** ~85,720 rows × 24 source columns

The complete source is retained in Raw and Standardized; only airport codes used by the flight dataset are selected in Curated.

---

## 3. Architecture

### 3.1 Workspace Structure

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
```

### 3.2 Lakehouse Schemas

```text
dbo          Default Fabric-managed schema
00control    Operational control and logging artefacts
01raw        Source-preserving ingestion with technical lineage only
02standardized   Technical renaming, normalization, typing, parse controls
03curated    Business enrichment, DQ logic, analysis eligibility, analytical aggregates, ML outputs
04trusted    Semantic-model-ready dimensions and facts with stable keys and validated foreign keys
```

### 3.3 End-to-End Lineage

```text
Flight source branch
====================

Files/landing/flight_operations/Data for Analysis.xlsx
    │
    v
NB01_Ingest_Raw
    │
    v
01raw.flight_raw
    │
    v
NB02_Standardize_Data
    │
    v
02standardized.flight_standardized
    │
    +---------------------------------------------+
                                                  │
Airport-reference branch                         │
========================                          │
                                                  │
Files/landing/reference/ourairports/airports.csv │
    │                                             │
    v                                             │
NB01R_Ingest_Airport_Reference                    │
    │                                             │
    v                                             │
01raw.airport_reference_raw                       │
    │                                             │
    v                                             │
NB02R_Standardize_Airport_Reference               │
    │                                             │
    v                                             │
02standardized.airport_reference_standardized     │
                                                  │
    +---------------------------------------------+
    │
    v
NB03_Curate_Quality_Features
    │
    +-- 03curated reference tables
    +-- 03curated.flight_enriched
    +-- 03curated DQ outputs
    │
    v
NB04_Analytical_Aggregates
    │
    +-- global distributions and percentile references
    +-- route/carrier/aircraft/month profiles
    +-- outlier, anomaly, correlation and population outputs
    │
    v
NB05_Predictive_Analytics
    │
    +-- planned-TOW model predictions
    +-- model metrics, feature importance and run summary
    │
    v
NB06_Publish_Trusted_Model
    │
    v
04trusted dimensions and facts
    │
    v
Power BI semantic model
    │
    v
Power BI report
```

### 3.4 Orchestration Architecture

The Fabric pipeline operationalizes the notebook lineage:

```text
ACT_NB01_Ingest_Raw ----------------> ACT_NB02_Standardize_Data -----------+
                                                                            │
                                                                            v
                                                                  ACT_NB03_Curate_Quality_Features
                                                                            │
ACT_NB01R_Ingest_Airport_Reference -> ACT_NB02R_Standardize_Airport_Reference
                                                                            │
                                                                            +--> ACT_NB04_Analytical_Aggregates
                                                                                       │
                                                                                       v
                                                                            ACT_NB05_Predictive_Analytics
                                                                                       │
                                                                                       v
                                                                            ACT_NB06_Publish_Trusted_Model
                                                                                       │
                                                                     Failed or Skipped
                                                                                       v
                                                                            ACT_FAIL_PIPELINE
```

The two source branches start independently and run in parallel. NB03 has two `Succeeded` dependencies and starts only after both Standardized branches complete. Subsequent notebooks execute sequentially through NB06.

**Activity policy:**

```text
Timeout:     0.01:00:00 (1 hour)
Retry:       0
```

Retries remain disabled because deterministic schema, parsing, and DQ failures should not be silently retried.

**Failure handling:**

```text
Activity:   ACT_FAIL_PIPELINE
Error code: EW-PIPELINE-001
Message:    The pipeline failed. One or more notebook activities failed or were skipped.
            Review the pipeline run history and the failed notebook activity.
```

### 3.5 Pipeline Execution Results

A full manual run succeeded with all eight notebook activities green.

| Activity | Duration |
|---|---:|
| `ACT_NB01_Ingest_Raw` | 1m 19s |
| `ACT_NB01R_Ingest_Airport_Reference` | 1m 34s |
| `ACT_NB02_Standardize_Data` | 1m 21s |
| `ACT_NB02R_Standardize_Airport_Reference` | 1m 36s |
| `ACT_NB03_Curate_Quality_Features` | 8m 39s |
| `ACT_NB04_Analytical_Aggregates` | 4m 09s |
| `ACT_NB05_Predictive_Analytics` | 4m 22s |
| `ACT_NB06_Publish_Trusted_Model` | 5m 07s |

Pipeline run ID: `17a617c3-ec61-4a75-9e1f-8fdf17e558fb`  
Pipeline status: **Succeeded**

---

## 4. Notebook Specifications

### 4.1 NB01 — Ingest Raw

**Purpose:** Ingest the supplied Excel workbook into Raw without business transformations.

**Target:** `01raw.flight_raw`

**Principles:**
- Preserve source values and source business-column names.
- Store all source values as strings except technical metadata.
- Add only lineage metadata: `_source_file_name`, `_source_sheet_name`, `_source_row_number`, `_ingestion_timestamp_utc`, `_pipeline_run_id`, `_source_record_hash`.
- No business renaming, type conversion, or quality filtering in Raw.

**Validation:**

```text
Expected rows:           15,931
Source column names:     Preserved exactly
Distinct source hashes:  15,931
```

---

### 4.2 NB01R — Ingest Airport Reference

**Purpose:** Preserve the complete OurAirports CSV source in Raw.

**Target:** `01raw.airport_reference_raw`

**Schema-drift handling:**

The source snapshot contained 24 columns. The notebook preserves all received source columns in original order, fails only if a required downstream column disappears, and reports additive columns without failing.

**Validation:** File exists, file not empty, no duplicate column names, required columns present, row count preserved, source-column order preserved, source-row number unique, source-record hash populated.

---

### 4.3 NB02 — Standardize Data

**Purpose:** Convert the flight Raw table into a technically consistent Standardized table.

**Target:** `02standardized.flight_standardized`

**Transformations:**
- Column renaming to lowercase snake_case (e.g., `Flight Date` → `flight_date`, `Pln TOW` → `planned_tow_weight`).
- Value normalization: trim text, collapse repeated whitespace, convert technical null tokens, uppercase codes, parse dates, parse whole-number fields.
- Lineage preservation via `_standardization_timestamp_utc`, `_standardization_run_id`, `_standardized_record_hash`, `standardization_error_count`, `standardization_error_reasons`, `standardization_status`.

**Result:**

```text
Rows:              15,931
Parse-error rows:      0
Validation:         PASS
```

---

### 4.4 NB02R — Standardize Airport Reference

**Purpose:** Create a typed and normalized reusable airport master while retaining all source records.

**Target:** `02standardized.airport_reference_standardized`

**Key logic:**
- Retain all airport records. Do not filter to flight-related codes in Standardized.
- Normalize code casing. Parse coordinates, scores, and timestamps.
- Report duplicate IATA mappings but resolve them in Curated.

**Boolean handling:** The source stores scheduled service as `0`/`1`. The parser accepts `yes/true/1/y` and `no/false/0/n`.

**Result:**

```text
Rows:                85,720
Parse-error rows:        0
Distinct IATA codes: 9,056
Validation:           PASS
```

---

### 4.5 NB03 — Curate Quality and Features

**Purpose:** Create the business-enriched, quality-controlled, analysis-ready Curated flight layer.

**Inputs:** `02standardized.flight_standardized`, `02standardized.airport_reference_standardized`

#### Reference Outputs

```text
03curated.ref_airline
03curated.ref_aircraft_type
03curated.ref_airport
```

**Airline reference:**

| Code | Name | Entity | Status |
|---|---|---|---|
| `EW` | Eurowings | Eurowings GmbH | `CURRENT` |
| `E6` | Eurowings Europe | Eurowings Europe Limited | `CURRENT` |
| `4U` | Germanwings | Germanwings GmbH | `LEGACY_REVIEW` |

**Aircraft reference:**

| Code | Aircraft | Family | Generation |
|---|---|---|---|
| `A319` | Airbus A319ceo | Airbus A320 Family | CEO |
| `A320` | Airbus A320ceo | Airbus A320 Family | CEO |
| `A20N` | Airbus A320neo | Airbus A320 Family | NEO |
| `A321` | Airbus A321ceo | Airbus A320 Family | CEO |
| `A21N` | Airbus A321neo | Airbus A320 Family | NEO |

**Aircraft configuration code:** `aircraft_type_code + "-" + aircraft_mtow_weight` (e.g., `A320-71500`, `A320-81000`). The two A320 MTOW values are not classified as errors — aircraft registration or fleet master data is required for validation.

#### Airport Reference Resolution

Curated logic filters the complete Standardized airport master to IATA codes used by the flight data and resolves duplicate IATA candidates deterministically:

1. Scheduled-service airport
2. Large airport
3. Medium airport
4. Small airport
5. Higher source score
6. Lower source ID

One Curated row is published per IATA code. The match resolution (unique or resolved from multiple candidates) is recorded.

#### Flight Enrichment Fields

**Identity and lineage:**

```text
flight_record_key
_source_... / _standardization_... / _curation_...
```

**Airline and aircraft:**

```text
airline_name, legal_entity_name, carrier_status, airline_master_match_status
aircraft_model_name, aircraft_family, aircraft_generation, manufacturer_name
aircraft_configuration_code, aircraft_master_match_status
```

**Airport and market:**

```text
origin_airport_reference_key, origin_airport_name, origin_city, origin_country_name
destination_airport_reference_key, destination_airport_name, destination_city
route_code, country_pair_code, market_scope (DOMESTIC / INTERNATIONAL / UNKNOWN)
```

**Date features:**

```text
flight_year, flight_quarter, flight_month_number, flight_month_name
flight_year_month, flight_week_of_year, flight_day_of_week_number, flight_day_of_week_name
```

**Weight features:**

```text
tow_utilisation_pct          = planned_tow_weight / aircraft_mtow_weight
remaining_mtow_margin_weight = aircraft_mtow_weight - planned_tow_weight
payload_to_tow_pct           = planned_payload_weight / planned_tow_weight
payload_to_mtow_pct          = planned_payload_weight / aircraft_mtow_weight
non_payload_planned_mass     = planned_tow_weight - planned_payload_weight
planned_payload_per_pax_proxy = planned_payload_weight / passenger_count
```

**Wind features:**

```text
absolute_wind_kts, wind_sign_category, wind_magnitude_category
```

Magnitude categories:

```text
0-40      NORMAL
41-60     HIGH
61-80     EXTREME
>80       CRITICAL_REVIEW
```

**Distribution bands:**

Wind (10-knot) and distance (100-NM) bands with keys, boundaries, labels, and sort orders.

---

### 4.6 NB03 — Data Quality Framework

#### Data-Quality Tables

| Table | Grain |
|---|---|
| `dq_rule_catalog` | One row per DQ rule |
| `dq_rule_results` | One row per triggered rule per flight (only triggered rules written) |
| `dq_record_summary` | One row per flight |
| `dq_batch_summary` | One row per curation execution |
| `column_profile_summary` | One row per profiled column per run |

#### Hard-Failure Rules

| Code | Rule |
|---|---|
| `DQ001` | Standardization status is not PASS |
| `DQ002` | Required business field missing |
| `DQ003` | Distance ≤ 0 |
| `DQ004` | Passenger count ≤ 0 |
| `DQ005` | Planned TOW ≤ 0 |
| `DQ006` | MTOW ≤ 0 |
| `DQ007` | Planned TOW > MTOW |
| `DQ008` | Planned payload < 0 |
| `DQ009` | Planned payload > planned TOW |
| `DQ010` | Invalid airport-code format |
| `DQ011` | Airport reference match missing |
| `DQ012` | Aircraft reference match missing |
| `DQ013` | Airline reference match missing |
| `DQ014` | Exact standardized duplicate |

#### Review Rules

| Code | Rule | Count |
|---|---|---:|
| `DQ101` | Payload = 0 with PAX > 0 | 128 |
| `DQ102` | Origin = destination | 5 |
| `DQ103` | Absolute wind > 60 kts | 175 |
| `DQ104` | Absolute wind > 80 kts | 5 |
| `DQ105` | TOW utilisation > 95% | 44 |
| `DQ106` | Legacy airline code | 1 |

#### Analysis-Specific Eligibility

```text
is_wind_distribution_eligible
is_distance_distribution_eligible
is_route_analysis_eligible
is_weight_analysis_eligible
is_payload_analysis_eligible
is_carrier_comparison_eligible
is_model_training_eligible
```

A row may be valid for the wind distribution but invalid for payload analysis. This replaces a single blanket "valid/invalid" flag.

#### NB03 Baseline Results

```text
Curated rows:                    15,931
Hard-failure records:                 0
Unique review records:              349
Records without review flags:     15,582
```

---

### 4.7 NB04 — Analytical Aggregates

**Purpose:** Create reusable analytical and statistical data products without replacing Power BI filter-context measures.

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

#### Key Published Benchmarks

```text
Wind P50:       -3 kts
Wind P85:       22 kts
Distance P50:   593 NM
Distance P85:   1,105 NM
```

#### Route-Distance Outlier Method

```text
IQR = P75 - P25
Lower bound = P25 - 1.5 × IQR
Upper bound = P75 + 1.5 × IQR
```

Outlier evaluation requires a minimum route sample of 10 flights.

#### Transparent Anomaly Features

```text
Route-distance outlier               +1
Absolute wind > 60                   +1
Absolute wind > 80                   +1
TOW utilisation > 95%                +1
Payload zero with PAX                +1
Same origin/destination              +1
```

Categories: `0 NORMAL`, `1 OBSERVE`, `2 REVIEW`, `3+ HIGH_REVIEW`

This is transparent rule-based anomaly engineering, not a trained AI model.

#### Metric Correlation Matrix

Pairwise-complete Pearson correlation across 7 metrics, producing 49 rows. Correlation is descriptive, not causal.

#### NB04 Results

```text
Source flight rows:               15,931
Analytical output tables:             11
Route-distance outlier rows:       2,041
Validation:                        PASS
```

---

### 4.8 NB05 — Predictive Analytics

**Purpose:** Demonstrate defensible data-science capability with a prediction target present in the dataset.

**Target:** `planned_tow_weight`  
**Eligibility:** `is_model_training_eligible = true`

**Population:**

```text
Source rows:           15,931
Model-eligible rows:   15,803
Date range:            2024-06-02 to 2024-12-31
```

#### Features

**Numeric:** `planned_payload_weight`, `airway_distance_nm`, `wind_component_kts`, `passenger_count`, `aircraft_mtow_weight`, `flight_month_number`

**Categorical:** `aircraft_type_code`, `route_code`, `airline_code`

Categorical processing: `StringIndexer(handleInvalid="keep")` → `OneHotEncoder(handleInvalid="keep", dropLast=True)` → `VectorAssembler(handleInvalid="error")`.

#### Chronological Split

```text
Strategy:   LAST_CALENDAR_MONTH_HOLDOUT
Training:   2024-06-02 to 2024-11-30 (13,984 rows)
Test:       2024-12-01 to 2024-12-31 (1,819 rows)
```

This avoids random leakage across time and keeps December as an unseen chronological holdout.

#### Models

| Model | Parameters |
|---|---|
| Linear Regression | maxIter=100, regParam=0.0, elasticNetParam=0.0, standardization=true |
| Gradient-Boosted Tree | maxIter=60, maxDepth=5, stepSize=0.05, subsamplingRate=0.8, seed=42 |

#### Results

| Model | MAE | RMSE | R² |
|---|---:|---:|---:|
| Linear Regression | 398.18 | 633.73 | 0.9924 |
| Gradient-Boosted Tree | 791.87 | 1,123.53 | 0.9761 |

Linear Regression is selected as the Trusted model based on lower chronological-test RMSE.

#### Outputs

```text
03curated.tow_model_predictions    (3,638 rows — 1,819 test flights × 2 models)
03curated.model_metrics            (6 rows — RMSE, MAE, R² per model)
03curated.model_feature_importance (162 rows — encoded and logical GBT importance)
03curated.model_run_summary        (2 rows — parameters, split dates, scope)
```

**Interpretation scope:** `ANALYTICAL_PROTOTYPE_NOT_OPERATIONAL_CERTIFICATION`

---

### 4.9 NB06 — Publish Trusted Model

**Purpose:** Publish a semantic-model-ready Trusted layer using stable keys, role-playing airport dimensions, validated foreign keys, and no hard-failure records.

**Trusted inclusion rule:** Include flights where `has_hard_failure = false`. Review-flagged records remain available with DQ flags and analysis-specific eligibility.

```text
Curated flight rows:      15,931
Hard-failure rows excluded:    0
Trusted fact-flight rows: 15,931
```

#### Key Strategy

- Source technical flight key: `flight_record_key`
- Dimension and aggregate-fact surrogate keys: deterministic `xxhash64` over normalized natural-key expressions
- Nulls in stable-key expressions normalized to `<NULL>`
- Every Trusted table receives `_trusted_run_id` and `_trusted_timestamp_utc`

#### Published Dimensions (11)

```text
dim_date                  Continuous calendar 2024-06-01 to 2024-12-31 (214 rows)
dim_airline
dim_aircraft
dim_origin_airport        Role-playing airport dimension
dim_destination_airport   Role-playing airport dimension
dim_route
dim_wind_band
dim_distance_band
dim_dq_rule
dim_analysis_population
dim_model                 Ranks models by chronological-test RMSE; one is_selected_model=true
```

Separate origin and destination dimensions were chosen over a shared airport table with an inactive relationship, keeping the semantic model simpler for report consumers.

#### Published Facts (11)

```text
fact_flight                        15,931 rows
fact_dq_rule_trigger                  358 rows
fact_route_month
fact_aircraft_route
fact_model_scoring                  3,638 rows
fact_model_metric                       6 rows
fact_model_feature_importance         162 rows
fact_percentile_reference
fact_analysis_population
fact_wind_distribution_global
fact_distance_distribution_global
```

#### Validation

- Trusted fact reconciles to the no-hard-failure Curated population.
- Unique primary keys for all dimensions and facts.
- No null dimension keys where prohibited.
- Foreign-key validation for the core fact and all supporting facts.
- No hard-failure rows in `fact_flight`.
- Exactly one selected model.
- Global cumulative percentages end at 100%.

```text
Status:               SUCCEEDED
Trusted run ID:       NB06_20260712T183117684404Z
Published dimensions: 11
Published facts:      11
Validation:           PASS
```

---

## 5. Data Quality Findings and Interpretation

### 5.1 Structural Integrity

```text
Rows:                          15,931
Missing/null source values:         0
Exact duplicate source rows:        0
Distance ≤ 0:                       0
PAX ≤ 0:                            0
Planned TOW ≤ 0:                    0
Planned TOW > MTOW:                 0
```

### 5.2 Review Conditions

#### Zero Planned Payload (128 records)

Condition: `planned_payload_weight = 0` and `passenger_count > 0`. Treatment: flag for business review, exclude from payload analysis and model training, retain eligibility for analyses not depending on payload.

#### Same Origin and Destination (5 records)

DUS-DUS records with non-zero distance. Possible explanations: return-to-origin, training/check flight, circular operation, source coding issue. Treatment: flag, exclude from route analysis, retain in the Curated population.

#### Wind Outliers (175 above 60 kts, 5 above 80 kts)

Range: -84 to +92 kts. Treatment: flag, do not remove, preserve signed value, avoid unsupported headwind/tailwind labelling.

#### TOW Utilisation

```text
>90%:   928 flights
>95%:    44 flights
>100%:    0 flights
```

Flights over 95% are operational review cases, not hard failures.

#### A320 MTOW Variants

A320 appears with MTOW values of 71,500 and 81,000. Treatment: create configuration code, classify neither as wrong, state that aircraft registration or fleet master is required.

#### DUS Concentration

The dataset is heavily concentrated on DUS destination operations. This is a sampling-bias warning, not a data-quality failure. Insights describe the supplied DUS-focused sample, not the entire network.

---

## 6. Unsupported Calculations Deliberately Rejected

### 6.1 Fuel

"Actual fuel" cannot be calculated as `planned TOW - generic OEW - planned payload`. Even with a correct aircraft-specific dry operating weight, this estimates fuel on board at take-off, not fuel burned. A defensible fuel model requires trip fuel, taxi fuel, block fuel, reserve fuel, landing fuel, aircraft registration, flight time, routing, altitude, weather profile, and operational restrictions.

### 6.2 Cargo

`Planned payload - PAX × assumed passenger weight` cannot be called actual cargo. This can only be an explicitly labelled scenario estimate.

### 6.3 Passenger Load Factor

True load factor requires actual sellable seat configuration. Generic aircraft maximum seating is insufficient.

### 6.4 Profitability and Market Share

The data does not contain revenue, fare, booking, sales channel, cost, yield, capacity, or competitor market totals. Therefore route profitability, market share, RASK, CASK, and competitor efficiency claims are not supported.

---

## 7. Lakehouse Table Inventory

### Control

```text
00control.notebook_step_log
```

### Raw

```text
01raw.flight_raw
01raw.airport_reference_raw
```

### Standardized

```text
02standardized.flight_standardized
02standardized.airport_reference_standardized
```

### Curated — References, Enrichment, and Quality

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

### Curated — Analytical Products

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

### Curated — Predictive Analytics

```text
03curated.tow_model_predictions
03curated.model_metrics
03curated.model_feature_importance
03curated.model_run_summary
```

### Trusted Dimensions

```text
04trusted.dim_date
04trusted.dim_airline
04trusted.dim_aircraft
04trusted.dim_origin_airport
04trusted.dim_destination_airport
04trusted.dim_route
04trusted.dim_wind_band
04trusted.dim_distance_band
04trusted.dim_dq_rule
04trusted.dim_analysis_population
04trusted.dim_model
```

### Trusted Facts

```text
04trusted.fact_flight
04trusted.fact_dq_rule_trigger
04trusted.fact_route_month
04trusted.fact_aircraft_route
04trusted.fact_model_scoring
04trusted.fact_model_metric
04trusted.fact_model_feature_importance
04trusted.fact_percentile_reference
04trusted.fact_analysis_population
04trusted.fact_wind_distribution_global
04trusted.fact_distance_distribution_global
```

---

## 8. Implementation Conventions

### Naming

```text
Notebooks:     NB01_... through NB06_...  (R suffix = reference branch)
Schemas:       00control, 01raw, 02standardized, 03curated, 04trusted
Tables:        lowercase_snake_case
Lineage:       _source_*, _ingestion_*, _standardization_*, _curation_*, _analytics_*, _trusted_*
```

### Error Handling

Each notebook uses a lightweight step wrapper:

```python
run_step(step_name, action)
```

Behaviour: prints `STARTED`, executes the cell function, prints `SUCCEEDED`. On failure, prints notebook name, step name, exception type, exception message, and re-raises a `RuntimeError` so the pipeline activity fails visibly.

### Delta Writes

```python
dataframe.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(target_table)
```

### Qualified Table References

Uses `` `{schema}`.`{table}` `` to protect numeric schema names such as `01raw`.

### Spark vs. Local Lakehouse Paths

```text
Spark:    Files/...
Pandas:   /lakehouse/default/Files/...
```

### Statistical Methods

- Global percentiles: `approxQuantile(..., relativeError=0.0)`
- Grouped percentiles: `percentile_approx(..., accuracy=100000)`
- Distribution accumulation: Spark window functions
- Correlation: pairwise-complete Pearson correlation
- Outlier method: route-specific 1.5 × IQR with minimum sample of 10
- Safe ratios: reusable `safe_ratio()` helper prevents divide-by-zero and null errors

---

## 9. Power BI Semantic Model Design

### Source and Storage Mode

Create the model from the `04trusted` schema using Direct Lake where supported. Initial table set: all 11 Trusted dimensions and all 11 Trusted facts.

### Relationships

Single-direction, one-to-many from dimensions to facts. No automatic bidirectional relationships. No unnecessary many-to-many.

**Core flight fact:**

```text
dim_date[date_key]                       1 → * fact_flight[date_key]
dim_airline[airline_key]                 1 → * fact_flight[airline_key]
dim_aircraft[aircraft_key]               1 → * fact_flight[aircraft_key]
dim_origin_airport[origin_airport_key]   1 → * fact_flight[origin_airport_key]
dim_destination_airport[destination_airport_key]
                                         1 → * fact_flight[destination_airport_key]
dim_route[route_key]                     1 → * fact_flight[route_key]
dim_wind_band[wind_band_key]             1 → * fact_flight[wind_band_key]
dim_distance_band[distance_band_key]     1 → * fact_flight[distance_band_key]
```

Both role-playing airport relationships remain active because origin and destination are separate dimensions.

### Model Properties

Dedicated measure table: `_Measures`

Display folders:

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

Sort-by settings applied to `dim_date[flight_month_name]`, `dim_date[flight_year_month]`, `dim_wind_band[wind_band_label]`, and `dim_distance_band[distance_band_label]`.

`dim_date` marked as the date table using `calendar_date`.

### Key DAX Measures

**Core:**

```DAX
Flight Count = SUM ( fact_flight[flight_count] )
Distinct Routes = DISTINCTCOUNT ( fact_flight[route_key] )
Review Rate = DIVIDE ( [Review Flights], [Flight Count] )
Average TOW Utilisation = AVERAGE ( fact_flight[tow_utilisation_pct] )
```

**Dynamic cumulative distributions:**

```DAX
Wind Cumulative % =
VAR CurrentBand = MAX ( dim_wind_band[wind_band_sort] )
RETURN
    DIVIDE (
        CALCULATE (
            [Flight Count],
            FILTER (
                ALLSELECTED ( dim_wind_band ),
                dim_wind_band[wind_band_sort] <= CurrentBand
            ),
            fact_flight[is_wind_distribution_eligible] = TRUE ()
        ),
        CALCULATE (
            [Flight Count],
            ALLSELECTED ( dim_wind_band ),
            fact_flight[is_wind_distribution_eligible] = TRUE ()
        )
    )
```

Equivalent distance measures use `dim_distance_band` and `is_distance_distribution_eligible`.

**Static global percentiles:**

```DAX
Global Wind P50 =
CALCULATE (
    MAX ( fact_percentile_reference[percentile_value] ),
    fact_percentile_reference[metric_code] = "WIND",
    fact_percentile_reference[percentile_code] = "P50"
)
```

**Predictive measures:**

```DAX
Selected Model RMSE =
CALCULATE (
    MAX ( fact_model_metric[metric_value] ),
    dim_model[is_selected_model] = TRUE (),
    fact_model_metric[metric_name] = "RMSE"
)

Prediction Within 1,000 =
VAR WithinRange =
    CALCULATE (
        COUNTROWS ( fact_model_scoring ),
        ABS ( fact_model_scoring[prediction_error_weight] ) <= 1000,
        dim_model[is_selected_model] = TRUE ()
    )
VAR TotalSelectedModel =
    CALCULATE (
        COUNTROWS ( fact_model_scoring ),
        dim_model[is_selected_model] = TRUE ()
    )
RETURN
    DIVIDE ( WithinRange, TotalSelectedModel )
```

---

## 10. Report Page Design

### Page 00 — Solution Overview

Business objective, original tasks, compact architecture diagram, dataset coverage, engineering status, Trusted-row reconciliation, important caveats (weight unit unconfirmed, signed wind definition unconfirmed, DUS-focused sample), and navigation buttons.

### Page 01 — Wind and Distance Distributions (Mandatory)

Wind flight-count distribution by 10-knot band with cumulative-percentage line on secondary axis. Clearly labelled P50 and P85 values. Distance flight-count distribution by 100-NM band with cumulative-percentage line. Slicers for date/month, airline, aircraft, and route. Optional selected-scope percentile cards.

### Page 02 — Network, Route, and Carrier

Route volume and sample share, origin/destination map, route-distance variability and outlier rate, monthly route profile, carrier comparison with sample-reliability warning, CEO/NEO deployment mix, DUS concentration disclosure.

### Page 03 — Aircraft, Weight, Payload, and Predictive Analytics

Aircraft and configuration deployment, TOW utilisation and remaining MTOW margin, payload/PAX relationship, A320 MTOW variants without declaring either incorrect, actual versus predicted planned TOW, error distribution and model metrics, GBT logical feature importance, explicit statement that the model is an analytical prototype.

### Page 04 — Data Quality and Trust

Total rows, hard failures, review flights, DQ trigger count, payload-zero flights, same-origin/destination flights, high/extreme wind flights, high-TOW-utilisation flights, unmatched reference rows, analysis-eligible populations, DQ rule breakdown, review trend, drill-through.

### Hidden Page — Flight Record Detail

Source lineage, flight attributes, reference enrichment, DQ rules triggered, eligibility fields, route/outlier features, selected-model scoring details.

---

## 11. Recommended Additional Data

### Operations

```text
flight number, flight-leg ID, aircraft registration, scheduled/actual times
delay codes, actual TOW, actual route trajectory, flight time, altitude profile, weather
```

### Fuel

```text
block fuel, taxi fuel, trip fuel, reserve fuel, landing fuel, fuel uplift
```

### Capacity and Payload

```text
sellable seats, cabin configuration, baggage weight, cargo weight, mail weight
cargo volume, MZFW, MLW, tail-specific DOW/OEW
```

### Commercial and Sales

```text
booking and flown revenue, fare family, sales channel, point of sale
ancillary revenue, no-show rate, airport charges, handling cost
```

These fields would enable route economics, true load factor, cargo analysis, fuel-efficiency analysis, and stronger commercial use cases.

---

## 12. Capabilities Demonstrated

```text
Business-requirement translation
Fabric workspace and Lakehouse design
Layered Raw / Standardized / Curated / Trusted architecture
Multi-source ingestion
Source lineage and hashing
Schema-drift handling
Technical standardization and parsing diagnostics
Reference-data integration
Data-quality rule design
Analysis-specific eligibility
PySpark transformations and window functions
Percentiles and cumulative distributions
Route, carrier, fleet, and monthly analytics
Route-specific IQR outlier detection
Transparent anomaly features
Correlation analysis
PySpark ML regression comparison
Chronological model validation
Feature-importance publication
Trusted dimensional modelling
Stable surrogate keys and foreign-key validation
Fabric pipeline orchestration
Parallel branches and dependency joins
Notebook and pipeline failure handling
Successful end-to-end execution
Power BI semantic model design
DAX measure layer design
Interactive dashboard development
```

---

## 13. Completion Boundary and Next Steps

### Completed

```text
Workspace and Lakehouse setup
Landing-folder structure
Flight and airport Raw ingestion
Flight and airport Standardization
Curated master-data references
Flight enrichment and analytical features
DQ framework and analysis eligibility
Analytical aggregates and percentile baselines
Route/carrier/aircraft/month profiles
Outlier, anomaly, and correlation outputs
Planned-TOW predictive prototype
Trusted dimensional publication
Fabric pipeline orchestration
Pipeline failure activity
Successful end-to-end run
Power BI semantic model
DAX measure layer
Dashboard report pages
```

### Engineering Boundary

The notebook and pipeline engineering layer is treated as **v1.0 frozen** unless semantic-model development exposes a specific schema defect. Avoid adding more upstream calculations merely to make report building easier; filter-responsive logic belongs in DAX.

### Future Enhancements

- Semantic-model refresh activity in the pipeline
- Scheduled pipeline execution
- Centralized run logging via `00control`
- Optional Fabric task flow for workspace navigation
- AI-assisted anomaly explanation after trusted data exists
- Production incremental/snapshot loading patterns
