# Lab 38 — Cost Visibility & Optimisation Tooling

| | |
|---|---|
| **Module** | 11 — Cost Optimisation & Capstone |
| **Exam mapping** | Domain 4 · Tasks 4.1–4.4 (cost-optimised storage, compute, database, and network) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.01 (Cost Explorer API calls are $0.01 each) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-37-hybrid-migration.md) · [Back to index](../README.md) · [Next lab ➡](lab-39-capstone-cloudformation.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Allocate costs with **cost allocation tags** and **cost categories**.
- Analyse and forecast spend with **Cost Explorer**, and catch surprises with **Cost Anomaly Detection**.
- Right-size with **Compute Optimizer** and **Trusted Advisor**, and buy commitments using **Savings Plans recommendations**.
- Find and remove **waste** (idle and unattached resources) with the CLI.
- Recognise **data transfer** cost patterns and the architectural fixes for them.

## Data transfer cost cheat sheet

| Path | Typical cost | Optimisation |
|---|---|---|
| Internet → AWS (ingress) | Free | — |
| AWS → Internet (egress) | $$ per GB (tiered) | **CloudFront** (cheaper egress plus caching) |
| Same AZ, private IP | Free | Keep chatty tiers in one AZ (trading off HA) |
| **Cross-AZ** | $ per GB **each direction** | Use AZ-aware routing. Remember Multi-AZ replication costs. |
| Cross-region | $$ per GB | Replicate only what you need |
| Through a **NAT gateway** | Hourly + **per GB processed** | **Gateway endpoints** for S3/DynamoDB, interface endpoints for others |
| S3 → CloudFront | **Free** | Serve S3 content through CloudFront |
| VPC peering in the same AZ | Free | Peering instead of TGW for high-volume pairs (TGW charges per GB) |

---

## Part A — Console

### A1. Cost allocation tags

- [ ] Tag a few resources (e.g. the `saa-vpc` subnets, a bucket) with `Project=saa-labs` and `Environment=lab`. Use **Resource Groups & Tag Editor → Tag Editor → Find resources → Manage tags**.
- [ ] Open **Billing → Cost allocation tags**, select `Project` and `Environment` under *User-defined*, and **Activate**. Data appears in Cost Explorer within 24 hours, and only from activation onwards.
- [ ] Look at the **AWS-generated** tags (`aws:createdBy`) and activate them too.

### A2. Cost Explorer

- [ ] Open **Billing → Cost Explorer** and try:
  - **Group by: Service**, last 3 months, monthly. Which services cost most during these labs?
  - **Group by: Usage type**, filter service **EC2-Other**. Find `NatGateway-Hours`, `NatGateway-Bytes`, `EBS:VolumeUsage`, and `DataTransfer-Regional-Bytes` (cross-AZ).
  - **Forecast** for the end of the month.
- [ ] Save the report as `saa-by-service`.

### A3. Cost Anomaly Detection

- [ ] Open **Billing → Cost Anomaly Detection → Create monitor**: type **AWS services**, name `saa-services-monitor`.
- [ ] Add an alert subscription: **individual alerts**, threshold **$5 impact**, email.

### A4. Compute Optimizer and Trusted Advisor

- [ ] Open **Compute Optimizer → Get started → Opt in** (free). Recommendations need about 14 hours of metrics, so check back later for EC2, ASG, EBS, Lambda, ECS on Fargate, and RDS.
- [ ] Open **Trusted Advisor → Cost optimization**. With Basic support you only see some checks. Note which checks Business/Enterprise support unlocks (idle load balancers, underutilised EC2, unassociated EIPs, RI optimisation…).

### A5. Savings Plans and purchase recommendations

- [ ] Open **Billing → Savings Plans → Recommendations**. Look at the inputs: term, payment option, and lookback period. With lab usage they'll be tiny, but learn the workflow.
- [ ] Open **Reservations → Recommendations** for **RDS, ElastiCache, OpenSearch, and Redshift**. EC2/Compute Savings Plans don't cover these, so they use Reserved Instances or reserved nodes.

### A6. Data exports (CUR 2.0)

- [ ] Open **Billing → Data Exports → Create**: **Standard data export (CUR 2.0)**, Parquet, delivered to a new bucket `saa-cur-<suffix>`. It's the most granular cost data, and you can query it with Athena (Lab 36).

---

## Part B — CLI

### B1. Cost and usage by service

- [ ] Get month-to-date cost by service:

```bash
START=$(date -u +%Y-%m-01)
END=$(date -u -v+1d +%F 2>/dev/null || date -u -d tomorrow +%F)

aws ce get-cost-and-usage --time-period Start=$START,End=$END --granularity MONTHLY \
  --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[?Metrics.UnblendedCost.Amount>`0.01`].{Service:Keys[0],USD:Metrics.UnblendedCost.Amount}' \
  --output table
```

### B2. Cost by tag

- [ ] Group by the `Project` tag:

```bash
aws ce get-cost-and-usage --time-period Start=$START,End=$END --granularity MONTHLY \
  --metrics UnblendedCost --group-by Type=TAG,Key=Project \
  --query 'ResultsByTime[0].Groups[].{Tag:Keys[0],USD:Metrics.UnblendedCost.Amount}'
```

### B3. Forecast

- [ ] Forecast to the end of the month:

```bash
EOM=$(date -u -v+1m -v1d +%F 2>/dev/null || date -u -d "$(date +%Y-%m-01) +1 month" +%F)
aws ce get-cost-forecast --time-period Start=$END,End=$EOM --metric UNBLENDED_COST --granularity MONTHLY \
  --query '{Forecast:Total.Amount,Unit:Total.Unit}'
```

### B4. Waste finder: unused resources

- [ ] Run this script in **each region you used** (us-east-1 and us-west-2):

```bash
cat > waste.sh <<'EOF'
#!/usr/bin/env bash
R=${1:-us-east-1}
echo "=== Region $R ==="
echo "-- Unattached EBS volumes";  aws ec2 describe-volumes --region $R --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size,VolumeType,CreateTime]' --output text
echo "-- Unassociated Elastic IPs"; aws ec2 describe-addresses --region $R --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output text
echo "-- Stopped instances (still paying for EBS)"; aws ec2 describe-instances --region $R --filters Name=instance-state-name,Values=stopped --query 'Reservations[].Instances[].[InstanceId,InstanceType]' --output text
echo "-- NAT gateways";            aws ec2 describe-nat-gateways --region $R --filter Name=state,Values=available --query 'NatGateways[].[NatGatewayId,SubnetId]' --output text
echo "-- Load balancers";          aws elbv2 describe-load-balancers --region $R --query 'LoadBalancers[].[LoadBalancerName,Type]' --output text
echo "-- Interface endpoints";     aws ec2 describe-vpc-endpoints --region $R --filters Name=vpc-endpoint-type,Values=Interface --query 'VpcEndpoints[].[VpcEndpointId,ServiceName]' --output text
echo "-- RDS instances";           aws rds describe-db-instances --region $R --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,DBInstanceStatus]' --output text
echo "-- RDS clusters";            aws rds describe-db-clusters --region $R --query 'DBClusters[].[DBClusterIdentifier,Status]' --output text
echo "-- ElastiCache";             aws elasticache describe-replication-groups --region $R --query 'ReplicationGroups[].[ReplicationGroupId,Status]' --output text
echo "-- Snapshots you own (>30 days)"; aws ec2 describe-snapshots --region $R --owner-ids self --query "Snapshots[?StartTime<='$(date -u -v-30d +%F 2>/dev/null || date -u -d '30 days ago' +%F)'].[SnapshotId,VolumeSize,StartTime]" --output text
echo "-- Transit gateways";        aws ec2 describe-transit-gateways --region $R --query 'TransitGateways[?State==`available`].TransitGatewayId' --output text
echo "-- Log groups with no retention"; aws logs describe-log-groups --region $R --query 'logGroups[?retentionInDays==null].logGroupName' --output text
EOF
chmod +x waste.sh
./waste.sh us-east-1
./waste.sh us-west-2
```

- [ ] **Clean up anything the script finds** that's left over from earlier labs.

### B5. Compute Optimizer recommendations (after the data period)

- [ ] Fetch recommendations:

```bash
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{Id:instanceArn,Current:currentInstanceType,Finding:finding,Top:recommendationOptions[0].instanceType}' --output table
aws compute-optimizer get-ebs-volume-recommendations --query 'volumeRecommendations[].{Vol:volumeArn,Finding:finding}' --output table
```

### B6. Anomaly monitors and budgets from the CLI

- [ ] List them:

```bash
aws ce get-anomaly-monitors --query 'AnomalyMonitors[].{Name:MonitorName,Type:MonitorType}'
aws budgets describe-budgets --account-id $ACCOUNT_ID --query 'Budgets[].{Name:BudgetName,Limit:BudgetLimit.Amount,Actual:CalculatedSpend.ActualSpend.Amount}'
```

### B7. Set log retention everywhere (a quick saving)

- [ ] Give every log group without retention a 30-day policy:

```bash
for G in $(aws logs describe-log-groups --query 'logGroups[?retentionInDays==null].logGroupName' --output text); do
  aws logs put-retention-policy --log-group-name "$G" --retention-in-days 30
done
```

---

## Cost optimisation patterns (review)

- [ ] Work through each pattern below and make sure you can name the lab where you practised it:

| Domain | Pattern | Lab |
|---|---|---|
| Storage (4.1) | S3 lifecycle and Intelligent-Tiering, gp2 → gp3, EFS IA, snapshot archive | 09, 17 |
| Compute (4.2) | Right-size, Graviton, Spot, Savings Plans, ASG scale-in, Lambda/Fargate for spiky loads | 19, 20, 26, 28 |
| Database (4.3) | Aurora Serverless v2 auto-pause, DynamoDB on-demand vs provisioned, RIs for RDS, caching | 22, 23, 24 |
| Network (4.4) | Gateway endpoints instead of NAT, CloudFront for egress, avoid cross-AZ chatter, one NAT vs per-AZ trade-off | 12, 14, 33 |

---

## Verify

- [ ] Cost allocation tags are activated, and Cost Explorer reports are saved.
- [ ] An anomaly monitor and alert subscription exist.
- [ ] `waste.sh` reports nothing unexpected in either region.

## Exam tips

- **"Track costs by department/project"** → **cost allocation tags** (activate them in Billing), plus **cost categories**, plus separate accounts in an Organization.
- **"Get alerted to unusual spend automatically"** → **Cost Anomaly Detection**. **"Alert on a fixed threshold"** → **AWS Budgets**.
- **"Recommendations to right-size EC2, EBS, Lambda"** → **Compute Optimizer**.
- **"Most granular billing data for custom analytics"** → **CUR 2.0 / Data Exports** + Athena.
- **"Estimate the cost of a new architecture"** → **AWS Pricing Calculator**.
- Exam questions often contrast "**most cost-effective**" with "**least operational overhead**". Read which one is being asked.

## Challenge (optional)

- [ ] Query your CUR 2.0 export with Athena to find the top 10 resource IDs by cost this month.

## Cleanup

- [ ] **Keep** the cost allocation tags, anomaly monitor, budgets, and Compute Optimizer opt-in. They're free and useful.
- [ ] Delete the CUR export and bucket if you don't want ongoing S3 storage: `aws s3 rb s3://saa-cur-<suffix> --force` (delete the export in the console first).
- [ ] `rm waste.sh`

---

## Knowledge check

**Q1.** A company's bill shows high `NatGateway-Bytes` charges. Most traffic from private subnets goes to S3 and DynamoDB. What's the most cost-effective fix?

<details><summary>Answer</summary>

Add **gateway VPC endpoints** for **S3 and DynamoDB**. They're free, and that traffic then skips the NAT gateway.

</details>

**Q2.** Finance wants each team to see its own AWS spend in Cost Explorer. Teams share one account. What should you do?

<details><summary>Answer</summary>

Enforce a **`Team` tag** (tag policies, or SCP conditions), **activate it as a cost allocation tag**, and filter or group by it in Cost Explorer. Better long term: **separate accounts per team** in AWS Organizations.

</details>

**Q3.** A web app serves 50 TB/month of images straight from S3 to global users. How can the company cut data-transfer cost and improve performance?

<details><summary>Answer</summary>

Serve the images through **Amazon CloudFront**. S3 → CloudFront transfer is free, CloudFront egress is cheaper than S3 internet egress, and caching cuts origin requests.

</details>
