# AWS Solutions Architect Associate (SAA-C03) — Hands-On Labs

A progressive, hands-on lab programme mapped to the **AWS Certified Solutions Architect – Associate (SAA-C03)** exam guide. Every lab has a **Console (UI)** walkthrough and an equivalent **CLI** walkthrough, so you learn the concept visually and then cement it in commands.

---

## How to use this programme

1. **Work in order.** Labs build on each other — fundamentals first, then networking, compute, data, decoupling, security, resilience, analytics, and cost. Later labs assume skills from earlier ones.
2. **Do both parts.** Do Part A (Console) to understand *what* you're building, then Part B (CLI) to understand *how it's really configured*.
3. **Always clean up.** Every lab ends with a cleanup checklist. Many labs use billable resources (NAT Gateways, RDS, ALBs, Transit Gateway, etc.) — each lab header shows an **estimated cost**.
4. **Answer the knowledge check** at the end of each lab before revealing the answers. These are written in the exam's scenario style.

### Marking progress

Progress is tracked with Markdown task checkboxes:

```markdown
- [ ] Not done
- [x] Done
```

- **This index** — tick the box beside each lab when you finish it.
- **Inside each lab** — tick each step, the cleanup items, and the `Lab complete` box at the top.

> **Tip (VS Code):** the *Markdown All in One* extension toggles a checkbox with <kbd>Alt</kbd>+<kbd>C</kbd> (<kbd>Option</kbd>+<kbd>C</kbd> on macOS). Commit your progress to git to keep a history of your study.

### Lab conventions

| Convention | Value |
|---|---|
| Default region | `us-east-1` (some labs use a second region, `us-west-2`) |
| CLI version | AWS CLI **v2** |
| Shell | bash / zsh (commands are POSIX-shell compatible) |
| Resource naming | `saa-labNN-<thing>` so you can find and delete everything easily |
| Uniqueness | Globally-unique names (e.g. S3 buckets) use a `$SUFFIX` variable set in Lab 02 |

Difficulty: ★☆☆ Foundational · ★★☆ Intermediate · ★★★ Advanced

---

## Exam domain coverage

| Domain | Weight | Labs |
|---|---|---|
| **1 — Design Secure Architectures** | 30% | 01, 03, 04, 05, 06, 08, 12, 13, 14, 29, 30, 31 |
| **2 — Design Resilient Architectures** | 26% | 10, 18, 19, 21, 22, 23, 25, 26, 27, 28, 32, 33, 34, 39 |
| **3 — Design High-Performing Architectures** | 24% | 07, 09, 15, 16, 17, 20, 22, 23, 24, 26, 33, 35, 36, 37 |
| **4 — Design Cost-Optimized Architectures** | 20% | 09, 12, 14, 17, 20, 23, 34, 37, 38 |

Task statement references (e.g. **1.1**, **3.4**) used throughout come from the official SAA-C03 exam guide:

<details>
<summary>Show all SAA-C03 task statements</summary>

| Ref | Task statement |
|---|---|
| 1.1 | Design secure access to AWS resources |
| 1.2 | Design secure workloads and applications |
| 1.3 | Determine appropriate data security controls |
| 2.1 | Design scalable and loosely coupled architectures |
| 2.2 | Design highly available and/or fault-tolerant architectures |
| 3.1 | Determine high-performing and/or scalable storage solutions |
| 3.2 | Design high-performing and elastic compute solutions |
| 3.3 | Determine high-performing database solutions |
| 3.4 | Determine high-performing and/or scalable network architectures |
| 3.5 | Determine high-performing data ingestion and transformation solutions |
| 4.1 | Design cost-optimized storage solutions |
| 4.2 | Design cost-optimized compute solutions |
| 4.3 | Design cost-optimized database solutions |
| 4.4 | Design cost-optimized network architectures |

</details>

---

## Module 1 — Getting Started & Account Foundations

- [ ] **[Lab 01 — Secure Account Baseline & Billing Guardrails](labs/lab-01-account-baseline.md)** ★☆☆
  - *Description:* Lock down the root user with MFA, create an admin identity, turn on billing alerts, and create a budget.
  - *Learning outcome:* Apply the AWS shared responsibility model and root-user best practices to secure a new account.
