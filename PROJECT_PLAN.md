# DataForge Covid-19 — Project Plan

## Context

This project builds a production-style Data Warehouse pipeline for the **Our World in Data (OWID) Covid-19 dataset** (`owid-covid-data.csv`, ~429K rows, 65 columns, country-date grain). The goal is to demonstrate end-to-end DWH and ETL skills: schema design in SQL Server, automated data loading via SSIS, analytical query layers answering 10 defined business questions, and a structured testing/validation framework.

---

## Dataset Quick Reference

| Property | Detail |
|---|---|
| File | `owid-covid-data.csv` |
| Rows | ~429,435 |
| Columns | 65 |
| Grain | One row per location per date |
| Locations | 255 (countries + OWID aggregate rows like `OWID_WRL`) |
| Date range | Jan 2020 → latest available |

**Key column groups:** iso_code, continent, location, date · new/total cases & deaths · new/total tests · vaccinations · ICU/hospital · demographics (population, gdp_per_capita, median_age, etc.) · excess mortality

---

## Tech Stack

| Layer | Tool |
|---|---|
| Database | SQL Server (SSMS) |
| ETL | SSIS (Visual Studio / SSDT) |
| Query/Logic | T-SQL stored procedures |
| Source control | Git (this repo) |

---

## Database Design

**Database name:** `Covid19DWH`

**Schemas:**
- `stg` — raw staging (all-NVARCHAR flat load, disposable)
- `dbo` — dimension and fact tables
- `rpt` — analytical results and validation output

**Tables:**

| Table | Schema | Purpose |
|---|---|---|
| `covid_raw` | `stg` | Flat load of every CSV row, all NVARCHAR |
| `covid_aggregates` | `stg` | OWID synthetic rows (iso_code starts with `OWID_`) routed here, NOT to fact |
| `dim_location` | `dbo` | One row per unique location with demographic attributes |
| `dim_date` | `dbo` | Calendar dimension covering full date range |
| `fact_covid_daily` | `dbo` | Daily metrics per location (FK to both dims) |
| `results_analytical` | `rpt` | Stored proc output — answers to Q01–Q10 |
| `results_validation` | `rpt` | Test results — pass/fail rows with counts and messages |

**Star schema:** `fact_covid_daily` → `dim_location` (location_key) + `dim_date` (date_key)

---

## SSIS Package Design

**Package:** `Covid19_ETL.dtsx`

| Task ID | Task Name | Description |
|---|---|---|
| CF_01 | Truncate Staging | Execute SQL: TRUNCATE stg.covid_raw, stg.covid_aggregates |
| CF_02 | Load CSV to Staging | Flat File Source → OLE DB Destination (stg.covid_raw) — all NVARCHAR |
| CF_03 | Route OWID Aggregates | INSERT into stg.covid_aggregates WHERE iso_code LIKE 'OWID\_%'; DELETE from stg.covid_raw |
| CF_04 | Load dim_location | Execute SQL: INSERT/MERGE from stg.covid_raw (deduplicated by location) |
| CF_05 | Load dim_date | Execute SQL: INSERT/MERGE date range from MIN(date) to MAX(date) in staging |
| CF_06 | Load fact_covid_daily | Execute SQL: INSERT from stg.covid_raw with CAST/CONVERT + FK lookups; MERGE dedup guard |
| CF_07 | Log ETL Run | Insert run metadata (timestamp, row counts, status) to rpt.results_validation |

---

## Analytical Questions & Stored Procedure Mapping

