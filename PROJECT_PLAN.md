# DataForge Covid-19 — Project Plan

**Source Dataset:** Our World in Data (OWID) `owid-covid-data.csv`  
**Stack:** SQL Server (SSMS), SSIS, T-SQL  
**Objective:** Build a production-style DWH pipeline that ingests raw COVID-19 data, structures it into fact/dimension tables, answers 10 analytical questions, and validates the entire pipeline with stored results.

---

## Dataset Overview

The CSV contains ~67 columns per country-date row including:

| Category | Key Columns |
|---|---|
| Identity | `iso_code`, `continent`, `location`, `date` |
| Cases | `total_cases`, `new_cases`, `new_cases_smoothed` |
| Deaths | `total_deaths`, `new_deaths`, `new_deaths_smoothed` |
| Hospitalisation | `icu_patients`, `hosp_patients` |
| Testing | `total_tests`, `new_tests`, `positive_rate` |
| Vaccination | `total_vaccinations`, `people_vaccinated`, `people_fully_vaccinated`, `total_boosters` |
| Demographics | `population`, `median_age`, `aged_65_older`, `gdp_per_capita` |
| Health System | `hospital_beds_per_thousand`, `life_expectancy`, `human_development_index` |
| Excess Mortality | `excess_mortality_cumulative_absolute`, `excess_mortality` |

---

## Phases

---

### Phase 1 — Data Profiling & Schema Design

**Goal:** Fully understand the source data and produce a signed-off DWH schema before any SQL is written.

#### Tasks

- [ ] **1. Profile the CSV**
  - [ ] Count total rows, distinct countries, date range, and continents.
  - [ ] Identify nullable columns and data types for each field.
  - [ ] Identify all OWID aggregate rows (e.g. `location = 'World'`, `'High income'`, etc.) that must be handled separately.
  - [ ] Document columns to be excluded from the DWH (low-fill-rate or out-of-scope).

- [ ] **2. Design Dimension Tables**

  | Table | Grain | Key Columns |
  |---|---|---|
  | `dim_location` | One row per country/region | `location_id` (surrogate), `iso_code`, `location`, `continent`, `population`, `median_age`, `aged_65_older`, `gdp_per_capita`, `hospital_beds_per_thousand`, `life_expectancy`, `human_development_index` |
  | `dim_date` | One row per calendar date | `date_id` (surrogate), `date`, `year`, `month`, `month_name`, `quarter`, `week_number`, `day_of_week` |

  - [ ] Define data types and nullability for all `dim_location` columns.
  - [ ] Define data types and nullability for all `dim_date` columns.
  - [ ] Define surrogate key strategy (`IDENTITY`) for both dimensions.
  - [ ] Define unique constraints for both dimensions.

- [ ] **3. Design Fact Table**

  | Table | Grain | Key Columns |
  |---|---|---|
  | `fact_covid_daily` | One row per location per date | `fact_id`, `location_id` (FK), `date_id` (FK), `total_cases`, `new_cases`, `total_deaths`, `new_deaths`, `total_vaccinations`, `people_vaccinated`, `people_fully_vaccinated`, `total_boosters`, `icu_patients`, `hosp_patients`, `total_tests`, `new_tests`, `positive_rate`, `stringency_index`, `reproduction_rate`, `excess_mortality` |

  - [ ] Define data types (`DECIMAL(18,4)` vs `BIGINT`) for every measure column.
  - [ ] Define FK relationships to `dim_location` and `dim_date`.
  - [ ] Define index strategy (clustered PK, non-clustered on FK columns).

- [ ] **4. Design Results & Validation Tables**

  | Table | Purpose |
  |---|---|
  | `results_analytical` | Stores the answer to each of the 10 questions |
  | `results_validation` | Stores test case ID, description, expected value, actual value, status (PASS/FAIL), run timestamp |

  - [ ] Define columns for `rpt.results_analytical` (`question_id`, `question_text`, `answer_value`, `answer_label`, `run_timestamp`).
  - [ ] Define columns for `rpt.results_validation` (`test_id`, `test_description`, `expected_value`, `actual_value`, `status`, `run_timestamp`).

- [ ] **5. Produce ERD** (entity-relationship diagram as a document or draw.io file).
  - [ ] Include all 6 tables with columns, data types, PKs, FKs, and constraints.
  - [ ] Review and sign off ERD before moving to Phase 2.

#### Exit Criteria — Phase 1
- [ ] CSV profiling report completed (row count, date range, null percentages per column).
- [ ] All OWID aggregate/non-country rows identified and documented (World, continents, income bands).
- [ ] `dim_location`, `dim_date`, `fact_covid_daily`, `results_analytical`, `results_validation` schemas finalised with data types and constraints.
- [ ] ERD reviewed and approved.
- [ ] Columns to be excluded from the DWH are listed with justification.

