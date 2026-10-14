# Lab 39 — Capstone: Highly Available 3-Tier Web App with CloudFormation

| | |
|---|---|
| **Module** | 11 — Cost Optimisation & Capstone |
| **Exam mapping** | **All domains**: 1 (secure), 2 (resilient), 3 (high-performing), 4 (cost-optimised) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 3–4 hours |
| **Estimated cost** | ~$0.30–0.50/hour while running (NAT, ALB, 2 × t3.micro, Aurora Serverless v2 at 2 × 0.5 ACU, CloudFront). **Delete the same day.** |

- [ ] **Lab complete**

[⬅ Previous lab](lab-38-cost-optimization.md) · [Back to index](../README.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Deploy a complete production-style architecture as **Infrastructure as Code** with **CloudFormation**.
- Manage the stack lifecycle: **parameters, outputs, change sets, rollbacks, drift detection, deletion policies**.
- **Test resilience** by injecting failures (instance loss, database failover, AZ impairment) and watching the system recover.
- **Load-test** and observe elasticity.
- Review the architecture against all four exam domains, and the **Well-Architected Framework**.

## Architecture

```
                        Users
                          │ HTTPS
                ┌─────────▼─────────┐
                │    CloudFront     │  (edge, TLS, Shield Standard)
                └─────────┬─────────┘
                          │ HTTP
 ┌────────────────────────┼────────────────── VPC 10.20.0.0/16 ─────────────────┐
 │   AZ a                 │                       AZ b                          │
 │ ┌──────────────┐ ┌─────▼─────────────────────────────┐ ┌──────────────┐      │
 │ │ public-a     │ │       Application Load Balancer   │ │ public-b     │      │
 │ │  NAT GW (a)  │ └─────┬───────────────────────┬─────┘ │ (NAT GW b)*  │      │
 │ └──────────────┘       │                       │       └──────────────┘      │
 │ ┌──────────────┐ ┌─────▼──────┐          ┌─────▼──────┐ ┌──────────────┐     │
 │ │ app-a        │ │ EC2 (ASG)  │          │ EC2 (ASG)  │ │ app-b        │     │
 │ └──────────────┘ └─────┬──────┘          └─────┬──────┘ └──────────────┘     │
 │ ┌──────────────┐ ┌─────▼──────────┐   ┌────────▼───────┐ ┌──────────────┐    │
 │ │ data-a       │ │ Aurora writer  │◄─►│ Aurora reader  │ │ data-b       │    │
 │ └──────────────┘ └────────────────┘   └────────────────┘ └──────────────┘    │
 │        S3 gateway endpoint · Secrets Manager · CloudWatch alarms → SNS       │
 └──────────────────────────────────────────────────────────────────────────────┘
 * second NAT only when HighlyAvailableNat=true
```

Template: [`capstone/saa-capstone.yaml`](capstone/saa-capstone.yaml)

---

## Prerequisites

- Every earlier module finished (or at least Modules 4–7).
- Read through the template **before** deploying. For each resource, write down which earlier lab taught it:

- [ ] VPC, subnets, route tables, NAT → Labs 11–12
- [ ] Security group chaining → Lab 13
- [ ] S3 gateway endpoint → Lab 14
- [ ] Launch template, user data, IMDSv2 → Lab 16
- [ ] ALB, target group, health checks → Lab 18
- [ ] ASG, target tracking, rolling updates → Lab 19
- [ ] Aurora Serverless v2, writer/reader → Lab 22
- [ ] Secrets Manager (managed master password) → Lab 29
- [ ] CloudWatch alarms + SNS → Lab 30
- [ ] CloudFront in front of a custom origin → Lab 33

---

## Part A — Console

### A1. Deploy the stack

- [ ] Open **CloudFormation → Create stack → With new resources → Upload a template file** → `saa-capstone.yaml`.
- [ ] Stack name `saa-capstone`. Parameters:
  - `HighlyAvailableNat`: **false** (cheaper. You'll change it with a change set later.)
  - `AlarmEmail`: your email
  - Leave the others at their defaults.
- [ ] Tick **I acknowledge that AWS CloudFormation might create IAM resources** and **Submit**.
- [ ] Watch the **Events** tab (about 20–25 minutes; Aurora and CloudFront take longest). Notice how CloudFormation works out the **dependency order**: VPC → subnets → NAT → DB → launch template → ASG.
- [ ] Confirm the SNS subscription email.

### A2. Explore the result

- [ ] Open the **Outputs** tab and browse to **CloudFrontUrl**. The page shows the instance ID, the AZ, and `Database: connected to Aurora writer`.
- [ ] Refresh several times. The instance and AZ alternate (ALB across two AZs, with CloudFront caching disabled).
- [ ] Open the **Resources** tab and click through to the ASG, ALB, and Aurora cluster.
- [ ] Open the template in **Infrastructure Composer** (stack → *Template* → *View in Infrastructure Composer*) to see it as a diagram.

### A3. Change set: make NAT highly available

- [ ] Choose **Stack actions → Create change set for current stack → Use current template**, and change `HighlyAvailableNat` to **true**.
- [ ] Review the **Changes**: *Add* `NatEipB` and `NatGatewayB`, *Modify* `PrivateRouteB`. **No replacement** of the database or ASG.
- [ ] **Execute** the change set.

> **Change sets** let you preview what an update will do (especially *Replacement: True*, which can mean data loss) before you apply it.

### A4. Drift detection

- [ ] Change something by hand, **outside CloudFormation**: open the `saa-capstone-alb-sg` security group and add an inbound rule, **TCP 22 from 0.0.0.0/0**.
- [ ] Open **Stack actions → Detect drift**, then **View drift results**. `AlbSecurityGroup` shows **MODIFIED**, with the difference.
- [ ] Remove the manual rule and detect drift again. It's back **IN_SYNC**.

---

## Part B — CLI: failure injection and operations

- [ ] Load the stack outputs into variables:

```bash
STACK=saa-capstone
out () { aws cloudformation describe-stacks --stack-name $STACK --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }
CF_URL=$(out CloudFrontUrl); ALB_URL=$(out AlbUrl); ASG=$(out AsgName); DB=$(out DbClusterId)
APP_SUBNETS=$(out AppSubnets)
echo "$CF_URL | $ALB_URL | $ASG | $DB"
```

- [ ] Keep a **monitoring loop** running in a second terminal for every test:

```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s -m 3 -o /tmp/p.html -w "%{http_code}" $ALB_URL/ && grep -oE '(i-[0-9a-f]+|us-east-1[a-f]|connected|UNREACHABLE)' /tmp/p.html | tr '\n' ' '
  echo; sleep 2
done
```

### B1. Validate and inspect the stack

- [ ] Run:

```bash
aws cloudformation validate-template --template-body file://labs/capstone/saa-capstone.yaml   # run from the repo root --query 'Parameters[].ParameterKey'
aws cloudformation describe-stack-resources --stack-name $STACK \
  --query 'StackResources[].{Logical:LogicalResourceId,Type:ResourceType,Status:ResourceStatus}' --output table
```

### B2. Test 1: instance failure (self-healing)

- [ ] Terminate one web instance:

```bash
VICTIM=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)
aws ec2 terminate-instances --instance-ids $VICTIM
```

- [ ] **Observe:** the monitoring loop keeps getting **200** from the surviving instance. Within about 3–5 minutes a replacement is **InService**.
- [ ] Record the time to full recovery: ________

### B3. Test 2: database failover

- [ ] Fail over Aurora:

```bash
aws rds failover-db-cluster --db-cluster-identifier $DB
aws rds describe-db-clusters --db-cluster-identifier $DB --query 'DBClusters[0].DBClusterMembers[].{Id:DBInstanceIdentifier,Writer:IsClusterWriter}'
```

- [ ] **Observe:** the writer role moves to the other instance (in the other AZ). The web tier keeps serving. New instances launched after this still report `connected`, because the cluster endpoint follows the writer.
- [ ] Record the failover duration (from `aws rds describe-events --source-identifier $DB --source-type db-cluster --duration 30`): ________

### B4. Test 3: AZ impairment (simulated)

- [ ] Remove AZ-a's app subnet from the ASG to simulate losing that AZ:

```bash
SUB_B=$(echo $APP_SUBNETS | cut -d, -f2)
aws autoscaling update-auto-scaling-group --auto-scaling-group-name $ASG --vpc-zone-identifier $SUB_B
```

- [ ] **Observe:** the ASG rebalances. Instances in AZ-a are terminated and replaced in AZ-b, and the loop shows only `us-east-1b` once that's done. Service continues throughout.
- [ ] Restore both AZs. Because this change was made outside CloudFormation, it's also a **drift**:

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name $ASG --vpc-zone-identifier $APP_SUBNETS
```

> For more realistic chaos testing, use **AWS Fault Injection Service (FIS)** with experiment templates, e.g. `aws:network:disrupt-connectivity` on a subnet, or an AZ power-interruption scenario.

### B5. Test 4: load and elasticity

- [ ] Generate CPU load **on the instances** through SSM. This works through the NAT gateway and avoids hammering CloudFront:

```bash
IDS=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].Instances[].InstanceId' --output text)
aws ssm send-command --instance-ids $IDS --document-name AWS-RunShellScript \
  --parameters 'commands=["timeout 600 bash -c \"for c in $(seq $(nproc)); do yes > /dev/null & done; wait\""]' \
  --query Command.CommandId --output text
```

- [ ] Watch scaling (about 5–10 minutes):

```bash
watch -n 20 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,InService:length(Instances[?LifecycleState==\`InService\`])}'"
```

- [ ] **Observe:** the target-tracking alarm fires and desired capacity goes above 2. After the 10-minute load ends, the ASG scales back in to 2.

### B6. Test 5: security posture checks

- [ ] Instances have **no public IPs**:

```bash
aws ec2 describe-instances --filters Name=tag:aws:autoscaling:groupName,Values=$ASG Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{Id:InstanceId,Public:PublicIpAddress,Subnet:SubnetId}' --output table
```

- [ ] The database is **not publicly accessible** and is **encrypted**:

```bash
aws rds describe-db-instances --filters Name=db-cluster-id,Values=$DB \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,Public:PubliclyAccessible,Encrypted:StorageEncrypted,AZ:AvailabilityZone}' --output table
```

- [ ] The app SG only allows the ALB SG, and the DB SG only allows the app SG:

```bash
aws ec2 describe-security-groups --filters Name=vpc-id,Values=$(out VpcId) \
  --query 'SecurityGroups[].{Name:GroupName,In:IpPermissions[].{Port:FromPort,FromSG:UserIdGroupPairs[0].GroupId,CIDR:IpRanges[0].CidrIp}}'
```

### B7. Update the stack from the CLI with a change set (scale up the instance type)

- [ ] Create the change set:

```bash
aws cloudformation create-change-set --stack-name $STACK --change-set-name bigger-instances \
  --use-previous-template --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.small \
               ParameterKey=HighlyAvailableNat,UsePreviousValue=true \
               ParameterKey=AlarmEmail,UsePreviousValue=true \
               ParameterKey=EnvName,UsePreviousValue=true \
               ParameterKey=VpcCidr,UsePreviousValue=true \
               ParameterKey=LatestAmiId,UsePreviousValue=true
aws cloudformation wait change-set-create-complete --stack-name $STACK --change-set-name bigger-instances
aws cloudformation describe-change-set --stack-name $STACK --change-set-name bigger-instances \
  --query 'Changes[].ResourceChange.{Action:Action,Resource:LogicalResourceId,Replace:Replacement}' --output table
```

- [ ] Execute it and watch the rolling update:

```bash
aws cloudformation execute-change-set --stack-name $STACK --change-set-name bigger-instances
aws cloudformation wait stack-update-complete --stack-name $STACK
```

> The ASG's **`AutoScalingRollingUpdate`** policy replaces one instance at a time, keeping at least one in service. Your monitoring loop should show **no failed requests**.

---

## Architecture review (self-assessment)

- [ ] Tick each item you can **justify out loud**, then write down one improvement for each domain.

### Domain 1: Secure

- [ ] Only CloudFront and the ALB are internet-facing. Instances and the DB are in private subnets.
- [ ] SG chaining: ALB → app → DB.
- [ ] DB credentials live in Secrets Manager, and the instance role can read **only** that secret.
- [ ] IMDSv2 is enforced, and the Aurora storage is encrypted.
- [ ] **Improvement ideas:** WAF on CloudFront, HTTPS between CloudFront and the ALB (ACM), restrict the ALB SG to the CloudFront origin-facing prefix list, Secrets Manager rotation, VPC Flow Logs, GuardDuty.

### Domain 2: Resilient

- [ ] Multi-AZ everything: ALB, ASG, Aurora writer/reader, optional NAT per AZ.
- [ ] Self-healing with ELB health checks, automatic Aurora failover, rolling updates.
- [ ] **Improvement ideas:** a cross-region DR strategy (Aurora Global Database, a second stack, Route 53 failover), AWS Backup, FIS experiments in CI.

### Domain 3: High-performing

- [ ] Elastic web tier with target tracking, Aurora Serverless v2 auto-scaling capacity, CloudFront edge.
- [ ] **Improvement ideas:** ElastiCache for sessions and queries, Aurora reader endpoint for read traffic, caching static assets at CloudFront, RDS Proxy.

### Domain 4: Cost-optimised

- [ ] Single NAT option, Serverless v2 with a low minimum ACU, burstable instances, S3 gateway endpoint, CloudFront PriceClass_100.
- [ ] **Improvement ideas:** Graviton (t4g), Savings Plans, Spot in a mixed-instances ASG, scheduled scale-down out of hours, Aurora auto-pause (0 ACU) for non-prod.

---

## Verify

- [ ] The stack deployed successfully and the CloudFront URL shows `connected`.
- [ ] You recorded recovery times for instance loss and DB failover.
- [ ] The ASG scaled out under load and back in.
- [ ] Drift detection found your manual change, and change sets previewed updates correctly.

## Exam tips

- **"Deploy the same infrastructure repeatedly and consistently across accounts/regions"** → **CloudFormation (StackSets for multi-account)**, or CDK, which generates CloudFormation.
- **"Preview the impact of an update"** → **change sets**. **"Detect manual changes"** → **drift detection**.
- **"Keep the database when the stack is deleted"** → **`DeletionPolicy: Retain` or `Snapshot`**.
- **"Developers need a simple way to deploy web apps without managing infrastructure"** → **Elastic Beanstalk** (or App Runner).
- **"Test how the architecture behaves when an AZ fails"** → **AWS Fault Injection Service**. **"Resilience assessment against RTO/RPO targets"** → **AWS Resilience Hub**.

## Challenge (optional)

- [ ] Extend the template with:
  - An **AWS WAF** web ACL on the CloudFront distribution (scope `CLOUDFRONT`, so it must be deployed in us-east-1).
  - An **ElastiCache Serverless** cache in the data subnets.
  - **Route 53** alias + **ACM** certificate for a custom domain.
  - A **StackSet** that deploys a read-only monitoring role to every account in your Organization.

## Cleanup

### Capstone teardown

- [ ] Delete the stack (about 15–20 minutes; CloudFront takes the longest):

```bash
aws cloudformation delete-stack --stack-name saa-capstone
aws cloudformation wait stack-delete-complete --stack-name saa-capstone
```

- [ ] Check what's left over. The Secrets Manager secret is removed with the cluster. CloudWatch log groups created by instances may remain.

### Whole-programme teardown (once you've finished every lab)

- [ ] Run the Lab 38 waste finder in both regions: `./waste.sh us-east-1 && ./waste.sh us-west-2`
- [ ] Delete the shared lab VPC `saa-vpc`. Delete its gateway endpoints and any remaining ENIs first, then **VPC → Actions → Delete VPC**, then `rm ~/saa-vpc.env`.
- [ ] Delete the shared IAM roles (`saa-ssm-ec2-role` and its instance profile, `ecsTaskExecutionRole` if unused) and any `saa-*` launch templates.
- [ ] Empty and delete any remaining `saa-*` S3 buckets: `aws s3 ls | grep saa-`
- [ ] Disable trial security services you don't want (Inspector, Macie, Security Hub), and keep GuardDuty and CloudTrail if you like.
- [ ] **Keep:** root MFA, admin user, budgets, billing alarm, Cost Anomaly Detection.
- [ ] Check **Billing → Bills** a few days later to confirm there are no unexpected charges.

---

## Knowledge check

**Q1.** A company wants every new AWS account in its Organization to get a standard VPC, IAM roles, and CloudTrail configuration automatically. What should it use?

<details><summary>Answer</summary>

**CloudFormation StackSets** with **service-managed permissions** and **automatic deployment** to the target OUs (or **Control Tower** customisations / Account Factory).

</details>

**Q2.** During a stack update, CloudFormation reports that the Aurora cluster will be **replaced**. What's the risk, and how should you protect the data?

<details><summary>Answer</summary>

Replacement creates a **new** cluster, and the old one is deleted. Without a snapshot, **the data is lost**. Review change sets before executing, and set **`DeletionPolicy: Snapshot`** and **`UpdateReplacePolicy: Snapshot`** (or Retain) on stateful resources.

</details>

**Q3.** Which combination in this capstone gives high availability for each tier? Name one service per tier.

<details><summary>Answer</summary>

- **Edge:** CloudFront (a global edge network).
- **Load balancing:** ALB across 2 AZs.
- **Compute:** ASG with ELB health checks across 2 AZs.
- **Database:** Aurora with a reader in a second AZ and automatic failover. Storage is 6-way replicated across 3 AZs.
- **Egress:** a NAT gateway per AZ (when `HighlyAvailableNat=true`).

</details>

---

🎉 **Congratulations on finishing the SAA-C03 lab programme!** Go back to the [index](../README.md), tick the exam readiness checklist, and book your exam.
