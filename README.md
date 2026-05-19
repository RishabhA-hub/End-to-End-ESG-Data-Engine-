# 🌱 GreenLife FMCG — ESG Data Engine

**A production-grade, end-to-end ESG data platform** simulating real sustainability reporting for an FMCG company. Aligned with GRI, BRSR, and SASB frameworks. EcoVadis-style scoring. Full audit trail.

---

## Table of Contents

1. [What This Is](#what-this-is)
2. [Architecture Overview](#architecture-overview)
3. [Folder Structure](#folder-structure)
4. [Data Sources](#data-sources)
5. [Module Guide](#module-guide)
6. [How to Run](#how-to-run)
7. [Pipeline Flow](#pipeline-flow)
8. [Database Schema](#database-schema)
9. [ESG Framework Mapping](#esg-framework-mapping)
10. [Scoring Logic](#scoring-logic)
11. [Emission Factors](#emission-factors)
12. [Business Value](#business-value)
13. [Tech Stack](#tech-stack)
14. [Extending the System](#extending-the-system)

---

## What This Is

The ESG Data Engine is a **modular Python + SQL platform** that:

- Ingests raw ESG data from Excel, CSV, and live API feeds
- Cleans, validates, and standardizes data with full error logging
- Maps metrics to GRI, BRSR (India), and SASB frameworks
- Calculates Scope 1 / 2 / 3 GHG emissions using IPCC AR6 emission factors
- Computes 20+ ESG KPIs across Environment, Social, and Governance
- Scores performance on a 0–100 EcoVadis-style scale with risk flags
- Maintains an immutable SQL audit trail for every data change
- Visualizes everything through an interactive React/HTML dashboard

**Simulated company:** GreenLife FMCG Ltd — a mid-large Indian FMCG company with ₹4,800 Cr revenue, 12,400 employees, and 5 facilities across India.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    DATA SOURCES                          │
│  Excel (energy, water, waste, HR)  CSV (supply chain)   │
│  JSON API feed (live ESG metrics)                        │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│              LAYER 1 · DATA INGESTION                    │
│  ingestion/ingester.py                                   │
│  · Reads all sources via pandas                          │
│  · Adds _source_file, _ingested_at metadata             │
│  · Unit standardization (kWh, tCO2e, kL, tonnes)        │
│  · Ingestion log with success/failure tracking           │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│         LAYER 2 · DATA CLEANING & VALIDATION             │
│  processing/cleaner.py                                   │
│  · Null checks + median/mode imputation                  │
│  · Range validation (negatives, outliers)                │
│  · Duplicate removal                                     │
│  · Schema consistency checks                             │
│  · DataQualityReport per dataset                         │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│            LAYER 3 · ESG METRICS ENGINE                  │
│  metrics/kpi_engine.py                                   │
│  · EmissionsEngine: Scope 1, 2, 3 (GHG Protocol)        │
│  · KPIEngine: 25+ annual KPIs per year                   │
│  · YoY change computation                                │
│  · 2030 target tracking                                  │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│            LAYER 4 · ESG SCORING ENGINE                  │
│  scoring/scorer.py                                       │
│  · EcoVadis-style 0–100 scoring                          │
│  · Category scores: E (40%), S (30%), G (30%)            │
│  · Threshold-based KPI normalization                     │
│  · Risk flag generation (Critical / High / Medium)       │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│               LAYER 5 · STORAGE (SQL)                    │
│  database/db.py  +  esg_engine.db (SQLite)               │
│  · raw_data, cleaned_data, esg_metrics                   │
│  · kpi_results, esg_scorecard                            │
│  · audit_log, data_quality_log, framework_mapping        │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│           LAYER 6 · VISUALIZATION DASHBOARD              │
│  Interactive HTML/React dashboard                        │
│  · ESG Scorecard overview                                │
│  · Scope 1/2/3 emissions breakdown                       │
│  · KPI trend charts (2022–2024)                          │
│  · GRI / BRSR / SASB framework mapping table             │
│  · Data quality issues panel + audit trail               │
│  · Scenario / what-if analysis with risk heatmap         │
└─────────────────────────────────────────────────────────┘
```

---

## Folder Structure

```
esg_engine/
│
├── pipeline.py                  ← Master orchestrator (run this)
│
├── config/
│   └── config.py                ← Emission factors, weights, thresholds,
│                                   framework mappings, validation rules
│
├── data/
│   ├── data_generator.py        ← Synthetic dataset generator
│   ├── energy_consumption.xlsx  ← 180 rows: monthly energy by facility
│   ├── water_usage.xlsx         ← 180 rows: monthly water by facility
│   ├── waste_generation.xlsx    ← 60 rows: quarterly waste by facility
│   ├── hr_demographics.xlsx     ← 12 rows: annual HR by business unit
│   ├── governance_metrics.xlsx  ← 3 rows: annual governance KPIs
│   ├── supply_chain_emissions.csv ← 30 rows: supplier-level Scope 3
│   └── live_api_feed.json       ← Simulated live ESG API response
│
├── ingestion/
│   └── ingester.py              ← ESGDataIngester class
│
├── processing/
│   └── cleaner.py               ← ESGDataCleaner + DataQualityReport
│
├── metrics/
│   └── kpi_engine.py            ← EmissionsEngine + KPIEngine
│
├── scoring/
│   └── scorer.py                ← ESGScoringEngine (EcoVadis-style)
│
├── database/
│   └── db.py                    ← ESGDatabase (SQLite ORM + audit trail)
│
├── outputs/
│   ├── kpi_results.xlsx         ← Computed KPIs exported
│   ├── esg_scorecard.xlsx       ← Scorecard exported
│   └── data_quality_issues.xlsx ← DQ issues exported
│
└── esg_engine.db                ← SQLite database (auto-created)
```

---

## Data Sources

### 1. `energy_consumption.xlsx` — 180 rows
Monthly energy data across 5 facilities × 3 years (2022–2024).

| Column | Unit | Description |
|--------|------|-------------|
| year, month | — | Time dimension |
| facility | — | Mumbai HQ, Pune Plant, Chennai Plant, Delhi DC, Kolkata DC |
| electricity_kwh | kWh | Grid electricity consumed |
| diesel_litres | litres | Diesel for generators/vehicles |
| natural_gas_scm | SCM | Natural gas for boilers |
| lpg_kg | kg | LPG for cooking/heating |
| renewable_energy_kwh | kWh | Solar/wind self-generation |
| data_source | — | Smart Meter / ERP |

**Intentional data quality issues injected:** 3 null electricity values, 2 negative diesel values (for validation demo).

---

### 2. `water_usage.xlsx` — 180 rows
Monthly water withdrawal and discharge by facility.

| Column | Unit | Description |
|--------|------|-------------|
| water_withdrawal_kl | kL | Total water drawn |
| water_recycled_kl | kL | Treated and reused on-site |
| water_discharged_kl | kL | Effluent discharged |
| water_source | — | Municipal / Groundwater / Surface |

**Intentional issues:** 4 null withdrawal values.

---

### 3. `waste_generation.xlsx` — 60 rows
Quarterly waste data by facility and disposal route.

| Column | Unit | Description |
|--------|------|-------------|
| total_waste_tonne | tonnes | All waste streams combined |
| waste_recycled_tonne | tonnes | Diverted for recycling |
| waste_incinerated_tonne | tonnes | Energy recovery / incineration |
| waste_landfill_tonne | tonnes | Residual to landfill |
| hazardous_waste_tonne | tonnes | Separately tracked hazardous |

---

### 4. `hr_demographics.xlsx` — 12 rows
Annual HR and social metrics by business unit.

| Column | Description |
|--------|-------------|
| total_employees | Headcount (permanent + contract) |
| female_employees | Female workforce count |
| new_hires / attrition | Hiring and departure counts |
| training_hours_total | All employee training hours |
| ltifr | Lost Time Injury Frequency Rate (per million hours) |
| fatalities | Work-related fatalities |
| avg_salary_inr_lpa | Average salary in LPA |
| ceo_to_median_pay_ratio | Pay equity metric |

---

### 5. `supply_chain_emissions.csv` — 30 rows
Supplier-level Scope 3 Category 1 data (purchased goods & services).

| Column | Description |
|--------|-------------|
| supplier_name | Supplier entity |
| category | Raw Materials / Packaging / Transport / Chemicals |
| supply_chain_tier | Tier 1 / 2 / 3 |
| spend_inr_crore | Annual procurement spend |
| reported_emissions_tco2e | Supplier-reported or estimated emissions |
| data_quality | Reported / Estimated / Modelled |
| certified_sustainable | Boolean — sustainability certification |
| audit_completed | Boolean — ESG audit done |

---

### 6. `governance_metrics.xlsx` — 3 rows
Annual board and governance KPIs.

| Column | Description |
|--------|-------------|
| board_total_members / board_female_members | Board composition |
| ethics_violations_reported / resolved_pct | Compliance tracking |
| data_privacy_breaches | DPDP Act compliance |
| anti_corruption_training_pct | % employees trained |
| supplier_code_signed_pct | Supplier CoC coverage |
| esg_linked_exec_pay_pct | % executive pay tied to ESG targets |
| third_party_assurance | Whether ESG data externally assured |

---

### 7. `live_api_feed.json` — Simulated live feed
```json
{
  "feed_id": "ESG-LIVE-2024-Q4",
  "real_time_metrics": {
    "solar_generation_kwh_today": 10240,
    "current_energy_intensity": 2.1,
    "water_recycling_rate_pct": 31.4,
    "carbon_credits_owned": 1850,
    "sdg_goals_mapped": [3, 6, 7, 8, 12, 13, 15]
  },
  "alerts": [...]
}
```

---

## Module Guide

### `config/config.py`
The central configuration file. Edit this to change:
- **Emission factors** — updated annually from CEA, IPCC, DEFRA
- **Scoring weights** — E/S/G split and sub-category weights
- **KPI thresholds** — what counts as Excellent / Good / Average / Poor
- **Framework mappings** — GRI codes, BRSR sections, SASB codes
- **Validation rules** — min/max ranges per metric type

### `ingestion/ingester.py`
`ESGDataIngester(data_dir)` — call `.ingest_all()` to load everything. Returns a dict of DataFrames. Supports `.standardize_units()` for on-the-fly unit conversion (e.g., MWh → kWh, gallons → litres).

### `processing/cleaner.py`
`ESGDataCleaner` — dataset-specific clean methods (`clean_energy_data`, `clean_water_data`, etc.). Each returns `(cleaned_df, DataQualityReport)`. Reports accumulate all issues and corrections with timestamps.

### `metrics/kpi_engine.py`
- `EmissionsEngine` — pure calculation class. `.calc_scope1()`, `.calc_scope2()`, `.calc_scope3_*()` add emission columns to DataFrames.
- `KPIEngine` — `.compute_all_kpis()` aggregates all datasets by year into a unified KPI table.

### `scoring/scorer.py`
`ESGScoringEngine` — `.compute_scorecard(df_kpis)` produces year-by-year scores. Uses `normalize_kpi()` to map raw values to 0–100 using configurable thresholds. Risk flags are auto-generated.

### `database/db.py`
`ESGDatabase` — wraps SQLite. All writes auto-generate audit log entries. `.log_manual_correction()` for human-initiated corrections. `.get_audit_log()` returns full immutable history.

---

## How to Run

### Prerequisites
```bash
pip install pandas openpyxl numpy
```

### Run the full pipeline
```bash
cd esg_engine/
python pipeline.py
```

This executes all 7 steps in sequence and prints:
```
📊 PIPELINE SUMMARY
  Status:    SUCCESS
  Company:   GreenLife FMCG Ltd
  Runtime:   0.73s

📈 ESG SCORES:
  2022:  68.5 (Engaged) | E:56.1 S:70.4 G:83.3
  2023:  69.6 (Engaged) | E:56.5 S:71.5 G:85.1
  2024:  67.3 (Engaged) | E:57.1 S:68.6 G:79.7
```

### Run individual modules
```python
from ingestion.ingester import ESGDataIngester
ingester = ESGDataIngester("data/")
raw = ingester.ingest_all()

from processing.cleaner import ESGDataCleaner
cleaner = ESGDataCleaner()
clean_energy, report = cleaner.clean_energy_data(raw["energy_consumption"])
print(report.summary())
```

### Query the database
```python
from database.db import ESGDatabase
db = ESGDatabase()
print(db.get_kpis())
print(db.get_scorecard())
print(db.get_audit_log(limit=20))
```

---

## Pipeline Flow

```
pipeline.py
    │
    ├─ Step 1: data_generator.py      → saves 7 files to data/
    ├─ Step 2: ingester.ingest_all()  → loads 6 DataFrames
    ├─ Step 3: cleaner.clean_*()      → validates + imputes + logs issues
    ├─ Step 4: kpi_engine.compute_all_kpis() → 25+ KPIs × 3 years
    ├─ Step 5: scorer.compute_scorecard()    → E/S/G scores + risk flags
    ├─ Step 6: db.insert_kpis() / insert_scorecard() → SQLite
    └─ Step 7: df.to_excel() → outputs/kpi_results.xlsx, esg_scorecard.xlsx
```

---

## Database Schema

Seven tables in `esg_engine.db`:

| Table | Purpose |
|-------|---------|
| `raw_data` | Snapshot of original ingested records as JSON |
| `cleaned_data` | Standardized metric-level records post-validation |
| `esg_metrics` | Framework-tagged metrics (GRI/BRSR/SASB) |
| `kpi_results` | Annual computed KPIs — one row per year |
| `esg_scorecard` | Annual ESG scores, bands, sub-scores, risk flags |
| `audit_log` | Immutable log of every INSERT / UPDATE / CORRECTION |
| `data_quality_log` | All validation issues found and corrections applied |
| `framework_mapping` | Lookup table: raw field → GRI code / BRSR section / SASB |

### Key relationships
```
raw_data (1) ──→ (N) cleaned_data
kpi_results (1) ──→ (1) esg_scorecard  [on year]
All writes ──→ audit_log  [always]
All issues ──→ data_quality_log  [always]
```

---

## ESG Framework Mapping

| Raw Field | ESG Metric | GRI Code | BRSR Section | SASB Code |
|-----------|-----------|----------|--------------|-----------|
| scope1_emissions_tco2e | Direct GHG Emissions | GRI 305-1 | Principle 6 | EM-FP-110a.1 |
| scope2_emissions_tco2e | Indirect GHG Emissions | GRI 305-2 | Principle 6 | EM-FP-110a.1 |
| scope3_emissions_tco2e | Other Indirect GHG | GRI 305-3 | Principle 6 | EM-FP-110a.1 |
| total_energy_kwh | Total Energy Consumption | GRI 302-1 | Principle 6 | FB-AG-130a.1 |
| renewable_energy_kwh | Renewable Energy Share | GRI 302-1 | Principle 6 | FB-AG-130a.1 |
| total_water_kl | Water Withdrawal | GRI 303-3 | Principle 6 | FB-AG-140a.1 |
| total_waste_tonne | Waste Generated | GRI 306-3 | Principle 6 | FB-AG-150a.1 |
| gender_diversity_pct | Gender Diversity | GRI 405-1 | Principle 5 | — |
| ltifr | Lost Time Injury Rate | GRI 403-9 | Principle 8 | FB-AG-320a.1 |
| training_hours | Training & Education | GRI 404-1 | Principle 5 | — |
| board_diversity_pct | Board Gender Diversity | GRI 405-1 | Principle 1 | — |
| ethics_violations | Ethics & Compliance | GRI 205-3 | Principle 1 | — |
| supplier_audits_pct | Supplier Sustainability Audits | GRI 308-1 | Principle 2 | — |

---

## Scoring Logic

### Overall ESG Score = E×40% + S×30% + G×30%

#### Environment (40%)
| Sub-metric | Weight | KPI Used |
|-----------|--------|---------|
| Emissions | 40% | Emission intensity (tCO2e/INR crore) |
| Energy | 25% | Renewable energy % |
| Water | 20% | Water intensity (kL/INR crore) |
| Waste | 15% | Waste recycling rate % |

#### Social (30%)
| Sub-metric | Weight | KPI Used |
|-----------|--------|---------|
| Diversity | 30% | Female workforce % |
| Safety | 30% | LTIFR (lower = better) |
| Training | 20% | Training hours per employee |
| Community | 20% | Anti-corruption training coverage % |

#### Governance (30%)
| Sub-metric | Weight | KPI Used |
|-----------|--------|---------|
| Ethics | 35% | Ethics violations rate per 1000 employees |
| Board Diversity | 25% | Board female members % |
| Data Privacy | 20% | Breach count (inverted) |
| Supplier Conduct | 20% | Supplier audit completion % |

### Score Bands
| Score | Band | Meaning |
|-------|------|---------|
| 90–100 | Outstanding | Industry leader |
| 70–90 | Advanced | Strong performance |
| 50–70 | Engaged | Making progress |
| 25–50 | Partial | Significant gaps |
| 0–25 | Insufficient | Material ESG risk |

---

## Emission Factors

All factors sourced from authoritative standards:

| Source | Scope | Factor | Standard |
|--------|-------|--------|---------|
| India grid electricity | Scope 2 | 0.000708 tCO2e/kWh | CEA India 2023 |
| Diesel | Scope 1 | 0.002693 tCO2e/litre | IPCC AR6 |
| Natural gas | Scope 1 | 0.001996 tCO2e/SCM | IPCC AR6 |
| LPG | Scope 1 | 0.002983 tCO2e/kg | IPCC AR6 |
| Waste — landfill | Scope 3 | 0.4680 tCO2e/tonne | GHG Protocol |
| Waste — incineration | Scope 3 | 0.9100 tCO2e/tonne | GHG Protocol |
| Air travel | Scope 3 | 0.000255 tCO2e/pax-km | DEFRA 2023 |
| Road freight | Scope 3 | 0.000062 tCO2e/tonne-km | DEFRA 2023 |
| Renewable electricity | Scope 2 | 0.000000 tCO2e/kWh | Market-based |

Update these in `config/config.py → EMISSION_FACTORS` as standards are revised annually.

---

## Business Value

| Stakeholder | Value Delivered |
|------------|----------------|
| **ESG / Sustainability Team** | Automated calculation replaces manual Excel work; audit-ready data |
| **Finance / CFO** | Emission intensity tied to revenue; scenario analysis for carbon pricing |
| **Board / Governance** | Real-time scorecard with risk flags; GRI/BRSR/SASB compliance view |
| **External Auditors** | Full immutable audit trail; data quality log with correction history |
| **Investors / ESG Analysts** | EcoVadis-comparable score; framework-mapped disclosures |
| **Regulators (SEBI BRSR)** | BRSR-mapped metrics ready for annual report submission |

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Data processing | Python 3.12 + pandas | Industry standard for ESG data pipelines |
| Storage | SQLite (upgradeable to PostgreSQL) | Zero-dependency, audit-friendly, SQL-queryable |
| Configuration | Python config module | Version-controllable, no JSON/YAML parsing overhead |
| Visualization | HTML + Chart.js | No server required; embeddable in any web context |
| Reporting export | openpyxl via pandas | Excel-native for finance/ESG teams |
| Emission factors | Hardcoded + config | Reproducible; update once, applied everywhere |

---

## Extending the System

### Add a new data source
```python
# In ingester.py
def ingest_new_source(self, filename):
    return self.ingest_excel(filename)   # or ingest_csv()

# In pipeline.py — add to ingest_all() and pass to kpi_engine
```

### Add a new KPI
```python
# In kpi_engine.py → compute_all_kpis(), add to `row` dict
"new_kpi": round(some_calc, 2),

# In config.py → KPI_THRESHOLDS, add thresholds
"new_kpi": {"excellent": ..., "good": ..., "average": ..., "poor": ...}

# In scorer.py → relevant score method, add normalize_kpi() call
```

### Change scoring weights
Edit `config/config.py`:
```python
SCORING_WEIGHTS = {"environment": 0.45, "social": 0.30, "governance": 0.25}
```

### Connect to PostgreSQL
In `database/db.py`, replace `sqlite3.connect(self.db_path)` with:
```python
import psycopg2
conn = psycopg2.connect(os.environ["DATABASE_URL"])
```

### Add Streamlit dashboard
```bash
pip install streamlit plotly
streamlit run dashboard/app.py
```
The `db.get_kpis()` and `db.get_scorecard()` methods provide ready-to-plot DataFrames.

---

## Output Files

After running `pipeline.py`:

| File | Contents |
|------|----------|
| `outputs/kpi_results.xlsx` | 3 rows × 35 KPI columns, one per year |
| `outputs/esg_scorecard.xlsx` | 3 rows — scores, bands, sub-scores, risk flags |
| `outputs/data_quality_issues.xlsx` | All validation issues with correction log |
| `esg_engine.db` | SQLite database with all 8 tables |
| `esg_pipeline.log` | Full pipeline execution log |

---

*Built as a production-grade ESG SaaS prototype. Ready for presentation to ESG consulting firms, climate-tech startups, and enterprise sustainability teams.*
