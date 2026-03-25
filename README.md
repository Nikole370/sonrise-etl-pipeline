# sonrise-etl-pipeline

ETL pipeline and Power BI reporting solution for a multi-branch dental clinic in Lima, Peru. Built to replace manual Excel-based analysis with automated data extraction, a local SQL Server data warehouse, and interactive dashboards for operational and executive monitoring.

---

## Background

The client operated **Dentalink**, a dental practice management SaaS, across 3 branches. All patient, appointment, payment, and treatment data lived inside Dentalink's CRM interface — visible per patient, but with no cross-clinic aggregated reporting. To analyze trends, staff had to manually export CSVs and work through them in Excel with no standardized metrics and no historical comparison.

The goal: build an automated pipeline that extracts data from Dentalink's API, loads it into a local SQL Server warehouse, and serves 4 Power BI report pages covering revenue, appointments, procedures, and goal tracking — refreshed daily with no manual intervention.

---

## Architecture

```
┌─────────────────────────────┐     ┌──────────────────────────────────────────┐
│       Data Sources          │     │              ETL Layer                   │
│                             │     │                                          │
│  Dentalink REST API  ───────┼────▶│  Python executables                      │
│  (token-authenticated,      │     │  · Scheduled via Windows Task Scheduler  │
│   paginated, 13 endpoints)  │     │  · 2 schedules: Mon (full) / Tue–Fri     │
│                             │     │  · BAT launchers → Python main logic     │
│  OneDrive Excel  ───────────┼────▶│  · Config-driven: JSON per table         │
│  (2 goal/target tables)     │     │  · Mapping layer: API fields → SQL cols  │
└─────────────────────────────┘     │  · Incremental + full load modes         │
                                    └──────────────────┬───────────────────────┘
                                                       │
                                    ┌──────────────────▼───────────────────────┐
                                    │         SQL Server Express (local)       │
                                    │         ~13 tables · 657K+ rows          │
                                    │         Data from 2016 to present        │
                                    └──────────────────┬───────────────────────┘
                                                       │
                                    ┌──────────────────▼───────────────────────┐
                                    │   Power BI (via local data gateway)      │
                                    │   Microsoft Fabric dataflow              │
                                    │   Daily scheduled refresh                │
                                    │   4 report pages · deployed to client   │
                                    └──────────────────────────────────────────┘
```

---

## Project Structure

