# Lab 20 — EC2 Purchasing Options & Placement Groups

| | |
|---|---|
| **Module** | 5 — Compute, Storage Volumes, Load Balancing & Scaling |
| **Exam mapping** | Domain 4 · Task 4.2 (cost-optimised compute) · Domain 3 · Task 3.2 (high-performing compute) · Domain 2 · Task 2.2 (spread for resilience) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.10 (short-lived Spot and On-Demand instances) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-19-auto-scaling.md) · [Back to index](../README.md) · [Next lab ➡](lab-21-rds.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Choose between **On-Demand, Savings Plans, Reserved Instances, Spot, Dedicated Hosts, Dedicated Instances**, and **Capacity Reservations**.
- Launch and manage **Spot Instances**, and design for **interruptions**.
- Build a cost-optimised **mixed-instances ASG** (an On-Demand base plus Spot across several types).
- Choose and create **cluster, spread, and partition** placement groups.

## Purchasing options

| Option | Discount vs On-Demand | Commitment | Notes / exam triggers |
|---|---|---|---|
| **On-Demand** | — | None | Short-term, unpredictable, can't be interrupted |
| **Compute Savings Plan** | up to ~66% | $/hour for 1 or 3 years | **Most flexible:** any family, region, OS, plus **Fargate and Lambda** |
| **EC2 Instance Savings Plan** | up to ~72% | $/hour for 1 or 3 years | Locked to an instance family in a region. Size, OS, and tenancy are flexible. |
| **Standard RI** | up to ~72% | 1 or 3 years | Specific type. Can be sold on the RI Marketplace. **Zonal RIs reserve capacity.** |
| **Convertible RI** | up to ~66% | 1 or 3 years | Can be exchanged for other families |
| **Spot** | up to **90%** | None | **Can be interrupted with a 2-minute warning.** For stateless, fault-tolerant, flexible jobs. |
| **Dedicated Host** | — | On-Demand or reservation | **BYOL** with per-socket/core licences, compliance, host visibility |
| **Dedicated Instance** | — | — | Hardware dedicated to your account, no host visibility |
| **On-Demand Capacity Reservation** | None (combine with a Savings Plan) | None, cancel any time | **Guarantee capacity in a specific AZ** (e.g. for a DR event) |

## Placement groups

| Type | Layout | Use case | Limits |
|---|---|---|---|
| **Cluster** | Packed close together in **one AZ** | HPC, lowest latency, 10–100 Gbps between nodes | Single AZ = correlated failure risk |
| **Spread** | Each instance on **distinct hardware** | Small number of critical instances | **7 instances per AZ** per group |
| **Partition** | Groups of instances on separate racks | Large distributed systems: **HDFS, HBase, Cassandra, Kafka** | Up to 7 partitions per AZ, hundreds of instances |

---

## Part A — Console

### A1. Explore Spot pricing

- [ ] Open **EC2 → Spot Requests → Pricing history**. Pick `m5.large`, Linux, last 3 months, and compare the AZs.
- [ ] Open **Spot placement score** (in the Spot Requests page). Enter a target of 10 instances across `m5.large`, `m5a.large`, `m6i.large` and see which regions or AZs score highest.

### A2. Launch a Spot Instance

- [ ] Open **EC2 → Launch instance**: `saa-spot-test`, AL2023, `t3.micro`, subnet **saa-public-a**.
- [ ] Under **Advanced details → Purchasing option**, choose **Spot instances**.
  - Request type: **One-time**, interruption behaviour: **Terminate**
- [ ] Launch, then open **Spot Requests** and look at the request state (`fulfilled`).
- [ ] Note how the instance's **Lifecycle** column says `spot`.

> Spot interruption notices arrive through **instance metadata** (`/latest/meta-data/spot/instance-action`) and **EventBridge** (`EC2 Spot Instance Interruption Warning`). Applications should checkpoint work when they see one.

### A3. Savings Plans and RI recommendations

- [ ] Open **Billing and Cost Management → Savings Plans → Recommendations**. They may be empty for a lab account, so read through the options: Compute vs EC2 Instance, 1 vs 3 years, No/Partial/All upfront.
- [ ] Open **EC2 → Reserved Instances → Purchase Reserved Instances**. Search `m5.large`, Linux, and compare 1-year Standard **No Upfront** with 3-year **All Upfront** effective hourly rates. **Don't purchase.**

### A4. Placement groups

- [ ] Open **EC2 → Placement groups → Create** and create:
  - `saa-pg-cluster`, strategy **Cluster**
  - `saa-pg-spread`, strategy **Spread**, spread level **Rack**
  - `saa-pg-partition`, strategy **Partition**, **3** partitions
- [ ] Launch 2 × `t3.micro` into `saa-pg-spread`, one in each public subnet.
- [ ] *(Note)* Cluster placement groups need instance types that support enhanced networking. `t3` is allowed, but `c5n` or `c7gn` is typical in production.

### A5. Capacity reservation

- [ ] Open **EC2 → Capacity Reservations → Create**: `t3.micro`, us-east-1a, Linux, **1 instance**, end **manually**.
- [ ] Look at the "Instance eligibility" options (*open* vs *targeted*).
- [ ] **Cancel it immediately.** You pay On-Demand rates for reserved capacity whether or not you use it.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
```

### B1. Spot price history

- [ ] Look at the last day of prices:

```bash
aws ec2 describe-spot-price-history --instance-types m5.large m6i.large c5.large \
  --product-descriptions "Linux/UNIX" --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%S) \
  --query 'SpotPriceHistory[].{Type:InstanceType,AZ:AvailabilityZone,Price:SpotPrice}' --output table | head -30
