# DataForge Covid-19 — DWH Rough Diagram

## Star Schema (Fact + Dims)

```
                    ┌──────────────────┐
                    │   dim_date       │
                    │──────────────────│
                    │ PK date_key (INT)│
                    │ full_date        │
                    │ year             │
                    │ quarter          │
                    │ month            │
                    │ day_of_week      │
                    │ is_weekend       │
                    └────────┬─────────┘
                          1  │ FK: date_key (Many-to-One)
                             │ one dim_date row → many fact rows
                             │ ∞
┌─────────────────────┐      │      ┌────────────────────────────────┐
│   dim_location      │      │      │      fact_covid_daily           │
│─────────────────────│      │      │────────────────────────────────│
│ PK location_key     │──────┴──────│ PK  fact_id                    │
│ iso_code            │  1          │ FK  date_key    → dim_date  N:1│
│ location (country)  │  FK:        │ FK  location_key→ dim_loc   N:1│
│ continent           │  location   │ UNIQUE(date_key, location_key) │
│ population          │  _key       │ = 1 row per country × day      │
│ gdp_per_capita      │  Many-to-One│────────────────────────────────│
│ median_age          │  ∞          │ CASES  : new_cases             │
│ life_expectancy     │  one country│         total_cases            │
│ hospital_beds_...   │  → many fact│         new_cases_smoothed     │
│ (+ other static     │  rows       │ DEATHS : new_deaths            │
│   country attrs)    │             │         total_deaths           │
└─────────────────────┘             │         new_deaths_smoothed    │
                                    │ TESTS  : new_tests             │
                                    │         total_tests            │
                                    │         positive_rate          │
                                    │ VACC   : new_vaccinations      │
                                    │         total_vaccinations     │
                                    │         people_vaccinated      │
                                    │         people_fully_vaccinated│
                                    │         total_boosters         │
                                    │ OTHER  : icu_patients          │
                                    │         hosp_patients          │
                                    │         stringency_index       │
                                    └────────────────────────────────┘
```


