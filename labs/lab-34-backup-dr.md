# Lab 34 — Backup & Disaster Recovery Strategies

| | |
|---|---|
| **Module** | 9 — High Availability, Disaster Recovery & Global Delivery |
| **Exam mapping** | Domain 2 · Task 2.2 (DR strategies, RPO/RTO, backups) · Domain 4 · Task 4.1/4.3 (cost of DR) · Domain 1 · Task 1.3 (immutable backups) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 120 minutes |
| **Estimated cost** | ~$0.50 (small EBS and DynamoDB backups, cross-region copies, a short-lived DR instance) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-33-cloudfront-global-accelerator.md) · [Back to index](../README.md) · [Next lab ➡](lab-35-kinesis.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Define **RPO** and **RTO**, and map business requirements onto the four AWS **DR strategies**.
- Centralise backups with **AWS Backup**: plans, vaults, tag-based selection, and **cross-region / cross-account copy**.
- Protect backups from deletion with **Vault Lock** and the **logically air-gapped vault** concept.
- **Restore** resources and **measure** the actual RTO.
- Build and test a **pilot light** environment in a second region.

## DR strategies

```
 Cost/complexity ─────────────────────────────────────────────────────►
 ┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
 │ Backup & Restore │   Pilot Light    │  Warm Standby    │ Multi-site       │
 │                  │                  │                  │ active/active    │
 │ RPO hours        │ RPO minutes      │ RPO seconds      │ RPO ~0           │
 │ RTO 24h+         │ RTO tens of mins │ RTO minutes      │ RTO ~0           │
 │ Backups only in  │ Data replicated, │ Scaled-down but  │ Full prod in 2+  │
 │ DR region        │ core infra off/  │ fully running    │ regions serving  │
 │                  │ minimal          │ copy of prod     │ traffic          │
 └──────────────────┴──────────────────┴──────────────────┴──────────────────┘
 ◄──────────────────────────────────────────────────── RTO / RPO (time)
```

| Strategy | Data layer in DR | Compute in DR | Typical services |
|---|---|---|---|
| **Backup & restore** | Snapshots / backups copied cross-region | None until disaster | AWS Backup, S3 CRR, AMI copies |
| **Pilot light** | **Live replication** (RDS cross-region replica, DynamoDB global table, S3 CRR) | **Off** or minimal (AMIs + IaC ready) | + CloudFormation, Route 53 failover |
| **Warm standby** | Live replication | **Running at reduced capacity** (scale up on failover) | + ASG min > 0, Aurora Global DB |
| **Multi-site active/active** | Multi-active (DynamoDB global tables, Aurora Global write forwarding) | Full capacity, serving users | + Route 53 latency/weighted, Global Accelerator |

**AWS Elastic Disaster Recovery (DRS)** does continuous block-level replication of servers (on-prem or cloud) into AWS, with low-cost staging and launch in minutes. It's the managed answer to "replicate on-prem servers for DR".

---

## Part A — Console: AWS Backup

### A1. Create backup vaults

- [ ] Open **AWS Backup → Backup vaults → Create** in **us-east-1**: `saa-primary-vault`, encryption key: default.
- [ ] Switch to **us-west-2** and create `saa-dr-vault`.
- [ ] Open **Settings → Advanced features for Amazon DynamoDB backups → Enable**. This is required for cross-region copy and cold storage of DynamoDB backups.
- [ ] *(Read)* Look at **Vault lock**: **governance mode** (users with permissions can remove it) vs **compliance mode** (immutable after a cooling-off period, so not even root can delete recovery points). You'll apply governance mode from the CLI.

### A2. Resources to protect

- [ ] Create, and tag `Backup=gold`:
  - An **EBS volume** (gp3, 1 GiB, us-east-1a)
  - A **DynamoDB table** `saa-dr-table` (on-demand, PK `id`) with a few items

```bash
VOLB=$(aws ec2 create-volume --availability-zone us-east-1a --size 1 --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Backup,Value=gold},{Key=Name,Value=saa-dr-vol}]' --query VolumeId --output text)
aws dynamodb create-table --table-name saa-dr-table --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH --billing-mode PAY_PER_REQUEST --tags Key=Backup,Value=gold
aws dynamodb wait table-exists --table-name saa-dr-table
for i in 1 2 3; do aws dynamodb put-item --table-name saa-dr-table --item "{\"id\":{\"S\":\"item-$i\"}}"; done
```

### A3. Backup plan with cross-region copy

- [ ] Open **Backup plans → Create → Build a new plan**: `saa-gold-plan`.
  - Rule `daily`: frequency **Daily**, window start 05:00 UTC, retention **35 days**, vault `saa-primary-vault`
  - **Copy to destination:** region **us-west-2**, vault `saa-dr-vault`, retention 35 days
  - Rule `monthly`: monthly, transition to **cold storage after 30 days**, retain **1 year** (cold storage applies to supported resources such as EBS and DynamoDB)
- [ ] **Assign resources:** name `gold-tagged`, IAM role **Default role**, **Include specific resource types → All**, condition **tag `Backup` = `gold`**.

### A4. Run an on-demand backup and a cross-region copy

- [ ] Open **Protected resources → Create on-demand backup**: the DynamoDB table `saa-dr-table`, vault `saa-primary-vault`, then do the same for the EBS volume.
- [ ] When the jobs finish, open **Backup vaults → saa-primary-vault → recovery point → Actions → Copy** to **us-west-2 / saa-dr-vault**.

### A5. Restore and measure RTO

- [ ] Delete an item from `saa-dr-table` (a simulated incident). **Note the time.**
- [ ] Open the DynamoDB recovery point and choose **Restore**: new table name `saa-dr-table-restored`.
- [ ] **Note the time when the restore completes.** That's your **measured RTO** for this resource, and the backup time gives your **RPO**.
- [ ] Open **Jobs → Restore jobs** and look at the duration.

### A6. Restore testing (automated DR validation)

- [ ] Open **Restore testing → Create restore testing plan**: weekly, include recovery points of type EBS from `saa-primary-vault`, and tear down after validation. This proves backups are restorable on a schedule.

---

## Part B — CLI: Vault Lock, reporting, and a pilot-light failover

### B1. Inspect backup jobs and recovery points

- [ ] List recent jobs and recovery points:

```bash
aws backup list-backup-jobs --by-state COMPLETED --max-results 5 \
  --query 'BackupJobs[].{Resource:ResourceArn,Type:ResourceType,Created:CreationDate,Size:BackupSizeInBytes}' --output table

aws backup list-recovery-points-by-backup-vault --backup-vault-name saa-primary-vault \
  --query 'RecoveryPoints[].{Arn:RecoveryPointArn,Type:ResourceType,Status:Status}' --output table
aws backup list-recovery-points-by-backup-vault --backup-vault-name saa-dr-vault --region us-west-2 \
  --query 'RecoveryPoints[].{Type:ResourceType,Status:Status}' --output table
```

### B2. Vault Lock (governance mode)

- [ ] Recovery points in the vault must be kept between 1 and 365 days:

```bash
aws backup put-backup-vault-lock-configuration --backup-vault-name saa-primary-vault \
  --min-retention-days 1 --max-retention-days 365
aws backup describe-backup-vault --backup-vault-name saa-primary-vault \
  --query '{Locked:Locked,Min:MinRetentionDays,Max:MaxRetentionDays}'
```

> Adding `--changeable-for-days 3` makes it **compliance mode**: after 3 days the lock can **never** be removed. **Don't use that flag in this lab.**

### B3. Deny deleting recovery points with a vault access policy

- [ ] Apply the policy:

```bash
aws backup put-backup-vault-access-policy --backup-vault-name saa-primary-vault --policy '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Deny","Principal":"*","Action":["backup:DeleteRecoveryPoint","backup:UpdateRecoveryPointLifecycle"],"Resource":"*"}]}'
```

### B4. Pilot light: prepare the DR region

The "app" is the web launch template from Lab 16. In a pilot light, **data is replicated** and **compute is pre-staged but off**.

- [ ] **Data:** make the DynamoDB table a **global table** with a replica in us-west-2, so the data is always live:

```bash
aws dynamodb update-table --table-name saa-dr-table \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
sleep 20
aws dynamodb update-table --table-name saa-dr-table --replica-updates '[{"Create":{"RegionName":"us-west-2"}}]'
aws dynamodb describe-table --table-name saa-dr-table --query 'Table.Replicas[].{Region:RegionName,Status:ReplicaStatus}'
```

- [ ] **Compute:** copy an AMI to us-west-2 (or rely on the public AL2023 AMI + user data) and pre-create a launch template in the DR region:

```bash
DR_AMI=$(aws ssm get-parameter --region us-west-2 --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
cat > dr-userdata.sh <<'EOF'
#!/bin/bash
dnf -y install httpd
echo "<h1>DR SITE us-west-2 is LIVE</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF
DR_SG=$(aws ec2 create-security-group --region us-west-2 --group-name saa-dr-web --description "DR web" \
  --vpc-id $(aws ec2 describe-vpcs --region us-west-2 --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text) \
  --query GroupId --output text)
aws ec2 authorize-security-group-ingress --region us-west-2 --group-id $DR_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 create-launch-template --region us-west-2 --launch-template-name saa-dr-web-lt --launch-template-data "{
  \"ImageId\":\"$DR_AMI\",\"InstanceType\":\"t3.micro\",\"SecurityGroupIds\":[\"$DR_SG\"],
  \"UserData\":\"$(base64 < dr-userdata.sh | tr -d '\n')\"}"
```

> In real life, the DR region would hold a **CloudFormation stack** (VPC, ALB, an ASG with desired = 0) that you scale up during failover.

### B5. Declare a disaster and fail over (measure RTO)

- [ ] Start the clock and launch the DR web tier:

```bash
START=$(date +%s)
DR_I=$(aws ec2 run-instances --region us-west-2 --launch-template LaunchTemplateName=saa-dr-web-lt \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --region us-west-2 --instance-ids $DR_I
DR_IP=$(aws ec2 describe-instances --region us-west-2 --instance-ids $DR_I --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
until curl -s -m 3 http://$DR_IP | grep -q LIVE; do sleep 5; done
echo "DR web tier serving after $(( $(date +%s) - START )) seconds"
```

- [ ] Confirm the data is already in the DR region (RPO ≈ seconds, thanks to the global table):

```bash
aws dynamodb scan --region us-west-2 --table-name saa-dr-table --query 'Count'
```

> In production you'd now switch DNS (**Route 53 failover**, Lab 32) or rely on **Global Accelerator** health checks.

### B6. Backup Audit Manager (compliance reporting)

- [ ] Create a framework that checks backup coverage:

```bash
aws backup create-framework --framework-name saa_backup_framework --framework-controls '[
  {"ControlName":"BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN"},
  {"ControlName":"BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK","ControlInputParameters":[{"ParameterName":"requiredRetentionDays","ParameterValue":"35"}]}
]'
```

> It needs AWS Config recording (Lab 30). The results show whether every resource is protected and kept long enough.

---

## Verify

- [ ] Recovery points exist in **both** us-east-1 and us-west-2 vaults.
- [ ] You restored a DynamoDB table and recorded the RTO.
- [ ] The pilot-light DR web tier came up in us-west-2, the measured RTO was printed, and the data was already there.

## Exam tips

- **"Lowest cost, RTO of a day is fine"** → **backup & restore**.
- **"RTO in minutes, keep cost low"** → **pilot light** (data replicated, compute off) or **warm standby** (compute running small).
- **"Near-zero RTO/RPO"** → **multi-site active/active**.
- **"Centrally manage and audit backups across services and accounts"** → **AWS Backup** (with Organizations backup policies).
- **"Backups must be immutable even against admins (ransomware)"** → **Backup Vault Lock in compliance mode**, ideally cross-account.
- **"Replicate on-prem servers to AWS for DR with minimal downtime"** → **AWS Elastic Disaster Recovery**.
- RPO = **how much data you can lose**. RTO = **how long you can be down**.

## Challenge (optional)

- [ ] Write a CloudFormation template for the DR region (VPC + ALB + ASG with desired 0) and turn the failover into a single `update-stack` that sets desired = 2.

## Cleanup

- [ ] Remove the DR instance, launch template, and SG:

```bash
aws ec2 terminate-instances --region us-west-2 --instance-ids $DR_I
aws ec2 delete-launch-template --region us-west-2 --launch-template-name saa-dr-web-lt
aws ec2 wait instance-terminated --region us-west-2 --instance-ids $DR_I
aws ec2 delete-security-group --region us-west-2 --group-id $DR_SG
```

- [ ] Remove the global table replica, then delete the tables:

```bash
aws dynamodb update-table --table-name saa-dr-table --replica-updates '[{"Delete":{"RegionName":"us-west-2"}}]'
sleep 120
aws dynamodb delete-table --table-name saa-dr-table
aws dynamodb delete-table --table-name saa-dr-table-restored
aws ec2 delete-volume --volume-id $VOLB
```

- [ ] Remove the vault access policy and lock, then delete the recovery points:

```bash
aws backup delete-backup-vault-access-policy --backup-vault-name saa-primary-vault
aws backup delete-backup-vault-lock-configuration --backup-vault-name saa-primary-vault
for V in "saa-primary-vault us-east-1" "saa-dr-vault us-west-2"; do
  set -- $V
  for RP in $(aws backup list-recovery-points-by-backup-vault --backup-vault-name $1 --region $2 --query 'RecoveryPoints[].RecoveryPointArn' --output text); do
    aws backup delete-recovery-point --backup-vault-name $1 --recovery-point-arn $RP --region $2
  done
done
```

- [ ] Delete the plan (selections first), the vaults, the framework, and the restore testing plan:

```bash
PLAN=$(aws backup list-backup-plans --query "BackupPlansList[?BackupPlanName=='saa-gold-plan'].BackupPlanId" --output text)
for S in $(aws backup list-backup-selections --backup-plan-id $PLAN --query 'BackupSelectionsList[].SelectionId' --output text); do
  aws backup delete-backup-selection --backup-plan-id $PLAN --selection-id $S; done
aws backup delete-backup-plan --backup-plan-id $PLAN
aws backup delete-backup-vault --backup-vault-name saa-primary-vault
aws backup delete-backup-vault --backup-vault-name saa-dr-vault --region us-west-2
aws backup delete-framework --framework-name saa_backup_framework
# delete the restore testing plan in the console
rm dr-userdata.sh
```

---

## Knowledge check

**Q1.** A company's RTO is 4 hours and its RPO is 1 hour. It wants the cheapest option that meets both. Which DR strategy?

<details><summary>Answer</summary>

**Pilot light.** Backup & restore would usually miss a 4-hour RTO for complex stacks, and warm standby costs more than needed. For a simple stack with a fast restore, backup & restore with hourly backups might do, but pilot light is the typical exam answer.

</details>

**Q2.** Ransomware compromised an admin account and deleted backups. How do you stop this happening again?

<details><summary>Answer</summary>

Use **AWS Backup Vault Lock (compliance mode)** and **cross-account copies** to a separate backup account (or a logically air-gapped vault), with SCPs restricting backup deletion.

</details>

**Q3.** A company runs 200 on-prem VMs and needs DR in AWS with an RPO of seconds and an RTO of minutes, without paying for full duplicate compute. Which service?

<details><summary>Answer</summary>

**AWS Elastic Disaster Recovery (DRS).**

</details>