```

### B2. Launch a Spot Instance from the CLI

- [ ] Launch it:

```bash
SPOT_ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PUB_A \
  --instance-market-options '{"MarketType":"spot","SpotOptions":{"SpotInstanceType":"one-time","InstanceInterruptionBehavior":"terminate"}}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-spot-cli}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 describe-instances --instance-ids $SPOT_ID \
  --query 'Reservations[0].Instances[0].{Lifecycle:InstanceLifecycle,SpotReq:SpotInstanceRequestId,State:State.Name}'
```

### B3. Spot placement score from the CLI

- [ ] Get scores per AZ:

```bash
aws ec2 get-spot-placement-scores --target-capacity 20 --single-availability-zone \
  --instance-types m5.large m5a.large m6i.large m6a.large --region-names us-east-1 \
  --query 'SpotPlacementScores[].{AZ:AvailabilityZoneId,Score:Score}' --output table
```

### B4. Mixed-instances ASG (On-Demand base + Spot)

- [ ] Create the ASG:

```bash
ASG_LT=$(aws ec2 describe-launch-templates --launch-template-names saa-asg-lt --query 'LaunchTemplates[0].LaunchTemplateId' --output text)

aws autoscaling create-auto-scaling-group --auto-scaling-group-name saa-mixed-asg \
  --vpc-zone-identifier "$APP_A,$APP_B" \
  --min-size 4 --max-size 8 --desired-capacity 4 \
  --capacity-rebalance \
  --mixed-instances-policy "{
    \"LaunchTemplate\": {
      \"LaunchTemplateSpecification\": {\"LaunchTemplateId\": \"$ASG_LT\", \"Version\": \"\$Latest\"},
      \"Overrides\": [
        {\"InstanceType\": \"t3.micro\"},
        {\"InstanceType\": \"t3a.micro\"},
        {\"InstanceType\": \"t3.small\"},
        {\"InstanceType\": \"t3a.small\"}
      ]
    },
    \"InstancesDistribution\": {
      \"OnDemandBaseCapacity\": 1,
      \"OnDemandPercentageAboveBaseCapacity\": 25,
      \"SpotAllocationStrategy\": \"price-capacity-optimized\"
    }
  }"
```

- [ ] After about a minute, see which instances are Spot and which are On-Demand:

```bash
sleep 60
IDS=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names saa-mixed-asg --query 'AutoScalingGroups[0].Instances[].InstanceId' --output text)
aws ec2 describe-instances --instance-ids $IDS \
  --query 'Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,AZ:Placement.AvailabilityZone,Lifecycle:InstanceLifecycle||`on-demand`}' --output table
```

> Expect **1 On-Demand** (the base), then 25% of the other 3 as On-Demand, so 1 more On-Demand and **2 Spot**. **Capacity Rebalance** launches a replacement *before* a Spot instance is interrupted.

### B5. Placement groups from the CLI

- [ ] Create a partition group and launch into a specific partition:

```bash
aws ec2 create-placement-group --group-name saa-pg-part-cli --strategy partition --partition-count 3

aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $APP_A --count 1 \
  --placement "GroupName=saa-pg-part-cli,PartitionNumber=2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-pg-test}]' --query 'Instances[0].InstanceId' --output text

aws ec2 describe-instances --filters Name=placement-group-name,Values=saa-pg-part-cli \
  --query 'Reservations[].Instances[].{Id:InstanceId,Partition:Placement.PartitionNumber,AZ:Placement.AvailabilityZone}'
```

- [ ] List every placement group:

```bash
aws ec2 describe-placement-groups --query 'PlacementGroups[].{Name:GroupName,Strategy:Strategy,Partitions:PartitionCount,State:State}' --output table
```

### B6. Spot interruption handling (EventBridge rule preview)

- [ ] Create a rule that catches Spot interruption warnings. You'll attach targets in Lab 27:

```bash
aws events put-rule --name saa-spot-interruption --event-pattern '{
  "source":["aws.ec2"],
  "detail-type":["EC2 Spot Instance Interruption Warning"]
}'
```

---

## Verify

- [ ] You launched Spot Instances from the console and the CLI.
- [ ] The mixed ASG runs a blend of On-Demand and Spot across several types.
- [ ] The placement groups exist and instances landed in the expected partition.

## Exam tips

- **Steady-state 24/7 workload for 3 years** → **Savings Plans or RIs** (All Upfront, 3-year = biggest discount).
- **Flexible across families, regions, and Fargate/Lambda** → **Compute Savings Plan**.
- **Fault-tolerant batch or CI jobs, cost is the priority** → **Spot** (with checkpointing and diversified instance types).
- **Must not be interrupted, short-term** → **On-Demand**.
- **Guaranteed capacity in an AZ without a long commitment** → **On-Demand Capacity Reservation**.
- **Server-bound software licences (BYOL per socket/core)** → **Dedicated Hosts**.
- **Lowest latency between HPC nodes** → **cluster placement group** (+ EFA). **Critical instances must not share hardware** → **spread**. **Hadoop/Cassandra rack awareness** → **partition**.

## Challenge (optional)

- [ ] Use the [AWS Pricing Calculator](https://calculator.aws/) to compare a 10-instance `m6i.large` fleet as On-Demand, 3-year Compute Savings Plan (no upfront), and 70% Spot / 30% On-Demand.

## Cleanup

- [ ] Delete the mixed ASG and terminate the other instances:

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name saa-mixed-asg --force-delete
aws ec2 terminate-instances --instance-ids $SPOT_ID $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-spot-test,saa-pg-test Name=instance-state-name,Values=running,pending,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
# terminate the two instances you launched into saa-pg-spread as well
```

- [ ] Cancel the one-time Spot request if it's still open:

```bash
aws ec2 describe-spot-instance-requests --query 'SpotInstanceRequests[?State==`open`].SpotInstanceRequestId' --output text | xargs -r aws ec2 cancel-spot-instance-requests --spot-instance-request-ids
```

- [ ] Delete the placement groups once their instances have terminated:

```bash
for PG in saa-pg-cluster saa-pg-spread saa-pg-partition saa-pg-part-cli; do aws ec2 delete-placement-group --group-name $PG; done
```

- [ ] Confirm the capacity reservation is **cancelled**.
- [ ] `aws events delete-rule --name saa-spot-interruption`

---

## Knowledge check

**Q1.** A company runs a nightly batch job that can be paused and resumed. It wants the lowest compute cost. What should it use?

<details><summary>Answer</summary>

**Spot Instances** (ideally in a mixed or diversified fleet with checkpointing).

</details>

**Q2.** A company has steady EC2 usage but plans to move some workloads to Fargate and Lambda over the next year. Which commitment saves money and stays flexible?

<details><summary>Answer</summary>

A **Compute Savings Plan**, which applies to EC2 in any family or region, plus Fargate and Lambda.

</details>

**Q3.** A distributed Cassandra cluster must be laid out so a single rack failure doesn't take down multiple replicas. Which placement strategy?

<details><summary>Answer</summary>

A **partition placement group**.

</details>
