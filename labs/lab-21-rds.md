# Lab 21 — Amazon RDS: Multi-AZ, Read Replicas & Backups

| | |
|---|---|
| **Module** | 6 — Databases |
| **Exam mapping** | Domain 2 · Task 2.2 (HA / fault-tolerant DB) · Domain 3 · Task 3.3 (high-performing DB) · Domain 4 · Task 4.3 (cost-optimised DB) · Domain 1 · Task 1.3 (encryption) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 120 minutes (mostly waiting for provisioning) |
| **Estimated cost** | ~$0.10–0.20/hour (db.t3.micro Multi-AZ + replica, client instance) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-20-ec2-pricing-placement.md) · [Back to index](../README.md) · [Next lab ➡](lab-22-aurora-rds-proxy.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Deploy RDS in **private DB subnets** with **security-group-based** access.
- Tell **Multi-AZ** (HA, synchronous, standby not readable) apart from **read replicas** (scaling, asynchronous, readable).
- Run a **forced failover** and watch the endpoint DNS move.
- Use **automated backups**, **point-in-time recovery (PITR)**, and **manual snapshots**, including cross-region copies.
- Manage the master password in **Secrets Manager** and understand **IAM DB authentication**.

## Multi-AZ vs read replicas

| | Multi-AZ DB instance | Multi-AZ DB cluster | Read replica |
|---|---|---|---|
| Purpose | **High availability** | HA + read scaling | **Read scaling**, DR (cross-region) |
| Replication | Synchronous | Semi-synchronous | **Asynchronous** |
| Standby readable? | **No** | Yes (2 readers) | Yes |
| Failover | Automatic, 60–120 s, same endpoint | Automatic, ~35 s | Manual **promotion** |
| Cross-region | No | No | **Yes** |
| Engines | All | MySQL, PostgreSQL | MySQL, MariaDB, PostgreSQL, Oracle, SQL Server |

---

## Prerequisites

- `saa-vpc` with data subnets, and `saa-app-sg` / `saa-db-sg` from Lab 13 (`saa-db-sg` allows 3306 from `saa-app-sg`).
- A **DB client** instance: AL2023 t3.micro in **saa-public-a**, public IP on, SG **`saa-app-sg`**, profile `saa-ssm-ec2-role`. Session Manager reaches it over the IGW.

```bash
source ~/saa-vpc.env
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
CLIENT=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PUB_A \
  --security-group-ids $APP_SG --iam-instance-profile Name=saa-ssm-ec2-role \
  --metadata-options HttpTokens=required \
  --user-data '#!/bin/bash
dnf -y install mariadb105 postgresql16 jq bind-utils' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-db-client}]' \
  --query 'Instances[0].InstanceId' --output text)
echo "export CLIENT=$CLIENT" >> ~/saa-vpc.env
```

---

## Part A — Console

### A1. DB subnet group

- [ ] Open **RDS → Subnet groups → Create**: name `saa-db-subnets`, VPC `saa-vpc`, AZs us-east-1a/b, subnets **saa-data-a** and **saa-data-b**.

### A2. Create a Multi-AZ MySQL instance

- [ ] Open **RDS → Create database → Standard create → MySQL 8.0**.
  - Templates: **Dev/Test**
  - Availability: **Multi-AZ DB instance** (one standby)
  - Identifier `saa-mysql`, master user `admin`, **Credentials management: Managed in AWS Secrets Manager**
  - Instance class **db.t3.micro** (burstable), storage **gp3 20 GiB**, storage autoscaling on (max 100)
  - Connectivity: VPC `saa-vpc`, subnet group `saa-db-subnets`, **Public access: No**, SG **`saa-db-sg`** (remove `default`)
  - Additional: initial DB name `saadb`, backup retention **7 days**, **Encryption** on (default `aws/rds` key), Performance Insights on (7 days, free), **Enhanced monitoring** 60 s, deletion protection **off** (for the lab)
- [ ] Wait about 15–20 minutes for **Available**.
- [ ] Look at the **Configuration** tab: *Multi-AZ: Yes* and *Secondary zone*.

### A3. Connect and load data

- [ ] Open **Secrets Manager → rds!db-...**, choose **Retrieve secret value**, and copy the password.
- [ ] Start a Session Manager session on `saa-db-client` and run:

```bash
mysql -h <saa-mysql-endpoint> -u admin -p saadb
```

```sql
CREATE TABLE orders (id INT AUTO_INCREMENT PRIMARY KEY, item VARCHAR(50), created TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO orders (item) VALUES ('keyboard'),('mouse'),('monitor');
SELECT * FROM orders;
```

### A4. Forced failover

- [ ] In a second terminal on the client, watch the endpoint:

```bash
while true; do echo "$(date +%T) $(dig +short <saa-mysql-endpoint> | tail -1)"; sleep 5; done
```

- [ ] In the console, choose **Actions → Reboot → Reboot with failover**.
- [ ] Watch the IP change after about 60–120 s. The **endpoint name stays the same**, which is why apps should always connect by DNS name.
- [ ] Open **Logs & events → Recent events** and find *"Multi-AZ instance failover started / completed"*.

### A5. Read replica

- [ ] Choose **Actions → Create read replica**: identifier `saa-mysql-replica`, db.t3.micro, same region, Single-AZ.
- [ ] When it's available, connect to the **replica endpoint**. `SELECT` works, and `INSERT` fails with *read-only*.
- [ ] Insert a row on the primary and read it from the replica. It's there within seconds, because replication is asynchronous.
- [ ] Look at the **ReplicaLag** CloudWatch metric.

### A6. Point-in-time restore

- [ ] On the primary, run `DELETE FROM orders;` (a simulated mistake). Note the time.
- [ ] Choose **Actions → Restore to point in time**, pick a **custom time** about 2 minutes *before* the delete, and name it `saa-mysql-pitr`.
- [ ] When it's ready, connect to it and confirm the rows are back. **PITR always creates a NEW instance** with a new endpoint.

---

## Part B — CLI

### B1. Inspect the instance

- [ ] Look at its key settings:

```bash
aws rds describe-db-instances --db-instance-identifier saa-mysql \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,MultiAZ:MultiAZ,AZ:AvailabilityZone,Secondary:SecondaryAvailabilityZone,Encrypted:StorageEncrypted,Backup:BackupRetentionPeriod,Secret:MasterUserSecret.SecretArn}'
```

### B2. Create a PostgreSQL instance entirely from the CLI

- [ ] Create it:

```bash
aws rds create-db-instance --db-instance-identifier saa-pg \
  --engine postgres --engine-version 16 \
  --db-instance-class db.t3.micro --allocated-storage 20 --storage-type gp3 \
  --master-username pgadmin --manage-master-user-password \
  --db-subnet-group-name saa-db-subnets --vpc-security-group-ids $DB_SG \
  --no-publicly-accessible --storage-encrypted \
  --backup-retention-period 3 --enable-iam-database-authentication \
  --tags Key=Name,Value=saa-pg
```

- [ ] Allow PostgreSQL from the app tier:

```bash
aws ec2 authorize-security-group-ingress --group-id $DB_SG \
  --ip-permissions "IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=$APP_SG}]"
aws rds wait db-instance-available --db-instance-identifier saa-pg
```

### B3. Convert to Multi-AZ with no downtime

- [ ] Modify it:

```bash
aws rds modify-db-instance --db-instance-identifier saa-pg --multi-az --apply-immediately
aws rds describe-db-instances --db-instance-identifier saa-pg --query 'DBInstances[0].{Status:DBInstanceStatus,Pending:PendingModifiedValues}'
```

### B4. Manual snapshot and cross-region copy

- [ ] Take a snapshot:

```bash
aws rds create-db-snapshot --db-instance-identifier saa-mysql --db-snapshot-identifier saa-mysql-manual-1
aws rds wait db-snapshot-available --db-snapshot-identifier saa-mysql-manual-1
```

- [ ] Copy it to us-west-2. Encrypted copies need a KMS key in the destination region:

```bash
aws rds copy-db-snapshot --region us-west-2 \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:$ACCOUNT_ID:snapshot:saa-mysql-manual-1 \
  --target-db-snapshot-identifier saa-mysql-dr-copy \
  --kms-key-id alias/aws/rds --source-region us-east-1
```

### B5. Cross-region read replica (optional, ~$0.02/h)

- [ ] Create it:

```bash
aws rds create-db-instance-read-replica --region us-west-2 \
  --db-instance-identifier saa-mysql-west-replica \
  --source-db-instance-identifier arn:aws:rds:us-east-1:$ACCOUNT_ID:db:saa-mysql \
  --db-instance-class db.t3.micro --kms-key-id alias/aws/rds
```

> In a regional disaster, **promote** it: `aws rds promote-read-replica --db-instance-identifier saa-mysql-west-replica --region us-west-2`.

### B6. IAM database authentication (PostgreSQL)

- [ ] On the **client instance**, connect as the master user and create an IAM-authenticated user:

```bash
PGPASS=$(aws secretsmanager get-secret-value --secret-id <saa-pg-secret-arn> --query SecretString --output text | jq -r .password)
PGPASSWORD=$PGPASS psql "host=<saa-pg-endpoint> user=pgadmin dbname=postgres sslmode=require" \
  -c "CREATE USER iam_reader; GRANT rds_iam TO iam_reader;"
```

- [ ] Add a policy to `saa-ssm-ec2-role` that allows `rds-db:connect` for `arn:aws:rds-db:us-east-1:<acct>:dbuser:<DbiResourceId>/iam_reader`.
- [ ] On the client, connect with a **15-minute auth token** instead of a password:

```bash
TOKEN=$(aws rds generate-db-auth-token --hostname <saa-pg-endpoint> --port 5432 --username iam_reader --region us-east-1)
PGPASSWORD=$TOKEN psql "host=<saa-pg-endpoint> user=iam_reader dbname=postgres sslmode=require" -c "select current_user;"
```

### B7. Look at the events and the CloudWatch storage metric

- [ ] Run:

```bash
aws rds describe-events --source-identifier saa-mysql --source-type db-instance --duration 120 \
  --query 'Events[].{Time:Date,Msg:Message}' --output table

aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=saa-mysql --statistics Minimum \
  --start-time $(date -u -v-1H +%FT%TZ 2>/dev/null || date -u -d '1 hour ago' +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 300
```

---

## Verify

- [ ] Forced failover moved the endpoint to a new IP without changing its name.
- [ ] The read replica serves reads and rejects writes.
- [ ] PITR recovered the deleted rows into a new instance.

## Exam tips

- **"HA / automatic failover"** → **Multi-AZ**. **"Offload read traffic / reporting"** → **read replicas**. **"Regional DR for RDS"** → **cross-region read replica** (promote it) or **cross-region automated backups or snapshot copies**.
- You can't encrypt an existing unencrypted RDS instance directly. Use **snapshot → copy with encryption → restore**.
- **Automated backups:** retention 1–35 days, PITR to within about 5 minutes. **Manual snapshots** are kept until you delete them.
- **Storage autoscaling** handles unpredictable growth.
- **RDS Custom** (Oracle/SQL Server) gives OS-level access. Plain RDS doesn't.
- **"Avoid storing DB credentials in code"** → **Secrets Manager** (with automatic rotation) or **IAM DB authentication**.

## Challenge (optional)

- [ ] Create a **Multi-AZ DB cluster** (MySQL, 2 readable standbys) and compare its failover time and reader endpoint with the Multi-AZ instance.

## Cleanup

> RDS charges per hour. Delete everything today.

- [ ] Delete the instances, skipping the final snapshot:

```bash
for DB in saa-mysql-replica saa-mysql-pitr saa-pg saa-mysql; do
  aws rds delete-db-instance --db-instance-identifier $DB --skip-final-snapshot --delete-automated-backups 2>/dev/null
done
aws rds delete-db-instance --db-instance-identifier saa-mysql-west-replica --region us-west-2 --skip-final-snapshot 2>/dev/null
```

- [ ] Delete the manual snapshots:

```bash
aws rds delete-db-snapshot --db-snapshot-identifier saa-mysql-manual-1
aws rds delete-db-snapshot --db-snapshot-identifier saa-mysql-dr-copy --region us-west-2
```

- [ ] **Keep** the `saa-db-subnets` subnet group (free), and keep the `saa-db-client` instance if you're going on to Lab 22 (otherwise terminate it).
- [ ] The Secrets Manager secrets created by RDS are deleted along with their instances.

---

## Knowledge check

**Q1.** A reporting workload is slowing down a production RDS MySQL database. What's the least disruptive fix?

<details><summary>Answer</summary>

Create a **read replica** and point the reporting queries at the replica endpoint.

</details>

**Q2.** An RDS database must survive the loss of an AZ with automatic failover and no application connection-string changes. What should you configure?

<details><summary>Answer</summary>

**RDS Multi-AZ.** Failover updates the same DNS endpoint to point at the promoted standby.

</details>

**Q3.** A company must encrypt an existing unencrypted RDS instance. How?

<details><summary>Answer</summary>

Take a **snapshot**, **copy the snapshot with encryption** turned on (choose a KMS key), **restore** a new instance from the encrypted copy, and cut over.

</details>
