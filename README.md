## `glue_crawler`

### Setup
1. Create an Amazon Web Services connection named `amazon_default`
1. Create an Airflow Variable named `s3_bucket` set to your student s3 bucket. 


### Airflow DAG (`raw.py`)

- Runs daily over a fixed historical window (Jan 1–2, 2026) with `catchup=True`, processing one date at a time and capping concurrency to a single active task and run.
- For each scheduled date, walks `s3://<bucket>/landing/<table>/ingested_date=<ds>/` to discover which raw tables have new partitions for that day.
- For every table with data, builds an S3 target at the **table directory level** (`landing/<table>/`) so the crawler treats `ingested_date=...` as a partition column rather than a separate table.
- Short-circuits the rest of the DAG if no landing data exists for the run's date — downstream tasks are skipped instead of running on an empty target list.
- Upserts a single Glue Crawler named `raw` against the discovered targets:
  - **`UpdateBehavior: UPDATE_IN_DATABASE`** so new columns (schema drift) automatically propagate into the Glue catalog.
  - **`DeleteBehavior: DEPRECATE_IN_DATABASE`** so removed source tables are flagged but not destroyed.
  - **`RecrawlBehavior: CRAWL_EVERYTHING`** to ensure existing partitions are re-classified when schemas change.
- Starts the crawler and waits synchronously for it to finish so downstream consumers see updated catalog state by the time the DAG run completes.

**Net effect**
Each daily run keeps the `raw` Glue database in sync with the landing layer: new tables are registered, new partitions are added, and schema drift is reflected automatically — making everything immediately queryable from Athena and consumable by downstream Iceberg pipelines.