# Lab 35 — Streaming Ingestion with Kinesis

| | |
|---|---|
| **Module** | 10 — Data Ingestion, Analytics & Hybrid |
| **Exam mapping** | Domain 3 · Task 3.5 (data ingestion and transformation) · Domain 2 · Task 2.1 (decoupled streaming) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.10–0.20 (one provisioned shard at $0.015/h, plus Firehose per GB) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-34-backup-dr.md) · [Back to index](../README.md) · [Next lab ➡](lab-36-glue-athena-datalake.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Produce to and consume from **Kinesis Data Streams (KDS)**, and explain **shards**, **partition keys**, and ordering.
- Size a stream (**provisioned vs on-demand**) and **reshard** it.
- Deliver streaming data to S3 with **Amazon Data Firehose**, including buffering, **Lambda transformation**, and format conversion.
- Choose between **KDS, Firehose, SQS, MSK**, and **Managed Service for Apache Flink**.

## Streaming services

| Service | What it is | Exam trigger |
|---|---|---|
| **Kinesis Data Streams** | Real-time, ordered (per shard), **replayable** stream. Retention 24 h – 365 days. Many consumers. | "Real-time processing", "multiple apps read the same stream", "replay" |
| **Amazon Data Firehose** | **Fully managed delivery** to S3, Redshift, OpenSearch, Splunk, HTTP. **Near real-time** (buffered). No consumers to manage. | "Load streaming data into S3/Redshift with no code", "convert to Parquet" |
| **Managed Service for Apache Flink** | SQL/Java/Python streaming analytics (windows, aggregations) | "Real-time analytics on streams" |
| **Amazon MSK** | Managed **Apache Kafka** (provisioned or serverless) | "Existing Kafka apps", "Kafka APIs" |
| **SQS** | Message queue: per-message delete, no replay, no ordering (standard) | "Decouple workers" |

**Shard capacity (provisioned):** **1 MB/s or 1,000 records/s in**, **2 MB/s out** (shared across consumers; **enhanced fan-out** gives each consumer 2 MB/s).

---

## Part A — Console

### A1. Create a data stream

- [ ] Open **Kinesis → Data streams → Create**: `saa-clickstream`, capacity mode **Provisioned**, **1 shard**.
- [ ] Look at the **Configuration** tab: retention (24 h default), encryption (on by default with `aws/kinesis`), and enhanced fan-out consumers.

### A2. Lambda transformation function for Firehose

- [ ] Create a Lambda function `saa-firehose-transform` (Python 3.13) that adds a field and filters out bot traffic:

```python
import base64, json

def lambda_handler(event, context):
    out = []
    for r in event["records"]:
        data = json.loads(base64.b64decode(r["data"]))
        if data.get("userAgent") == "bot":
            out.append({"recordId": r["recordId"], "result": "Dropped", "data": r["data"]})
            continue
        data["processed"] = True
        payload = (json.dumps(data) + "\n").encode()
        out.append({"recordId": r["recordId"], "result": "Ok", "data": base64.b64encode(payload).decode()})
    return {"records": out}
```

- [ ] Set the timeout to **1 minute**.

### A3. Firehose: stream → transform → S3

- [ ] Create bucket `saa-lab35-lake-<suffix>`.
- [ ] Open **Amazon Data Firehose → Create Firehose stream**.
  - Source **Amazon Kinesis Data Streams** → `saa-clickstream`
  - Destination **Amazon S3**, name `saa-clickstream-to-s3`
  - **Transform records with Lambda:** ✔ `saa-firehose-transform`
  - S3 bucket `saa-lab35-lake-<suffix>`, prefix `clicks/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/`, error prefix `errors/!{firehose:error-output-type}/`
  - Buffer hints: **1 MiB / 60 seconds** (low, for the lab)
  - Compression **GZIP**
- [ ] *(Read)* **Convert record format** to **Parquet/ORC** needs a Glue table schema. You'll cover that pattern in Lab 36.

### A4. Send test data

- [ ] In the Firehose stream, choose **Test with demo data** → **Start sending**. Or use the CLI producer in Part B. Wait about 2 minutes.
- [ ] Browse `s3://saa-lab35-lake-<suffix>/clicks/year=.../`. GZIP objects appear, partitioned by date.

### A5. Monitoring

- [ ] Open the data stream's **Monitoring** tab and watch `IncomingRecords`, `GetRecords.IteratorAgeMilliseconds` (consumer lag), and `WriteProvisionedThroughputExceeded`.

---

## Part B — CLI

### B1. Producer: put records with partition keys

- [ ] Send 200 click events:

```bash
for i in $(seq 1 200); do
  USER="user-$(( RANDOM % 20 ))"
  UA=$([ $((RANDOM % 10)) -eq 0 ] && echo bot || echo browser)
  DATA=$(printf '{"user":"%s","page":"/p/%d","userAgent":"%s","ts":%d}' $USER $((RANDOM % 50)) $UA $(date +%s))
  aws kinesis put-record --stream-name saa-clickstream --partition-key $USER \
    --data "$DATA" --cli-binary-format raw-in-base64-out --query SequenceNumber --output text >/dev/null
done
```

> The **partition key** decides the shard. All events for one user go to one shard, so they stay **in order**.

### B2. Batch producer (more efficient)

- [ ] Send 100 records in one call:

```bash
RECS=$(for i in $(seq 1 100); do printf '{"Data":"%s","PartitionKey":"batch-%d"},' \
  "$(printf '{"user":"batch","n":%d,"userAgent":"browser"}' $i | base64)" $i; done | sed 's/,$//')
aws kinesis put-records --stream-name saa-clickstream --records "[$RECS]" --query 'FailedRecordCount'
```

### B3. Consumer: read from a shard

- [ ] Read the first 5 records from the start of the shard:

```bash
SHARD=$(aws kinesis list-shards --stream-name saa-clickstream --query 'Shards[0].ShardId' --output text)
IT=$(aws kinesis get-shard-iterator --stream-name saa-clickstream --shard-id $SHARD \
  --shard-iterator-type TRIM_HORIZON --query ShardIterator --output text)
aws kinesis get-records --shard-iterator $IT --limit 5 --query 'Records[].Data' --output text | \
  tr '\t' '\n' | while read d; do echo $d | base64 --decode; echo; done
```

> `TRIM_HORIZON` = from the oldest record (replay). `LATEST` = only new records. `AT_TIMESTAMP` = replay from a point in time.

### B4. Reshard (scale up)

- [ ] Double the shard count:

```bash
aws kinesis update-shard-count --stream-name saa-clickstream --target-shard-count 2 --scaling-type UNIFORM_SCALING
aws kinesis wait stream-exists --stream-name saa-clickstream
aws kinesis describe-stream-summary --stream-name saa-clickstream \
  --query 'StreamDescriptionSummary.{Status:StreamStatus,OpenShards:OpenShardCount,Mode:StreamModeDetails.StreamMode}'
```

- [ ] Or switch to **on-demand** (no shard management; scales automatically):

```bash
STREAM_ARN=$(aws kinesis describe-stream-summary --stream-name saa-clickstream --query StreamDescriptionSummary.StreamARN --output text)
# aws kinesis update-stream-mode --stream-arn $STREAM_ARN --stream-mode-details StreamMode=ON_DEMAND
```

### B5. Extend retention (for replay)

- [ ] Keep records for 3 days:

```bash
aws kinesis increase-stream-retention-period --stream-name saa-clickstream --retention-period-hours 72
```

### B6. Enhanced fan-out consumer

- [ ] Register a consumer with its own 2 MB/s pipe:

```bash
aws kinesis register-stream-consumer --stream-arn $STREAM_ARN --consumer-name saa-analytics-efo
aws kinesis list-stream-consumers --stream-arn $STREAM_ARN --query 'Consumers[].{Name:ConsumerName,Status:ConsumerStatus}'
```

### B7. Direct-PUT Firehose (no stream in front)

- [ ] Create a Firehose role that can write to the bucket:

```bash
cat > fh-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"firehose.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-firehose-s3 --assume-role-policy-document file://fh-trust.json
aws iam put-role-policy --role-name saa-firehose-s3 --policy-name s3 --policy-document "{
 \"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":[\"s3:PutObject\",\"s3:GetBucketLocation\",\"s3:ListBucket\",\"s3:AbortMultipartUpload\"],
 \"Resource\":[\"arn:aws:s3:::saa-lab35-lake-$SUFFIX\",\"arn:aws:s3:::saa-lab35-lake-$SUFFIX/*\"]}]}"
sleep 10
```

- [ ] Create a direct-PUT Firehose stream and send a record to it:

```bash
aws firehose create-delivery-stream --delivery-stream-name saa-direct-put \
  --delivery-stream-type DirectPut \
  --extended-s3-destination-configuration "{
    \"RoleARN\":\"arn:aws:iam::$ACCOUNT_ID:role/saa-firehose-s3\",
    \"BucketARN\":\"arn:aws:s3:::saa-lab35-lake-$SUFFIX\",
    \"Prefix\":\"direct/\",\"ErrorOutputPrefix\":\"direct-errors/\",
    \"BufferingHints\":{\"SizeInMBs\":1,\"IntervalInSeconds\":60},
    \"CompressionFormat\":\"GZIP\"}"

sleep 30
aws firehose put-record --delivery-stream-name saa-direct-put \
  --record "{\"Data\":\"$(echo '{"event":"direct","ok":true}' | base64)\"}"
```

---

## Verify

- [ ] Records go into KDS and can be read back from `TRIM_HORIZON`.
- [ ] Firehose delivered transformed, GZIP'd, date-partitioned objects to S3, with the bot records dropped.
- [ ] The stream was resharded to 2 shards.

## Exam tips

- **"Ingest clickstream / IoT data in real time and process it with several applications"** → **Kinesis Data Streams**.
- **"Load streaming data into S3 / Redshift / OpenSearch with the least operational overhead"** → **Data Firehose** (near real-time, buffered, can transform with Lambda and convert to Parquet).
- **"Real-time SQL analytics / anomaly detection on streams"** → **Managed Service for Apache Flink**.
- **"Records for the same device must be processed in order"** → same **partition key**, so the same shard.
- **`ProvisionedThroughputExceeded`** → add shards, switch to **on-demand**, or use a better partition key (a hot shard).
- **"Each consumer needs dedicated throughput / low latency"** → **enhanced fan-out**.
- **Firehose can't do real-time** (minimum buffer is about 0–60 s). If sub-second is needed, use KDS + Lambda or Flink.

## Challenge (optional)

- [ ] Add a **Lambda event source mapping** on `saa-clickstream` with a **tumbling window** of 30 s to count clicks per user.

## Cleanup

- [ ] Delete the Firehose streams first, then the data stream:

```bash
aws firehose delete-delivery-stream --delivery-stream-name saa-direct-put
aws firehose delete-delivery-stream --delivery-stream-name saa-clickstream-to-s3
aws kinesis deregister-stream-consumer --stream-arn $STREAM_ARN --consumer-name saa-analytics-efo
aws kinesis delete-stream --stream-name saa-clickstream --enforce-consumer-deletion
```

- [ ] Delete the Lambda function and roles:

```bash
aws lambda delete-function --function-name saa-firehose-transform
aws iam delete-role-policy --role-name saa-firehose-s3 --policy-name s3 && aws iam delete-role --role-name saa-firehose-s3
# delete the console-created Firehose and Lambda roles too
```

- [ ] **Keep** bucket `saa-lab35-lake-<suffix>` for Lab 36, or delete it: `aws s3 rb s3://saa-lab35-lake-$SUFFIX --force`
- [ ] `rm fh-trust.json`

---

## Knowledge check

**Q1.** A company collects telemetry from 100,000 devices. Three teams need to process the same data independently in real time, and data must be replayable for 7 days. Which service?

<details><summary>Answer</summary>

**Kinesis Data Streams** with 7-day retention (and enhanced fan-out consumers if needed).

</details>

**Q2.** An application must load log records into S3 in Parquet format, partitioned by date, with no servers or custom consumers to manage. What should it use?

<details><summary>Answer</summary>

**Amazon Data Firehose** with **record format conversion** (to Parquet, using a Glue table schema) and dynamic/timestamp-based S3 prefixes.

</details>

**Q3.** A Kinesis stream returns `ProvisionedThroughputExceededException` even though the total traffic is under the stream's capacity. What's the likely cause?

<details><summary>Answer</summary>

A **hot shard** caused by a low-cardinality **partition key**. Use a higher-cardinality key (or add a random suffix), or switch to on-demand mode.

</details>