---

### Phase 2 — DWH Build in SSMS

**Goal:** Create the database, schemas, and all tables in SQL Server with correct constraints, indexes, and documentation.

#### Tasks

- [ ] **1. Create Database & Schemas**
  - [ ] Create `Covid19DWH` database.
  - [ ] Create schema `stg` (staging).
  - [ ] Create schema `dbo` (dimensions + fact).
  - [ ] Create schema `rpt` (results + validation).
  - [ ] Save script as `/sql/ddl/01_create_database_and_schemas.sql`.

- [ ] **2. Create Staging Table**
  - [ ] Create `stg.covid_raw` — flat table mirroring all CSV columns as `NVARCHAR` for safe bulk load.
  - [ ] Save script as `/sql/ddl/02_create_stg_covid_raw.sql`.

- [ ] **3. Create Dimension Tables**
  - [ ] Create `dbo.dim_location` with `IDENTITY` surrogate key and unique constraint on `iso_code + location`.
  - [ ] Create `dbo.dim_date` with `IDENTITY` surrogate key and unique constraint on `date`.
  - [ ] Write date generation script to pre-populate `dim_date` from `2020-01-01` to dataset end date.
  - [ ] Run date generation script and verify row count.
  - [ ] Save scripts as `/sql/ddl/03_create_dim_location.sql` and `/sql/ddl/04_create_dim_date.sql`.

- [ ] **4. Create Fact Table**
  - [ ] Create `dbo.fact_covid_daily` with `IDENTITY` PK, FK constraints to both dimensions.
  - [ ] Add non-clustered indexes on `location_id` and `date_id`.
  - [ ] Save script as `/sql/ddl/05_create_fact_covid_daily.sql`.

- [ ] **5. Create Results & Validation Tables**
  - [ ] Create `rpt.results_analytical` with columns: `question_id`, `question_text`, `answer_value`, `answer_label`, `run_timestamp`.
  - [ ] Create `rpt.results_validation` with columns: `test_id`, `test_description`, `expected_value`, `actual_value`, `status`, `run_timestamp`.
  - [ ] Create `rpt.etl_log` with columns: `run_id`, `run_timestamp`, `status`, `staging_rows`, `fact_rows`, `dim_location_rows`, `notes`.
  - [ ] Save script as `/sql/ddl/06_create_rpt_tables.sql`.

- [ ] **6. Write & Verify DDL Scripts**
  - [ ] Ensure all scripts are numbered and re-runnable in order on a clean database.
  - [ ] Test full teardown and rebuild (DROP + re-run all scripts) once to confirm.

#### Exit Criteria — Phase 2
- [ ] `Covid19DWH` database exists in SSMS with schemas `stg`, `dbo`, `rpt`.
- [ ] All 6 tables created with correct column names, data types, nullability, and constraints.
- [ ] `dim_date` pre-populated covering the full dataset date range.
- [ ] FK relationships enforced between fact and both dimensions.
- [ ] All DDL saved as `.sql` scripts (re-runnable from scratch).
- [ ] Scripts execute without errors on a clean database.

---

### Phase 3 — ETL via SSIS

**Goal:** Build an SSIS package that reliably loads the CSV into staging, then transforms and moves data into dimension and fact tables.

#### Package Structure — `Covid19_ETL.dtsx`

- [ ] **CF_01 — Truncate Staging**
  - [ ] Add Execute SQL Task: `TRUNCATE TABLE stg.covid_raw`.
  - [ ] Test in isolation; confirm table is empty after execution.

- [ ] **CF_02 — Load Staging**
  - [ ] Add Data Flow Task with Flat File Source pointing to `owid-covid-data.csv`.
  - [ ] Map all CSV columns to `stg.covid_raw` (all as `NVARCHAR`).
  - [ ] Configure error output to redirect bad rows to a flat file log.
  - [ ] Test load; verify row count in `stg.covid_raw` matches CSV row count minus header.

- [ ] **CF_03 — Load `dim_location`**
  - [ ] Add Execute SQL Task: upsert distinct non-OWID locations from staging into `dbo.dim_location` using `MERGE`.
  - [ ] Test; verify distinct location count.

- [ ] **CF_04 — Load `dim_date`**
  - [ ] Add Execute SQL Task: gap-fill any dates from staging not already in `dbo.dim_date`.
  - [ ] Test; verify no missing dates after load.