| Q# | Question | Stored Proc | Key Logic |
|---|---|---|---|
| Q01 | Total worldwide COVID cases | `rpt.usp_Q01_TotalWorldCases` | MAX(total_cases) WHERE location = 'World' |
| Q02 | Total worldwide deaths | `rpt.usp_Q02_TotalWorldDeaths` | MAX(total_deaths) WHERE location = 'World' |
| Q03 | Country with most cases | `rpt.usp_Q03_MostCasesCountry` | MAX(total_cases) per location, TOP 1 |
| Q04 | Country with fewest cases | `rpt.usp_Q04_FewestCasesCountry` | MIN(total_cases) per location WHERE total_cases > 0, TOP 1 |
| Q05 | Number of countries in dataset | `rpt.usp_Q05_CountryCount` | COUNT(DISTINCT location) from dim_location WHERE is_aggregate = 0 |
| Q06 | Deadliest year of the pandemic | `rpt.usp_Q06_DeadliestYear` | SUM(new_deaths) GROUP BY YEAR(date), TOP 1 |
| Q07 | Continent with most deaths | `rpt.usp_Q07_DeadliestContinent` | SUM(new_deaths) GROUP BY continent, TOP 1 |
| Q08 | People fully vaccinated globally | `rpt.usp_Q08_GlobalFullyVaccinated` | MAX(people_fully_vaccinated) WHERE location = 'World' |
| Q09 | Country that vaccinated first | `rpt.usp_Q09_FirstVaccinatingCountry` | MIN(date) WHERE new_vaccinations > 0 per country, TOP 1 |
| Q10 | Highest single-day new case count | `rpt.usp_Q10_PeakDailyNewCases` | MAX(new_cases) globally across all country rows |

**Master proc:** `rpt.usp_Run_All_Questions` — calls Q01–Q10 and inserts each answer into `rpt.results_analytical` (question_id, question_text, answer_value, answer_label, run_timestamp).

---

## File/Folder Structure

```
/DataForge Covid-19
  /sql
    /ddl
      01_create_database.sql
      02_create_stg_tables.sql
      03_create_dim_tables.sql
      04_create_fact_table.sql
      05_create_rpt_tables.sql
    /stored_procs
      Q01_TotalWorldCases.sql
      Q02_TotalWorldDeaths.sql
      Q03_MostCasesCountry.sql
      Q04_FewestCasesCountry.sql
      Q05_CountryCount.sql
      Q06_DeadliestYear.sql
      Q07_DeadliestContinent.sql
      Q08_GlobalFullyVaccinated.sql
      Q09_FirstVaccinatingCountry.sql
      Q10_PeakDailyNewCases.sql
      Run_All_Questions.sql
    /validation
      run_validation_suite.sql
  /ssis
    Covid19_ETL.dtsx
  /docs
    ERD.png
  PROJECT_PLAN.md
  Requirement.txt
  Covid-19 Questions.txt
  owid-covid-data.csv
```

---

## Phase 1 — Data Profiling & Schema Design

### Tasks
- [ ] Profile the CSV: identify NULL%, data type, min/max, distinct count for each of the 65 columns
- [ ] Identify which columns go to `dim_location` (static per-country attributes) vs `fact_covid_daily` (daily metrics)
- [ ] Confirm OWID aggregate rows (iso_code LIKE 'OWID_%') and how to handle them
- [ ] Design and draw the star schema ERD (`/docs/ERD.png`)
- [ ] Write DDL scripts for all 7 tables (under `/sql/ddl/`)
- [ ] Peer-review DDL — verify data types, NULLability, PKs, FKs, indexes
- [ ] Commit DDL scripts to git

### Exit Criteria
- [ ] ERD diagram exists at `/docs/ERD.png` showing all 7 tables with PKs, FKs, and column names
- [ ] All 5 DDL `.sql` files exist and are syntactically valid (parse without error in SSMS)
- [ ] `dim_location` column list is finalized with correct source-column mapping
- [ ] `fact_covid_daily` column list is finalized covering all 10 question-relevant metrics
- [ ] OWID aggregate handling strategy is documented in a comment in `02_create_stg_tables.sql`

---

## Phase 2 — DWH Database Build in SSMS

