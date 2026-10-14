# Lab 22 — Amazon Aurora, Aurora Serverless v2 & RDS Proxy

| | |
|---|---|
| **Module** | 6 — Databases |
| **Exam mapping** | Domain 3 · Task 3.3 (high-performing DB) · Domain 2 · Task 2.2 (HA, global DB) · Domain 4 · Task 4.3 (serverless DB cost) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 120 minutes |
| **Estimated cost** | ~$0.20–0.40/hour (Serverless v2 at 0.5–2 ACU × 2 instances, plus RDS Proxy) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-21-rds.md) · [Back to index](../README.md) · [Next lab ➡](lab-23-dynamodb.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Explain Aurora's architecture: **shared cluster storage** (6 copies across 3 AZs), separate compute, and up to **15 replicas**.
- Use **cluster (writer)**, **reader**, and **custom** endpoints.
- Run **Aurora Serverless v2** that scales capacity (ACUs) up and down with load.
- **Clone** a cluster instantly with copy-on-write, and recognise **Backtrack** and **Global Database**.
- Pool and share connections with **RDS Proxy** to improve failover times and handle Lambda connection storms.

## Aurora quick facts

| Feature | Detail |
|---|---|
| Storage | Auto-grows to 128 TiB. 6 copies in 3 AZs. Tolerates losing 2 copies for writes and 3 for reads. |
| Replicas | Up to 15, with replica lag usually **under 10 ms**. Any replica can become the writer. |
| Failover | Usually **under 30 s** (faster with RDS Proxy) |
| **Global Database** | 1 primary region + up to 10 secondary regions, typically under 1 s replication lag, cross-region failover RTO around **1 minute** |
| **Serverless v2** | Scales in 0.5-ACU steps (1 ACU ≈ 2 GiB RAM), from **0 ACU (auto-pause)** up to 256 ACU |
| Backtrack (MySQL) | "Rewind" the cluster in place without restoring |
| Cloning | Copy-on-write. Fast and cheap for test copies of prod. |
| I/O-Optimized | A storage config for I/O-heavy workloads: no per-I/O charge, higher storage price |

---

## Prerequisites

- `saa-db-subnets`, `saa-db-sg`, and the `saa-db-client` instance from Lab 21.

---

## Part A — Console

### A1. Create an Aurora MySQL Serverless v2 cluster

- [ ] Open **RDS → Create database → Standard create → Aurora (MySQL Compatible)**.
  - Templates **Dev/Test**, identifier `saa-aurora`, master `admin`, **Managed in Secrets Manager**
  - Cluster storage configuration: **Aurora Standard**
  - Instance configuration: **Serverless v2**, min **0.5** ACU, max **4** ACU
  - Availability: **Create an Aurora Replica in a different AZ** ✔
  - Connectivity: `saa-vpc`, `saa-db-subnets`, not public, SG `saa-db-sg`
  - Initial DB name `saadb`, encryption on, backups 1 day
- [ ] Wait about 15 minutes. Open the cluster and look at:
  - **Writer instance** and **Reader instance** in different AZs
  - **Endpoints**: the *cluster endpoint* (writer) and the *reader endpoint* (load-balanced across readers)

### A2. Connect to the writer and reader endpoints

- [ ] From `saa-db-client` (Session Manager), run:

```bash
mysql -h <cluster-endpoint> -u admin -p -e "SELECT @@aurora_server_id, @@innodb_read_only;"
mysql -h <reader-endpoint>  -u admin -p -e "SELECT @@aurora_server_id, @@innodb_read_only;"
```

> The writer shows `innodb_read_only = 0` and the reader shows `1`.

### A3. Watch Serverless v2 scale

- [ ] Generate CPU load from the client with 8 parallel CPU-heavy queries (about 2–3 minutes):

```bash
read -s -p "Password: " PW; echo
for i in $(seq 1 8); do
  mysql -h <cluster-endpoint> -u admin -p"$PW" -e "SELECT BENCHMARK(400000000, SHA2('saa',256));" &
done; wait
```

- [ ] Open **Monitoring** on the writer instance and watch **ServerlessDatabaseCapacity (ACU)** and **CPUUtilization** go up, then back down to 0.5 ACU once the load stops.

### A4. Failover

- [ ] Select the **writer** instance and choose **Actions → Failover**.
- [ ] Re-run the `@@aurora_server_id` query against the **cluster endpoint**. A different instance is now the writer, and the endpoint followed it automatically.

### A5. Create an RDS Proxy

- [ ] Open **RDS → Proxies → Create proxy**.
  - Engine **MariaDB/MySQL**, identifier `saa-aurora-proxy`
  - Target: cluster `saa-aurora`
  - Secrets: the cluster's Secrets Manager secret. IAM role: **Create new**.
  - Subnets: data subnets, SG `saa-db-sg`, **Require TLS** ✔
- [ ] When it's available (about 10 minutes), connect through the **proxy endpoint**:

```bash
mysql -h <proxy-endpoint> -u admin -p --ssl -e "SELECT @@aurora_server_id;"
```

- [ ] Run a failover again while looping queries through the proxy. With the proxy, the client sees a **shorter interruption** than when connecting straight to the cluster endpoint.

### A6. Clone the cluster

- [ ] Select the cluster and choose **Actions → Create clone**, `saa-aurora-clone`, Serverless v2 (0.5–1 ACU).
- [ ] Notice how quickly it's created. Pages are only copied when either cluster changes them.

---

## Part B — CLI

### B1. Describe the cluster and its endpoints

- [ ] Look at the members and endpoints:

```bash
aws rds describe-db-clusters --db-cluster-identifier saa-aurora \
  --query 'DBClusters[0].{Writer:Endpoint,Reader:ReaderEndpoint,Members:DBClusterMembers[].{Id:DBInstanceIdentifier,Writer:IsClusterWriter},Scaling:ServerlessV2ScalingConfiguration}'
```

### B2. Change the Serverless v2 scaling range (including auto-pause)

- [ ] Allow scaling to zero after 10 idle minutes. 0 ACU means auto-pause, and you pay only for storage:

```bash
aws rds modify-db-cluster --db-cluster-identifier saa-aurora \
  --serverless-v2-scaling-configuration MinCapacity=0,MaxCapacity=4,SecondsUntilAutoPause=600 \
  --apply-immediately
```

> Auto-pause suits dev/test. The first connection after a pause waits for a resume (roughly 15 s).

### B3. Add a reader and a custom endpoint

- [ ] Add a third (reporting) instance:

```bash
aws rds create-db-instance --db-instance-identifier saa-aurora-reporting \
  --db-cluster-identifier saa-aurora --engine aurora-mysql \
  --db-instance-class db.serverless --promotion-tier 15
aws rds wait db-instance-available --db-instance-identifier saa-aurora-reporting
```

- [ ] Create a custom endpoint that only includes the reporting instance:

```bash
aws rds create-db-cluster-endpoint --db-cluster-identifier saa-aurora \
  --db-cluster-endpoint-identifier saa-reporting --endpoint-type READER \
  --static-members saa-aurora-reporting
aws rds describe-db-cluster-endpoints --db-cluster-identifier saa-aurora \
  --query 'DBClusterEndpoints[].{Type:EndpointType,Custom:CustomEndpointType,Endpoint:Endpoint}' --output table
```

> **Promotion tier 15** (the lowest priority) makes failover pick other readers first. A custom endpoint isolates analytics queries from the app's readers.

### B4. Replica auto scaling (Application Auto Scaling)

- [ ] Register the cluster and add a policy:

```bash
aws application-autoscaling register-scalable-target --service-namespace rds \
  --resource-id cluster:saa-aurora --scalable-dimension rds:cluster:ReadReplicaCount \
  --min-capacity 1 --max-capacity 3

aws application-autoscaling put-scaling-policy --service-namespace rds \
  --resource-id cluster:saa-aurora --scalable-dimension rds:cluster:ReadReplicaCount \
  --policy-name reader-cpu-60 --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{"TargetValue":60,"PredefinedMetricSpecification":{"PredefinedMetricType":"RDSReaderAverageCPUUtilization"}}'
```

### B5. Clone from the CLI

- [ ] Create the clone cluster and give it an instance:

```bash
aws rds restore-db-cluster-to-point-in-time --source-db-cluster-identifier saa-aurora \
  --db-cluster-identifier saa-aurora-clone-cli --restore-type copy-on-write --use-latest-restorable-time \
  --db-subnet-group-name saa-db-subnets --vpc-security-group-ids $DB_SG \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=1

aws rds create-db-instance --db-instance-identifier saa-aurora-clone-cli-1 \
  --db-cluster-identifier saa-aurora-clone-cli --engine aurora-mysql --db-instance-class db.serverless
```

### B6. RDS Proxy details

- [ ] Inspect the proxy and its targets:

```bash
aws rds describe-db-proxies --db-proxy-name saa-aurora-proxy \
  --query 'DBProxies[0].{Endpoint:Endpoint,Status:Status,TLS:RequireTLS,Idle:IdleClientTimeout}'
aws rds describe-db-proxy-targets --db-proxy-name saa-aurora-proxy \
  --query 'Targets[].{Id:RdsResourceId,Role:Role,Health:TargetHealth.State}' --output table
```

### B7. Global Database (read-through, or try it, ~$0.10/h extra)

```bash
# aws rds create-global-cluster --global-cluster-identifier saa-global --source-db-cluster-identifier arn:aws:rds:us-east-1:$ACCOUNT_ID:cluster:saa-aurora
# aws rds create-db-cluster --region us-west-2 --db-cluster-identifier saa-aurora-west \
#   --global-cluster-identifier saa-global --engine aurora-mysql --engine-version <same>
# Managed planned failover / switchover:
# aws rds switchover-global-cluster --global-cluster-identifier saa-global --target-db-cluster-identifier <west-arn>
```

- [ ] Read the Global Database commands above. Optionally run them if you're happy with the extra cost.

---

## Verify

- [ ] The writer and reader endpoints resolve to different instances, and the writer endpoint followed a failover.
- [ ] Serverless v2 capacity went up under load and back down afterwards.
- [ ] Connections through RDS Proxy work over TLS.

## Exam tips

- **"MySQL/PostgreSQL with the highest performance and availability, up to 15 low-lag replicas"** → **Aurora**.
- **"Intermittent or unpredictable DB load, pay for what you use"** → **Aurora Serverless v2**.
- **"Global app with sub-second cross-region reads and fast regional DR"** → **Aurora Global Database**.
- **"Lambda functions exhaust DB connections"** or **"reduce failover time for apps"** → **RDS Proxy**.
- **"Create a production-like test database quickly and cheaply"** → **Aurora cloning**.
- **"Undo a bad write quickly without a restore"** (Aurora MySQL) → **Backtrack**.
- Use **custom endpoints** to send analytics to a dedicated set of larger readers.

## Challenge (optional)

- [ ] Turn on **Aurora zero-ETL integration with Amazon Redshift** (concept) and explain when it would replace a nightly ETL pipeline.

## Cleanup

> Aurora and RDS Proxy cost money every hour. Delete everything now.

- [ ] Delete the RDS Proxy:

```bash
aws rds delete-db-proxy --db-proxy-name saa-aurora-proxy
```

- [ ] Delete the clones (instances first, then clusters):

```bash
aws rds delete-db-instance --db-instance-identifier saa-aurora-clone-cli-1 --skip-final-snapshot
# delete the console clone's instance(s) too
aws rds wait db-instance-deleted --db-instance-identifier saa-aurora-clone-cli-1
aws rds delete-db-cluster --db-cluster-identifier saa-aurora-clone-cli --skip-final-snapshot
aws rds delete-db-cluster --db-cluster-identifier saa-aurora-clone --skip-final-snapshot 2>/dev/null
```

- [ ] Delete the main cluster's instances, then the cluster:

```bash
aws application-autoscaling deregister-scalable-target --service-namespace rds --resource-id cluster:saa-aurora --scalable-dimension rds:cluster:ReadReplicaCount
for I in $(aws rds describe-db-clusters --db-cluster-identifier saa-aurora --query 'DBClusters[0].DBClusterMembers[].DBInstanceIdentifier' --output text); do
  aws rds delete-db-instance --db-instance-identifier $I --skip-final-snapshot
done
for I in $(aws rds describe-db-clusters --db-cluster-identifier saa-aurora --query 'DBClusters[0].DBClusterMembers[].DBInstanceIdentifier' --output text); do
  aws rds wait db-instance-deleted --db-instance-identifier $I
done
aws rds delete-db-cluster --db-cluster-identifier saa-aurora --skip-final-snapshot
```

- [ ] Delete the IAM role the RDS Proxy wizard created.
- [ ] Keep `saa-db-client` if you're going on to Lab 24 (ElastiCache). Otherwise terminate it.

---

## Knowledge check

**Q1.** A serverless app on Lambda opens thousands of short-lived connections to Aurora MySQL during peaks and hits `Too many connections`. What fixes this with the fewest code changes?

<details><summary>Answer</summary>

Put **Amazon RDS Proxy** in front of Aurora. It pools and reuses connections. Point the Lambda function at the proxy endpoint.

</details>

**Q2.** A global e-commerce company needs a relational database with low-latency reads in three continents and recovery in another region within about a minute. What should it use?

<details><summary>Answer</summary>

**Aurora Global Database**, with secondary regions for local reads and managed cross-region failover.

</details>

**Q3.** A development database is used for a few hours a day at unpredictable times. How do you minimise cost while keeping it MySQL-compatible?

<details><summary>Answer</summary>

**Aurora Serverless v2** with a minimum of **0 ACU (auto-pause)**, so you pay only for capacity while it's in use, plus storage.

</details>
