# DataForge Covid-19 — Project Plan

## Context

This project builds a production-style **Data Warehouse pipeline** for the Our World in Data (OWID) Covid-19 dataset using **SQL Server + SSIS**. The primary learning objective is hands-on experience with **SSIS data transformation** — SSIS reads the CSV, applies all transformation rules in-flight, and writes clean data into a buffer table. SQL then loads dimensions and the fact table from that buffer.

**Source file:** `owid-covid-data.csv` — ~429,000 rows × 65 columns, country-date grain  
**Output:** A clean DWH (`Covid19DWH`) that answers Q01–Q10 without any post-load fixes needed  
**Stack:** SQL Server 2019+, SSIS (Visual Studio / SSDT), T-SQL, SSMS

Column classification (which of the 65 columns to keep vs skip) — see **Appendix A**.

---

## Phase Structure

| Phase | Focus | Core Output |
|---|---|---|
| **1 — Schema & DQ Design** | DDL, partitioning, DQ rules, reject/log tables, verification SP | `sql/create_tables.sql` + `sql/usp_verify_etl_load.sql` |
| **2 — ETL Pipeline (SSIS)** | All DQ rules enforced in-flight | `Covid19_ETL.dtsx` running idempotently |
| **3 — Analytical Query Layer** | Q01–Q10 stored procs | `rpt.results_analytical` with 10 answers |
| **4 — Testing & Validation** | Formal quality gate | `rpt.results_validation` all PASS |

---

## Project Folder Structure (target)

```
DataForge Covid-19/
├── owid-covid-data.csv
├── Covid-19 Questions.txt
├── Requirement.txt
├── PROJECT_PLAN.md
├── sql/
│   ├── create_tables.sql              ← single idempotent DDL script (Phase 1)
│   ├── usp_verify_etl_load.sql        ← 11-check verification SP (Phase 1)
│   ├── dml/
│   │   ├── load_dim_location.sql
│   │   ├── load_dim_date.sql
│   │   └── load_fact_covid_daily.sql
│   └── analytical/
│       ├── Q01_total_cases.sql  ...  Q10_highest_single_day.sql
│       └── run_all_questions.sql
├── ssis/
│   └── Covid19_ETL/
│       ├── Covid19_ETL.sln
│       ├── Covid19_ETL.dtproj
│       └── Covid19_ETL.dtsx
└── tests/
    └── validation_queries.sql
```

---

## Architecture Overview

```
owid-covid-data.csv
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  CF_02  Data Flow Task  (SSIS — all transformation here) │
│                                                          │
│  Flat File Source                                        │
│      │                                                   │
│      ▼                                                   │
│  Conditional Split ── [IsOWID / DQ-09] ──► stg.covid_aggregates │
│      │ [NotOWID]                                         │
│      ▼                                                   │
│  Conditional Split ── [InvalidDate / DQ-05] ──► dq_rejected_rows │
│      │ [ValidDate]                                        │
│      ▼                                                   │
│  Derived Column  (DQ-02: blank→NULL, DQ-04: N/A→NULL)   │
│      │                                                   │
│      ▼                                                   │
│  OLE DB Destination → stg.fact_stage                    │
└─────────────────────────────────────────────────────────┘
        │
        ▼
CF_03  MERGE → dbo.dim_location   (from stg.fact_stage)
CF_04  INSERT → dbo.dim_date      (from stg.fact_stage, DQ-06)
CF_05  MERGE → dbo.fact_covid_daily (TRY_CAST = DQ-CAST, MERGE = DQ-10)
CF_06  Update etl_run_log
```

---

## Phase 1 — Schema & DQ Design

**Goal:** Design the star schema, write all DDL in one idempotent script, define the DQ rule taxonomy, and write the post-load verification SP. Phase 1 output is SQL scripts only — no data loaded.

### Step 1 — Design the Star Schema ✅

- [ ] Draw ERD (paper or draw.io) showing:
  - **2 dimensions:** `dbo.dim_location`, `dbo.dim_date`
  - **1 fact:** `dbo.fact_covid_daily` — grain = one row per (country × calendar day)
  - **2 staging buffers:** `stg.fact_stage`, `stg.covid_aggregates`
  - **2 support tables:** `dbo.dq_rejected_rows`, `dbo.etl_run_log`
  - **2 reporting tables:** `rpt.results_analytical`, `rpt.results_validation`
