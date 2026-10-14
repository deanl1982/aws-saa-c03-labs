# Lab 10 — S3 Replication, Object Lock, Events & Performance

| | |
|---|---|
| **Module** | 3 — Amazon S3 Object Storage |
| **Exam mapping** | Domain 2 · Task 2.2 (DR, fault tolerance) · Domain 1 · Task 1.3 (retention, WORM) · Domain 3 · Task 3.1 (performance) · Task 2.1 (event-driven) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.10 (cross-region transfer, Transfer Acceleration test) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-09-s3-storage-classes-lifecycle.md) · [Back to index](../README.md) · [Next lab ➡](lab-11-custom-vpc.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Set up **Cross-Region Replication (CRR)** and **Same-Region Replication (SRR)**, including Replication Time Control and replica storage-class choice.
- Make objects immutable with **S3 Object Lock** (Governance or Compliance mode, legal holds).
- Trigger downstream processing with **S3 Event Notifications** (SQS, SNS, Lambda, EventBridge).
- Improve throughput with **multipart upload**, **byte-range fetches**, prefix parallelism, and **Transfer Acceleration**.

## Replication facts

- **Versioning is required** on both source and destination.
- Only **new** objects replicate once the rule is in place. Use **S3 Batch Replication** for existing objects.
- **Delete markers** are *not* replicated unless you turn it on. **Version deletes are never replicated**, which protects against malicious deletes.
- **No chaining:** A→B→C doesn't replicate A's objects to C.
- **RTC (Replication Time Control):** SLA that 99.99% of objects replicate within **15 minutes**.

---

## Part A — Console

### A1. Cross-Region Replication

- [ ] Create source bucket `saa-lab10-src-<suffix>` in **us-east-1** with versioning enabled.
- [ ] Create destination bucket `saa-lab10-dst-<suffix>` in **us-west-2** with versioning enabled.
- [ ] On the source bucket, open **Management → Replication rules → Create replication rule**.
  - Name: `crr-to-west`, Scope: **all objects**
  - Destination: `saa-lab10-dst-<suffix>`
  - IAM role: **Create new role**
  - Destination storage class: **Standard-IA** (a cheaper DR copy)
  - Turn on **Replication Time Control** and **Delete marker replication**
- [ ] When asked whether to **replicate existing objects** with Batch Operations, choose *No* for this lab.
- [ ] Upload a file to the source, wait about a minute, and check the destination. On the source object, look at **Properties → Replication status**: `COMPLETED`. On the replica it shows `REPLICA`.

### A2. Object Lock (WORM)

> Object Lock can only be turned on **when you create the bucket**, or later through a support process. It needs versioning.

- [ ] Create bucket `saa-lab10-lock-<suffix>` and, under **Advanced settings**, turn on **Object Lock**.
- [ ] Open **Properties → Object Lock → Edit**:
  - Default retention: **Governance mode**, **1 day**
- [ ] Upload a file and try to delete its **specific version**. It's denied.
- [ ] Turn on a **legal hold** for an object (Object actions → Edit legal hold). Legal holds have no expiry and stay until removed.

> ⚠️ **Don't use Compliance mode in this lab.** Not even the root user can delete a Compliance-mode object before its retention date. Compliance mode is the answer to "no one, including root, can delete" exam questions.

### A3. Event notification to SQS

- [ ] Create an **SQS standard queue** `saa-lab10-events`.
- [ ] Edit the queue's **Access policy** so the S3 bucket can send messages. Replace the placeholders:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "s3.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:<acct>:saa-lab10-events",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:s3:::saa-lab10-src-<suffix>" },
      "StringEquals": { "aws:SourceAccount": "<acct>" }
    }
  }]
}
```

- [ ] On the source bucket, open **Properties → Event notifications → Create event notification**.
  - Prefix `uploads/`, suffix `.jpg`, event types **All object create events**
  - Destination: **SQS queue** `saa-lab10-events`
- [ ] Upload `uploads/cat.jpg` (any file renamed). In **SQS → Send and receive messages → Poll**, you'll see the event JSON.
- [ ] Turn on **Amazon EventBridge** under the same Properties section. Every event now also goes to EventBridge for advanced routing (Lab 27).

### A4. Transfer Acceleration

- [ ] Open the source bucket's **Properties → Transfer acceleration → Enable**.
- [ ] Use the [S3 Transfer Acceleration Speed Comparison tool](https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html) to see the speedup from your location.

---

## Part B — CLI

```bash
SRC=saa-lab10-cli-src-$SUFFIX
DST=saa-lab10-cli-dst-$SUFFIX
aws s3 mb s3://$SRC --region us-east-1
aws s3 mb s3://$DST --region us-west-2
aws s3api put-bucket-versioning --bucket $SRC --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket $DST --versioning-configuration Status=Enabled --region us-west-2
```

### B1. Replication role and configuration

- [ ] Create the role S3 uses to replicate:

```bash
cat > s3-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-lab10-replication --assume-role-policy-document file://s3-trust.json
```

- [ ] Give it the minimum permissions it needs:

```bash
cat > s3-repl-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["s3:GetReplicationConfiguration","s3:ListBucket"],"Resource":"arn:aws:s3:::$SRC"},
    {"Effect":"Allow","Action":["s3:GetObjectVersionForReplication","s3:GetObjectVersionAcl","s3:GetObjectVersionTagging"],"Resource":"arn:aws:s3:::$SRC/*"},
    {"Effect":"Allow","Action":["s3:ReplicateObject","s3:ReplicateDelete","s3:ReplicateTags"],"Resource":"arn:aws:s3:::$DST/*"}
  ]
}
EOF
aws iam put-role-policy --role-name saa-lab10-replication --policy-name repl --policy-document file://s3-repl-policy.json
```

- [ ] Apply the replication configuration:

```bash
cat > replication.json <<EOF
{
  "Role": "arn:aws:iam::$ACCOUNT_ID:role/saa-lab10-replication",
  "Rules": [{
    "ID": "crr-all",
    "Priority": 1,
    "Status": "Enabled",
    "Filter": {},
    "DeleteMarkerReplication": { "Status": "Enabled" },
    "Destination": { "Bucket": "arn:aws:s3:::$DST", "StorageClass": "STANDARD_IA" }
  }]
}
EOF
sleep 10
aws s3api put-bucket-replication --bucket $SRC --replication-configuration file://replication.json
```

- [ ] Test it:

```bash
echo "replicate me" > r.txt && aws s3 cp r.txt s3://$SRC/r.txt
sleep 30
aws s3api head-object --bucket $SRC --key r.txt --query ReplicationStatus
aws s3api head-object --bucket $DST --key r.txt --region us-west-2 --query '{Status:ReplicationStatus,Class:StorageClass}'
```

### B2. Object Lock with the CLI

- [ ] Create a locked bucket with a default retention:

```bash
LOCK=saa-lab10-cli-lock-$SUFFIX
aws s3api create-bucket --bucket $LOCK --object-lock-enabled-for-bucket
aws s3api put-object-lock-configuration --bucket $LOCK --object-lock-configuration \
  '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":1}}}'
