# Flight Operations Analytics — Data Modelling and Dashboard

End-to-end flight operations analytics data product built in **Microsoft Fabric** and **Power BI**. Covers data engineering, semantic star-schema modelling, predictive ML, and interactive dashboard delivery — from raw Excel through Trusted publication.

---

## Overview

This project transforms a flight operations dataset (15,931 records, 11 business columns) into a production-grade analytical data product. Rather than a direct Excel-to-Power-BI import, the solution implements a full layered data pipeline demonstrating scalable data-product thinking: business-requirement translation, data engineering, semantic modelling, DAX measure design, data-quality governance, and predictive analytics.

An external airport reference source (85,720 rows from OurAirports) is integrated through its own separate ingestion lineage branch with deterministic IATA conflict resolution.

---

## Architecture

```
Raw → Standardized → Curated → Trusted → Power BI Semantic Model → Dashboard
```

### Layered Lakehouse Schemas

| Layer | Responsibility |
|---|---|
| `01raw` | Source-preserving ingestion with technical lineage only |
| `02standardized` | Technical renaming, normalization, typing, parse controls, row reconciliation |
| `03curated` | Business enrichment, DQ logic, analysis eligibility, analytical aggregates, ML outputs |
| `04trusted` | Semantic-model-ready dimensions and facts with stable keys and validated foreign keys |

### End-to-End Pipeline

8 PySpark notebooks orchestrated by a Fabric Data Factory pipeline with parallel source branches, dependency joins, fail-fast behaviour, and explicit failure handling.

```
NB01 Ingest Raw ──────────────► NB02 Standardize ────────────┐
                                                               ► NB03 Curate ► NB04 Analytics ► NB05 ML ► NB06 Trusted
NB01R Ingest Airport Ref ─────► NB02R Standardize Airport ──┘
```

All 8 activities succeeded on a full manual pipeline run.

---

## Key Results

| Metric | Value |
|---|---|
| Flight records processed | 15,931 |
| Airport reference rows integrated | 85,720 |
| Lakehouse tables published | 40+ across 4 layers |
| Trusted star schema | 11 dimensions, 11 facts |
| Data-quality rules | 14 hard-failure + 6 review |
| Hard failures found | 0 |
| ML model (Linear Regression) | R² = 0.9924, MAE = 398, RMSE = 634 |
| Wind P50 / P85 | -3 kts / 22 kts |
| Distance P50 / P85 | 593 NM / 1,105 NM |
| Pipeline status | Succeeded (all 8 activities green) |

---

## Tech Stack

| Area | Technology |
|---|---|
| Data platform | Microsoft Fabric (Lakehouse, Data Factory, Notebooks) |
| Data processing | PySpark, Delta Lake |
| Semantic model | Power BI (Direct Lake, DAX, star schema) |
| Machine learning | PySpark ML (Linear Regression, GBT) |
| Reference data | OurAirports public dataset |
| Version control | Git |

---

## Repository Structure

```
.
├── notebooks/                     # 8 PySpark notebook HTML exports with executed code and outputs
│   ├── NB01_Ingest_Raw.html
│   ├── NB01R_Ingest_Airport_Reference.html
│   ├── NB02_Standardize_Data.html
│   ├── NB02R_Standardize_Airport_Reference.html
│   ├── NB03_Curate_Quality_Features.html
│   ├── NB04_Analytical_Aggregates.html
│   ├── NB05_Predictive_Analytics.html
│   └── NB06_Publish_Trusted_Model.html
│
├── data-engineering/              # Pipeline definition and snapshots
│   ├── PL_EW_Flight_Operations_EndToEnd.json
│   └── Data_Pipeline_Snapshot.png
│
├── semantic-modelling/            # Power BI semantic model design
│   └── Semantic_Model_Snapshot.png
│
├── powerbi-dashboard/             # Power BI report files
│   ├── RP_EW_Flight_Operations.pbix
│   └── RP_EW_Flight_Operations.pdf
│
├── case-study-report/             # Formal case study report
│   └── Vikas_Rathore_Eurowings_Digital_Use_Case_Report.pdf
│
├── technical-documentation/       # Full engineering documentation
│   └── Flight_Operations_Analytics_Technical_Documentation.md
│
├── fabric-workspace/              # Workspace and Lakehouse configuration snapshots
│   ├── Fabrics_Workspace_Snapshot.png
│   └── Lakehouse_Snapshot.png
│
└── data-samples/                  # Source data files
    ├── Data for Analysis.xlsx     # Flight operations data (15,931 rows)
    └── airports.csv               # OurAirports reference (85,720 rows)
```

---

## Notebook Pipeline

