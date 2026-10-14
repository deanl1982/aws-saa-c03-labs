# Lab 07 — S3 Fundamentals & Static Website Hosting

| | |
|---|---|
| **Module** | 3 — Amazon S3 Object Storage |
| **Exam mapping** | Domain 3 · Task 3.1 (scalable storage) · Domain 2 · Task 2.2 (durability, versioning) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 45 minutes |
| **Estimated cost** | ~$0.01 |

- [ ] **Lab complete**

[⬅ Previous lab](lab-06-organizations-scps.md) · [Back to index](../README.md) · [Next lab ➡](lab-08-s3-security-encryption.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Explain S3's **object model**: bucket, key, prefix, object, metadata, version ID.
- State S3's **durability** (11 nines), **availability**, and **strong read-after-write consistency**.
- Turn on **versioning** and recover deleted or overwritten objects.
- Host a **static website** on S3 and explain its limits (HTTP only; use CloudFront for HTTPS).
- Choose between `aws s3` (high-level) and `aws s3api` (low-level) commands.

## Key facts

| Fact | Value |
|---|---|
| Max object size | 5 TB (single PUT max 5 GB; use **multipart upload** above 100 MB) |
| Bucket names | Globally unique, 3–63 chars, lowercase, DNS-compliant |
| Buckets per account | 10,000 default (soft limit) |
| Consistency | Strong read-after-write for PUT/DELETE on all objects |
| Durability | 99.999999999% (11 9s) across ≥3 AZs (except One Zone classes) |
| Request rate | 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second **per prefix** |

---

## Part A — Console

### A1. Create a bucket and upload objects

- [ ] Open **S3 → Create bucket**.
  - Name: `saa-lab07-<suffix>`, Region: `us-east-1`
  - Object Ownership: **ACLs disabled** (recommended)
  - Block Public Access: **on** (for now)
  - Bucket Versioning: **Enable**
  - Default encryption: **SSE-S3**
- [ ] Upload a few files, putting one in a "folder" `docs/`. A folder is just a key prefix: the object key becomes `docs/file.txt`.
- [ ] Select an object and look at its **Properties**: storage class, ETag, server-side encryption, and metadata.
- [ ] Add user-defined metadata `x-amz-meta-owner = <you>`.

### A2. Versioning in action

- [ ] Upload `hello.txt` containing `v1`, then upload it again containing `v2`.
- [ ] Turn on **Show versions**. You'll see two version IDs.
- [ ] Delete `hello.txt` with *Show versions* **off**. S3 adds a **delete marker**.
- [ ] Turn *Show versions* back on, then delete the **delete marker**. The object comes back.
- [ ] Download the older `v1` version.

### A3. Static website hosting

- [ ] Create `index.html` and `error.html` locally:

```html
<!-- index.html -->
<!doctype html><html><body><h1>SAA Lab 07 — hosted on S3</h1></body></html>
```

```html
<!-- error.html -->
<!doctype html><html><body><h1>404 — not found</h1></body></html>
```

- [ ] Create a **second** bucket `saa-lab07-web-<suffix>` with **Block all public access unticked** (acknowledge the warning).
- [ ] Upload both files.
- [ ] Open **Properties → Static website hosting → Enable**. Set the index document to `index.html` and the error document to `error.html`.
- [ ] Open **Permissions → Bucket policy** and add:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadForWebsite",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::saa-lab07-web-<suffix>/*"
  }]
}
```

- [ ] Go to the **bucket website endpoint** (`http://saa-lab07-web-<suffix>.s3-website-us-east-1.amazonaws.com`), then try a missing page to see `error.html`.

> **Exam note:** the website endpoint supports **HTTP only**. For HTTPS and a custom domain, put **CloudFront** in front (Lab 33).

---

## Part B — CLI

```bash
BUCKET=saa-lab07-cli-$SUFFIX
```

### B1. High-level `aws s3` commands

- [ ] Create a bucket and copy files in:

```bash
aws s3 mb s3://$BUCKET --region us-east-1

mkdir -p site/assets && echo "hello" > site/a.txt && echo "img" > site/assets/b.txt
aws s3 cp site/a.txt s3://$BUCKET/
aws s3 sync site/ s3://$BUCKET/site/
aws s3 ls s3://$BUCKET --recursive --human-readable --summarize
```

- [ ] Run `sync` again to see that only changes are copied:

```bash
echo "changed" > site/a.txt
aws s3 sync site/ s3://$BUCKET/site/ --dryrun
```

### B2. Low-level `aws s3api` commands: versioning

- [ ] Turn on versioning and write two versions:

```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api get-bucket-versioning --bucket $BUCKET

echo v1 > hello.txt && aws s3api put-object --bucket $BUCKET --key hello.txt --body hello.txt
echo v2 > hello.txt && aws s3api put-object --bucket $BUCKET --key hello.txt --body hello.txt

aws s3api list-object-versions --bucket $BUCKET --prefix hello.txt \
  --query 'Versions[].{Id:VersionId,Latest:IsLatest,Modified:LastModified}' --output table
```