- [ ] Confirm FKs: `fact_covid_daily` → `dim_location(location_key)` and `dim_date(date_key)`
- [ ] Confirm surrogate keys: `dim_location` uses `INT IDENTITY`, `dim_date` uses `INT YYYYMMDD`

### Step 2 — Define All Table Columns, Data Types, Constraints ✅

- [ ] For each table decide: column names, SQL Server data types, NULLability, PKs, FKs, UNIQUE constraints
- [ ] Columns and types are written directly into `sql/create_tables.sql` in Step 3

### Step 3 — Write `sql/create_tables.sql` ✅

- [ ] Single idempotent script — `DROP … IF EXISTS` then `CREATE` for every object
- [ ] Script order: schemas (`stg`, `rpt`) → staging tables → dims → fact → support tables → reporting tables
- [ ] Header comment block listing all skipped CSV columns with rationale (see Appendix A)
- [ ] SQL Server syntax only
- [ ] Run in SSMS; zero errors

### Step 4 — Add Partition Function and Scheme ✅

- [ ] Define partition function `pf_covid_year` on `INT` (date_key), RANGE RIGHT, boundaries at each year from 2021 to 2026 — producing 7 partitions covering 2020 through 2026+
- [ ] Define partition scheme `ps_covid_year` mapping all partitions to `[PRIMARY]`
- [ ] Add both objects to `sql/create_tables.sql`
- [ ] Clustered index on `fact_covid_daily(date_key, location_key)` aligned to `ps_covid_year`
- [ ] Verify after script runs: `sys.partitions` shows 7 rows for `fact_covid_daily`

### Step 5 — Design DQ Rules (10 rules) ✅

Document as a comment block in `create_tables.sql` header so the taxonomy is source-controlled with the schema.

| Rule ID | Where Enforced | What It Catches |
|---|---|---|
| DQ-CAST | MERGE SQL (`TRY_CAST`) | Non-castable value → NULL instead of load failure |
| DQ-01 | Appendix A / schema skip | Columns with >85% NULL excluded from all tables |
| DQ-02 | SSIS Derived Column | Blank `""` in numeric column → NULL |
| DQ-03 | SSIS Derived Column (pass-through) | Negative `new_cases`/`new_deaths` kept — valid corrections |
| DQ-04 | SSIS Derived Column | Literal `"N/A"` in numeric column → NULL |
| DQ-05 | SSIS Conditional Split | Date not matching `YYYY-MM-DD` format → `dq_rejected_rows` |
| DQ-06 | `load_dim_date.sql` | Date < 2020-01-01 or > today → excluded from dim_date and fact |
| DQ-07 | `load_dim_location.sql` MERGE | Duplicate `iso_code` → MERGE inserts only once |
| DQ-09 | SSIS Conditional Split | `iso_code LIKE 'OWID_%'` → `stg.covid_aggregates`, never reaches fact |
| DQ-10 | UNIQUE constraint + MERGE | Duplicate (location_key, date_key) in fact → MERGE skips; idempotency guaranteed |

*DQ-08 reserved.*

### Step 6 — Design Reject Table (`dq_rejected_rows`) ✅

- [ ] Table captures rows that fail DQ-02, DQ-04, or DQ-05 instead of silently dropping them
- [ ] Columns: `reject_id`, `run_id` (FK → etl_run_log), `rule_id`, `iso_code`, `date`, `column_name`, `raw_value`, `rejected_at`
- [ ] Add to `sql/create_tables.sql`

### Step 7 — Design ETL Run Log Table (`etl_run_log`) ✅

- [ ] One row per ETL package execution; tracks row counts per table and overall status
- [ ] Columns: `run_id`, `run_at`, `rows_read_csv`, `rows_to_aggregates`, `rows_to_fact_stage`, `rows_rejected`, `rows_loaded_dim_loc`, `rows_loaded_dim_date`, `rows_loaded_fact`, `run_duration_sec`, `status` ('SUCCESS' | 'FAILED' | 'PARTIAL')
- [ ] Add to `sql/create_tables.sql`
- [ ] SSIS CF_01 inserts a row with `status = 'RUNNING'`; CF_06 updates it to final status