| Notebook | Purpose | Key Output |
|---|---|---|
| **NB01** Ingest Raw | Source-preserving Excel ingestion | `01raw.flight_raw` (15,931 rows) |
| **NB01R** Ingest Airport Reference | OurAirports CSV ingestion | `01raw.airport_reference_raw` (85,720 rows) |
| **NB02** Standardize Data | Type conversion, renaming, normalization | `02standardized.flight_standardized` (0 parse errors) |
| **NB02R** Standardize Airport Reference | Typed airport master | `02standardized.airport_reference_standardized` (9,056 IATA codes) |
| **NB03** Curate Quality & Features | Enrichment, DQ framework, eligibility flags | `03curated.flight_enriched` + 5 DQ tables (0 hard failures) |
| **NB04** Analytical Aggregates | Distributions, profiles, outliers, anomalies | 11 analytical tables, P50/P85 benchmarks |
| **NB05** Predictive Analytics | ML model comparison (LR vs GBT) | TOW prediction (R²=0.99), feature importance, model metrics |
| **NB06** Publish Trusted Model | Star-schema-ready dimensions and facts | 11 dimensions, 11 facts, validated foreign keys |

---

## Data Quality Framework

### Hard-Failure Rules (14)

Records violating these are excluded from the Trusted layer.

| Code | Rule |
|---|---|
| DQ001–DQ006 | Standardization failure, missing fields, non-positive distances/weights |
| DQ007 | Planned TOW > MTOW |
| DQ008–DQ009 | Invalid payload values |
| DQ010–DQ013 | Invalid airport/aircraft/airline reference matches |
| DQ014 | Exact standardized duplicate |

### Review Rules (6)

Records are flagged but retained with analysis-specific eligibility.

| Code | Rule | Count |
|---|---|---:|
| DQ101 | Payload = 0 with PAX > 0 | 128 |
| DQ102 | Origin = destination | 5 |
| DQ103 | Absolute wind > 60 kts | 175 |
| DQ104 | Absolute wind > 80 kts | 5 |
| DQ105 | TOW utilisation > 95% | 44 |
| DQ106 | Legacy airline code | 1 |

### Analysis-Specific Eligibility

Each analytical question gets its own clean population instead of a blanket valid/invalid flag:

```
is_wind_distribution_eligible
is_distance_distribution_eligible
is_route_analysis_eligible
is_weight_analysis_eligible
is_payload_analysis_eligible
is_carrier_comparison_eligible
is_model_training_eligible
```

---

## Predictive Analytics

Planned take-off weight prediction using PySpark ML with chronological holdout validation.

| Model | MAE | RMSE | R² |
|---|---:|---:|---:|
| **Linear Regression** (selected) | 398.18 | 633.73 | **0.9924** |
| Gradient-Boosted Tree | 791.87 | 1,123.53 | 0.9761 |

**Split:** Training Jun–Nov 2024 (13,984 rows) → Test Dec 2024 (1,819 rows)

**Features:** payload, distance, wind, passengers, MTOW, month, aircraft type, route, airline

---

## Power BI Dashboard

### Report Pages

| Page | Content |
|---|---|
| **00 — Solution Overview** | Business objective, architecture diagram, caveats, navigation |
| **01 — Wind & Distance Distributions** | Mandatory distributions with cumulative %, P50/P85 markers, slicers |
| **02 — Network, Route & Carrier** | Route volume, map, carrier comparison, CEO/NEO mix |
| **03 — Aircraft, Weight & Predictive** | TOW utilisation, payload analysis, actual vs predicted TOW, ML metrics |
| **04 — Data Quality & Trust** | DQ rule breakdown, review trends, eligible populations |

### Semantic Model

- Single-direction dimension-to-fact relationships
- Role-playing origin and destination airport dimensions (both active)
- Dedicated `_Measures` table with 8 display folders
- Dynamic DAX cumulative distributions responding to slicers
- Static global percentiles and dynamic selected-scope percentiles
- Predictive model scoring measures

---

## Technical Documentation

The full engineering documentation is in [`technical-documentation/Flight_Operations_Analytics_Technical_Documentation.md`](technical-documentation/Flight_Operations_Analytics_Technical_Documentation.md). It covers:

- Complete architecture and lineage diagrams
- Every notebook specification with validation results
- Full data-quality rule catalogue with counts
- Trusted layer key strategy and foreign-key validation
- DAX measure definitions with code
- Report page design specifications
- Implementation conventions and statistical methods
- Deliberately rejected calculations with rationale
- Recommended additional data for production use

---

## Calculation Placement Principle

```
Heavy, stable, governed, reusable logic   →  Fabric notebooks and Delta tables
Dynamic filter-context calculations        →  Power BI DAX measures
Formatting, narrative, visual interaction  →  Power BI report layer
```

---

## Author

**Vikas Rathore**  
M.Sc. Information Technology | Data Analytics & Engineering  
Aviation analytics background with Lufthansa Group experience

---

## License

This project is shared for portfolio and demonstration purposes. The source data is a sample dataset. The OurAirports reference data is from the public OurAirports open dataset.