```

- [ ] Upload an object and inspect its retention:

```bash
echo "immutable" > lock.txt
VID=$(aws s3api put-object --bucket $LOCK --key lock.txt --body lock.txt \
  --checksum-algorithm SHA256 --query VersionId --output text)
aws s3api get-object-retention --bucket $LOCK --key lock.txt --version-id $VID
```

- [ ] Try to delete that version. It's denied:

```bash
aws s3api delete-object --bucket $LOCK --key lock.txt --version-id $VID || echo "Blocked by Object Lock"
```

- [ ] Governance mode can be bypassed by users with `s3:BypassGovernanceRetention`:

```bash
aws s3api delete-object --bucket $LOCK --key lock.txt --version-id $VID --bypass-governance-retention
```

### B3. Event notifications to SQS from the CLI

- [ ] Create the queue and allow S3 to send to it:

```bash
Q_URL=$(aws sqs create-queue --queue-name saa-lab10-cli-events --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

POLICY=$(jq -n --arg q "$Q_ARN" --arg b "arn:aws:s3:::$SRC" '{
  Version:"2012-10-17",
  Statement:[{Effect:"Allow",Principal:{Service:"s3.amazonaws.com"},Action:"sqs:SendMessage",
              Resource:$q,Condition:{ArnEquals:{"aws:SourceArn":$b}}}]}')
aws sqs set-queue-attributes --queue-url $Q_URL --attributes "$(jq -n --arg p "$POLICY" '{Policy:$p}')"
```

- [ ] Configure the notification and trigger it:

```bash
aws s3api put-bucket-notification-configuration --bucket $SRC --notification-configuration "{
  \"QueueConfigurations\":[{\"QueueArn\":\"$Q_ARN\",\"Events\":[\"s3:ObjectCreated:*\"],
  \"Filter\":{\"Key\":{\"FilterRules\":[{\"Name\":\"prefix\",\"Value\":\"incoming/\"}]}}}]
}"