- [ ] Delete the object, which creates a delete marker:

```bash
aws s3api delete-object --bucket $BUCKET --key hello.txt
aws s3api list-object-versions --bucket $BUCKET --prefix hello.txt --query 'DeleteMarkers[].VersionId'
```

- [ ] Restore it by deleting the delete marker:

```bash
DM=$(aws s3api list-object-versions --bucket $BUCKET --prefix hello.txt --query 'DeleteMarkers[0].VersionId' --output text)
aws s3api delete-object --bucket $BUCKET --key hello.txt --version-id $DM
aws s3 cp s3://$BUCKET/hello.txt -    # prints v2
```

- [ ] Fetch the old v1 version:

```bash
V1=$(aws s3api list-object-versions --bucket $BUCKET --prefix hello.txt --query 'Versions[-1].VersionId' --output text)
aws s3api get-object --bucket $BUCKET --key hello.txt --version-id $V1 v1-copy.txt && cat v1-copy.txt
```

### B3. Static website from the CLI

- [ ] Create the website bucket and upload the pages:

```bash
WEB=saa-lab07-cliweb-$SUFFIX
aws s3 mb s3://$WEB
echo '<h1>CLI site</h1>' > index.html
echo '<h1>Oops 404</h1>'  > error.html
aws s3 cp index.html s3://$WEB/ --content-type text/html
aws s3 cp error.html s3://$WEB/ --content-type text/html
```

- [ ] Allow a public bucket policy on this bucket only:

```bash
aws s3api put-public-access-block --bucket $WEB --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

- [ ] Add the bucket policy and enable website hosting:

```bash
aws s3api put-bucket-policy --bucket $WEB --policy "{
  \"Version\":\"2012-10-17\",
  \"Statement\":[{\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::$WEB/*\"}]
}"

aws s3 website s3://$WEB/ --index-document index.html --error-document error.html
curl -s http://$WEB.s3-website-us-east-1.amazonaws.com
curl -s http://$WEB.s3-website-us-east-1.amazonaws.com/nope
```

### B4. Object metadata and HEAD

- [ ] Add metadata and inspect it:

```bash
aws s3 cp site/a.txt s3://$BUCKET/meta.txt --metadata owner=$USER,project=saa --cache-control "max-age=300"
aws s3api head-object --bucket $BUCKET --key meta.txt \
  --query '{Size:ContentLength,ETag:ETag,Meta:Metadata,Cache:CacheControl,SSE:ServerSideEncryption}'
```

> **Note:** metadata can't be edited in place. You change it by copying the object onto itself (`aws s3 cp s3://b/k s3://b/k --metadata-directive REPLACE ...`).

---

## Verify

- [ ] You recovered a deleted object by removing its delete marker.
- [ ] Your website endpoint serves `index.html` and returns `error.html` for missing keys.
- [ ] `head-object` shows your custom metadata.

## Exam tips

- **Versioning** protects against accidental overwrites and deletes. Once turned on, it can only be **suspended**, never fully turned off. It's required for replication and Object Lock.
- **MFA Delete** (root only, CLI only) adds MFA to permanent version deletion and to changing versioning state.
- S3 is **object storage**: no file locking, no partial in-place updates. For a shared POSIX file system, use **EFS**. For Windows SMB, use **FSx for Windows**.
- Static site needs HTTPS or a custom domain? Use **CloudFront + OAC** with a private bucket.

## Challenge (optional)

- [ ] Add a **redirection rule** to the website configuration that redirects `old/*` to `new/*`.

## Cleanup

- [ ] Delete every version of every object, then the bucket. `rb --force` doesn't delete old versions in a versioned bucket:

```bash
for B in $BUCKET saa-lab07-<suffix>; do
  aws s3api delete-objects --bucket $B --delete "$(aws s3api list-object-versions --bucket $B \
    --query '{Objects: [Versions, DeleteMarkers][].{Key:Key,VersionId:VersionId}}' --output json)" 2>/dev/null
  aws s3 rb s3://$B --force
done
aws s3 rb s3://$WEB --force
aws s3 rb s3://saa-lab07-web-<suffix> --force
```

- [ ] Or, in the console, use **Empty** and then **Delete** on each bucket.
- [ ] `rm -rf site index.html error.html hello.txt v1-copy.txt`

---

## Knowledge check

**Q1.** A user accidentally deleted a critical object from a versioned bucket. How do you recover it?

<details><summary>Answer</summary>

Delete the **delete marker** (the current "version" of the object). The previous version becomes current again.

</details>

**Q2.** A marketing team wants to host a static site with HTTPS on a custom domain at the lowest cost and operational overhead. What should you recommend?

<details><summary>Answer</summary>

A private S3 bucket as the origin, **CloudFront** with **Origin Access Control**, an **ACM certificate** (in us-east-1), and a **Route 53 alias** record.

</details>

**Q3.** An application writes an object and immediately reads it back. Can it get stale data?

<details><summary>Answer</summary>

**No.** S3 gives strong read-after-write consistency for all PUTs and DELETEs, and for LIST operations too.

</details>