### Step 8 — Write `sql/usp_verify_etl_load.sql` (11-check SP) ✅

- [ ] Post-load quality gate — run after every ETL execution; results land in `rpt.results_validation`
- [ ] Each check inserts one row: `PASS` if condition met, `FAIL` otherwise
- [ ] SP must run without error even on empty tables — returns FAIL, not an exception
- [ ] 11 checks:
  1. `stg.fact_stage` row count > 400,000
  2. `stg.covid_aggregates` row count > 5,000
  3. `dim_location` row count between 200 and 300
  4. `dim_date` row count between 700 and 1,500
  5. `fact_covid_daily` row count > 300,000
  6. No OWID rows in fact (join to `dim_location`, filter `iso_code LIKE 'OWID_%'`) → 0
  7. Referential integrity — `location_key` orphans in fact → 0
  8. Referential integrity — `date_key` orphans in fact → 0
  9. Date range — no `date_key` < 20200101 or > today in fact → 0
  10. Data quality — no blank or `'N/A'` text in any fact numeric column → 0
  11. Idempotency — fact row count matches previous run in `etl_run_log`

### Phase 1 Exit Criteria

- [ ] `sql/create_tables.sql` runs cleanly from scratch in SSMS (drop + recreate, zero errors)
- [ ] All 9 objects exist in SSMS: `dim_location`, `dim_date`, `fact_covid_daily`, `stg.fact_stage`, `stg.covid_aggregates`, `dq_rejected_rows`, `etl_run_log`, `rpt.results_analytical`, `rpt.results_validation`
- [ ] `pf_covid_year` and `ps_covid_year` visible in `sys.partition_functions` / `sys.partition_schemes`
- [ ] `fact_covid_daily` shows 7 partitions in `sys.partitions`
- [ ] DQ rule table (DQ-CAST, DQ-01 to DQ-07, DQ-09, DQ-10) in `create_tables.sql` header comment
- [ ] `sql/usp_verify_etl_load.sql` executes without error; returns 11 FAIL rows (correct — tables empty)
- [ ] Both scripts committed to git under `sql/`
- [ ] Zero data loaded

---

## Phase 2 — ETL Pipeline (SSIS)

**Goal:** Build the SSIS package that reads the CSV, enforces all DQ rules in-flight inside the Data Flow, and writes clean data into `stg.fact_stage`. SQL tasks then load dims and fact from there.

### SSIS Components Used

| Component | Task | What You Learn |
|---|---|---|
| Flat File Connection Manager | CF_02 | Wiring a delimited CSV as a data source |
| OLE DB Connection Manager | All tasks | Connecting SSIS to SQL Server |
| Execute SQL Task | CF_01, CF_03–CF_06 | Calling T-SQL from the Control Flow |
| Data Flow Task | CF_02 | Container where all row-level transformation happens |
| Flat File Source | CF_02 DFT | Parsing CSV rows into the pipeline |
| Conditional Split | CF_02 DFT | Routing rows by expression (DQ-09, DQ-05) |
| Derived Column | CF_02 DFT | Expression-based NULL replacement (DQ-02, DQ-04) |
| OLE DB Destination | CF_02 DFT | Bulk insert to `stg.fact_stage` / `stg.covid_aggregates` |
| Row Count | CF_02 DFT | Counting discarded rows so nothing disappears silently |
| Sequence Container | CF_03–04 | Grouping dim-load tasks visually |

### Tasks

#### 2.1 Create SSIS Project

- [ ] Visual Studio → New Project → **Integration Services Project**
- [ ] Name: `Covid19_ETL`; save under `ssis/Covid19_ETL/`
- [ ] Add **OLE DB Connection Manager**: Server = your instance, Database = `Covid19DWH`
- [ ] Add **Flat File Connection Manager** for `owid-covid-data.csv`:
  - Delimiter: `,` | Text qualifier: `"` | Header row: yes
  - All columns: `DT_WSTR` length 255 — type casting happens later in SQL

#### 2.2 Build the Control Flow (6 Tasks)

Connect all tasks sequentially with the green success arrow.

**CF_01 — Truncate Staging + Start Log**
- [ ] Execute SQL Task — truncates `stg.covid_aggregates` and `stg.fact_stage`; inserts a 'RUNNING' row into `etl_run_log`