- [ ] **[Lab 02 — AWS CLI Setup, Profiles & Querying](labs/lab-02-cli-setup.md)** ★☆☆
  - *Description:* Install and configure AWS CLI v2, named profiles, and output formats, and filter results with JMESPath `--query`.
  - *Learning outcome:* Use the CLI confidently, which every later lab depends on, and understand how credentials are resolved.

## Module 2 — Identity & Access Management (Domain 1.1)

- [ ] **[Lab 03 — IAM Users, Groups & Policies](labs/lab-03-iam-users-groups-policies.md)** ★☆☆
  - *Description:* Create users and groups, and attach AWS-managed, customer-managed, and inline policies. Enforce a password policy and MFA.
  - *Learning outcome:* Design least-privilege access using IAM identities and policy types.
- [ ] **[Lab 04 — IAM Roles, STS & Cross-Account Access](labs/lab-04-iam-roles-sts.md)** ★★☆
  - *Description:* Create roles with trust policies, assume them with STS, and attach an instance profile to EC2.
  - *Learning outcome:* Use temporary credentials and roles instead of long-term access keys, both for workloads and across accounts.
- [ ] **[Lab 05 — Advanced Policies: Conditions, Resource Policies & Boundaries](labs/lab-05-advanced-iam-policies.md)** ★★☆
  - *Description:* Write condition-based policies, compare identity-based and resource-based policies, apply permission boundaries, and test with the Policy Simulator.
  - *Learning outcome:* Explain how IAM evaluates policies and get the result you intend when several policy types combine.
- [ ] **[Lab 06 — AWS Organizations, SCPs & IAM Identity Center](labs/lab-06-organizations-scps.md)** ★★☆
  - *Description:* Create an organization and OUs, apply Service Control Policies, and set up IAM Identity Center for SSO.
  - *Learning outcome:* Design multi-account governance and centralised access for an enterprise.

## Module 3 — Amazon S3 Object Storage (Domains 1.3, 3.1, 4.1)

- [ ] **[Lab 07 — S3 Fundamentals & Static Website Hosting](labs/lab-07-s3-fundamentals.md)** ★☆☆
  - *Description:* Create buckets, upload objects, enable versioning, and host a static website.
  - *Learning outcome:* Describe S3's object model, durability, and consistency, and serve static content from S3.
- [ ] **[Lab 08 — S3 Security & Encryption](labs/lab-08-s3-security-encryption.md)** ★★☆
  - *Description:* Configure Block Public Access, bucket policies, SSE-S3/SSE-KMS encryption, enforced TLS, and presigned URLs.
  - *Learning outcome:* Pick and apply the right S3 data-protection controls for a given compliance requirement.
- [ ] **[Lab 09 — S3 Storage Classes & Lifecycle Management](labs/lab-09-s3-storage-classes-lifecycle.md)** ★☆☆
  - *Description:* Compare storage classes, build lifecycle transition and expiration rules, and enable Intelligent-Tiering.
  - *Learning outcome:* Cut storage costs by matching access patterns to the right storage class.
- [ ] **[Lab 10 — S3 Replication, Object Lock, Events & Performance](labs/lab-10-s3-replication-advanced.md)** ★★☆
  - *Description:* Set up Cross-Region Replication, apply Object Lock (WORM), trigger event notifications, and use Transfer Acceleration and multipart upload.
  - *Learning outcome:* Design S3 for disaster recovery, compliance retention, and high-throughput transfers.

## Module 4 — Networking with Amazon VPC (Domains 1.2, 3.4, 4.4)

- [ ] **[Lab 11 — Build a Custom VPC from Scratch](labs/lab-11-custom-vpc.md)** ★☆☆
  - *Description:* Create a VPC with public and private subnets across two AZs, plus an internet gateway and route tables.
  - *Learning outcome:* Design a multi-AZ VPC with a clear CIDR plan and correct routing.
- [ ] **[Lab 12 — Private Subnets, NAT Gateway & Session Manager](labs/lab-12-nat-private-access.md)** ★★☆
  - *Description:* Give private instances outbound internet through a NAT gateway, and administer them with SSM Session Manager instead of a bastion host.
  - *Learning outcome:* Provide secure outbound connectivity and admin access without exposing instances publicly.
