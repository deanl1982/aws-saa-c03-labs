# Lab 33 — CloudFront & Global Accelerator

| | |
|---|---|
| **Module** | 9 — High Availability, Disaster Recovery & Global Delivery |
| **Exam mapping** | Domain 3 · Task 3.4 (edge networking, global performance) · Domain 1 · Task 1.2 (origin protection) · Domain 4 · Task 4.4 (data transfer cost) · Domain 2 · Task 2.2 (origin failover) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.10–0.50 (CloudFront is mostly free tier; Global Accelerator is $0.025/h plus data transfer premium) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-32-route53.md) · [Back to index](../README.md) · [Next lab ➡](lab-34-backup-dr.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Serve a **private S3 bucket** through **CloudFront** with **Origin Access Control (OAC)**.
- Tune caching with **cache policies**, TTLs, and **invalidations**, and read cache hit/miss headers.
- Configure **origin groups** for origin failover, and **geo restriction**.
- Restrict content with **signed URLs/cookies**, and run code at the edge with **CloudFront Functions** or **Lambda@Edge**.
- Choose between **CloudFront** and **Global Accelerator**.

## CloudFront vs Global Accelerator

| | CloudFront | Global Accelerator |
|---|---|---|
| Layer | HTTP/HTTPS (L7) **CDN** | TCP/UDP (L4) **network accelerator** |
| Caching | **Yes**, at 600+ edge locations | No caching |
| IPs | Dynamic | **2 static anycast IPs** |
| Failover | Origin groups (seconds) | **Instant** regional failover with health checks |
| Use | Static and dynamic web content, APIs, video | Gaming, IoT, VoIP, non-HTTP traffic, needs static IPs or fast regional failover |

---

## Part A — Console: CloudFront + private S3

### A1. Private origin bucket

- [ ] Create bucket `saa-lab33-origin-<suffix>` (us-east-1) with **Block Public Access ON** (it stays private).
- [ ] Upload content:

```bash
mkdir site && cd site
echo '<html><body><h1>CloudFront v1</h1><img src="logo.svg"></body></html>' > index.html
echo '<svg xmlns="http://www.w3.org/2000/svg" width="80" height="80"><circle cx="40" cy="40" r="35" fill="orange"/></svg>' > logo.svg
aws s3 sync . s3://saa-lab33-origin-<suffix>/ && cd ..
```

### A2. Create the distribution with OAC

- [ ] Open **CloudFront → Create distribution**.
  - Origin domain: pick the **S3 bucket** (not the website endpoint)
  - Origin access: **Origin access control settings (recommended)** → **Create new OAC** (sign requests)
  - Viewer protocol policy: **Redirect HTTP to HTTPS**
  - Cache policy: **CachingOptimized**
  - WAF: **Do not enable** (to save cost)
  - Default root object: `index.html`
- [ ] Copy the **bucket policy** CloudFront offers and apply it to the bucket. It only allows `cloudfront.amazonaws.com` with `AWS:SourceArn` = your distribution.
- [ ] Wait about 5 minutes for **Deployed**, then browse to `https://dxxxx.cloudfront.net`.
- [ ] Try the S3 object URL directly. It's **AccessDenied**, because only CloudFront can read the bucket.

### A3. Caching behaviour

- [ ] Look at the cache headers:

```bash
CF=dxxxx.cloudfront.net
curl -sI https://$CF/index.html | grep -iE 'x-cache|age|via'
curl -sI https://$CF/index.html | grep -iE 'x-cache|age'   # second request: Hit from cloudfront
```

- [ ] Change `index.html` to **v2** and upload it. CloudFront **still serves v1** (cached).
- [ ] Open **Invalidations → Create**, path `/index.html`. Refresh and you get v2.

> **Better practice:** **versioned file names** (`app.v2.js`) instead of invalidations. The first 1,000 invalidation paths per month are free.

### A4. Behaviours, geo restriction, and error pages

- [ ] Add a **behaviour** for path `*.svg` with cache policy **CachingOptimized** and a longer TTL (e.g. a custom policy with min 1 day).
- [ ] Open **Security → CloudFront geographic restrictions → Edit** and allow-list your country only. Optionally block one country to test with a VPN.
- [ ] Open **Error pages → Create custom error response**: 403 → `/index.html` with 200. This is the common SPA pattern.

### A5. Origin failover (origin group)

- [ ] Create a second bucket `saa-lab33-failover-<suffix>` in **us-west-2** containing `index.html` that says `FAILOVER ORIGIN`. Add it as an origin with OAC and update its bucket policy.
- [ ] Open **Origins → Create origin group**: primary = the us-east-1 bucket, secondary = the us-west-2 bucket, failover criteria **403, 404, 500, 502, 503, 504**.
- [ ] Edit the **default behaviour** to use the origin group.
- [ ] Delete `index.html` from the primary bucket, **invalidate** `/*`, and browse. You get `FAILOVER ORIGIN`.

---

## Part B — CLI

```bash
DIST_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName,'saa-lab33')].Id | [0]" --output text)
```

### B1. Inspect the distribution

- [ ] Look at its key settings:

```bash
aws cloudfront get-distribution --id $DIST_ID \
  --query 'Distribution.{Status:Status,Domain:DomainName,Origins:DistributionConfig.Origins.Items[].DomainName,OAC:DistributionConfig.Origins.Items[0].OriginAccessControlId,PriceClass:DistributionConfig.PriceClass}'
```

### B2. Create an invalidation and wait for it

- [ ] Invalidate everything:

```bash
INV=$(aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*" --query Invalidation.Id --output text)
aws cloudfront wait invalidation-completed --distribution-id $DIST_ID --id $INV
```

### B3. CloudFront Function: add security headers at the edge

- [ ] Write the function:

```bash
cat > headers.js <<'EOF'
function handler(event) {
  var response = event.response;
  var h = response.headers;
  h['strict-transport-security'] = { value: 'max-age=63072000; includeSubDomains; preload' };
  h['x-content-type-options']    = { value: 'nosniff' };
  h['x-frame-options']           = { value: 'DENY' };
  h['x-saa-lab']                 = { value: 'edge-function' };
  return response;
}
EOF
```

- [ ] Create, test, and publish it:

```bash
ETAG=$(aws cloudfront create-function --name saa-sec-headers \
  --function-config Comment="security headers",Runtime=cloudfront-js-2.0 \
  --function-code fileb://headers.js --query ETag --output text)
aws cloudfront test-function --name saa-sec-headers --if-match $ETAG --stage DEVELOPMENT \
  --event-object '{"version":"1.0","context":{"eventType":"viewer-response"},"viewer":{"ip":"1.2.3.4"},"request":{"method":"GET","uri":"/","headers":{}},"response":{"statusCode":200,"headers":{}}}' \
  --query 'TestResult.FunctionOutput' --output text | jq .
aws cloudfront publish-function --name saa-sec-headers --if-match $ETAG
```

- [ ] Associate it with the default behaviour's **viewer response** in the console (Behaviours → Edit → Function associations), then:

```bash
curl -sI https://$CF/ | grep -i x-saa-lab
```

> **CloudFront Functions:** lightweight JS, sub-millisecond, viewer events only, very cheap (header rewrites, redirects, URL normalisation). **Lambda@Edge:** Node/Python, up to 5–30 s, network access, origin events too (auth, dynamic origin selection).

### B4. Signed URLs (concept + key setup)

- [ ] Generate a key pair and register the public key:

```bash
openssl genrsa -out cf_private.pem 2048
openssl rsa -pubout -in cf_private.pem -out cf_public.pem
PK_ID=$(aws cloudfront create-public-key --public-key-config "CallerReference=saa-$(date +%s),Name=saa-pubkey,EncodedKey=$(cat cf_public.pem | sed ':a;N;$!ba;s/\n/\\n/g')" --query PublicKey.Id --output text 2>/dev/null) || echo "Use the console (Public keys → Create) if the CLI encoding gives you trouble"
```

- [ ] Remember the rest of the flow: create a **key group** with the key, set the behaviour to **Restrict viewer access → key group**, then sign URLs with `aws cloudfront sign --url https://$CF/index.html --key-pair-id $PK_ID --private-key file://cf_private.pem --date-less-than 2026-12-31`.

> **Signed URL** = one file. **Signed cookies** = many files (e.g. a premium video library).

### B5. Global Accelerator in front of an EC2 endpoint (~$0.03)

- [ ] Launch a public web instance with an EIP:

```bash
source ~/saa-vpc.env
GA_I=$(aws ec2 run-instances --launch-template LaunchTemplateId=$LT_ID,Version=1 --subnet-id $PUB_A \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-running --instance-ids $GA_I
```

- [ ] Create the accelerator (its API lives in **us-west-2**):

```bash
GA=$(aws globalaccelerator create-accelerator --name saa-ga --ip-address-type IPV4 --enabled \
  --region us-west-2 --query 'Accelerator.AcceleratorArn' --output text)
LIS=$(aws globalaccelerator create-listener --accelerator-arn $GA --protocol TCP \
  --port-ranges FromPort=80,ToPort=80 --region us-west-2 --query 'Listener.ListenerArn' --output text)
aws globalaccelerator create-endpoint-group --listener-arn $LIS --endpoint-group-region us-east-1 \
  --endpoint-configurations EndpointId=$GA_I,Weight=100,ClientIPPreservationEnabled=true \
  --health-check-protocol TCP --health-check-port 80 --region us-west-2
```

- [ ] Get the two static anycast IPs and call them:

```bash
aws globalaccelerator describe-accelerator --accelerator-arn $GA --region us-west-2 \
  --query 'Accelerator.IpSets[0].IpAddresses'
# after about 2-5 minutes:
curl -s http://<ga-ip-1>/
```

> Users reach the nearest AWS edge on the **anycast IP**, then ride the **AWS backbone** to the endpoint. For multi-region, add an endpoint group per region and control traffic with **traffic dials**.

---

## Verify

- [ ] S3 objects are only reachable through CloudFront (direct S3 access is denied).
- [ ] `x-cache: Hit from cloudfront` appears on repeat requests, and an invalidation brought in fresh content.
- [ ] Origin failover served the secondary bucket, and Global Accelerator served the EC2 page on a static IP.

## Exam tips

- **"Serve static content globally with low latency, and keep the S3 bucket private"** → **CloudFront + OAC** (OAI is legacy).
- **"Reduce load on the origin / cut data-transfer-out cost"** → CloudFront caching (data transfer from S3 to CloudFront is free).
- **"Users in some countries must be blocked"** → **CloudFront geo restriction** (or WAF geo match).
- **"Premium content only for paying users"** → **signed URLs or signed cookies**.
- **"Static IPs for a global app, fast failover, TCP/UDP"** → **Global Accelerator**.
- **"Upload acceleration to S3"** → **S3 Transfer Acceleration** (it uses CloudFront edges).
- **Custom HTTPS domain on CloudFront** → ACM certificate in **us-east-1** + Route 53 alias.
- **"Protect a custom origin (ALB) so it only accepts CloudFront traffic"** → CloudFront **managed prefix list** in the ALB SG + a secret custom header checked by an ALB rule or WAF. VPC origins also let CloudFront reach a **private** ALB.

## Challenge (optional)

- [ ] Put CloudFront in front of an **ALB** origin with a **VPC origin** (a private ALB), and compare the response times of cacheable and dynamic paths.

## Cleanup

- [ ] **Disable** the distribution, wait for **Deployed**, then delete it:

```bash
CFG=$(aws cloudfront get-distribution-config --id $DIST_ID)
ETAG=$(echo "$CFG" | jq -r .ETag)
echo "$CFG" | jq '.DistributionConfig.Enabled=false | .DistributionConfig' > dist.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG --distribution-config file://dist.json
aws cloudfront wait distribution-deployed --id $DIST_ID
aws cloudfront delete-distribution --id $DIST_ID --if-match $(aws cloudfront get-distribution --id $DIST_ID --query ETag --output text)
```

- [ ] Delete the function, public key, OAC, and buckets:

```bash
aws cloudfront delete-function --name saa-sec-headers --if-match $(aws cloudfront describe-function --name saa-sec-headers --query ETag --output text)
# delete the public key and key group (console) and the OAC (Security → Origin access)
aws s3 rb s3://saa-lab33-origin-<suffix> --force
aws s3 rb s3://saa-lab33-failover-<suffix> --force --region us-west-2
```

- [ ] Delete Global Accelerator. It must be disabled before deletion:

```bash
EG=$(aws globalaccelerator list-endpoint-groups --listener-arn $LIS --region us-west-2 --query 'EndpointGroups[0].EndpointGroupArn' --output text)
aws globalaccelerator delete-endpoint-group --endpoint-group-arn $EG --region us-west-2
aws globalaccelerator delete-listener --listener-arn $LIS --region us-west-2
aws globalaccelerator update-accelerator --accelerator-arn $GA --no-enabled --region us-west-2
sleep 60
aws globalaccelerator delete-accelerator --accelerator-arn $GA --region us-west-2
aws ec2 terminate-instances --instance-ids $GA_I
```

- [ ] `rm -rf site headers.js cf_*.pem dist.json`

---

## Knowledge check

**Q1.** A company serves images from S3 to users worldwide. It wants lower latency and to make sure users can't bypass the CDN to reach S3 directly. What should it do?

<details><summary>Answer</summary>

Use **CloudFront** with the S3 bucket as origin and **Origin Access Control**, keep the bucket private, and allow only the distribution in the bucket policy.

</details>

**Q2.** A multiplayer game uses UDP, needs static IP addresses for allow-listing, and must fail over between regions within seconds. Which service?

<details><summary>Answer</summary>

**AWS Global Accelerator.**

</details>

**Q3.** After deploying a new `app.js` to S3, users still get the old version through CloudFront. What are two ways to fix this?

<details><summary>Answer</summary>

**Create an invalidation** for `/app.js`, or (better) use **versioned file names** (`app.v2.js`) so new content gets a new cache key.

</details>