**CF_02 — Transform and Load** ← *all DQ rules live inside this one Data Flow Task*
- [ ] Data Flow Task — see section 2.3 for the full pipeline inside

**CF_03 — Load dim_location** and **CF_04 — Load dim_date**
*(both inside a Sequence Container named "Load Dimensions")*
- [ ] CF_03: Execute SQL Task calling `sql/dml/load_dim_location.sql` — MERGE from `stg.fact_stage` into `dim_location` (DQ-07)
- [ ] CF_04: Execute SQL Task calling `sql/dml/load_dim_date.sql` — INSERT distinct valid dates from `stg.fact_stage` into `dim_date`, applying date range guard (DQ-06)

**CF_05 — Load fact_covid_daily**
- [ ] Execute SQL Task calling `sql/dml/load_fact_covid_daily.sql` — joins `stg.fact_stage` to both dims, applies `TRY_CAST` on all numeric columns (DQ-CAST), MERGEs into `fact_covid_daily` (DQ-10)

**CF_06 — Finalise Log**
- [ ] Execute SQL Task — updates `etl_run_log` with final row counts per table and sets `status = 'SUCCESS'`

#### 2.3 Build the Data Flow Inside CF_02

Build components in this order:

**Component 1 — Flat File Source**
- [ ] Select Flat File Connection Manager; click Preview to confirm columns arrive as `DT_WSTR`
- [ ] DQ-01 applied here: skipped columns (Appendix A) simply not mapped to any downstream destination

**Component 2 — Conditional Split: OWID Routing (DQ-09)**
- [ ] Name: `Split OWID vs Country`
- [ ] Condition `IsOWID`: checks whether `iso_code` starts with `"OWID_"` using `FINDSTRING`
- [ ] Default output: `NotOWID` → continues down the pipeline
- [ ] `IsOWID` output → **OLE DB Destination** → `stg.covid_aggregates`
- [ ] *SSIS lesson: Conditional Split routes entire rows — every row goes to exactly one output*

**Component 3 — Conditional Split: Date Validation (DQ-05)**
- [ ] Name: `Validate Date`
- [ ] Condition `ValidDate`: checks that `date` is not null, is 10 characters, and has dashes at positions 5 and 8 (YYYY-MM-DD structure)
- [ ] Default output: `InvalidDate` → **Row Count** transform (variable `@[User::InvalidDateCount]`) — makes discarded count visible
- [ ] *Note: SSIS expressions have no `ISDATE()` function — structural format check here; date range guard (DQ-06) applied in SQL*

**Component 4 — Derived Column: NULL Cleanup (DQ-02 & DQ-04)**
- [ ] Name: `Clean Nulls`
- [ ] For every numeric column (fact metrics + demographic columns), replace in-place using the ternary expression pattern: if value is NULL, blank, or "N/A" → output NULL; else output trimmed value
- [ ] In Advanced Editor → Output Columns: set `IsNullable = True` for each replaced column
- [ ] Wire error output → **Flat File Destination** → `error_rows.txt` (makes expression failures visible)
- [ ] *SSIS lesson: ternary `condition ? trueValue : falseValue` is the core Derived Column expression pattern*

**Component 5 — OLE DB Destination → `stg.fact_stage`**
- [ ] Access mode: Table or view — fast load
- [ ] Map all 33 kept columns by name

#### 2.4 Package Settings

- [ ] Package parameter `CSVFilePath` (String) → Flat File Connection Manager `ConnectionString`
- [ ] Package parameter `ServerConnection` → OLE DB Connection Manager
- [ ] User variable `@[User::InvalidDateCount]` (Int32) for Row Count transform
- [ ] Set `DefaultBufferMaxRows = 100000` and `DefaultBufferSize = 104857600` on CF_02 Data Flow Task

#### 2.5 Run and Verify

- [ ] Run in SSDT debug mode; verify all 6 tasks show green checkmarks
- [ ] Check row counts in SSMS after first run; note `InvalidDateCount` in the package log
- [ ] Run `EXEC rpt.usp_verify_etl_load` — all 11 checks should return PASS
- [ ] Run package a **second time** — dim and fact row counts must be identical (DQ-10)

### Phase 2 Exit Criteria

