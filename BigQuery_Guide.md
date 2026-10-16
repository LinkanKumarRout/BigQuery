# BigQuery: a beginner guide

BigQuery is Google Cloud’s warehouse for large amounts of data. You ask it questions in SQL, and it reads huge tables quickly. You do not install it on a server, create indexes the way you do in PostgreSQL, or keep it running overnight. Google runs the machines. You pay mainly for the data each query reads, and for storage.

This guide explains what BigQuery is for, how it relates to a PostgreSQL database (including AlloyDB, which speaks PostgreSQL), and the everyday operations: create a place for tables, load data, query, schedule work, and control cost.

In this project (SAF), PostgreSQL-compatible **AlloyDB** holds the live operational data (forms, users, calculations). **BigQuery** holds history for dashboards. Looker Studio reads BigQuery, so heavy history questions do not slow down the live database.

---

## 1. A simple picture

Think of two rooms:

| Room | Product in SAF | What it is good at |
|------|----------------|--------------------|
| The shop counter | AlloyDB (PostgreSQL) | Saving one order, updating one row, logging a user in. Fast for small, frequent writes. |
| The archive and research room | BigQuery | “How did power use change over two years?” Fast for big reads across millions of rows. |

People call the shop counter **OLTP** (online transaction processing) and the archive **OLAP** (online analytical processing). You keep both. You copy data from the shop into the archive. You rarely run the yearly report directly on the shop database.

```text
Plant / app writes
        |
        v
PostgreSQL or AlloyDB          <-- live system of record
        |
        |  copy changes (Datastream) or ask live (federated query)
        v
BigQuery                       <-- history, big reports, Looker Studio
```

---

## 2. Words you will see everywhere

| Word | Plain meaning |
|------|----------------|
| **Project** | Your Google Cloud account folder. Billing and permissions live here. |
| **Dataset** | A folder of tables inside one region, like a PostgreSQL schema. Example: `saf_raw`. |
| **Table** | Rows and columns, same idea as PostgreSQL. |
| **View** | A saved `SELECT`. It does not store rows. Each use runs the query again. |
| **Materialized view** | A saved query whose result Google keeps updated for you. |
| **Job** | One piece of work: a query, a load, or an export. Every question you ask is a job. |
| **Partition** | The table is split by a value, usually a date. A query for one day can skip the other days. |
| **Cluster** | Inside a partition, rows with the same plant or line sit near each other, so filters on those columns read less data. |
| **Slot** | A unit of computing power BigQuery uses while a query runs. |
| **Location** | The region where the dataset lives, such as `us-central1` or `asia-south1`. A connection and a dataset must share a location when you query PostgreSQL through BigQuery. |

---

## 3. What people use BigQuery for

- **History.** Years of telemetry, daily KPIs, form submissions, and lab results, kept after the live database only needs recent rows.
- **Dashboards.** Looker Studio, Connected Sheets, and similar tools run SQL against BigQuery.
- **Ad-hoc questions.** An analyst writes SQL for a date range without adding load to the app database.
- **Combining sources.** One query can join application tables with a CSV in Cloud Storage, a Google Sheet, or another dataset.
- **Scheduled numbers.** Every night, a saved query builds a daily summary table.
- **Machine learning (later).** BigQuery ML can train simple models with SQL. You can ignore this until the warehouse itself is in place.

BigQuery is a weak place to run the application itself. It is not built for “update this one user’s password” thousands of times per minute. Keep that in PostgreSQL.

---

## 4. How BigQuery is different from PostgreSQL

The SQL looks familiar. A few habits change:

| Habit in PostgreSQL | Habit in BigQuery |
|---------------------|-------------------|
| You design indexes so a query is fast. | You partition and cluster so a query **reads fewer bytes**. Speed follows from that. |
| `UPDATE` and `DELETE` are normal all day. | They work, but they are for corrections and rebuilds. Streaming new facts in is the usual pattern. |
| `SERIAL` / `IDENTITY` creates ids. | You usually generate the id in the app, or use `GENERATE_UUID()`. |
| Tables live in a **schema**. | Tables live in a **dataset**. You write `` `project.dataset.table` ``. |
| You can `JOIN` freely. The cost is CPU on your server. | Every column you touch can add **bytes billed**. Select only the columns you need. |
| `information_schema` is in the database. | BigQuery has `INFORMATION_SCHEMA` too, plus a job history in the console. |

