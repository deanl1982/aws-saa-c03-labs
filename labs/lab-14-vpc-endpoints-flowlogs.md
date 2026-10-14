# Lab 14 — VPC Endpoints, PrivateLink & Flow Logs

| | |
|---|---|
| **Module** | 4 — Networking with Amazon VPC |
| **Exam mapping** | Domain 1 · Task 1.2 (private connectivity) · Domain 4 · Task 4.4 (cost-optimised network) · Domain 1 · Task 1.3 (data access restriction) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.10–0.20 (interface endpoints are $0.01/h per AZ each; gateway endpoints are free) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-13-sg-vs-nacl.md) · [Back to index](../README.md) · [Next lab ➡](lab-15-peering-transit-gateway.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Compare **gateway endpoints** (S3, DynamoDB) with **interface endpoints** (PrivateLink ENIs).
- Reach AWS services from fully private subnets **without NAT or an IGW**.
- Lock an S3 bucket to **only** accept traffic from your VPC endpoint (`aws:SourceVpce`).
- Expose your own service to other VPCs with **PrivateLink** (endpoint service + NLB).
- Capture and query **VPC Flow Logs** to troubleshoot and audit traffic.

## Endpoint comparison

| | Gateway endpoint | Interface endpoint (PrivateLink) |
|---|---|---|
| Services | **S3, DynamoDB** only | 100+ AWS services, SaaS, your own services |
| Mechanism | Route table entry (prefix list) | ENI with a private IP in your subnet |
| Cost | **Free** | Per hour per AZ + per GB |
| Reachable from on-prem / peered VPC | No | **Yes** (over VPN/DX/TGW) |
| Security | Endpoint policy | Endpoint policy + **security group** |
| DNS | Uses the public service DNS | **Private DNS** names resolve to the ENI IPs |

---

## Prerequisites

- Labs 11–13. A private instance (`saa-private-a` or a new one) in **saa-app-a** with the `saa-ssm-ec2-role` profile.

---

## Part A — Console

### A1. Gateway endpoint for S3

- [ ] Open **VPC → Endpoints → Create endpoint**.
  - Name `saa-s3-gw`, Service: **com.amazonaws.us-east-1.s3**, Type **Gateway**
  - VPC `saa-vpc`, route tables **saa-private-rt-a** and **saa-private-rt-b**
  - Policy: **Full access** (you'll tighten it later)
- [ ] Open **Route tables → saa-private-rt-a → Routes**. A `pl-xxxx` (S3 prefix list) → `vpce-xxxx` route has appeared.

### A2. Interface endpoints for Session Manager

Session Manager needs `ssm`, `ssmmessages`, and `ec2messages`. With these endpoints, private instances work **without NAT**.

- [ ] Create SG `saa-endpoint-sg` in `saa-vpc` with inbound **HTTPS 443 from 10.0.0.0/16**.
- [ ] Create **three interface endpoints**, one each for `com.amazonaws.us-east-1.ssm`, `...ssmmessages`, and `...ec2messages`:
  - Subnets **saa-app-a** and **saa-app-b**
  - Security group `saa-endpoint-sg`
  - **Enable private DNS name**: ✔
- [ ] Wait for **Available**.

### A3. Remove the NAT and prove the endpoints work

- [ ] Delete the `0.0.0.0/0 → NAT` route from **saa-private-rt-a** and **saa-private-rt-b**, and delete the NAT gateways (to save money).
- [ ] Reboot the private instance so the SSM agent re-registers through the endpoints. Then **Connect → Session Manager**. **It works with no internet path.**
- [ ] In the session, run:

```bash
curl -s -m 5 https://checkip.amazonaws.com || echo "No internet - expected"
aws s3 ls --region us-east-1        # works via the gateway endpoint
nslookup ssm.us-east-1.amazonaws.com # resolves to 10.0.x.x private IPs
```

### A4. Restrict the bucket to the endpoint

- [ ] Create bucket `saa-lab14-<suffix>` and upload a file.
- [ ] Add this bucket policy, replacing `<vpce-id>`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "OnlyFromVpce",
    "Effect": "Deny",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::saa-lab14-<suffix>/*",
    "Condition": { "StringNotEquals": { "aws:SourceVpce": "<vpce-id>" } }
  }]
}
```

- [ ] From the private instance, `aws s3 cp s3://saa-lab14-<suffix>/<file> -` **works**.
- [ ] From your laptop or the console, downloading the object is **denied**.

> Be careful: a bucket policy like this can lock **you** out of the console too. You can always change or delete the bucket policy as an admin, because `PutBucketPolicy` isn't in the deny.

### A5. VPC Flow Logs

- [ ] Open **VPC → saa-vpc → Flow logs → Create flow log**.
  - Filter **All**, aggregation **1 minute**
  - Destination **CloudWatch Logs**, log group `saa-vpc-flowlogs`, IAM role: **Create new** (or choose an existing one with `logs:*` permissions)
- [ ] Generate some traffic (browse to an instance, or ping something), wait 5 minutes, and open the log group.
- [ ] In **CloudWatch → Logs Insights**, query rejected traffic:

```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) as attempts by srcAddr, dstPort
| sort attempts desc
| limit 20
```

> On the public internet, you'll see scanners probing your IPs within minutes.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
```

### B1. DynamoDB gateway endpoint with a restrictive policy

- [ ] Write the endpoint policy:

```bash
cat > ddb-endpoint-policy.json <<EOF
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["dynamodb:GetItem","dynamodb:PutItem","dynamodb:Query","dynamodb:DescribeTable"],
    "Resource": "arn:aws:dynamodb:us-east-1:${ACCOUNT_ID}:table/saa-*"
  }]
}
EOF
```

- [ ] Create the endpoint:

```bash
DDB_VPCE=$(aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids $APP_RT_A $APP_RT_B \
  --policy-document file://ddb-endpoint-policy.json \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=saa-ddb-gw}]' \
  --query VpcEndpoint.VpcEndpointId --output text)
```

### B2. List endpoints and their DNS names

- [ ] Look at what you've built:

```bash
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID \
  --query 'VpcEndpoints[].{Id:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State}' --output table

# Private DNS entries for an interface endpoint
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.ssm \
  --query 'VpcEndpoints[0].DnsEntries[].DnsName'
```

### B3. Add another interface endpoint (Secrets Manager) from the CLI

- [ ] Create it:

```bash
EP_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-endpoint-sg --query 'SecurityGroups[0].GroupId' --output text)
SM_VPCE=$(aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids $APP_A $APP_B --security-group-ids $EP_SG --private-dns-enabled \
  --query VpcEndpoint.VpcEndpointId --output text)
echo "export EP_SG=$EP_SG" >> ~/saa-vpc.env
```

### B4. Flow logs to S3 in Parquet

- [ ] Create a bucket and send flow logs to it:

```bash
FL_BUCKET=saa-lab14-flowlogs-$SUFFIX
aws s3 mb s3://$FL_BUCKET

aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type REJECT \
  --log-destination-type s3 --log-destination arn:aws:s3:::$FL_BUCKET \
  --destination-options FileFormat=parquet,HiveCompatiblePartitions=true,PerHourPartition=true \
  --max-aggregation-interval 60

aws ec2 describe-flow-logs --filter Name=resource-id,Values=$VPC_ID \
  --query 'FlowLogs[].{Id:FlowLogId,Dest:LogDestinationType,Traffic:TrafficType,Status:FlowLogStatus}' --output table
```

> Flow logs in **S3 Parquet** with Hive partitions are the cheapest way to keep them long term, and you can query them with Athena (Lab 36).

### B5. Query flow logs in CloudWatch from the CLI

- [ ] Run a Logs Insights query:

```bash
QID=$(aws logs start-query --log-group-name saa-vpc-flowlogs \
  --start-time $(( $(date +%s) - 3600 )) --end-time $(date +%s) \
  --query-string 'filter action="REJECT" | stats count(*) by dstPort | sort count(*) desc | limit 10' \
  --query queryId --output text)
sleep 5
aws logs get-query-results --query-id $QID --query 'results'
```

### B6. PrivateLink concept: expose your own service (read-through)

To share a service privately with other VPCs or accounts:

1. Put the service behind a **Network Load Balancer** (or Gateway LB).
2. `aws ec2 create-vpc-endpoint-service-configuration --network-load-balancer-arns <nlb-arn> --acceptance-required`
3. Consumers run `create-vpc-endpoint --vpc-endpoint-type Interface --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxxx`
4. No VPC peering, **overlapping CIDRs are fine**, and traffic goes one way (consumer → provider).

- [ ] Read and understand the four steps above. You'll build an NLB in Lab 18.

---

## Verify

- [ ] Private instances use Session Manager and S3 with **no NAT and no IGW route**.
- [ ] The S3 bucket only serves objects through your gateway endpoint.
- [ ] Flow logs arrive in CloudWatch and in S3.

## Exam tips

- **"Private subnet instances must reach S3 without going over the internet"** → **S3 gateway endpoint**. It's free and the most cost-effective answer.
- **"On-prem servers over Direct Connect need private access to S3"** → **S3 interface endpoint**. Gateway endpoints can't be reached from on-prem.
- **"Share an application with hundreds of VPCs that have overlapping CIDRs"** → **PrivateLink** (endpoint service + NLB).
- **"Make sure S3 data is only reachable from our VPC"** → bucket policy with **`aws:SourceVpce`** (or `aws:SourceVpc`).
- **Flow logs** capture metadata (IPs, ports, ACCEPT/REJECT), **not packet contents**. For packet capture, use **Traffic Mirroring**.

## Challenge (optional)

- [ ] Add an **endpoint policy** to the S3 gateway endpoint that only allows access to buckets in your own account (`aws:ResourceAccount`). This stops data exfiltration to attacker-owned buckets.

## Cleanup

- [ ] Delete the interface endpoints (they cost money) and the flow logs. **Keep** the S3 and DynamoDB gateway endpoints, since they're free:

```bash
SSM_EPS=$(aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID Name=vpc-endpoint-type,Values=Interface \
  --query 'VpcEndpoints[].VpcEndpointId' --output text)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $SSM_EPS

aws ec2 delete-flow-logs --flow-log-ids $(aws ec2 describe-flow-logs --filter Name=resource-id,Values=$VPC_ID --query 'FlowLogs[].FlowLogId' --output text)
aws logs delete-log-group --log-group-name saa-vpc-flowlogs
aws s3 rb s3://$FL_BUCKET --force
```

- [ ] Remove the bucket policy (or delete the bucket) `saa-lab14-<suffix>`: `aws s3 rb s3://saa-lab14-<suffix> --force`
- [ ] Terminate the private instance(s).
- [ ] If the NAT gateways still exist, delete them now (see the Lab 12 cleanup).

> Later labs that need private instances to install packages will say whether to recreate a NAT gateway or the SSM endpoints.

---

## Knowledge check

**Q1.** A company processes terabytes of S3 data from EC2 instances in private subnets. Most of the monthly bill is NAT gateway data processing. What's the most cost-effective fix?

<details><summary>Answer</summary>

Create an **S3 gateway VPC endpoint** and add it to the private subnets' route tables.

</details>

**Q2.** A SaaS provider wants to offer its API privately to customers' VPCs, and many customers use the same 10.0.0.0/16 CIDR. Which solution works?

<details><summary>Answer</summary>

**AWS PrivateLink**: expose the API through an NLB-backed **VPC endpoint service**. Customers create **interface endpoints**. Overlapping CIDRs don't matter.

</details>

**Q3.** Security needs to see which IP addresses are being rejected when they try to reach EC2 instances. What should you enable?

<details><summary>Answer</summary>

**VPC Flow Logs** (filter REJECT), analysed with **CloudWatch Logs Insights** or Athena.

</details>
