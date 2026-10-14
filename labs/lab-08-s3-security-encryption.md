# Lab 08 — S3 Security & Encryption

| | |
|---|---|
| **Module** | 3 — Amazon S3 Object Storage |
| **Exam mapping** | Domain 1 · Task 1.3 (data security controls) · Task 1.1 (resource policies) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$1 (one KMS customer-managed key is $1/month, prorated) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-07-s3-fundamentals.md) · [Back to index](../README.md) · [Next lab ➡](lab-09-s3-storage-classes-lifecycle.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Layer S3 access controls: **Block Public Access**, **Object Ownership**, **bucket policies**, IAM, and **access points**.
- Compare the encryption options **SSE-S3**, **SSE-KMS** (with S3 Bucket Keys), **DSSE-KMS**, **SSE-C**, and client-side encryption.
- Enforce **encryption in transit** (TLS) and **a specific KMS key** with bucket policies.
- Share private objects for a limited time with **presigned URLs**.

## Encryption decision table

| Option | Who manages keys | Audit trail of key use | Typical exam trigger |
|---|---|---|---|
| **SSE-S3** (AES-256) | AWS, fully | No | "Encrypt at rest, minimal effort" (default since Jan 2023) |
| **SSE-KMS** | AWS KMS (AWS-managed or **customer-managed** key) | **Yes**, CloudTrail logs each key use | "Audit who used the key", "control key rotation and policy", "separation of duties" |
| **DSSE-KMS** | KMS, two layers | Yes | "Dual-layer encryption for compliance" |
| **SSE-C** | **You** provide the key on every request (HTTPS required) | No | "Company must keep keys outside AWS but have S3 encrypt" |
| **Client-side** | You encrypt before upload | Your choice | "Data must be encrypted before it leaves the premises" |

> **S3 Bucket Keys** cut KMS request costs by up to 99% for SSE-KMS by issuing a bucket-level data key.

---

## Part A — Console

### A1. Account-level Block Public Access

- [ ] Open **S3 → Block Public Access settings for this account** and confirm **all four** settings are on.
- [ ] Notice how this overrides any bucket-level setting. It's the strongest guardrail.

### A2. Create a KMS key and an encrypted bucket

- [ ] Open **KMS → Customer managed keys → Create key**: symmetric, encrypt/decrypt, alias `saa-lab08-s3`, and your admin user as key administrator and user.
- [ ] Create bucket `saa-lab08-<suffix>`:
  - Default encryption: **SSE-KMS**, choose `alias/saa-lab08-s3`
  - **Bucket Key: Enable**
- [ ] Upload a file and look at **Properties → Server-side encryption** to confirm the KMS key ARN.

### A3. Enforce TLS and the KMS key with a bucket policy

- [ ] Add this bucket policy (replace `<suffix>` and `<key-arn>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::saa-lab08-<suffix>", "arn:aws:s3:::saa-lab08-<suffix>/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::saa-lab08-<suffix>/*",
      "Condition": {
        "StringNotEqualsIfExists": { "s3:x-amz-server-side-encryption-aws-kms-key-id": "<key-arn>" }
      }
    },
    {
      "Sid": "DenyOldTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::saa-lab08-<suffix>", "arn:aws:s3:::saa-lab08-<suffix>/*"],
      "Condition": { "NumericLessThan": { "s3:TlsVersion": "1.2" } }
    }
  ]
}
```

### A4. Create an S3 access point

- [ ] Open the bucket's **Access Points → Create access point**, name it `saa-analytics-ap`, and set **Network origin: Internet**.
- [ ] Attach an access point policy that lets a single IAM role read only the `analytics/` prefix.
- [ ] Look at the access point's own ARN and alias. Access points simplify bucket policies for shared data sets with many teams.

### A5. Presigned URL (console)

- [ ] Select an object, choose **Object actions → Share with a presigned URL**, and set it to expire in 5 minutes.
- [ ] Open the link in a private window. It works.
- [ ] Wait until it expires and try again. **AccessDenied / Request has expired.**

### A6. Review findings

- [ ] Open **S3 → IAM Access Analyzer for S3** and check whether any bucket is public or shared outside the account.

---

## Part B — CLI

```bash
BUCKET=saa-lab08-cli-$SUFFIX
aws s3 mb s3://$BUCKET
```

### B1. Create a KMS key and set default SSE-KMS with a bucket key

- [ ] Create the key and alias:

```bash
KEY_ARN=$(aws kms create-key --description "SAA lab08 CLI S3 key" --query KeyMetadata.Arn --output text)
aws kms create-alias --alias-name alias/saa-lab08-cli --target-key-id $KEY_ARN
```

- [ ] Set default encryption on the bucket:

```bash
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration "{
  \"Rules\": [{
    \"ApplyServerSideEncryptionByDefault\": {\"SSEAlgorithm\": \"aws:kms\", \"KMSMasterKeyID\": \"$KEY_ARN\"},
    \"BucketKeyEnabled\": true
  }]
}"
aws s3api get-bucket-encryption --bucket $BUCKET
```

- [ ] Upload an object and confirm it's encrypted with your key:

```bash
echo "secret data" > secret.txt
aws s3 cp secret.txt s3://$BUCKET/
aws s3api head-object --bucket $BUCKET --key secret.txt \
  --query '{SSE:ServerSideEncryption,Key:SSEKMSKeyId,BucketKey:BucketKeyEnabled}'
```

### B2. Compare SSE-S3 and SSE-C on individual objects

- [ ] Override with SSE-S3 for one object:

```bash
aws s3 cp secret.txt s3://$BUCKET/sse-s3.txt --sse AES256
```

- [ ] Use SSE-C, where you supply the key:

```bash
openssl rand -out ssec.key 32
aws s3 cp secret.txt s3://$BUCKET/sse-c.txt --sse-c AES256 --sse-c-key fileb://ssec.key
```

- [ ] Reading the SSE-C object **without** the key fails:

```bash
aws s3 cp s3://$BUCKET/sse-c.txt - || echo "Expected failure: need the customer key"
aws s3 cp s3://$BUCKET/sse-c.txt - --sse-c AES256 --sse-c-key fileb://ssec.key
```

### B3. Enforce TLS with a bucket policy

- [ ] Apply the policy:

```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy "{
  \"Version\":\"2012-10-17\",
  \"Statement\":[{
    \"Sid\":\"DenyNonTLS\",\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"s3:*\",
    \"Resource\":[\"arn:aws:s3:::$BUCKET\",\"arn:aws:s3:::$BUCKET/*\"],
    \"Condition\":{\"Bool\":{\"aws:SecureTransport\":\"false\"}}
  }]
}"
```

- [ ] Test it. HTTPS (the default) works and HTTP is denied:

```bash
aws s3 ls s3://$BUCKET/
aws s3 ls s3://$BUCKET/ --endpoint-url http://s3.us-east-1.amazonaws.com || echo "Denied over HTTP as expected"
```

### B4. Object Ownership and Block Public Access

- [ ] Inspect the bucket's settings:

```bash
aws s3api get-bucket-ownership-controls --bucket $BUCKET
aws s3api get-public-access-block --bucket $BUCKET
aws s3control get-public-access-block --account-id $ACCOUNT_ID   # account-level
```

### B5. Presigned URLs

- [ ] Generate a URL valid for 60 seconds and use it:

```bash
URL=$(aws s3 presign s3://$BUCKET/secret.txt --expires-in 60)
curl -s "$URL"            # works
sleep 65
curl -s "$URL" | head -3  # AccessDenied / Request has expired
```

> A presigned URL carries the **permissions of whoever signed it**. If that user loses access, or the signing role's session expires, the URL stops working early.

### B6. Access point

- [ ] Create one from the CLI:

```bash
aws s3control create-access-point --account-id $ACCOUNT_ID --name saa-cli-ap --bucket $BUCKET
aws s3control get-access-point --account-id $ACCOUNT_ID --name saa-cli-ap --query '{Alias:Alias,Network:NetworkOrigin}'
aws s3 ls s3://arn:aws:s3:us-east-1:$ACCOUNT_ID:accesspoint/saa-cli-ap/
```

---

## Verify

- [ ] Objects show `aws:kms` encryption with your key and Bucket Key enabled.
- [ ] HTTP requests are denied by the bucket policy.
- [ ] A presigned URL works, then expires.

## Exam tips

- **Audit key usage**, **rotate keys yourself**, or **separate key admins from data admins**: use **SSE-KMS with a customer-managed key**.
- High request volume with SSE-KMS hits **KMS request quotas** and costs. Enable **S3 Bucket Keys**.
- Encrypt existing unencrypted objects with **S3 Batch Operations** (copy in place) or a re-upload.
- Temporary upload/download access without AWS credentials: **presigned URLs**. Many files or paths behind a CDN: **CloudFront signed URLs or cookies**.
- **Amazon Macie** finds PII in S3 (Lab 31).
- **ACLs are legacy.** Use **Object Ownership = Bucket owner enforced**.

## Challenge (optional)

- [ ] Write a bucket policy that only allows access through a specific **VPC endpoint** (`aws:SourceVpce`) **or** from your corporate CIDR.
- [ ] Use **S3 Batch Operations** to re-encrypt every object in a bucket from SSE-S3 to SSE-KMS.

## Cleanup

- [ ] Delete the access point and bucket:

```bash
aws s3control delete-access-point --account-id $ACCOUNT_ID --name saa-cli-ap
aws s3 rb s3://$BUCKET --force
```

- [ ] Delete the console access point and `saa-lab08-<suffix>` bucket (empty it first).
- [ ] Schedule deletion of both KMS keys. The minimum is 7 days, and you aren't billed while deletion is pending:

```bash
aws kms schedule-key-deletion --key-id $KEY_ARN --pending-window-in-days 7
aws kms delete-alias --alias-name alias/saa-lab08-cli
# repeat for alias/saa-lab08-s3
```

- [ ] `rm secret.txt ssec.key`

---

## Knowledge check

**Q1.** A financial company must encrypt S3 data at rest, control the key's rotation schedule, and audit every use of the key. Which option meets all three needs?

<details><summary>Answer</summary>

**SSE-KMS with a customer-managed key.** You control the key policy and rotation, and each use is logged in CloudTrail.

</details>

**Q2.** After switching a high-traffic bucket to SSE-KMS, the app sees `ThrottlingException` from KMS. What's the most cost-effective fix?

<details><summary>Answer</summary>

Enable **S3 Bucket Keys**, which drastically cuts the number of KMS calls. Requesting a KMS quota increase is the alternative.

</details>

**Q3.** A partner without AWS credentials needs to download one file for the next 24 hours. What should you do?

<details><summary>Answer</summary>

Generate an S3 **presigned URL** with a 24-hour expiry, signed by an IAM user or role whose credentials stay valid for that long.

</details>