- [ ] **CF_05 — Load Fact Table**
  - [ ] Add Execute SQL Task joining staging → both dims.
  - [ ] Cast all numeric columns from `NVARCHAR` to `DECIMAL(18,4)` or `BIGINT`.
  - [ ] Replace empty string `''` with `NULL` for all numeric columns.
  - [ ] Exclude OWID aggregate rows (`iso_code LIKE 'OWID_%'`) — route them to `stg.covid_aggregates`.
  - [ ] Apply dedup guard using `WHERE NOT EXISTS` or `MERGE` on `(location_id, date_id)`.
  - [ ] Test; verify fact row count and spot-check a sample country.

- [ ] **CF_06 — Log Run**
  - [ ] Add Execute SQL Task: insert row into `rpt.etl_log` with run timestamp, row counts, and status.
  - [ ] Test; verify `rpt.etl_log` has one new row per run.

- [ ] **Transformation Rules Verification**
  - [ ] Confirm no negative `total_cases` or `total_deaths` after load.
  - [ ] Confirm `stg.covid_aggregates` contains OWID aggregate rows and fact does not.
  - [ ] Confirm all numeric NULLs are `NULL` (not `0` or empty string) in fact.

- [ ] **End-to-End Package Test**
  - [ ] Run full package once on clean DB; confirm all 6 tasks succeed.
  - [ ] Run package a second time; confirm no duplicate rows added to fact (idempotency check).
  - [ ] Save package to `/ssis/Covid19_ETL.dtsx`.

#### Exit Criteria — Phase 3
- [ ] SSIS package executes end-to-end without errors or warnings on a full CSV load.
- [ ] `stg.covid_raw` row count matches CSV row count (minus header).
- [ ] `dbo.dim_location` contains the correct number of distinct countries/regions (non-OWID rows only).
- [ ] `dbo.fact_covid_daily` row count matches staging row count for non-aggregate rows.
- [ ] Re-running the package produces no duplicate rows in the fact table.
- [ ] OWID aggregate rows are isolated in `stg.covid_aggregates`.
- [ ] `rpt.etl_log` records each run with timestamp and row counts.
- [ ] Package is saved and runnable from SSIS Catalog or file system.

---

### Phase 4 — Analytical Query Layer

**Goal:** Write T-SQL stored procedures that answer all 10 questions and persist results into `rpt.results_analytical`.

#### Query Map

| Question | Approach |
|---|---|
| Q1 — Total worldwide cases | `MAX(total_cases)` for `location = 'World'` in staging aggregates, or `SUM` of latest country snapshots from fact |
| Q2 — Total worldwide deaths | Same pattern as Q1 using `total_deaths` |
| Q3 — Country with most cases | `TOP 1` by `MAX(total_cases)` grouped by `location`, excluding aggregates |
| Q4 — Country with fewest cases | `TOP 1` by `MAX(total_cases)` grouped by `location`, excluding zero-data rows |
| Q5 — Number of countries in dataset | `COUNT(DISTINCT location_id)` in `dim_location` |
| Q6 — Deadliest year | `SUM(new_deaths)` grouped by `year` via `dim_date` join |
| Q7 — Continent with most deaths | `SUM(new_deaths)` grouped by `continent` via `dim_location` join |
| Q8 — People fully vaccinated globally | `MAX(people_fully_vaccinated)` from OWID `World` aggregate row |
| Q9 — Country that started vaccinating first | `MIN(date)` where `new_vaccinations > 0`, grouped by `location` |
| Q10 — Highest single-day new case count | `TOP 1` by `new_cases` across entire fact table |

#### Tasks

- [ ] **Write individual stored procedures (Q01–Q10)**
  - [ ] `rpt.usp_Q01` — Total worldwide cases.
  - [ ] `rpt.usp_Q02` — Total worldwide deaths.
  - [ ] `rpt.usp_Q03` — Country with most cases.
  - [ ] `rpt.usp_Q04` — Country with fewest cases.
  - [ ] `rpt.usp_Q05` — Number of countries in dataset.
  - [ ] `rpt.usp_Q06` — Deadliest year.
  - [ ] `rpt.usp_Q07` — Continent with most deaths.
  - [ ] `rpt.usp_Q08` — People fully vaccinated globally.
  - [ ] `rpt.usp_Q09` — Country that started vaccinating first.
  - [ ] `rpt.usp_Q10` — Highest single-day new case count.

- [ ] **Write master stored procedure**
  - [ ] `rpt.usp_Run_All_Questions` — calls Q01–Q10 in sequence and upserts each result into `rpt.results_analytical`.