- [ ] **[Lab 13 — Security Groups vs Network ACLs](labs/lab-13-sg-vs-nacl.md)** ★★☆
  - *Description:* Build layered network security, and see stateful versus stateless filtering, rule order, and ephemeral ports in action.
  - *Learning outcome:* Pick and configure the right network-layer control for a security requirement.
- [ ] **[Lab 14 — VPC Endpoints, PrivateLink & Flow Logs](labs/lab-14-vpc-endpoints-flowlogs.md)** ★★☆
  - *Description:* Reach S3 and other AWS services privately with gateway and interface endpoints, and capture traffic with VPC Flow Logs.
  - *Learning outcome:* Keep traffic on the AWS network for security and lower NAT cost, and audit traffic flows.
- [ ] **[Lab 15 — VPC Peering & Transit Gateway](labs/lab-15-peering-transit-gateway.md)** ★★★
  - *Description:* Connect VPCs with peering, then scale up to a hub-and-spoke Transit Gateway. Covers VPN and Direct Connect concepts.
  - *Learning outcome:* Design scalable multi-VPC and hybrid network connectivity.

## Module 5 — Compute, Storage Volumes, Load Balancing & Scaling (Domains 2.2, 3.1, 3.2, 4.2)

- [ ] **[Lab 16 — EC2 Fundamentals: Launch, User Data & AMIs](labs/lab-16-ec2-fundamentals.md)** ★☆☆
  - *Description:* Launch instances, bootstrap them with user data, read instance metadata (IMDSv2), and create a custom AMI.
  - *Learning outcome:* Choose instance families and launch repeatable, pre-configured EC2 workloads.
- [ ] **[Lab 17 — Block & File Storage: EBS, Snapshots, EFS & Instance Store](labs/lab-17-ebs-efs-storage.md)** ★★☆
  - *Description:* Compare EBS volume types, resize volumes live, take and copy snapshots, and share an EFS file system across AZs.
  - *Learning outcome:* Pick the right block or file storage for performance, durability, and sharing needs.
- [ ] **[Lab 18 — Elastic Load Balancing with an Application Load Balancer](labs/lab-18-elastic-load-balancing.md)** ★★☆
  - *Description:* Build an ALB with target groups, health checks, and path-based routing. Compares ALB, NLB, and GWLB.
  - *Learning outcome:* Spread traffic across AZs for availability and pick the right load-balancer type.
- [ ] **[Lab 19 — EC2 Auto Scaling](labs/lab-19-auto-scaling.md)** ★★☆
  - *Description:* Create launch templates and Auto Scaling groups with target-tracking, scheduled, and step policies, and watch self-healing happen.
  - *Learning outcome:* Design elastic compute that scales with demand and recovers from failure.
- [ ] **[Lab 20 — EC2 Purchasing Options & Placement Groups](labs/lab-20-ec2-pricing-placement.md)** ★★☆
  - *Description:* Launch Spot Instances and mixed-instance ASGs, and compare Savings Plans and Reserved Instances. Covers cluster, spread, and partition placement groups.
  - *Learning outcome:* Choose the most cost-effective purchasing model and the best-performing placement for a workload.

## Module 6 — Databases (Domains 2.2, 3.3, 4.3)

- [ ] **[Lab 21 — Amazon RDS: Multi-AZ, Read Replicas & Backups](labs/lab-21-rds.md)** ★★☆
  - *Description:* Deploy RDS MySQL with Multi-AZ, force a failover, add a read replica, and do a point-in-time restore.
  - *Learning outcome:* Tell high availability apart from read scaling, and design for database recovery.
- [ ] **[Lab 22 — Amazon Aurora, Aurora Serverless v2 & RDS Proxy](labs/lab-22-aurora-rds-proxy.md)** ★★★
  - *Description:* Build an Aurora cluster with reader endpoints and Serverless v2 capacity, and pool connections through RDS Proxy.
  - *Learning outcome:* Choose Aurora features for performance, elasticity, and serverless-friendly connection handling.
