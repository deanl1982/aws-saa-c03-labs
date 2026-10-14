# Lab 09 — S3 Storage Classes & Lifecycle Management

| | |
|---|---|
| **Module** | 3 — Amazon S3 Object Storage |
| **Exam mapping** | Domain 4 · Task 4.1 (cost-optimized storage) · Domain 3 · Task 3.1 |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 45 minutes |
| **Estimated cost** | ~$0.01 |

- [ ] **Lab complete**

[⬅ Previous lab](lab-08-s3-security-encryption.md) · [Back to index](../README.md) · [Next lab ➡](lab-10-s3-replication-advanced.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Pick the right **S3 storage class** from access frequency, retrieval time, and resilience needs.
- Build **lifecycle rules** for transitions, expirations, noncurrent versions, and incomplete multipart uploads.
- Use **S3 Intelligent-Tiering** for unknown or changing access patterns.
- Restore archived objects from **Glacier Flexible Retrieval / Deep Archive**.
- Find savings with **Storage Class Analysis** and **S3 Storage Lens**.

## Storage class cheat sheet

| Class | AZs | Min duration | Min billable size | Retrieval | Use case |
|---|---|---|---|---|---|
| **Standard** | ≥3 | — | — | ms | Frequently accessed |
| **Intelligent-Tiering** | ≥3 | — | (objects <128 KB not auto-tiered) | ms (archive tiers optional) | Unknown or changing access. Small monthly monitoring fee per object. |
| **Standard-IA** | ≥3 | 30 days | 128 KB | ms + per-GB retrieval fee | Infrequent but needs fast access (backups, DR) |
| **One Zone-IA** | **1** | 30 days | 128 KB | ms + fee | Re-creatable infrequent data, secondary backup copies |
| **Glacier Instant Retrieval** | ≥3 | 90 days | 128 KB | **ms** | Archive accessed about once a quarter, needs instant access |
| **Glacier Flexible Retrieval** | ≥3 | 90 days | 40 KB overhead | Expedited 1–5 min · Standard 3–5 h · Bulk 5–12 h | Archives, backups |
| **Glacier Deep Archive** | ≥3 | 180 days | 40 KB overhead | Standard ≤12 h · Bulk ≤48 h | 7–10 year compliance retention, the lowest cost |
| **Express One Zone** | 1 (you choose the AZ) | — | — | single-digit ms | Ultra-low-latency, high-request-rate workloads (ML training, analytics). Uses *directory buckets*. |

**Lifecycle transition waterfall** (you can only move *down*):
`Standard → Standard-IA / Intelligent-Tiering → One Zone-IA → Glacier IR → Glacier Flexible → Deep Archive`
Transitions to Standard-IA or One Zone-IA need objects to be **at least 30 days old**.

---

## Part A — Console

### A1. Upload with different storage classes

- [ ] Create bucket `saa-lab09-<suffix>` with versioning **enabled**.
- [ ] Upload `report.csv`. Under **Properties → Storage class**, choose **Standard-IA**.
- [ ] Upload `archive.zip` as **Glacier Flexible Retrieval**.
- [ ] Try to **Open** or download `archive.zip`. You can't until it's restored.

### A2. Restore an archived object

- [ ] Select `archive.zip`, choose **Actions → Initiate restore**, set **Number of days 1**, and pick the **Bulk** retrieval tier.
- [ ] Note that the restored copy is temporary. The archived object stays in Glacier.

### A3. Lifecycle rules

- [ ] Open **Management → Lifecycle rules → Create rule**.
  - Name: `logs-tiering`
  - Scope: prefix filter `logs/`
  - Actions:
    - Move **current** versions → Standard-IA after **30** days → Glacier Flexible after **90** days → Deep Archive after **365** days
    - **Expire** current versions after **2555** days (7 years)
    - Permanently delete **noncurrent** versions **30** days after they become noncurrent
    - Delete **expired delete markers** and **incomplete multipart uploads** after **7** days
- [ ] Look at the **timeline visualisation** the console shows when you review the rule.

### A4. Intelligent-Tiering archive configuration

- [ ] Open **Properties → Intelligent-Tiering Archive configurations → Create**. Turn on **Archive Access tier** after 90 days and **Deep Archive Access tier** after 180 days.

### A5. Analyse access patterns

- [ ] Open **Metrics → Storage Class Analysis → Create analytics configuration** for the whole bucket. After about 30 days it recommends when to move data to IA.
- [ ] Open **S3 → Storage Lens → Dashboards → default-account-dashboard** and review the cost-efficiency metrics (e.g. % noncurrent bytes, incomplete MPU bytes).

---

## Part B — CLI

```bash
BUCKET=saa-lab09-cli-$SUFFIX
aws s3 mb s3://$BUCKET
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

### B1. Upload to specific storage classes

- [ ] Upload a file to each of several classes:

```bash
for C in STANDARD STANDARD_IA ONEZONE_IA INTELLIGENT_TIERING GLACIER_IR GLACIER DEEP_ARCHIVE; do
  echo "data for $C" > f.txt
  aws s3 cp f.txt s3://$BUCKET/classes/$C.txt --storage-class $C
done

aws s3api list-objects-v2 --bucket $BUCKET --prefix classes/ \
  --query 'Contents[].{Key:Key,Class:StorageClass}' --output table
```

### B2. Restore from Glacier and check status

- [ ] Request a restore:

```bash
aws s3api restore-object --bucket $BUCKET --key classes/GLACIER.txt \
  --restore-request '{"Days":1,"GlacierJobParameters":{"Tier":"Standard"}}'
```

- [ ] Check progress. It shows `ongoing-request="true"` until the restore finishes:

```bash
aws s3api head-object --bucket $BUCKET --key classes/GLACIER.txt --query '{Class:StorageClass,Restore:Restore}'
```

### B3. Apply a lifecycle configuration

- [ ] Write the rules:

```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "logs-tiering",
      "Filter": { "Prefix": "logs/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 },
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER_IR" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90, "NewerNoncurrentVersions": 3 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    },
    {
      "ID": "tmp-cleanup",
      "Filter": { "And": { "Prefix": "tmp/", "Tags": [ { "Key": "ttl", "Value": "short" } ] } },
      "Status": "Enabled",
      "Expiration": { "Days": 1 }
    },
    {
      "ID": "large-objects-to-it",
      "Filter": { "ObjectSizeGreaterThan": 1048576 },
      "Status": "Enabled",
      "Transitions": [ { "Days": 0, "StorageClass": "INTELLIGENT_TIERING" } ]
    }
  ]
}
EOF
```

- [ ] Apply and review them:

```bash
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET --query 'Rules[].{ID:ID,Status:Status}'
```

- [ ] See which lifecycle rule applies to an object (the `x-amz-expiration` header):

```bash
echo log > app.log && aws s3 cp app.log s3://$BUCKET/logs/app.log
aws s3api head-object --bucket $BUCKET --key logs/app.log --query Expiration
```

### B4. Intelligent-Tiering archive config

- [ ] Add the configuration:

```bash
aws s3api put-bucket-intelligent-tiering-configuration --bucket $BUCKET --id archive-it \
  --intelligent-tiering-configuration '{
    "Id":"archive-it","Status":"Enabled",
    "Tierings":[{"Days":90,"AccessTier":"ARCHIVE_ACCESS"},{"Days":180,"AccessTier":"DEEP_ARCHIVE_ACCESS"}]
  }'