- [ ] **Cross-verify results**
  - [ ] Spot-check Q1 (total cases) against OWID published summary.
  - [ ] Spot-check Q2 (total deaths) against OWID published summary.
  - [ ] Spot-check Q3 (most cases country) against OWID dashboard.
  - [ ] Spot-check Q6 (deadliest year) against known pandemic timeline.

- [ ] **Save all procs** under `/sql/procs/rpt.usp_Q*.sql` and `/sql/procs/rpt.usp_Run_All_Questions.sql`.

#### Exit Criteria — Phase 4
- [ ] All 10 stored procedures created and execute without errors.
- [ ] `rpt.results_analytical` contains one row per question after running `usp_Run_All_Questions`.
- [ ] Results are manually cross-verified against OWID's published summary statistics (spot-check at least Q1, Q2, Q3, Q6).
- [ ] `run_timestamp` is populated so results are traceable to a specific ETL run.

---

### Phase 5 — Testing & Validation

**Goal:** Systematically verify data quality, ETL correctness, and analytical accuracy, with all outcomes persisted to `rpt.results_validation`.

#### Tasks

- [ ] **T1 — Row Count Validation**
  - [ ] Test: Staging row count = CSV row count (minus header).
  - [ ] Test: Fact row count = staging row count for non-OWID-aggregate rows.
  - [ ] Test: `dim_location` distinct count matches expected country count.

- [ ] **T2 — Referential Integrity**
  - [ ] Test: No orphaned `location_id` in fact (every FK resolves to `dim_location`).
  - [ ] Test: No orphaned `date_id` in fact (every FK resolves to `dim_date`).

- [ ] **T3 — Null / Data Quality**
  - [ ] Test: `total_cases` is never negative in fact.
  - [ ] Test: `new_cases` for a given date ≤ `total_cases` for that date.
  - [ ] Test: Date range in fact spans `2020-01-01` to the expected end date.
  - [ ] Test: No duplicate `(location_id, date_id)` pairs in fact.

- [ ] **T4 — Analytical Sanity Checks**
  - [ ] Test: Q1 worldwide total cases > 600 million (known lower bound).
  - [ ] Test: Q2 worldwide total deaths > 6 million (known lower bound).
  - [ ] Test: Q3 most-cases country is United States or China.
  - [ ] Test: Q6 deadliest year is 2021 or 2022.
  - [ ] Test: Q9 first vaccination country is United Kingdom or USA (December 2020).

- [ ] **T5 — ETL Idempotency**
  - [ ] Test: Run SSIS package twice; fact table row count does not increase on second run.

- [ ] **Write `rpt.usp_Run_All_Tests`**
  - [ ] Implement all T1–T5 test cases as parameterised checks inside the proc.
  - [ ] Each test evaluates expected vs actual and writes `PASS` or `FAIL` to `rpt.results_validation`.
  - [ ] Save as `/sql/procs/rpt.usp_Run_All_Tests.sql`.

- [ ] **Run all tests and review results**
  - [ ] Execute `rpt.usp_Run_All_Tests` on clean load.
  - [ ] Confirm all rows in `rpt.results_validation` show `status = 'PASS'`.
  - [ ] Document root cause and resolution for any `FAIL` result.
  - [ ] Export final validation result set to CSV or screenshot from SSMS.

#### Exit Criteria — Phase 5
- [ ] All T1–T5 test cases implemented in `usp_Run_All_Tests`.
- [ ] All tests PASS on a clean load.
- [ ] `rpt.results_validation` contains one row per test case with `status`, `expected_value`, `actual_value`, and `run_timestamp`.
- [ ] Any FAIL result has a documented root cause and resolution.
- [ ] Idempotency test (T5) confirmed PASS.
- [ ] Final validation report exported (query result to CSV or printed from SSMS).

---

## Deliverables Summary

| Deliverable | Location |
|---|---|
| Project Plan | `PROJECT_PLAN.md` |
| DDL Scripts | `/sql/ddl/*.sql` |
| ETL SSIS Package | `/ssis/Covid19_ETL.dtsx` |
| Analytical Stored Procs | `/sql/procs/rpt.usp_Q*.sql` |
| Validation Stored Proc | `/sql/procs/rpt.usp_Run_All_Tests.sql` |
| Results (live in DB) | `rpt.results_analytical`, `rpt.results_validation` |

---

## Definition of Done

- [ ] SSIS package loads the full CSV end-to-end without errors.
- [ ] All 10 questions have answers stored in `rpt.results_analytical`.
- [ ] All validation tests in `rpt.results_validation` show status = `PASS`.
- [ ] Re-running the pipeline from scratch (truncate → reload) produces identical results.
- [ ] All SQL scripts are saved and re-runnable in order on a clean `Covid19DWH` database.