### Tasks
- [ ] Execute `01_create_database.sql` — creates `Covid19DWH` database with `stg`, `dbo`, `rpt` schemas
- [ ] Execute `02_create_stg_tables.sql` — creates `stg.covid_raw` and `stg.covid_aggregates`
- [ ] Execute `03_create_dim_tables.sql` — creates `dbo.dim_location` and `dbo.dim_date`
- [ ] Execute `04_create_fact_table.sql` — creates `dbo.fact_covid_daily` with FK constraints
- [ ] Execute `05_create_rpt_tables.sql` — creates `rpt.results_analytical` and `rpt.results_validation`
- [ ] Verify all objects in SSMS Object Explorer
- [ ] Run a quick sanity SELECT on each empty table to confirm schema matches ERD

### Exit Criteria
- [ ] `Covid19DWH` database exists in SQL Server with schemas `stg`, `dbo`, `rpt`
- [ ] All 7 tables exist with correct columns, data types, and constraints
- [ ] `SELECT * FROM <each_table>` returns 0 rows with no error (empty tables, schema confirmed)
- [ ] FK constraints on `fact_covid_daily` reference `dim_location` and `dim_date`
- [ ] All DDL scripts are committed to `/sql/ddl/` in git

---

## Phase 3 — ETL Pipeline via SSIS

### Tasks
- [ ] Create SSIS project in Visual Studio (SSDT) and add `Covid19_ETL.dtsx` package
- [ ] Configure OLE DB Connection Manager pointing to `Covid19DWH`
- [ ] Configure Flat File Connection Manager for `owid-covid-data.csv` (header row, comma delimiter)
- [ ] Build CF_01: Execute SQL Task — TRUNCATE `stg.covid_raw`, `stg.covid_aggregates`
- [ ] Build CF_02: Data Flow Task — Flat File Source → OLE DB Destination (`stg.covid_raw`), all columns NVARCHAR
- [ ] Build CF_03: Execute SQL Task — route OWID rows to `stg.covid_aggregates`, delete from `stg.covid_raw`
- [ ] Build CF_04: Execute SQL Task — MERGE/INSERT into `dbo.dim_location` from distinct location rows
- [ ] Build CF_05: Execute SQL Task — populate `dbo.dim_date` for the full date range in staging
- [ ] Build CF_06: Execute SQL Task — INSERT into `dbo.fact_covid_daily` with CAST/CONVERT + MERGE dedup guard
- [ ] Build CF_07: Execute SQL Task — log run metadata to `rpt.results_validation`
- [ ] Run full package end-to-end; fix any cast errors or truncation issues
- [ ] Commit `.dtsx` file to `/ssis/` in git

### Exit Criteria
- [ ] SSIS package runs to completion with no red X tasks (all tasks green checkmarks)
- [ ] `SELECT COUNT(*) FROM stg.covid_raw` ≈ 429,435 (all source rows loaded)
- [ ] `SELECT COUNT(*) FROM stg.covid_aggregates` > 0 (OWID rows correctly routed)
- [ ] `SELECT COUNT(*) FROM dbo.dim_location` = 255 (one row per unique location)
- [ ] `SELECT COUNT(*) FROM dbo.dim_date` covers the full date range in the source file
- [ ] `SELECT COUNT(*) FROM dbo.fact_covid_daily` ≈ total non-OWID country-date rows
- [ ] Running the package a second time produces **identical row counts** (idempotency confirmed)
- [ ] No NULL `location_key` or `date_key` in `fact_covid_daily`

---

## Phase 4 — Analytical Query Layer