- [ ] `Covid19_ETL.dtsx` saved with all 6 Control Flow tasks connected and named
- [ ] CF_02 Data Flow has all 5 components wired correctly
- [ ] All 10 DQ rules verified after load:
  - [ ] DQ-CAST — no load failures from type mismatches
  - [ ] DQ-01 — skipped columns absent from all tables
  - [ ] DQ-02 — no blank strings in any numeric column of `stg.fact_stage`
  - [ ] DQ-03 — at least one negative `new_cases` exists in `fact_covid_daily`
  - [ ] DQ-04 — no `'N/A'` text in any column of `stg.fact_stage`
  - [ ] DQ-05 — `InvalidDateCount` inspected; rows in `error_rows.txt` reviewed
  - [ ] DQ-06 — no `date_key` outside 20200101–today in fact
  - [ ] DQ-07 — `COUNT(*) FROM dim_location` = `COUNT(DISTINCT iso_code) FROM stg.fact_stage`
  - [ ] DQ-09 — zero OWID rows in `fact_covid_daily` (verified via join to `dim_location`)
  - [ ] DQ-10 — second ETL run produces identical fact row count
- [ ] `error_rows.txt` exists; contents reviewed and explained
- [ ] Package uses parameters — no hardcoded paths or server names
- [ ] `etl_run_log` has one row per run showing `status = 'SUCCESS'`
- [ ] `EXEC rpt.usp_verify_etl_load` returns 11 PASS rows

---

## Phase 3 — Analytical Query Layer

**Goal:** Write T-SQL stored procedures that answer Q01–Q10 from the clean DWH. Each proc inserts its result into `rpt.results_analytical`.

### Q01–Q10 Answer Mapping

| # | Question | Tables | SQL Logic |
|---|---|---|---|
| Q01 | Total worldwide cases | fact + dim_location | `SUM(MAX(total_cases))` per country |
| Q02 | Total worldwide deaths | fact + dim_location | `SUM(MAX(total_deaths))` per country |
| Q03 | Country with most cases | fact + dim_location | `TOP 1 MAX(total_cases)` by location |
| Q04 | Country with fewest cases | fact + dim_location | `MIN` of `MAX(total_cases)` per country, exclude 0 |
| Q05 | Number of countries | dim_location | `COUNT(*)` |
| Q06 | Deadliest year | fact + dim_date | `SUM(new_deaths) GROUP BY year`, top 1 |
| Q07 | Continent with most deaths | fact + dim_location | `SUM(MAX(total_deaths))` by continent |
| Q08 | People fully vaccinated globally | fact + dim_location | `SUM(MAX(people_fully_vaccinated))` per country |
| Q09 | First country to vaccinate | fact + dim_location + dim_date | `MIN(full_date)` where `new_vaccinations > 0` |
| Q10 | Highest single-day new cases | fact + dim_date + dim_location | `MAX(new_cases)` with location and date |

### Tasks

- [ ] Create one `.sql` file per question under `sql/analytical/` → one stored proc each (`rpt.usp_Q01_*` through `rpt.usp_Q10_*`)
- [ ] Each proc inserts one row into `rpt.results_analytical(question_id, question_text, answer)`
- [ ] Create `sql/analytical/run_all_questions.sql` → proc `rpt.usp_Run_All_Questions` that truncates the table then calls all 10 procs in sequence
- [ ] Run `EXEC rpt.usp_Run_All_Questions`; verify 10 rows in `rpt.results_analytical`

### Phase 3 Exit Criteria

- [ ] All 10 stored procedures run without error
- [ ] Exactly 10 rows in `rpt.results_analytical`, all non-NULL
- [ ] Q05 country count is between 200 and 250
- [ ] Q09 answer is a country name with a date in 2020
- [ ] Q10 answer includes country name, date, and a numeric count

---

## Phase 4 — Testing & Validation

**Goal:** Run the verification SP and analytical sanity checks; confirm everything is PASS.

### Tasks

- [ ] Run `EXEC rpt.usp_verify_etl_load` — confirm all 11 checks PASS
- [ ] Create `tests/validation_queries.sql` with additional analytical sanity checks:
  - Sum of max total deaths per country > 1,000,000
  - Count of countries with `people_fully_vaccinated > 0` is > 100
  - Earliest date with `new_vaccinations > 0` is before June 2021