A first query:

```sql
SELECT
  business_date,
  plant_code,
  power_kwh
FROM `my_project.saf_curated.daily_power`
WHERE business_date BETWEEN '2026-10-01' AND '2026-10-15'
ORDER BY business_date;
```

The backticks are required around `project.dataset.table`.

---

## 5. Connecting BigQuery to PostgreSQL

“Attach” can mean three different jobs. Pick the one that matches the question you are asking.

| Goal | Method | When to use it |
|------|--------|----------------|
| Run a SQL question in BigQuery that reads PostgreSQL **right now**, without copying the table | **Federated query** (`EXTERNAL_QUERY`) | Cloud SQL for PostgreSQL. Small lookups, a few joins, checking a live row. |
| Keep a BigQuery copy that follows inserts, updates, and deletes | **Datastream** (change data capture) | The normal SAF path: AlloyDB history into `saf_raw`. |
| Copy a file once, or on a schedule | **Load from Cloud Storage** | Backfill, CSV exports, nightly snapshots. |

AlloyDB is PostgreSQL-compatible, and SAF uses it as the live database. The supported history path in this project is **Datastream from an AlloyDB read replica into BigQuery**. Federated query below is the Cloud SQL for PostgreSQL feature. Use that when the instance really is Cloud SQL.

### 5.1 One-time setup (Google Cloud)

You need:

1. A Google Cloud project with billing.
2. These APIs enabled: BigQuery, and either Cloud SQL Admin (for a live attach) or Datastream (for continuous copy).
3. The `gcloud` CLI, logged in:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud services enable bigquery.googleapis.com
```

4. A PostgreSQL user used only by the pipeline. Grant it `SELECT` on the tables BigQuery should see. Do not use the application superuser, and do not put the password in git.

### 5.2 Live attach: BigQuery reads Cloud SQL for PostgreSQL

This creates a **connection** in BigQuery. Queries can then call `EXTERNAL_QUERY(...)` and send a PostgreSQL statement to Cloud SQL. The rows come back as a BigQuery result. Nothing is stored unless you `CREATE TABLE ... AS SELECT`.

The dataset location and the Cloud SQL region must match. Example: both in `asia-south1`.

Create the connection (run this once):

```bash
bq mk --connection \
  --connection_type=CLOUD_SQL \
  --properties='{"instanceId":"YOUR_PROJECT_ID:asia-south1:YOUR_INSTANCE","database":"saf","type":"POSTGRES"}' \
  --connection_credential='{"username":"bq_reader","password":"CHOOSE_A_PASSWORD"}' \
  --project_id=YOUR_PROJECT_ID \
  --location=asia-south1 \
  saf_postgres_conn
```

Ask PostgreSQL a question from BigQuery:

```sql
SELECT *
FROM EXTERNAL_QUERY(
  'YOUR_PROJECT_ID.asia-south1.saf_postgres_conn',
  '''
  SELECT id, plant_code, business_date, power_kwh
  FROM public.daily_power
  WHERE business_date >= CURRENT_DATE - INTERVAL '7 days'
  '''
);
```

The inner string is **PostgreSQL** SQL. The outer statement is **BigQuery** SQL.

Save a live lookup as a view:

```sql
CREATE OR REPLACE VIEW `YOUR_PROJECT_ID.saf_raw.daily_power_live` AS
SELECT *
FROM EXTERNAL_QUERY(
  'YOUR_PROJECT_ID.asia-south1.saf_postgres_conn',
  '''SELECT id, plant_code, business_date, power_kwh FROM public.daily_power'''
);
```

Copy the rows into a real BigQuery table when you want a snapshot:

```sql
CREATE TABLE `YOUR_PROJECT_ID.saf_raw.daily_power_snapshot`
PARTITION BY business_date AS
SELECT *
FROM `YOUR_PROJECT_ID.saf_raw.daily_power_live`;
```

What to remember:

- Each call opens work on PostgreSQL. A dashboard that refreshes every minute can overload the live database. Prefer a copied table for heavy or repeated reads.
- PostgreSQL types land as BigQuery types (`timestamptz` becomes `TIMESTAMP`, `numeric` often becomes `NUMERIC` or `FLOAT`). Check a few rows after the first query.
- The password lives in the connection, not in the SQL text. Rotate it in the BigQuery connection settings when you rotate the database user.

### 5.3 Continuous copy: PostgreSQL or AlloyDB into BigQuery (Datastream)

This is the method to use when BigQuery should **keep its own tables** and stay up to date. SAF’s pipeline is: AlloyDB primary → read replica → Datastream → BigQuery dataset `saf_raw` → scheduled SQL or Dataform → `saf_curated` → Looker Studio.

Datastream reads the PostgreSQL write-ahead log (logical replication) and applies changes in BigQuery.

**On PostgreSQL / AlloyDB**

1. Allow logical replication. On AlloyDB and Cloud SQL this is a flag, often `cloudsql.logical_decoding=on`, plus `wal_level=logical` where you control it. Restart if the flag requires it.
2. Create a publication for the tables you want copied:

```sql
CREATE PUBLICATION saf_to_bigquery FOR TABLE
  public.daily_power,
  public.telemetry_reading;
