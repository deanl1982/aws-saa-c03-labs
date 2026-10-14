# Lab 36 — Serverless Data Lake: Glue, Athena & Lake Formation

| | |
|---|---|
| **Module** | 10 — Data Ingestion, Analytics & Hybrid |
| **Exam mapping** | Domain 3 · Task 3.5 (data transformation, analytics) · Domain 4 · Task 4.1 (columnar formats, partitioning) · Domain 1 · Task 1.3 (fine-grained data access) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.20 (Glue crawler DPU-minutes; Athena at $5/TB scanned, so fractions of a cent here) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-35-kinesis.md) · [Back to index](../README.md) · [Next lab ➡](lab-37-hybrid-migration.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Catalogue S3 data automatically with a **Glue crawler** and the **Glue Data Catalog**.
- Query S3 in place with **Athena**, using **workgroups** to control cost.
- Cut query cost and time with **columnar formats (Parquet)**, **compression**, and **partitioning** (CTAS).
- Explain **Lake Formation** fine-grained permissions (database, table, column, row).
- Choose between **Athena, Redshift, EMR, OpenSearch, QuickSight**, and **Glue ETL**.

## Analytics service map

| Need | Service |
|---|---|
| Ad-hoc SQL on S3, serverless, pay per query | **Athena** |
| Data warehouse, complex joins, BI at scale, provisioned or serverless | **Redshift** (+ **Redshift Spectrum** to query S3) |
| Managed Hadoop/Spark for big data processing | **EMR** (on EC2, EKS, or Serverless) |
| Serverless ETL, data catalog, crawlers, DataBrew (no-code prep) | **AWS Glue** |
| Centralised data-lake permissions and governance | **Lake Formation** |
| Search and log analytics, dashboards | **OpenSearch Service** |
| BI dashboards | **QuickSight** |

---

## Part A — Console

### A1. Generate and upload a dataset

- [ ] Generate a sales CSV with 20,000 rows across 4 regions and 3 months:

```bash
B=saa-lab36-lake-$SUFFIX
aws s3 mb s3://$B
python3 - <<'EOF'
import csv, random, datetime
regions = ["us","eu","apac","latam"]
with open("sales.csv","w",newline="") as f:
    w = csv.writer(f); w.writerow(["order_id","order_date","region","product","qty","price","customer_email"])
    d0 = datetime.date(2026,7,1)
    for i in range(20000):
        d = d0 + datetime.timedelta(days=random.randint(0,91))
        w.writerow([i, d.isoformat(), random.choice(regions), f"prod-{random.randint(1,40)}",
                    random.randint(1,5), round(random.uniform(5,500),2), f"user{random.randint(1,3000)}@example.com"])
EOF
aws s3 cp sales.csv s3://$B/raw/sales/sales.csv
```

- [ ] Create a separate bucket for Athena results:

```bash
aws s3 mb s3://saa-lab36-athena-results-$SUFFIX
```

### A2. Glue crawler

- [ ] Open **AWS Glue → Data Catalog → Databases → Add database**: `saa_lake`.
- [ ] Open **Crawlers → Create crawler**: `saa-raw-crawler`.
  - Data source: S3 `s3://saa-lab36-lake-<suffix>/raw/`
  - IAM role: **Create new** `AWSGlueServiceRole-saa`
  - Target database `saa_lake`, schedule **On demand**
- [ ] **Run crawler** (about 1–2 minutes). Open **Tables → `sales`** and look at the inferred schema (`order_date` might be inferred as a string, which is fine).

### A3. Query with Athena

- [ ] Open **Athena → Query editor → Settings → Manage**, and set the result location to `s3://saa-lab36-athena-results-<suffix>/`.
- [ ] Run:

```sql
SELECT region, COUNT(*) AS orders, ROUND(SUM(qty*price),2) AS revenue
FROM saa_lake.sales
GROUP BY region ORDER BY revenue DESC;
```

- [ ] Note **Data scanned** under the results (about 1 MB).

### A4. Convert to partitioned Parquet with CTAS

- [ ] Run the CTAS:

```sql
CREATE TABLE saa_lake.sales_parquet
WITH (
  format = 'PARQUET',
  write_compression = 'SNAPPY',
  external_location = 's3://saa-lab36-lake-<suffix>/curated/sales_parquet/',
  partitioned_by = ARRAY['region']
) AS
SELECT order_id, CAST(order_date AS date) AS order_date, product, qty, price, customer_email, region
FROM saa_lake.sales;
```

- [ ] Re-run the revenue query against `sales_parquet` with `WHERE region = 'eu'`, and compare **Data scanned** for:
  - `SELECT * FROM sales WHERE region='eu'` (CSV, full scan)
  - `SELECT SUM(qty*price) FROM sales_parquet WHERE region='eu'` (Parquet + partition pruning + column pruning)

> Expect **10× or more less data scanned**. That's the exam's "reduce Athena cost" answer.

### A5. Workgroup with a data-scan limit

- [ ] Open **Athena → Workgroups → Create** `saa-analysts`: result location, **enforce workgroup settings**, **per-query data usage control** limit **10 MB**.
- [ ] Switch to it and run `SELECT * FROM saa_lake.sales` repeatedly with a CROSS JOIN to blow past 10 MB. The query is **cancelled**.

### A6. Lake Formation fine-grained access (guided)

- [ ] Open **Lake Formation → Get started** and add yourself as a **Data lake administrator**.
- [ ] Open **Data lake locations → Register** `s3://saa-lab36-lake-<suffix>/curated/`.
- [ ] Open **Data permissions → Grant** to an IAM role or user (e.g. `saa-ssm-ec2-role`, or a test user):
  - Database `saa_lake`, table `sales_parquet`
  - **Column-based access:** include `order_date, region, product, qty, price` (**exclude `customer_email`**)
  - Permissions: **Select**
- [ ] *(Optional)* Create a **data filter** with row-level filter `region = 'eu'`.

> Lake Formation centralises permissions for Athena, Redshift Spectrum, EMR, and Glue, so you don't have to manage separate S3 bucket policies per team.

---

## Part B — CLI

```bash
B=saa-lab36-lake-$SUFFIX
RES=s3://saa-lab36-athena-results-$SUFFIX/
```

### B1. Run the crawler and inspect the catalog

- [ ] Run the crawler and look at the tables:

```bash
aws glue start-crawler --name saa-raw-crawler
sleep 90
aws glue get-crawler --name saa-raw-crawler --query 'Crawler.{State:State,Last:LastCrawl.Status}'
aws glue get-tables --database-name saa_lake --query 'TableList[].{Name:Name,Format:StorageDescriptor.InputFormat,Location:StorageDescriptor.Location}' --output table
```

### B2. Run Athena queries from the CLI

- [ ] Define a helper that runs a query and reports bytes scanned:

```bash
run_q () {
  QID=$(aws athena start-query-execution --query-string "$1" \
    --query-execution-context Database=saa_lake --result-configuration OutputLocation=$RES \
    --query QueryExecutionId --output text)
  while true; do
    S=$(aws athena get-query-execution --query-execution-id $QID --query QueryExecution.Status.State --output text)
    case $S in QUEUED|RUNNING) sleep 2 ;; *) break ;; esac
  done
  aws athena get-query-execution --query-execution-id $QID \
    --query 'QueryExecution.{State:Status.State,BytesScanned:Statistics.DataScannedInBytes,ms:Statistics.EngineExecutionTimeInMillis}'
  aws athena get-query-results --query-execution-id $QID --max-items 5 --query 'ResultSet.Rows[].Data[].VarCharValue' --output text
}
```

- [ ] Compare CSV with partitioned Parquet:

```bash
run_q "SELECT COUNT(*) FROM sales WHERE region='eu'"
run_q "SELECT COUNT(*) FROM sales_parquet WHERE region='eu'"
```

### B3. List partitions

- [ ] Show them:

```bash
run_q "SHOW PARTITIONS sales_parquet"
aws s3 ls s3://$B/curated/sales_parquet/
```

### B4. Partition projection (no crawler needed for new partitions)

- [ ] Create a table where Athena computes partitions from a rule instead of looking them up:

```bash
run_q "CREATE EXTERNAL TABLE IF NOT EXISTS clicks_projected (user string, page string, ts bigint)
PARTITIONED BY (year string, month string, day string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://saa-lab35-lake-$SUFFIX/clicks/'
TBLPROPERTIES (
 'projection.enabled'='true',
 'projection.year.type'='integer','projection.year.range'='2025,2030',
 'projection.month.type'='integer','projection.month.range'='1,12','projection.month.digits'='2',
 'projection.day.type'='integer','projection.day.range'='1,31','projection.day.digits'='2',
 'storage.location.template'='s3://saa-lab35-lake-$SUFFIX/clicks/year=\${year}/month=\${month}/day=\${day}/')"
```

> If you kept the Lab 35 bucket, query it with `SELECT * FROM clicks_projected LIMIT 10`. Firehose output is GZIP JSON, which Athena reads natively.

### B5. Glue ETL job (concept, plus a minimal script)

- [ ] Write a minimal PySpark job that converts CSV to Parquet:

```bash
cat > etl.py <<'EOF'
import sys
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.utils import getResolvedOptions
args = getResolvedOptions(sys.argv, ["bucket"])
gc = GlueContext(SparkContext.getOrCreate())
df = gc.create_dynamic_frame.from_catalog(database="saa_lake", table_name="sales")
gc.write_dynamic_frame.from_options(frame=df, connection_type="s3",
    connection_options={"path": f"s3://{args['bucket']}/curated/glue_job/", "partitionKeys": ["region"]},
    format="parquet")
EOF
aws s3 cp etl.py s3://$B/scripts/etl.py
```

- [ ] Create the job, run it (it costs a few cents), and check its state:

```bash
aws glue create-job --name saa-csv-to-parquet --role AWSGlueServiceRole-saa \
  --command Name=glueetl,ScriptLocation=s3://$B/scripts/etl.py,PythonVersion=3 \
  --glue-version 5.0 --worker-type G.1X --number-of-workers 2 \
  --default-arguments "{\"--bucket\":\"$B\"}"
RUN=$(aws glue start-job-run --job-name saa-csv-to-parquet --query JobRunId --output text)
sleep 120
aws glue get-job-run --job-name saa-csv-to-parquet --run-id $RUN --query 'JobRun.{State:JobRunState,Secs:ExecutionTime}'
```

> The Glue role created by the console wizard may only have access to the crawler path. If the job fails with AccessDenied, add `s3:*` on `arn:aws:s3:::saa-lab36-lake-*` to `AWSGlueServiceRole-saa`.

---

## Verify

- [ ] The crawler created a `sales` table, and Athena queries run against it.
- [ ] The partitioned Parquet table scans far less data than the CSV.
- [ ] The workgroup scan limit cancelled an oversized query.

## Exam tips

- **"Run SQL on S3 data with no infrastructure, pay per query"** → **Athena**. **Reduce Athena cost** → **Parquet/ORC + compression + partitioning** (and partition projection).
- **"Discover schema of S3 data and make it queryable"** → **Glue crawler + Data Catalog**.
- **"Serverless ETL to transform data in S3"** → **Glue ETL jobs** (Spark). **No-code data prep** → **Glue DataBrew**.
- **"Column- or row-level security on the data lake across services"** → **Lake Formation**.
- **"Petabyte-scale warehouse with complex joins and BI"** → **Redshift**. **Query S3 from Redshift** → **Redshift Spectrum**.
- **"Big data with Spark/Hadoop and custom frameworks"** → **EMR**.
- **"Federated queries across RDS, DynamoDB, and on-prem"** → Athena **federated query** connectors.

## Challenge (optional)

- [ ] Build a **QuickSight** dashboard on the `sales_parquet` table (QuickSight has a free trial; cancel afterwards).

## Cleanup

- [ ] Delete the Glue job, crawler, database, and role:

```bash
aws glue delete-job --job-name saa-csv-to-parquet
aws glue delete-crawler --name saa-raw-crawler
aws glue delete-database --name saa_lake
# delete the IAM role AWSGlueServiceRole-saa (detach its policies first)
```

- [ ] Delete the Athena workgroup and the buckets:

```bash
aws athena delete-work-group --work-group saa-analysts --recursive-delete-option
aws s3 rb s3://$B --force
aws s3 rb s3://saa-lab36-athena-results-$SUFFIX --force
aws s3 rb s3://saa-lab35-lake-$SUFFIX --force 2>/dev/null
```

- [ ] Lake Formation: deregister the data location and revoke the grants (console). You can keep yourself as data lake admin.
- [ ] `rm -f sales.csv etl.py`

---

## Knowledge check

**Q1.** Analysts query 5 TB of CSV logs in S3 with Athena and the costs are high. Most queries filter on date and select 3 of 40 columns. How do you cut cost?

<details><summary>Answer</summary>

Convert to **Parquet** (columnar, compressed), **partition by date**, and use partition projection or pruning. Athena then scans only the needed columns and partitions.

</details>

**Q2.** A company wants to give its marketing team access to a sales table without exposing the `customer_email` column, in Athena and Redshift Spectrum. What should it use?

<details><summary>Answer</summary>

**AWS Lake Formation** column-level permissions (or data filters).

</details>

**Q3.** New CSV files land in S3 daily with a schema that sometimes changes. The team wants them catalogued automatically for SQL queries. What should it use?

<details><summary>Answer</summary>

A scheduled **AWS Glue crawler** that updates the **Glue Data Catalog**, then **Athena** for queries.

</details>