- [ ] **[Lab 23 — Amazon DynamoDB](labs/lab-23-dynamodb.md)** ★★☆
  - *Description:* Design partition and sort keys, query with GSIs, compare on-demand and provisioned capacity, and use TTL, Streams, and global tables.
  - *Learning outcome:* Model and scale a NoSQL workload, and know when DynamoDB fits better than a relational database.
- [ ] **[Lab 24 — Caching with Amazon ElastiCache](labs/lab-24-elasticache.md)** ★★☆
  - *Description:* Deploy ElastiCache (Redis OSS / Valkey) and implement lazy-loading and write-through caching. Covers when DAX or CloudFront fit better.
  - *Learning outcome:* Use caching to cut latency and database load.

## Module 7 — Decoupling, Serverless & Containers (Domains 2.1, 3.2)

- [ ] **[Lab 25 — Decoupling with SQS & SNS](labs/lab-25-sqs-sns.md)** ★★☆
  - *Description:* Build standard and FIFO queues, dead-letter queues, and an SNS-to-SQS fan-out with message filtering.
  - *Learning outcome:* Design loosely coupled, fault-tolerant messaging between components.
- [ ] **[Lab 26 — Serverless APIs with Lambda & API Gateway](labs/lab-26-lambda-api-gateway.md)** ★★☆
  - *Description:* Build a Lambda-backed REST API with DynamoDB, and configure throttling, concurrency, and a Lambda that runs inside a VPC.
  - *Learning outcome:* Design scalable, pay-per-use serverless applications.
- [ ] **[Lab 27 — Event-Driven Orchestration: EventBridge & Step Functions](labs/lab-27-eventbridge-step-functions.md)** ★★★
  - *Description:* Route events with EventBridge rules and schedules, and orchestrate a workflow with Step Functions, including retries and error handling.
  - *Learning outcome:* Pick between choreography (events) and orchestration (workflows) in distributed systems.
- [ ] **[Lab 28 — Containers: ECR & ECS on Fargate](labs/lab-28-ecs-fargate.md)** ★★★
  - *Description:* Push an image to ECR and run it as an ECS Fargate service behind an ALB with service auto scaling. Covers EKS concepts.
  - *Learning outcome:* Deploy containerised workloads without managing servers, and choose between ECS, EKS, Fargate, and EC2.

## Module 8 — Data Protection, Monitoring & Threat Detection (Domains 1.2, 1.3)

- [ ] **[Lab 29 — Encryption & Secrets: KMS, Secrets Manager & Parameter Store](labs/lab-29-kms-secrets.md)** ★★☆
  - *Description:* Create customer-managed KMS keys and key policies, do envelope encryption, and rotate secrets. Compares Secrets Manager and Parameter Store.
  - *Learning outcome:* Protect data at rest and manage credentials securely.
- [ ] **[Lab 30 — Observability & Audit: CloudWatch, CloudTrail & AWS Config](labs/lab-30-cloudwatch-cloudtrail-config.md)** ★★☆
  - *Description:* Collect metrics, logs, and alarms, run a CloudTrail trail and Lake queries, and apply Config rules with remediation.
  - *Learning outcome:* Monitor, audit, and enforce compliance across AWS resources.
- [ ] **[Lab 31 — Perimeter Security: ACM, WAF, Shield & GuardDuty](labs/lab-31-waf-shield-guardduty.md)** ★★★
  - *Description:* Add HTTPS with ACM, protect an ALB with WAF managed rules and rate limits, and enable GuardDuty, Inspector, and Macie.
  - *Learning outcome:* Protect internet-facing applications and detect threats across an account.

## Module 9 — High Availability, Disaster Recovery & Global Delivery (Domains 2.2, 3.4)

- [ ] **[Lab 32 — Route 53: DNS Routing Policies & Health Checks](labs/lab-32-route53.md)** ★★☆
  - *Description:* Set up hosted zones, alias records, and weighted, latency, failover, and geolocation routing, all with health checks.
  - *Learning outcome:* Use DNS to steer traffic and fail over between regions.