### Tasks
- [ ] Write and test `rpt.usp_Q01_TotalWorldCases`
- [ ] Write and test `rpt.usp_Q02_TotalWorldDeaths`
- [ ] Write and test `rpt.usp_Q03_MostCasesCountry`
- [ ] Write and test `rpt.usp_Q04_FewestCasesCountry`
- [ ] Write and test `rpt.usp_Q05_CountryCount`
- [ ] Write and test `rpt.usp_Q06_DeadliestYear`
- [ ] Write and test `rpt.usp_Q07_DeadliestContinent`
- [ ] Write and test `rpt.usp_Q08_GlobalFullyVaccinated`
- [ ] Write and test `rpt.usp_Q09_FirstVaccinatingCountry`
- [ ] Write and test `rpt.usp_Q10_PeakDailyNewCases`
- [ ] Write `rpt.usp_Run_All_Questions` (calls Q01–Q10, inserts to `rpt.results_analytical`)
- [ ] Execute master proc and verify 10 rows in `rpt.results_analytical`
- [ ] Spot-check 3+ answers against public data (WHO, OWID website)
- [ ] Commit all stored proc `.sql` files to `/sql/stored_procs/` in git

### Exit Criteria
- [ ] All 10 individual stored procs exist and execute without error
- [ ] `EXEC rpt.usp_Run_All_Questions` inserts exactly 10 rows into `rpt.results_analytical`
- [ ] `SELECT * FROM rpt.results_analytical` shows a non-NULL, non-zero answer for all 10 questions
- [ ] Q01 answer is in the hundreds of millions range (plausibility check)
- [ ] Q02 answer is in the millions range (plausibility check)
- [ ] Q09 answer is a date in late 2020 or early 2021
- [ ] Running master proc a second time does NOT create duplicate rows
- [ ] All stored proc files committed to git

---

## Phase 5 — Testing & Validation

### Test Categories

| Category | Description |
|---|---|
| Row Count | Verify staging, dim, and fact row counts match expectations |
| Referential Integrity | No orphan FKs in fact_covid_daily |
| Data Quality | No all-NULL records in fact; dates in valid range; no future dates |
| Analytical Sanity | Cross-check SUM(new_cases) in fact vs MAX(total_cases) in OWID World row |
| Idempotency | Re-run SSIS + master proc; counts and answers must be identical |

### Tasks
- [ ] Write `/sql/validation/run_validation_suite.sql` with one test case per assertion
- [ ] Row count test: `stg.covid_raw` count = expected total
- [ ] Row count test: `dim_location` count = distinct location count in staging
- [ ] Row count test: `fact_covid_daily` count = staging minus OWID rows
- [ ] RI test: LEFT JOIN fact → dim_location WHERE dim key IS NULL = 0
- [ ] RI test: LEFT JOIN fact → dim_date WHERE dim key IS NULL = 0
- [ ] Data quality test: no row in fact WHERE date > GETDATE()
- [ ] Data quality test: MIN(date) in fact ≥ '2020-01-01'
- [ ] Analytical sanity test: SUM(new_cases) in fact ≈ OWID World total_cases (within 1%)
- [ ] Idempotency test: run SSIS twice, assert row counts unchanged
- [ ] Insert all results (test_name, status PASS/FAIL, actual_value, expected_value, run_timestamp) into `rpt.results_validation`
- [ ] Commit validation script to git

### Exit Criteria
- [ ] Validation script runs end-to-end without unhandled errors
- [ ] `SELECT * FROM rpt.results_validation WHERE status = 'FAIL'` returns 0 rows
- [ ] All 5 test categories have at least one PASS recorded in `rpt.results_validation`
- [ ] Idempotency test explicitly recorded as PASS
- [ ] Validation script committed to `/sql/validation/` in git

---

## Overall Project Completion Checklist

- [ ] Phase 1 — Data Profiling & Schema Design: all exit criteria met
- [ ] Phase 2 — DWH Build in SSMS: all exit criteria met
- [ ] Phase 3 — ETL via SSIS: all exit criteria met
- [ ] Phase 4 — Analytical Query Layer: all exit criteria met
- [ ] Phase 5 — Testing & Validation: all exit criteria met
- [ ] All SQL scripts, SSIS package, and ERD committed to git
- [ ] Final `git log` shows clean commit history with at least one commit per phase
