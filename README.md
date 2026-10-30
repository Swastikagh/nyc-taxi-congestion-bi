# NYC Taxi and Ride-hail: What Changed After Congestion Pricing?

**Python (DuckDB) + Oracle SQL + Power BI** | NYC TLC trip records | Feb 2024 vs Feb 2025

## Business question

New York's congestion fee began in January 2025. Using the city's public trip records, how did trip volume, market share, fares and fees change between Feb 2024 (before) and Feb 2025 (after), for yellow taxis, Uber and Lyft?

The pilot compares the same month a year apart, so seasonality is not mixed in.

## Key findings (trips per day)

Feb 2024 has 29 days and Feb 2025 has 28, so all comparisons use daily averages.

| Service | Feb 2024 | Feb 2025 | Change |
|---|---|---|---|
| Uber | 496,496 | 514,846 | +3.7% |
| Lyft | 170,657 | 175,699 | +3.0% |
| Yellow taxi | 98,829 | 116,381 | +17.8% |
| **All three** | **765,982** | **806,927** | **+5.3%** |

- **Yellow's growth is almost entirely one payment category.** Flex Fare trips went from about 5,100 to about 22,400 per day (+337%). Excluding them, yellow was flat: 93,693 to 93,936 trips per day (+0.3%). Flex Fare accounts for about 99% of yellow's increase. Cash trips fell 16.9% and credit-card trips rose 3.2%. I have not verified why Flex Fare jumped; it may reflect a change in how trips are recorded, not new demand.
- **Congestion fees collected in Feb 2025 were about $12.0M** across these three services (about $430K per day): Uber $7.55M, Lyft $2.69M, yellow $1.81M.
- **Pickups outside Manhattan grew faster than Manhattan** (Manhattan +3.2%; Brooklyn +6.6%, Queens +8.1%, Bronx +7.2%, Staten Island +8.8%). Borough is not the congestion relief zone, so this is a pattern to examine, not proof of an effect of the fee.
- **Weekday mornings (5-9 a.m.) are up roughly 7-9%.**

This is a two-month comparison. It shows what changed, not why: weather, fares, economic conditions and the Flex Fare reporting change are not controlled.

## Architecture

```
TLC Parquet files --> Python + DuckDB --> small CSVs --> Oracle SQL (APEX) --> Power BI
 45.3M rows           clean, aggregate     star schema     validate, analyse     dashboard
                      log exclusions                       answer key for DAX
```

1. **Prepare (Colab or local):** `notebooks/01_data_prep.ipynb` downloads the files, applies the cleaning rules in SQL (DuckDB) and writes facts, dimensions, a raw 100K-trip sample and a data-quality log. Oracle's free workspace cannot hold 45M rows, so aggregation happens here.
2. **Load and validate (Oracle):** `sql/01` creates the star schema; `sql/03` re-applies the rules to the raw sample in SQL and reconciles every fact table.
3. **Analyse (Oracle):** `sql/04` answers the business questions and doubles as the answer key for the dashboard.
4. **Report (Power BI):** `powerbi/BUILD_GUIDE.md` has the model, DAX measures (`measures.dax`) and page designs.

## Data model

| Table | Grain | Purpose |
|---|---|---|
| `fact_daily_trips` | date x service x pickup zone | volume, fares, distance, congestion fee |
| `fact_hourly_trips` | date x hour x service | time-of-day profile |
| `fact_daily_payment` | date x service x payment type | the Flex Fare analysis |
| `dim_date`, `dim_zone`, `dim_services` | | calendar (with before/after flag), zones and boroughs, service |
| `trips_sample` | raw yellow trip | 100K uncleaned trips for SQL cleaning practice |
| `data_quality_log` | file x service | rows read and excluded per rule |

## Data quality rules

A trip is kept only if: pickup is inside the file's month; pickup zone is present; payment type is not 3, 4 or 6 (no charge, dispute, voided); fare is positive, except Flex Fare trips with a zero fare; distance is greater than 0; duration is between 0 and 360 minutes.

Why Flex Fare is treated differently: in the 2025 yellow file, 15.8% of Flex Fare rows had a fare of 0 but averaged about 16 miles with a total of about $4, which looks like an unpopulated fare field, not a missing trip. My first rule (fare > 0 for every trip) removed about 128K real trips from 2025 and distorted the comparison, so I changed it and added `fare_trips` so average fares use only trips with a recorded fare.

Result: **45.3M rows read, 44.8M kept (98.9%)**, every exclusion counted in `data_quality_log`.


## Dashboard preview

### Page 1: Overview & Market Share
![Overview]("C:\Users\sharm\OneDrive\Desktop\NYC-TAXI PROJ\screenshots\Screenshot (54).png")

### Page 2: Yellow Taxi & Flex Fare Analysis
![Yellow & Flex Fare]("C:\Users\sharm\OneDrive\Desktop\NYC-TAXI PROJ\screenshots\Screenshot (55).png")

### Page 3: Congestion Fee Collections
![Congestion Fee]("C:\Users\sharm\OneDrive\Desktop\NYC-TAXI PROJ\screenshots\Screenshot (56).png")

### Page 4: Geographic & Hourly Demand
![Where & When]("C:\Users\sharm\OneDrive\Desktop\NYC-TAXI PROJ\screenshots\Screenshot (57).png")

### Page 5: Data Quality & Reconciliation
![Data Quality]("C:\Users\sharm\OneDrive\Desktop\NYC-TAXI PROJ\screenshots\Screenshot (58).png")

## Validation

- Daily, hourly, payment and quality-log totals reconcile exactly (44,807,411 valid trips), and `sql/03` checks every date and service.
- Power BI measures must reproduce the SQL results in `sql/04` (table in `powerbi/BUILD_GUIDE.md`).

## Limitations

- Two months only; before/after is not causal.
- Yellow taxi tips are recorded for credit-card payments only; Uber/Lyft have no payment-type field.
- No cap on speed, fare or distance beyond the rules above.
- - **Yellow taxi distance and speed distortion resolved:** Diagnostic query (`sql/05_speed_diagnostic.sql`) revealed that standard yellow trips were consistent year-over-year (12.5 mph in 2024 vs. 12.2 mph in 2025). However, 2025 Flex Fare records exhibited corrupted distance telematics, averaging 35.2 miles in 15.7 minutes (~134 mph). Because Flex Fare distance reporting in 2025 is unreliable, Page 6 (Speed & Trip Economics) was excluded to preserve analytical integrity.
- The fee split by borough, the airport query and the screenshots below are still to be added.

## Reproduce

1. Open `notebooks/01_data_prep.ipynb` in Google Colab (or locally; `pip install -r requirements.txt`) and run all cells.
2. In Oracle APEX, run `sql/01_create_tables.sql`, then load the CSVs from `data/processed/` in this order: `dim_zone`, `dim_date`, the fact tables, `trips_sample`, `data_quality_log`. See `docs/oracle_load_guide.md`.
3. Run `sql/03` and `sql/04`, then follow `powerbi/BUILD_GUIDE.md`.

## Repository

```
notebooks/01_data_prep.ipynb    download, clean, aggregate (DuckDB)
sql/01-05                       schema, quality checks, analysis queries, speed diagnostic
powerbi/                        build guide, DAX measures, theme
docs/                           data dictionary, Oracle load guide, resume bullets
data/processed/                 CSVs for Oracle and Power BI (raw TLC files are not committed)
screenshots/                    dashboard pages (add after the build)
```

Data: NYC Taxi & Limousine Commission trip records (public). Dashboard screenshots: _add after building the Power BI pages_.