```

3. Create a replication user:

```sql
CREATE USER datastream_user WITH REPLICATION LOGIN PASSWORD 'CHOOSE_A_PASSWORD';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO datastream_user;
GRANT USAGE ON SCHEMA public TO datastream_user;
```

**In Google Cloud**

1. Enable the API: `gcloud services enable datastream.googleapis.com`.
2. Open Datastream → **Create stream**.
3. Source: PostgreSQL or AlloyDB. Point it at the **read replica** when you have one, so the copy does not compete with application writes.
4. Connectivity: private networking (VPC peering or Private Service Connect). Avoid a public IP with a world-open firewall.
5. Destination: BigQuery. Choose a dataset such as `saf_raw` in the same region as the source.
6. Pick the tables. Start with a few, not the whole database. Leave passwords, session tokens, and user secrets out of the publication.
7. Start the stream. The first run **backfills** existing rows, then switches to ongoing changes.

After it is healthy, `saf_raw` tables gain BigQuery metadata columns Datastream maintains (a change timestamp and whether the row was deleted). Downstream models should keep the latest version of each primary key and ignore deleted rows.

**Turn landing tables into curated tables** with a scheduled query (see section 6.8) or with Dataform. Curated tables are what dashboards should read: clean names, one row per business key, partitioned by `business_date`.

### 5.4 Batch copy: export rows and load a file

Useful for a one-time backfill or a nightly full snapshot of a small table.

From PostgreSQL, write a CSV (or use `gcloud sql export` on Cloud SQL, which writes to Cloud Storage):

```bash
psql "host=127.0.0.1 dbname=saf user=saf" \
  -c "\copy (SELECT id, plant_code, business_date, power_kwh FROM public.daily_power) TO 'daily_power.csv' WITH CSV HEADER"
```

Upload and load:

```bash
gcloud storage cp daily_power.csv gs://YOUR_BUCKET/backfill/daily_power.csv

bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  --replace \
  YOUR_PROJECT_ID:saf_raw.daily_power \
  gs://YOUR_BUCKET/backfill/daily_power.csv \
  id:STRING,plant_code:STRING,business_date:DATE,power_kwh:FLOAT64
