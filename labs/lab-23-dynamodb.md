# Lab 23 — Amazon DynamoDB

| | |
|---|---|
| **Module** | 6 — Databases |
| **Exam mapping** | Domain 3 · Task 3.3 (high-performing DB) · Domain 4 · Task 4.3 (capacity modes) · Domain 2 · Task 2.2 (global tables, PITR) · Task 2.1 (streams) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.05 (on-demand requests; a global table replica adds a few cents) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-22-aurora-rds-proxy.md) · [Back to index](../README.md) · [Next lab ➡](lab-24-elasticache.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Design **partition keys** and **sort keys** for access patterns, and avoid **hot partitions**.
- Use **Query** vs **Scan**, and add **GSIs/LSIs** for extra access patterns.
- Choose **on-demand** vs **provisioned** capacity (with auto scaling), and do RCU/WCU maths.
- Turn on **TTL**, **Streams**, **PITR**, and **global tables**.
- Know when to add **DAX**, and when DynamoDB is (or isn't) the right database.

## Key facts

| Item | Detail |
|---|---|
| Item size | Max **400 KB** (store bigger objects in S3 and keep a pointer) |
| Read capacity unit | 1 strongly consistent read/s of ≤4 KB, or **2** eventually consistent reads/s. Transactional reads cost 2× |
| Write capacity unit | 1 write/s of ≤1 KB. Transactional writes cost 2× |
| GSI | Different PK/SK. **Add any time.** Has its own capacity. Eventually consistent only. |
| LSI | Same PK, different SK. **Only at table creation.** Strong consistency possible. |
| DAX | In-memory cache, **microsecond** reads, API-compatible |
| Global tables | Multi-region, **multi-active** (read/write everywhere) |
| Streams | 24 h ordered change log → Lambda, Kinesis |

---

## Data model for this lab

**Table `saa-orders`**

| Attribute | Role |
|---|---|
| `customerId` (S) | **Partition key** |
| `orderDate` (S, ISO-8601) | **Sort key** |
| `status` (S) | GSI partition key, for "all PENDING orders" |
| `total` (N), `expiresAt` (N, epoch) | Attributes. `expiresAt` drives TTL. |

---

## Part A — Console

### A1. Create the table

- [ ] Open **DynamoDB → Tables → Create table**: name `saa-orders`, partition key `customerId` (String), sort key `orderDate` (String).
- [ ] Table settings **Customize**: capacity mode **On-demand**, encryption **AWS owned key**.
- [ ] Create it.

### A2. Add items

- [ ] Open **Explore items → Create item**, switch to the **JSON view**, and add items such as:

```json
{ "customerId": {"S":"C001"}, "orderDate": {"S":"2026-10-01T10:00:00Z"}, "status": {"S":"SHIPPED"}, "total": {"N":"59.99"} }
```

- [ ] Add 5–6 more items across customers `C001`–`C003` with different dates and statuses (`PENDING`, `SHIPPED`, `CANCELLED`).

### A3. Query vs Scan

- [ ] **Query:** partition key `C001`, sort key **begins with** `2026-10`. Note the **consumed capacity** shown.
- [ ] **Scan** with a filter `status = PENDING`. Notice it reads **every item** and then filters. On a big table, that's slow and expensive.

### A4. Add a GSI

- [ ] Open **Indexes → Create index**: partition key `status`, sort key `orderDate`, name `status-orderDate-index`, projection **All**.
- [ ] When it's **Active**, query the index for `status = PENDING`. That's an efficient access pattern a Scan can't match.

### A5. TTL, PITR, and Streams

- [ ] **Additional settings → Time to Live → Turn on**, attribute `expiresAt`.
- [ ] Add an item with `expiresAt` = now + 120 seconds (Unix epoch). It's deleted **within a few days, usually much sooner**. TTL deletes are free and best-effort.
- [ ] **Backups → Point-in-time recovery → Edit → Turn on**, so you can restore to any second in the last 35 days.
- [ ] **Exports and streams → DynamoDB stream → Turn on**, view type **New and old images**.

### A6. Global table

- [ ] Open **Global tables → Create replica** → **us-west-2**. (This needs Streams, which the console turns on for you.)
- [ ] When it's active, switch to us-west-2, add an item there, then switch back to us-east-1 and see it **replicated within about a second**.

---

## Part B — CLI

### B1. Create a provisioned table with an LSI

- [ ] Create the table:

```bash
aws dynamodb create-table --table-name saa-sessions \
  --attribute-definitions AttributeName=userId,AttributeType=S AttributeName=sessionStart,AttributeType=S AttributeName=device,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH AttributeName=sessionStart,KeyType=RANGE \
  --local-secondary-indexes '[{
    "IndexName":"device-index",
    "KeySchema":[{"AttributeName":"userId","KeyType":"HASH"},{"AttributeName":"device","KeyType":"RANGE"}],
    "Projection":{"ProjectionType":"KEYS_ONLY"}}]' \
  --billing-mode PROVISIONED --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
aws dynamodb wait table-exists --table-name saa-sessions
```

### B2. Auto scaling for provisioned capacity

- [ ] Register the table and add a target-tracking policy:

```bash
aws application-autoscaling register-scalable-target --service-namespace dynamodb \
  --resource-id table/saa-sessions --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --min-capacity 5 --max-capacity 50

aws application-autoscaling put-scaling-policy --service-namespace dynamodb \
  --resource-id table/saa-sessions --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --policy-name rcu-70 --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{"TargetValue":70,"PredefinedMetricSpecification":{"PredefinedMetricType":"DynamoDBReadCapacityUtilization"}}'
```

### B3. Write and read items

- [ ] Write a batch of orders into `saa-orders`:

```bash
for i in $(seq 1 20); do
  C=$(printf "C%03d" $(( (i % 4) + 1 )))
  S=$([ $((i % 3)) -eq 0 ] && echo PENDING || echo SHIPPED)
  aws dynamodb put-item --table-name saa-orders --item "{
    \"customerId\":{\"S\":\"$C\"},
    \"orderDate\":{\"S\":\"2026-10-$(printf %02d $i)T09:00:00Z\"},
    \"status\":{\"S\":\"$S\"},
    \"total\":{\"N\":\"$((i * 7)).50\"}}"
done
```

- [ ] Query (efficient) with a key condition and a projection:

```bash
aws dynamodb query --table-name saa-orders \
  --key-condition-expression "customerId = :c AND orderDate BETWEEN :a AND :b" \
  --expression-attribute-values '{":c":{"S":"C002"},":a":{"S":"2026-10-01"},":b":{"S":"2026-10-15"}}' \
  --projection-expression "orderDate, #s, total" --expression-attribute-names '{"#s":"status"}' \
  --return-consumed-capacity TOTAL --query '{Items:Items,Consumed:ConsumedCapacity}'
```

- [ ] Query the GSI:

```bash
aws dynamodb query --table-name saa-orders --index-name status-orderDate-index \
  --key-condition-expression "#s = :p" --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":p":{"S":"PENDING"}}' --query 'Count'
```

- [ ] Compare with a Scan:

```bash
aws dynamodb scan --table-name saa-orders --filter-expression "#s = :p" \
  --expression-attribute-names '{"#s":"status"}' --expression-attribute-values '{":p":{"S":"PENDING"}}' \
  --return-consumed-capacity TOTAL --query '{Count:Count,Scanned:ScannedCount,Consumed:ConsumedCapacity.CapacityUnits}'
```

> Compare `ScannedCount` with `Count`. You paid to read every item.

### B4. Conditional writes and transactions

- [ ] Optimistic concurrency: update only if the status is still PENDING:

```bash
aws dynamodb update-item --table-name saa-orders \
  --key '{"customerId":{"S":"C004"},"orderDate":{"S":"2026-10-03T09:00:00Z"}}' \
  --update-expression "SET #s = :new" --condition-expression "#s = :old" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":new":{"S":"SHIPPED"},":old":{"S":"PENDING"}}'
```

- [ ] Run the same command again. It fails with `ConditionalCheckFailedException`.
- [ ] Run an all-or-nothing transaction across two items:

```bash
aws dynamodb transact-write-items --transact-items '[
 {"Put":{"TableName":"saa-orders","Item":{"customerId":{"S":"C009"},"orderDate":{"S":"2026-10-20T00:00:00Z"},"status":{"S":"PENDING"},"total":{"N":"10"}}}},
 {"Update":{"TableName":"saa-orders","Key":{"customerId":{"S":"C001"},"orderDate":{"S":"2026-10-04T09:00:00Z"}},
   "UpdateExpression":"SET total = total + :x","ExpressionAttributeValues":{":x":{"N":"1"}}}}]'
```

### B5. Backups and restore

- [ ] Take an on-demand backup and check PITR:

```bash
aws dynamodb create-backup --table-name saa-orders --backup-name saa-orders-manual
aws dynamodb describe-continuous-backups --table-name saa-orders \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription.{Status:PointInTimeRecoveryStatus,Earliest:EarliestRestorableDateTime}'
```

- [ ] Restore to a new table. As with RDS, a restore always creates a new table:

```bash
aws dynamodb restore-table-to-point-in-time --source-table-name saa-orders \
  --target-table-name saa-orders-restored --use-latest-restorable-time
```

### B6. Read the stream

- [ ] Fetch a few change records:

```bash
STREAM=$(aws dynamodb describe-table --table-name saa-orders --query Table.LatestStreamArn --output text)
SHARD=$(aws dynamodbstreams describe-stream --stream-arn $STREAM --query 'StreamDescription.Shards[0].ShardId' --output text)
IT=$(aws dynamodbstreams get-shard-iterator --stream-arn $STREAM --shard-id $SHARD --shard-iterator-type TRIM_HORIZON --query ShardIterator --output text)
aws dynamodbstreams get-records --shard-iterator $IT --limit 3 --query 'Records[].{Event:eventName,Keys:dynamodb.Keys}'
```

> In production, a **Lambda trigger** consumes the stream (Lab 26), e.g. to send an order-confirmation email when an item is inserted.

### B7. Capacity maths

- [ ] Work it out: an app does **100 strongly consistent reads/s of 6 KB items** and **50 writes/s of 2.5 KB items**.
  - RCU = 100 × ceil(6/4) = 100 × 2 = **200 RCU** (eventually consistent: **100**)
  - WCU = 50 × ceil(2.5/1) = 50 × 3 = **150 WCU**

---

## Verify

- [ ] Queries on the base table and the GSI return the expected items. The Scan cost far more capacity.
- [ ] The conditional update failed the second time.
- [ ] The global table replica in us-west-2 shows items written in us-east-1.

## Exam tips

- **"Serverless key-value at any scale with single-digit ms latency"** → **DynamoDB**. **"Microsecond reads"** → add **DAX**.
- **Unpredictable or spiky traffic, or a new app** → **on-demand**. **Steady, predictable traffic** → **provisioned + auto scaling** (cheaper), optionally with **reserved capacity**.
- **"Multi-region active-active with local reads and writes"** → **global tables**.
- **"Automatically delete old sessions/records at no cost"** → **TTL**.
- **"React to item changes"** → **DynamoDB Streams + Lambda**. **Kinesis Data Streams for DynamoDB** gives longer retention and more consumers.
- **"Export to S3 for analytics without consuming RCUs"** → **Export to S3** (needs PITR).
- **Complex joins, ad-hoc SQL, relational integrity** → not DynamoDB. Use RDS or Aurora.

## Challenge (optional)

- [ ] Use **PartiQL** (`aws dynamodb execute-statement --statement "SELECT * FROM \"saa-orders\" WHERE customerId='C001'"`) and compare it with the native query.

## Cleanup

- [ ] Remove the global table replica first:

```bash
aws dynamodb update-table --table-name saa-orders --replica-updates '[{"Delete":{"RegionName":"us-west-2"}}]'
```

- [ ] Delete the tables:

```bash
aws dynamodb wait table-exists --table-name saa-orders
sleep 60
for T in saa-orders saa-orders-restored saa-sessions; do aws dynamodb delete-table --table-name $T; done
```

- [ ] Delete the backup and deregister auto scaling:

```bash
for B in $(aws dynamodb list-backups --query 'BackupSummaries[?starts_with(BackupName,`saa-`)].BackupArn' --output text); do aws dynamodb delete-backup --backup-arn $B; done
aws application-autoscaling deregister-scalable-target --service-namespace dynamodb --resource-id table/saa-sessions --scalable-dimension dynamodb:table:ReadCapacityUnits 2>/dev/null
```

---

## Knowledge check

**Q1.** A gaming leaderboard needs reads in microseconds from a DynamoDB table that's read millions of times per second. What should you add?

<details><summary>Answer</summary>

**DynamoDB Accelerator (DAX).**

</details>

**Q2.** A startup is launching a new app with unknown traffic patterns and wants to avoid capacity planning. Which DynamoDB capacity mode?

<details><summary>Answer</summary>

**On-demand.**

</details>

**Q3.** A table uses `orderDate` (YYYY-MM-DD) as its partition key, and all writes go to today's date, causing throttling. What's wrong, and how do you fix it?

<details><summary>Answer</summary>

It's a **hot partition**, because the key has low cardinality at write time. Use a **high-cardinality partition key** (e.g. `customerId`, or add a random/calculated suffix with write sharding) and move the date to the sort key.

</details>