aws s3 cp r.txt s3://$SRC/incoming/r.txt
sleep 5
aws sqs receive-message --queue-url $Q_URL --query 'Messages[0].Body' --output text | jq '.Records[0] | {event:.eventName,key:.s3.object.key}'
```

### B4. Performance: multipart and byte-range

- [ ] Create a 200 MB file and upload it. The high-level `aws s3 cp` uses multipart automatically above 8 MB:

```bash
dd if=/dev/urandom of=big.bin bs=1M count=200
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
aws configure set default.s3.max_concurrent_requests 20
time aws s3 cp big.bin s3://$SRC/big.bin
```

- [ ] Byte-range fetch, downloading only the first 1 KB:

```bash
aws s3api get-object --bucket $SRC --key big.bin --range bytes=0-1023 first1k.bin && ls -l first1k.bin
```

- [ ] Transfer Acceleration:

```bash
aws s3api put-bucket-accelerate-configuration --bucket $SRC --accelerate-configuration Status=Enabled
time aws s3 cp big.bin s3://$SRC/big-accel.bin --endpoint-url https://s3-accelerate.amazonaws.com
```

---

## Verify

- [ ] Replicated objects appear in us-west-2 as `REPLICA` in `STANDARD_IA`.
- [ ] Object Lock blocks deleting a locked version unless governance is bypassed.
- [ ] SQS receives S3 event messages.

## Exam tips

- "Keep a copy of data in another Region for DR or compliance" → **CRR**. "Aggregate logs from several buckets into one bucket in the same Region" or "copy between prod and test accounts" → **SRR**.
- "Replication must finish within 15 minutes with an SLA" → **S3 RTC**.
- "No one, including root, can delete for 7 years" → **Object Lock Compliance mode**. "Admins can override" → **Governance mode**.
- "Slow uploads from distant users to a single bucket" → **Transfer Acceleration** (uses CloudFront edge locations). "Very large files" → **multipart upload**.
- Need more than 5,500 GETs per second? **Spread keys across prefixes** (the limits apply per prefix) or cache with **CloudFront**.
- Event notifications go to **SQS, SNS, Lambda, or EventBridge**. EventBridge gives advanced filtering, many targets, archive, and replay.

## Challenge (optional)

- [ ] Configure **S3 Multi-Region Access Points** over your two buckets and read through the global endpoint.
- [ ] Run **S3 Batch Replication** to copy objects that existed before the replication rule.

## Cleanup

- [ ] Remove replication and notifications, then delete the buckets. A locked object must wait 1 day or be removed with `--bypass-governance-retention`:

```bash
aws s3api delete-bucket-replication --bucket $SRC
aws s3api put-bucket-notification-configuration --bucket $SRC --notification-configuration '{}'
for B in $SRC; do
  aws s3api delete-objects --bucket $B --delete "$(aws s3api list-object-versions --bucket $B --query '{Objects: [Versions, DeleteMarkers][].{Key:Key,VersionId:VersionId}}')"
  aws s3 rb s3://$B
done
aws s3api delete-objects --bucket $DST --region us-west-2 --delete "$(aws s3api list-object-versions --bucket $DST --region us-west-2 --query '{Objects: [Versions, DeleteMarkers][].{Key:Key,VersionId:VersionId}}')"
aws s3 rb s3://$DST --region us-west-2
```

- [ ] Delete the lock bucket the next day, once retention expires, or bypass governance on each version first.
- [ ] Delete the queues, IAM role, and console buckets:

```bash
aws sqs delete-queue --queue-url $Q_URL
aws iam delete-role-policy --role-name saa-lab10-replication --policy-name repl
aws iam delete-role --role-name saa-lab10-replication
```

- [ ] Delete the console resources: `saa-lab10-src/dst/lock-<suffix>` buckets, the `saa-lab10-events` queue, and the auto-created replication role.
- [ ] `rm -f big.bin first1k.bin r.txt lock.txt s3-*.json replication.json`
- [ ] Reset the CLI tuning: `aws configure set default.s3.max_concurrent_requests 10`

---

## Knowledge check

**Q1.** A company must keep a copy of all S3 objects in a second Region, and 99.99% of objects must arrive within 15 minutes. What should it configure?

<details><summary>Answer</summary>

**S3 Cross-Region Replication with Replication Time Control (RTC)**, with versioning enabled on both buckets.

</details>

**Q2.** Regulators require that trade records can't be changed or deleted by anyone for 7 years. What should you configure?

<details><summary>Answer</summary>

**S3 Object Lock in Compliance mode** with a 7-year retention period.

</details>

**Q3.** Users around the world upload large files to a bucket in us-east-1 and complain about slow uploads. What improves upload speed with the fewest changes?

<details><summary>Answer</summary>

Enable **S3 Transfer Acceleration** and use **multipart upload**.

</details>