```

`--replace` overwrites the table. Drop that flag if you mean to append.

### 5.5 The other direction: BigQuery results back into PostgreSQL

Sometimes a nightly summary should land back in the app database so the API can serve it without calling BigQuery.

1. Export from BigQuery to Cloud Storage:

```sql
EXPORT DATA OPTIONS(
  uri='gs://YOUR_BUCKET/exports/daily_summary_*.csv',
  format='CSV',
  overwrite=true,
  header=true
) AS
SELECT business_date, plant_code, power_kwh
FROM `YOUR_PROJECT_ID.saf_curated.daily_power`
WHERE business_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY);
```

2. Load the CSV into PostgreSQL with `\copy`, or `gcloud sql import csv` for Cloud SQL.

Do this for **small summary tables**. Copying a full BigQuery warehouse back into PostgreSQL removes the reason you split the two systems.

---

## 6. Everyday operations

Replace `YOUR_PROJECT_ID` and region names before you run anything.

### 6.1 Create a dataset

A dataset is the container. Create one per layer so raw landing data and cleaned tables stay apart.

```bash
bq mk --dataset --location=asia-south1 YOUR_PROJECT_ID:saf_raw
bq mk --dataset --location=asia-south1 YOUR_PROJECT_ID:saf_curated
```

Suggested layers, matching SAF:

| Dataset | What goes in it |
|---------|-----------------|
| `saf_raw` | Exact copies from Datastream or files. Little renaming. |
| `saf_curated` | Clean fact and dimension tables for dashboards. |
| `saf_mart` | Optional extra-wide tables built for one report. |

### 6.2 Create a table

```sql
CREATE TABLE `YOUR_PROJECT_ID.saf_curated.daily_power` (
  business_date DATE NOT NULL,
  plant_code STRING NOT NULL,
  line_code STRING,
  power_kwh FLOAT64,
  loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY business_date
CLUSTER BY plant_code, line_code;
```

- **Partition by date** so “last 7 days” does not scan 2024.
- **Cluster** by the columns you filter on most often (plant, line).

### 6.3 Insert and query

```sql
INSERT INTO `YOUR_PROJECT_ID.saf_curated.daily_power`
  (business_date, plant_code, line_code, power_kwh)
VALUES
  (DATE '2026-10-15', 'P1', 'L1', 1520.5);

SELECT plant_code, SUM(power_kwh) AS power_kwh
FROM `YOUR_PROJECT_ID.saf_curated.daily_power`
WHERE business_date >= DATE '2026-10-01'
GROUP BY plant_code
ORDER BY power_kwh DESC;
```

Preview a table in the console, or:

```bash
bq query --use_legacy_sql=false \
  'SELECT * FROM `YOUR_PROJECT_ID.saf_curated.daily_power` LIMIT 20'
```

Always set a `LIMIT` while exploring. A `SELECT *` with no filter can scan the whole table and show up on the bill.

### 6.4 Load, append, and replace

| Need | What to do |
|------|------------|
| Add rows from a file | `bq load` without `--replace` |
| Rebuild the table from a file | `bq load --replace` |
| Add rows from a query | `INSERT INTO ... SELECT ...` |
| Rebuild from a query | `CREATE OR REPLACE TABLE ... AS SELECT ...` |

Partitioned loads can replace **one day** without touching the rest:

```sql
DELETE FROM `YOUR_PROJECT_ID.saf_curated.daily_power`
WHERE business_date = DATE '2026-10-15';

INSERT INTO `YOUR_PROJECT_ID.saf_curated.daily_power`
SELECT * FROM `YOUR_PROJECT_ID.saf_raw.daily_power_staging`
WHERE business_date = DATE '2026-10-15';
```

### 6.5 Views

A view is a name for a query. Use it to hide raw column names from dashboard authors.

```sql
CREATE OR REPLACE VIEW `YOUR_PROJECT_ID.saf_curated.v_daily_power` AS
SELECT
  business_date,
  plant_code,
  power_kwh
FROM `YOUR_PROJECT_ID.saf_curated.daily_power`
WHERE business_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY);
```

### 6.6 Check size, schema, and recent jobs

```sql
SELECT table_name, row_count, ROUND(size_bytes / POW(1024, 3), 2) AS size_gb
FROM `YOUR_PROJECT_ID.saf_curated.INFORMATION_SCHEMA.TABLE_STORAGE`
ORDER BY size_bytes DESC;
```

```bash
bq show --schema --format=prettyjson YOUR_PROJECT_ID:saf_curated.daily_power
bq ls -j -a -n 10 YOUR_PROJECT_ID
```

The console’s **Job history** shows bytes billed for each query. That number is the one to watch.

### 6.7 Dry-run a query before you pay for it

```bash
bq query --dry_run --use_legacy_sql=false \
  'SELECT * FROM `YOUR_PROJECT_ID.saf_curated.daily_power`'
```

The reply includes bytes that **would** be processed. If the number is huge, add a date filter or drop columns from the `SELECT`.

### 6.8 Schedule a query

In the BigQuery console: write the SQL, choose **Schedule**, pick a time, and set the destination table. A typical nightly job rebuilds yesterday’s curated rows after the plant’s business day has closed.

You can also deploy the same SQL with Dataform if many tables depend on each other. Dataform is a better fit once you have more than a handful of transforms. One scheduled query is enough to start.

### 6.9 Share access

BigQuery uses Google Cloud IAM. Typical roles:

| Who | Role | Why |
|-----|------|-----|
| Pipeline service account | `roles/bigquery.dataEditor` on `saf_raw` | Datastream and load jobs write landing tables. |
| Transform job | `roles/bigquery.dataEditor` on `saf_curated`, `dataViewer` on `saf_raw` | Read landing data, write clean tables. |
| Dashboard viewers | `roles/bigquery.dataViewer` on `saf_curated` only | They can read reports and cannot browse raw tables or secrets. |
| Analysts who write SQL | `roles/bigquery.user` plus `dataViewer` | `bigquery.user` lets them run jobs. `dataViewer` lets them read tables. |

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member='user:analyst@example.com' \
  --role='roles/bigquery.dataViewer'
```