```
SONRISE-ETL-Dentalink/
├── bat/
│   ├── run_diario.bat       # Tue–Fri daily extraction launcher
│   └── run_semanal.bat      # Monday full-refresh launcher
├── config/
│   ├── config.json          # Global settings: tables enabled, load mode, token path
│   ├── run_diario.json      # Table config for daily schedule
│   ├── run_semanal.json     # Table config for weekly schedule
│   └── token.txt            # API token (not committed to repo)
├── logs/
│   └── log.txt              # Execution logs per run
├── mappings/
│   └── *.json               # Field mapping: API response keys → SQL column names
├── src/
│   ├── db.py                # SQL Server connection, upsert, table operations
│   ├── logging_utils.py     # Logging setup and helpers
│   └── main.py              # Core extraction, transformation, and load logic
└── README.md
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data sources | Dentalink REST API · OneDrive Excel |
| Extraction | Python · `requests` · `pandas` · `openpyxl` |
| Database | SQL Server Express (local install on client machine) |
| Scheduling | Windows Task Scheduler · BAT scripts |
| Visualization | Power BI Desktop · Microsoft Fabric dataflows |
| Gateway | Power BI local data gateway (daily refresh) |

---

## Data Sources

### Dentalink API
- Token-authenticated REST API with full documentation provided by client
- **13 endpoints** mapped to SQL tables
- Paginated responses — extraction handles full pagination automatically
- Data volume: largest table has **657,773 rows** spanning records from 2016
- Load strategy varies by table:
  - Static/reference tables: occasional full refresh
  - Transactional tables: daily incremental load (Tue–Fri)
  - High-volume historical table: weekly full refresh (Monday)

### OneDrive Excel
- 2 sheets with monthly targets set by clinic management
- Used in the "Objectives vs Actual" report page to compare goals against real Dentalink data

---

## Key Engineering Decisions

### Config-driven table loading
Each table has a JSON config entry specifying: endpoint, load mode (full/incremental), enabled/disabled. Adding or disabling a table requires only a config change — no code modification.

### Parameterized field mapping
API response keys differ from SQL column names. A `mappings/` folder holds one JSON per table with the field mapping. The merge/upsert logic reads from these files, making schema changes manageable without touching core logic.

### Pagination handling
Dentalink returns paginated results. The extractor loops through all pages automatically and stops when the API signals the last page, regardless of total record count.

### Flexible date filtering
Supports dynamic date expressions in filters:
```
fecha_inicio: now-2y     # 2 years back from today
fecha_inicio: now-30d    # last 30 days
fecha_inicio: 2023-02-13 # absolute date
```
This allows each table to define its own historical lookback window in config.

### Rate limiting with jitter
To avoid thundering herd problems when multiple API calls fire in sequence, the extractor sleeps a randomized interval between requests. This prevents hitting Dentalink's rate limits and makes the extraction pattern less predictable and more resilient.

### Data quality normalization
The API returns inconsistent formats across endpoints. The pipeline normalizes:
- **Dates:** accepts `yyyy-mm-dd`, `dd/mm/yyyy`, and other formats → converts to standard SQL date
- **Times:** normalizes to `HH:MM:SS`
- **Empty/null values:** maps empty strings, `"null"`, `None`, and equivalent representations to proper SQL NULLs
- **Booleans and enums:** standardized across endpoints

---

## Data Model

SQL Server warehouse with ~13 tables covering:

| Domain | Tables |
|---|---|
| Patients | Registration, demographics, recapture status |
| Appointments | Scheduling, attendance status, specialty, assigned dentist |
| Payments | Transaction records, amounts, branch, date |
| Procedures & treatments | Procedure catalog, completion status, procedure-payment link |
| Targets | Monthly goals per branch and coordinator (from Excel) |

**Notable modeling challenge:** several KPIs required joining tables that had no direct relationship — connections had to route through 2–3 intermediate tables. This required careful relationship design in Power BI to avoid ambiguous paths and incorrect aggregations.

---

## Power BI Report

4 pages built for clinic management and the CEO:

**Page 1 — Patient & Revenue Overview**
KPIs: registered patients, patients with no prior payment, total revenue, active dentists. Trends: monthly revenue evolution, appointment volume by month, payments by branch, average days from first appointment to first payment, average ticket by dentist, hours attended by dentist.

**Page 2 — Appointment Operations**
KPIs: total appointments, attended, scheduled, first-visit appointments. Breakdowns by appointment status, monthly evolution, recurring patients by status, appointments per dentist, by coordinator/user, by specialty.

**Page 3 — Treatments & Procedures**
KPIs: procedures completed, average procedure ticket, pending procedures. Charts: billed amount vs procedures performed (monthly), revenue by fee category, procedure mix by specialty over time, top procedures by volume, top ticket by dentist, billing by branch.

**Page 4 — Objectives vs Actual**
Date-range filter. Tracks: procedures completed vs monthly target (%), average ticket vs target. Visual: billing by branch (target vs actual bar), appointments by coordinator (first-visit and recurrent targets vs actual).

Gateway configured for daily scheduled refresh via Microsoft Fabric dataflow.

---

## Challenges

**Requirement definition under ambiguity**
Requirements were scoped from a single initial brief with no iterative feedback loop. KPI definitions and data model design were derived independently from Dentalink's API documentation and the original spec — without clarification calls or prototyping cycles. This required strong ownership of technical decisions throughout the project.

**Complex Power BI data model**
Several requested KPIs involved metrics that couldn't be computed from directly related tables. Relationships had to be designed through bridge tables, and DAX measures were written to handle cross-filter paths correctly.

**API data quality**
Each Dentalink endpoint returned slightly different date formats, inconsistent null representations, and varying field structures. The normalization layer was built to handle this generically across all tables rather than with per-table patches.

**Local infrastructure**
The warehouse runs on SQL Server Express installed on the client's own machine. Setting up the Power BI local data gateway, configuring Windows credentials, and ensuring the scheduled tasks ran reliably in that environment added complexity beyond standard cloud deployments.

---

## Scale

- ~13 SQL tables loaded from API + Excel
- Largest table: **657,773 rows** (appointment history from 2016)
- Daily incremental loads for transactional tables
- Weekly full refresh for high-volume historical data
- Single-user deployment (CEO) for executive monitoring

---

## Note

This repository documents the pipeline architecture and engineering decisions. No client data is included. All metrics shown are structural (row counts, table names, KPI categories) — no patient, financial, or identifying information from the client is present. Shared for portfolio purposes.

---

*Developed as part of a client engagement at Qualis Tech · Lima, Peru · Python · SQL Server · Power BI · Microsoft Fabric · Windows Task Scheduler*
