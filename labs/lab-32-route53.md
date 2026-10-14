# Lab 32 — Route 53: DNS Routing Policies & Health Checks

| | |
|---|---|
| **Module** | 9 — High Availability, Disaster Recovery & Global Delivery |
| **Exam mapping** | Domain 2 · Task 2.2 (DNS failover, multi-region HA) · Domain 3 · Task 3.4 (latency-based global routing) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$1 (hosted zone at $0.50/month, health checks at $0.50–0.75/month prorated, one t3.micro) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-31-waf-shield-guardduty.md) · [Back to index](../README.md) · [Next lab ➡](lab-33-cloudfront-global-accelerator.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Create **public** and **private** hosted zones, and explain **alias** vs **CNAME** records.
- Configure **simple, weighted, latency, failover, geolocation, geoproximity, multivalue**, and **IP-based** routing.
- Attach **health checks** (endpoint, calculated, CloudWatch alarm) to drive DNS failover.
- Explain how **TTL** affects failover speed and caching.

## Routing policies

| Policy | Behaviour | Exam trigger |
|---|---|---|
| **Simple** | One record, one or more values (random order), **no health checks** | Single resource |
| **Weighted** | Split traffic by weight | Blue/green, canary, A/B testing |
| **Latency** | Route to the region with the lowest latency **for the user** | Global app, best performance |
| **Failover** | Primary/secondary based on a health check | Active-passive DR |
| **Geolocation** | By user's continent or country (plus a default) | Localised content, legal or licensing restrictions |
| **Geoproximity** | By distance, with an adjustable **bias** (Traffic Flow) | Shift more traffic to one region |
| **Multivalue answer** | Up to 8 **healthy** records returned | Simple client-side load balancing with health checks |
| **IP-based** | By client CIDR | Route ISPs or known networks to specific endpoints |

**Alias vs CNAME:** an alias is free to query, works at the **zone apex** (`example.com`), and points to AWS resources (ALB, CloudFront, S3 website, API Gateway, another record). A CNAME can't be used at the apex.

---

## Lab setup: two "regional" web endpoints

You'll use **S3 static website endpoints** in two regions as stand-ins for regional applications. Everyone can reach them, and Route 53 health checks can monitor them.

- [ ] Create two website buckets, one per region:

```bash
for R in us-east-1 us-west-2; do
  B=saa-lab32-$R-$SUFFIX
  aws s3 mb s3://$B --region $R
  echo "<h1>Served from $R</h1>" > index.html
  aws s3 cp index.html s3://$B/ --content-type text/html --region $R
  aws s3api put-public-access-block --bucket $B --region $R --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false
  aws s3api put-bucket-policy --bucket $B --region $R --policy "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::$B/*\"}]}"
  aws s3 website s3://$B/ --index-document index.html --region $R
done
EAST=saa-lab32-us-east-1-$SUFFIX.s3-website-us-east-1.amazonaws.com
WEST=saa-lab32-us-west-2-$SUFFIX.s3-website-us-west-2.amazonaws.com
curl -s http://$EAST; curl -s http://$WEST
```

### Choose your zone path

| Path | Requirement | How you test |
|---|---|---|
| **A: Private hosted zone** (default) | Nothing extra | `dig` from an instance inside `saa-vpc` |
| **B: Public hosted zone** | A domain you own (or register one in **Route 53 → Registered domains**; prices vary by TLD) | `dig` and a browser from anywhere |

The steps below use `ZONE=saa.internal` (path A). For path B, substitute your domain.

- [ ] Launch a test client for path A (public subnet, Session Manager):

```bash
source ~/saa-vpc.env
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
DNS_CLIENT=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PUB_A \
  --iam-instance-profile Name=saa-ssm-ec2-role --user-data '#!/bin/bash
dnf -y install bind-utils' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-dns-client}]' --query 'Instances[0].InstanceId' --output text)
```

---

## Part A — Console

### A1. Create the hosted zone

- [ ] Open **Route 53 → Hosted zones → Create hosted zone**.
  - Path A: domain `saa.internal`, type **Private**, VPC `saa-vpc` (us-east-1)
  - Path B: your domain, type **Public**. If the domain is registered elsewhere, update its **NS records** at the registrar.
- [ ] Look at the default **SOA** and **NS** records.

### A2. Weighted routing

- [ ] Create two records named `weighted.saa.internal`, type **CNAME**, TTL **30**:
  - Value `$EAST`, routing **Weighted**, weight **80**, record ID `east`
  - Value `$WEST`, routing **Weighted**, weight **20**, record ID `west`
- [ ] On `saa-dns-client`, run:

```bash
for i in $(seq 1 20); do dig +short weighted.saa.internal | head -1; done | sort | uniq -c
```

> Roughly 80/20. The resolver cache and TTL affect how evenly it splits.

### A3. Health check and failover routing

- [ ] Open **Health checks → Create health check**:
  - Name `saa-hc-east`, endpoint **domain name** `$EAST`, protocol **HTTP**, port 80, path `/index.html`
  - Advanced: request interval **10 s** (fast), failure threshold **2**
  - Create an alarm: **No** (optional)
- [ ] Create two records named `app.saa.internal`, type CNAME, TTL **10**:
  - Value `$EAST`, routing **Failover**, type **Primary**, health check `saa-hc-east`, ID `primary`
  - Value `$WEST`, routing **Failover**, type **Secondary**, ID `secondary`
- [ ] Run `dig +short app.saa.internal`. It returns the **east** endpoint.
- [ ] **Simulate a regional failure:** delete `index.html` from the **east** bucket. The health check gets a 404 and goes **Unhealthy** in about 30 s.
- [ ] Run `dig +short app.saa.internal` again. It now returns **west**.
- [ ] Re-upload `index.html` to east and watch it fail back.

> Route 53 health checkers run on the **public internet**, so they can't reach private IPs directly. For private resources, use a **CloudWatch-alarm-based health check**.

### A4. Latency routing

- [ ] Create two records named `fast.saa.internal`, type CNAME:
  - `$EAST`, routing **Latency**, region **us-east-1**, ID `lat-east`
  - `$WEST`, routing **Latency**, region **us-west-2**, ID `lat-west`
- [ ] From the client in us-east-1, `dig +short fast.saa.internal` returns **east**.

### A5. Geolocation (public zone, path B only)

- [ ] Create `geo.<domain>` records: **Europe** → west, **North America** → east, plus **Default** → east.
- [ ] Test with an EDNS client-subnet query, e.g. `dig +short geo.<domain> @<ns-server> +subnet=81.2.69.0/24` (a UK range).

> Always add a **Default** geolocation record, or users from unmatched locations get no answer.

---

## Part B — CLI

```bash
ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name saa.internal \
  --query 'HostedZones[0].Id' --output text | cut -d/ -f3)
```

### B1. List records

- [ ] Look at what's in the zone:

```bash
aws route53 list-resource-record-sets --hosted-zone-id $ZONE_ID \
  --query 'ResourceRecordSets[].{Name:Name,Type:Type,Id:SetIdentifier,Weight:Weight,Failover:Failover,Region:Region,Value:ResourceRecords[0].Value}' \
  --output table
```

### B2. Create a multivalue answer set with health checks

- [ ] Create a health check for the west endpoint:

```bash
HC_WEST=$(aws route53 create-health-check --caller-reference saa-west-$(date +%s) \
  --health-check-config "Type=HTTP,FullyQualifiedDomainName=$WEST,Port=80,ResourcePath=/index.html,RequestInterval=30,FailureThreshold=3" \
  --query HealthCheck.Id --output text)
HC_EAST=$(aws route53 list-health-checks --query "HealthChecks[?HealthCheckConfig.FullyQualifiedDomainName=='$EAST'].Id | [0]" --output text)
```

- [ ] Create two multivalue A records, each tied to a health check. Documentation IPs stand in for servers:

```bash
cat > mv.json <<EOF
{"Changes":[
 {"Action":"UPSERT","ResourceRecordSet":{"Name":"multi.saa.internal","Type":"A","SetIdentifier":"mv-1","MultiValueAnswer":true,"TTL":30,
   "ResourceRecords":[{"Value":"192.0.2.10"}],"HealthCheckId":"$HC_EAST"}},
 {"Action":"UPSERT","ResourceRecordSet":{"Name":"multi.saa.internal","Type":"A","SetIdentifier":"mv-2","MultiValueAnswer":true,"TTL":30,
   "ResourceRecords":[{"Value":"192.0.2.20"}],"HealthCheckId":"$HC_WEST"}}
]}
EOF
CHANGE=$(aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch file://mv.json --query ChangeInfo.Id --output text)
aws route53 wait resource-record-sets-changed --id $CHANGE
```

> On the client, `dig +short multi.saa.internal` returns **both** IPs while both health checks pass, and only the healthy one otherwise.

### B3. Check health check status from the CLI

- [ ] See what each Route 53 checker region reports:

```bash
aws route53 get-health-check-status --health-check-id $HC_EAST \
  --query 'HealthCheckObservations[].{Region:Region,Status:StatusReport.Status}' --output table
```

### B4. Calculated health check

- [ ] Create a check that's healthy only if **both** child checks are healthy:

```bash
aws route53 create-health-check --caller-reference saa-calc-$(date +%s) \
  --health-check-config "{\"Type\":\"CALCULATED\",\"ChildHealthChecks\":[\"$HC_EAST\",\"$HC_WEST\"],\"HealthThreshold\":2}" \
  --query HealthCheck.Id --output text
```

### B5. Shift weights for a canary release

- [ ] Change the weighted records to send 100% to west:

```bash
cat > shift.json <<EOF
{"Changes":[
 {"Action":"UPSERT","ResourceRecordSet":{"Name":"weighted.saa.internal","Type":"CNAME","SetIdentifier":"east","Weight":0,"TTL":30,"ResourceRecords":[{"Value":"$EAST"}]}},
 {"Action":"UPSERT","ResourceRecordSet":{"Name":"weighted.saa.internal","Type":"CNAME","SetIdentifier":"west","Weight":100,"TTL":30,"ResourceRecords":[{"Value":"$WEST"}]}}
]}
EOF
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch file://shift.json
```

### B6. Route 53 Resolver for hybrid DNS (read-through)

- [ ] Remember the two hybrid patterns:
  - **Inbound endpoint:** on-prem DNS servers forward queries **into** the VPC, to resolve private hosted zones.
  - **Outbound endpoint + forwarding rules:** VPC resources resolve **on-prem** domains (e.g. `corp.example.com`).

```bash
aws route53resolver list-resolver-endpoints
```

---

## Verify

- [ ] Weighted queries split roughly by weight.
- [ ] Failover switched to west when the east health check failed, then switched back.
- [ ] Multivalue returns only the healthy IPs.

## Exam tips

- **"Active-passive DR across regions"** → **failover routing** + health checks. **"Active-active, best performance"** → **latency routing** (with health checks).
- **"Serve country-specific content / block countries at DNS"** → **geolocation**. **"Gradually move traffic to a new region"** → **weighted** (or geoproximity bias).
- **Zone apex to an ALB or CloudFront** → **alias record**, never a CNAME.
- **Lower TTLs** = faster failover but more DNS queries. Clients and resolvers may cache longer than the TTL.
- **Hybrid DNS** → **Route 53 Resolver inbound and outbound endpoints**.
- **Health checks for private resources** → **CloudWatch alarm-based** health checks.

## Challenge (optional)

- [ ] Build a **Traffic Flow** policy with geoproximity and a bias of +25 towards us-west-2, and visualise it in the editor. Traffic policy records cost $50/month, so delete it the same day.

## Cleanup

- [ ] Delete all non-default records (everything except SOA and NS):

```bash
aws route53 list-resource-record-sets --hosted-zone-id $ZONE_ID --output json | \
  jq '{Changes: [.ResourceRecordSets[] | select(.Type != "SOA" and .Type != "NS") | {Action:"DELETE", ResourceRecordSet: .}]}' > del.json
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch file://del.json
```

- [ ] Delete the hosted zone (unless it's your public domain's zone and you want to keep it) and all health checks:

```bash
aws route53 delete-hosted-zone --id $ZONE_ID
for H in $(aws route53 list-health-checks --query 'HealthChecks[].Id' --output text); do aws route53 delete-health-check --health-check-id $H; done
```

- [ ] Delete the buckets and the test client:

```bash
aws s3 rb s3://saa-lab32-us-east-1-$SUFFIX --force
aws s3 rb s3://saa-lab32-us-west-2-$SUFFIX --force --region us-west-2
aws ec2 terminate-instances --instance-ids $DNS_CLIENT
```

- [ ] `rm -f mv.json shift.json del.json index.html`

---

## Knowledge check

**Q1.** A company runs its application in us-east-1 and wants traffic to fail over automatically to a standby stack in eu-west-1 if us-east-1 becomes unhealthy. Which Route 53 configuration?

<details><summary>Answer</summary>

**Failover routing**: the primary record (us-east-1) is associated with a **health check**, and the secondary record points to eu-west-1.

</details>

**Q2.** A global application runs in three regions. Users should be sent to the region that gives them the fastest response. Which routing policy?

<details><summary>Answer</summary>

**Latency-based routing** (with health checks).

</details>

**Q3.** How do you point `example.com` (the zone apex) at an Application Load Balancer?

<details><summary>Answer</summary>

Create a Route 53 **alias A (and AAAA) record** targeting the ALB. CNAMEs aren't allowed at the zone apex.

</details>