```

### B5. Estimate the savings yourself

- [ ] Work it out: 10 TB of logs, read heavily for the first 30 days, rarely afterwards, kept for 7 years. Using current [S3 pricing](https://aws.amazon.com/s3/pricing/) for us-east-1, roughly compare:
  - Leaving everything in Standard for 7 years.
  - The lifecycle policy above.

> Expect roughly an order-of-magnitude difference. That's why the exam leans so heavily on lifecycle questions.

---

## Verify

- [ ] Objects exist in each storage class.
- [ ] The lifecycle rules apply (the `head-object` `Expiration` header shows `rule-id="logs-tiering"`).
- [ ] The Glacier restore has finished, or is in progress.

## Exam tips

- **Unknown or unpredictable access** → **Intelligent-Tiering** (no retrieval fees).
- **Accessed rarely, but needs millisecond access** → **Glacier Instant Retrieval** (or Standard-IA if accessed more often than quarterly).
- **Data can be re-created, infrequent access** → **One Zone-IA**.
- **Compliance archive, retrieval within 12–48 h is fine** → **Deep Archive**.
- Small objects (<128 KB) in IA classes are billed as 128 KB, so IA can cost *more* for tiny files.
- Always add **AbortIncompleteMultipartUpload**. Incomplete parts are invisible but billed.
- Need a **retrieval within minutes** from Glacier Flexible? Use **Expedited** retrieval (optionally with provisioned capacity).

## Challenge (optional)

- [ ] Set up **S3 Inventory** to deliver a daily CSV/Parquet report of objects and storage classes, and query it with Athena (after Lab 36).

## Cleanup

- [ ] Delete all versions, then the bucket:

```bash
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: [Versions, DeleteMarkers][].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3 rb s3://$BUCKET --force
```

> Deleting objects in IA or Glacier classes before their minimum duration incurs a prorated **early deletion charge**. It's pennies here.

- [ ] Empty and delete `saa-lab09-<suffix>` in the console.
- [ ] Delete the Storage Class Analysis configuration (it's removed with the bucket).
- [ ] `rm f.txt app.log lifecycle.json`

---

## Knowledge check

**Q1.** A media company stores videos that are accessed heavily for the first week and then almost never, but when one is requested it must play immediately. Which lifecycle design costs least?

<details><summary>Answer</summary>

Standard for the first 30 days (IA transitions need objects at least 30 days old), then **Glacier Instant Retrieval**, which gives millisecond access at archive pricing.

</details>

**Q2.** A data lake has millions of objects with unpredictable access patterns. The team doesn't want to analyse access or manage lifecycle rules. Which storage class?

<details><summary>Answer</summary>

**S3 Intelligent-Tiering.**

</details>

**Q3.** A company must keep audit logs for 10 years at the lowest possible cost. Retrieval within 48 hours is acceptable. Where should the logs go?

<details><summary>Answer</summary>

**S3 Glacier Deep Archive**, using a lifecycle transition, and optionally **Object Lock** for immutability (Lab 10).

</details>