Prefer binding viewer access on the **dataset**, not the whole project, so `saf_raw` stays limited to the pipeline.

### 6.10 Export and delete

Export was shown in section 5.5. To remove a table or a dataset:

```bash
bq rm -t YOUR_PROJECT_ID:saf_raw.daily_power_snapshot
bq rm -r -d YOUR_PROJECT_ID:saf_raw
```

`bq rm -r -d` deletes the dataset and every table in it. Read the name twice.

---

## 7. Using BigQuery from Python

The application can ask BigQuery a question when the API needs a long history. Install the client in the same environment as the app:

```bash
pip install google-cloud-bigquery
```

Authenticate locally once with `gcloud auth application-default login`. On Cloud Run, attach a service account that has `roles/bigquery.jobUser` and `roles/bigquery.dataViewer`. The key file should not sit in the repository.

```python
from google.cloud import bigquery

client = bigquery.Client(project="YOUR_PROJECT_ID")

query = """
SELECT business_date, plant_code, power_kwh
FROM `YOUR_PROJECT_ID.saf_curated.daily_power`
WHERE business_date BETWEEN @start AND @end
  AND plant_code = @plant
ORDER BY business_date
"""

job = client.query(
    query,
    job_config=bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("start", "DATE", "2026-10-01"),
            bigquery.ScalarQueryParameter("end", "DATE", "2026-10-15"),
            bigquery.ScalarQueryParameter("plant", "STRING", "P1"),
        ]
    ),
)

for row in job.result():
    print(row.business_date, row.plant_code, row.power_kwh)
```

Parameters (`@start`, `@plant`) keep values out of the SQL string. Filter on the partition column (`business_date`) so the job stays small.

---

## 8. Cost and safety habits

- **Filter on the partition column** in every dashboard query.
- **Name columns in `SELECT`.** `SELECT *` reads every column, including large strings you do not display.
- **Dry-run** a new query once.
- **Set a project quota** (BigQuery → Quotas, or a custom cost alert in Billing) so a mistaken query cannot run unbounded.
- **Keep secrets out of BigQuery.** Password hashes, session tokens, and API keys stay in the application database.
- **Point dashboards at `saf_curated`.** Raw landing tables change shape when the stream changes.
- **Read replicas for Datastream** so the copy does not sit on the primary that the app writes to.
- **A deleted BigQuery row is not a PostgreSQL rollback.** Fixing history means writing a new transform, not editing the live form.

---

## 9. A first path you can follow

1. Create `saf_raw` and `saf_curated` in one region.
2. Create a read-only PostgreSQL user.
3. If the database is **Cloud SQL for PostgreSQL** and you only need a live lookup, create the BigQuery connection and run one `EXTERNAL_QUERY`.
4. If you need history (the SAF case, AlloyDB), create a Datastream stream from the read replica into `saf_raw` for one small table.
5. Write one scheduled query that copies that table into a partitioned `saf_curated` table.
6. In the console, run a 7-day `SELECT` with a dry run, then for real.
7. Give dashboard users viewer access on `saf_curated` only.

When that one table looks right, add the next table to the publication. Expanding slowly makes type mismatches and missing keys obvious.

---

## 10. Where this sits in SAF

| Piece | Role |
|-------|------|
| AlloyDB primary | Application reads and writes. |
| AlloyDB read replica | Source for the copy into BigQuery. |
| Datastream | Near-real-time changes into `saf_raw`. |
| Scheduled SQL or Dataform | Builds `saf_curated`. |
| Looker Studio | Dashboards on `saf_curated` only. |
| Django analytics API | Serves the app’s own analytics screens from the application database. It does not replace the warehouse. |

More detail on that pipeline is in `docs/03_Architecture/SAF_System_Architecture.md` (section 4.9).