- [ ] **[Lab 33 — CloudFront & Global Accelerator](labs/lab-33-cloudfront-global-accelerator.md)** ★★☆
  - *Description:* Put CloudFront in front of a private S3 origin with OAC, tune caching and invalidations, and compare it with Global Accelerator.
  - *Learning outcome:* Cut global latency and protect origins with edge services.
- [ ] **[Lab 34 — Backup & Disaster Recovery Strategies](labs/lab-34-backup-dr.md)** ★★★
  - *Description:* Centralise backups with AWS Backup plans and cross-region copy, then build a pilot-light DR setup and test RTO and RPO.
  - *Learning outcome:* Match DR strategies (backup & restore, pilot light, warm standby, multi-site) to RTO/RPO and cost needs.

## Module 10 — Data Ingestion, Analytics & Hybrid (Domains 3.1, 3.5)

- [ ] **[Lab 35 — Streaming Ingestion with Kinesis](labs/lab-35-kinesis.md)** ★★☆
  - *Description:* Produce to and consume from Kinesis Data Streams, and deliver to S3 with Amazon Data Firehose, including a Lambda transform.
  - *Learning outcome:* Design real-time ingestion pipelines and pick between Kinesis, SQS, and MSK.
- [ ] **[Lab 36 — Serverless Data Lake: Glue, Athena & Lake Formation](labs/lab-36-glue-athena-datalake.md)** ★★☆
  - *Description:* Catalog S3 data with a Glue crawler, query it with Athena, convert it to partitioned Parquet, and look at Lake Formation permissions.
  - *Learning outcome:* Build a cost-efficient, queryable data lake on S3.
- [ ] **[Lab 37 — Hybrid Storage & Migration: DataSync, Storage Gateway, FSx & DMS](labs/lab-37-hybrid-migration.md)** ★★★
  - *Description:* Move data with DataSync, deploy FSx file systems, and explore Storage Gateway, Snow Family, and DMS migration patterns.
  - *Learning outcome:* Pick the right service to move or extend on-premises data and databases to AWS.

## Module 11 — Cost Optimisation & Capstone (Domain 4 + all domains)

- [ ] **[Lab 38 — Cost Visibility & Optimisation Tooling](labs/lab-38-cost-optimization.md)** ★★☆
  - *Description:* Use cost allocation tags, Cost Explorer, Budgets with actions, Compute Optimizer, and Trusted Advisor. Covers data-transfer cost patterns.
  - *Learning outcome:* Find savings and build cost governance into an architecture.
- [ ] **[Lab 39 — Capstone: Highly Available 3-Tier Web App with CloudFormation](labs/lab-39-capstone-cloudformation.md)** ★★★
  - *Description:* Deploy a full multi-AZ VPC, ALB, Auto Scaling, Aurora, and CloudFront stack as code, then run failure scenarios against it.
  - *Learning outcome:* Pull every exam domain together into one secure, resilient, high-performing, cost-optimised architecture.

---

## Progress summary

| Module | Labs | Done |
|---|---|---|
| 1 — Foundations | 01–02 | ☐ |
| 2 — IAM | 03–06 | ☐ |
| 3 — S3 | 07–10 | ☐ |
| 4 — VPC | 11–15 | ☐ |
| 5 — Compute | 16–20 | ☐ |
| 6 — Databases | 21–24 | ☐ |
| 7 — Decoupling & Serverless | 25–28 | ☐ |
| 8 — Security & Monitoring | 29–31 | ☐ |
| 9 — HA, DR & Global | 32–34 | ☐ |
| 10 — Data & Hybrid | 35–37 | ☐ |
| 11 — Cost & Capstone | 38–39 | ☐ |

> Swap a ☐ for ☑ when you finish a whole module.

## Exam readiness checklist

- [ ] Finished all 39 labs
- [ ] Can whiteboard a multi-AZ 3-tier architecture from memory
- [ ] Can explain each DR strategy and its RTO/RPO trade-offs
- [ ] Can pick S3/EBS/EFS/FSx storage for a given scenario
- [ ] Can choose between SQS, SNS, EventBridge, Kinesis, and Step Functions
- [ ] Scoring 80%+ on full-length practice exams
- [ ] Exam booked