- [ ] Idempotency test: record fact count → run SSIS again → assert count unchanged → write PASS/FAIL to `rpt.results_validation`
- [ ] For any FAIL: trace back to the responsible DQ rule and fix in the relevant phase

### Phase 4 Exit Criteria

- [ ] `EXEC rpt.usp_verify_etl_load` returns 11 PASS rows
- [ ] All additional sanity checks pass
- [ ] Idempotency explicitly shows PASS in `rpt.results_validation`
- [ ] `rpt.results_analytical` (10 rows) and `rpt.results_validation` (all PASS) both present

---

## Project Completion Checklist

- [ ] Phase 1 — `sql/create_tables.sql` and `sql/usp_verify_etl_load.sql` written and committed
- [ ] Phase 2 — SSIS package runs end-to-end, all 10 DQ rules enforced, idempotent
- [ ] Phase 3 — 10 stored procs answer Q01–Q10, results in `rpt.results_analytical`
- [ ] Phase 4 — all 11 verification checks PASS, idempotency confirmed
- [ ] All `.sql` files committed to git under `sql/`
- [ ] SSIS package committed under `ssis/`

---

## Key Design Decisions (Reference)

| Decision | Rationale |
|---|---|
| Transform in-flight in SSIS (no raw staging table) | All CSV rows are cleaned by Derived Column and routed by Conditional Split before touching any typed table |
| `stg.fact_stage` holds all kept columns as NVARCHAR | Bridge between SSIS output and FK-constrained fact table; dims also sourced from here — CSV read once |
| Conditional Split for OWID before Derived Column | Route OWID rows first so NULL-cleanup expressions only run on country rows |
| Derived Column replaces columns in-place | Downstream components see the same column names — no suffix aliasing needed |
| `TRY_CAST` in dim and fact MERGE SQL (DQ-CAST) | Final safety net — anything Derived Column missed becomes NULL, not a load failure |
| MERGE for all dim and fact loads | First run and all re-runs handled identically — idempotency built in |
| Negative `new_cases` kept (DQ-03) | Health authorities issue negative corrections; they are valid data |
| OWID rows to `stg.covid_aggregates` (DQ-09) | Kept for auditability; mixing into fact would double-count Q01, Q02, Q07, Q08 |
| `date_key` as INT YYYYMMDD | Timezone-safe; fast integer equality joins; partition boundary aligned |
| Fact partitioned by year | Enables partition elimination on date-range queries; prepares schema for future archiving |
| `dq_rejected_rows` + `etl_run_log` | Makes ETL behaviour auditable — every run's row counts and rejections are on record |

---

## Appendix A — Column Classification (65 columns)

Skipped columns must not appear in any table or SSIS column mapping.

| Column(s) | Classification | Rationale |
|---|---|---|
| iso_code, continent, location | Keep-Dim | Location identity |
| population, population_density, median_age, aged_65_older, aged_70_older, gdp_per_capita, extreme_poverty, cardiovasc_death_rate, diabetes_prevalence, female_smokers, male_smokers, handwashing_facilities, hospital_beds_per_thousand, life_expectancy, human_development_index | Keep-Dim | Static demographics |
| date | Grain key | Bridged to dim_date via date_key |
| total_cases, new_cases | Keep-Fact | Core for Q01, Q03, Q04, Q10 |
| total_deaths, new_deaths | Keep-Fact | Core for Q02, Q06, Q07 |
| people_fully_vaccinated, total_vaccinations, new_vaccinations, people_vaccinated, total_boosters | Keep-Fact | Core for Q08, Q09 |
| total_tests, new_tests, positive_rate | Keep-Fact | Supporting analytics |
| stringency_index, reproduction_rate | Keep-Fact | Policy & spread metrics |
| new_cases_smoothed, new_deaths_smoothed, `*_per_million`, `*_per_hundred`, `*_per_thousand` | **Skip** | Derived from kept columns |
| icu_patients, hosp_patients, weekly_icu_admissions, weekly_hosp_admissions, `weekly_*_per_million` | **Skip** | >80% NULL |
| excess_mortality_cumulative_absolute, excess_mortality_cumulative, excess_mortality, excess_mortality_cumulative_per_million | **Skip** | >80% NULL; not needed for Q01–Q10 |
| tests_units | **Skip** | Free-text, inconsistent |