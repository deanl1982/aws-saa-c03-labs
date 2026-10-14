# Lab 37 — Hybrid Storage & Migration: DataSync, Storage Gateway, FSx & DMS

| | |
|---|---|
| **Module** | 10 — Data Ingestion, Analytics & Hybrid |
| **Exam mapping** | Domain 3 · Task 3.1 (hybrid storage, file systems) · Task 3.5 (data transfer and migration) · Domain 4 · Task 4.1 (storage cost) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 120 minutes |
| **Estimated cost** | ~$0.50–1.50 (DataSync per GB, a Storage Gateway EC2 host at m5.xlarge ~$0.19/h, optional FSx for Lustre ~$0.20/h) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-36-glue-athena-datalake.md) · [Back to index](../README.md) · [Next lab ➡](lab-38-cost-optimization.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Move data online with **AWS DataSync** (scheduling, verification, filters).
- Extend on-prem storage into S3 with **Storage Gateway** (S3 File, FSx File, Volume, and Tape gateways).
- Pick the right **FSx** file system (Windows, Lustre, NetApp ONTAP, OpenZFS).
- Choose between online and offline transfer: DataSync, Transfer Family, Direct Connect, Snow Family, and **Data Transfer Terminal**.
- Plan database migrations with **DMS** and **SCT**, and server migrations with **Application Migration Service (MGN)**.

## Decision tables

**Moving files and data**

| Scenario | Service |
|---|---|
| Recurring or one-time **online** transfer between on-prem NFS/SMB/HDFS/object and S3/EFS/FSx, with verification | **DataSync** |
| On-prem apps need **local NFS/SMB access** with data stored in S3, plus a local cache | **S3 File Gateway** |
| Windows users need low-latency local access to **FSx for Windows** shares | **FSx File Gateway** (or FSx with a caching option) |
| Block storage (iSCSI) backed by S3 snapshots | **Volume Gateway** (cached or stored) |
| Replace physical tape backups | **Tape Gateway** (VTL → S3 Glacier) |
| Partners send files over **SFTP/FTPS/AS2** into S3/EFS | **AWS Transfer Family** |
| **Tens of TB to PB**, limited bandwidth, or no network | **Snowball Edge** (offline; check current availability) / **Data Transfer Terminal** |
| Continuous, consistent high bandwidth | **Direct Connect** + DataSync |

**FSx family**

| FSx type | Protocol | Exam trigger |
|---|---|---|
| **FSx for Windows File Server** | SMB, NTFS, **Active Directory**, DFS | "Windows file shares", "lift-and-shift Windows apps" |
| **FSx for Lustre** | Lustre (POSIX), **S3 integration** | "HPC", "ML training", "sub-ms, hundreds of GB/s" |
| **FSx for NetApp ONTAP** | NFS, SMB, **iSCSI**, multi-protocol, snapshots, dedupe | "Migrate NetApp", "multi-protocol" |
| **FSx for OpenZFS** | NFS, ZFS snapshots/clones | "Migrate ZFS/Linux NAS" |

**Migration (the 7 Rs)**: Retire, Retain, **Rehost** (lift-and-shift → **MGN**), **Relocate** (VMware Cloud on AWS), **Replatform** (e.g. to RDS), **Repurchase** (SaaS), **Refactor** (cloud-native).

**Databases:** **DMS** = data migration and **CDC** (ongoing replication, minimal downtime). **SCT** / **DMS Schema Conversion** = converts schemas for **heterogeneous** moves (Oracle → Aurora PostgreSQL).

---

## Prerequisites

- `saa-vpc` and `saa-ssm-ec2-role`.
- Source data: a bucket with sample files, which stands in for an "on-prem" share.

```bash
SRC=saa-lab37-src-$SUFFIX; DST=saa-lab37-dst-$SUFFIX
aws s3 mb s3://$SRC && aws s3 mb s3://$DST --region us-west-2
mkdir -p data && for i in $(seq 1 50); do head -c 200000 /dev/urandom > data/file-$i.bin; done
mkdir -p data/tmp && echo junk > data/tmp/ignore.me
aws s3 sync data s3://$SRC/share/
```

---

## Part A — Console

### A1. DataSync: S3 → S3 (cross-region, agentless)

> For a real on-prem NFS/SMB source, you'd deploy a **DataSync agent** (VM or EC2) next to the data. AWS storage-to-storage transfers within AWS don't need an agent.

- [ ] Open **DataSync → Tasks → Create task**.
  - Source location: **Create new → Amazon S3**, bucket `saa-lab37-src-<suffix>`, folder `/share/`, IAM role **Autogenerate**
  - Destination: **Amazon S3** in us-west-2, bucket `saa-lab37-dst-<suffix>`, folder `/migrated/`, role **Autogenerate**
  - Settings: name `saa-s3-migration`, **verify only the data transferred**, **exclude pattern** `/tmp*`, transfer mode **Transfer only data that has changed**
  - Task logging: CloudWatch, log level **Basic**
  - Schedule: none (or **Daily** for recurring syncs)
- [ ] **Start** the task, then watch **Task execution** for files transferred, throughput, and the verification status.
- [ ] Confirm that `tmp/` was excluded in the destination.

### A2. S3 File Gateway on EC2

- [ ] Find the gateway AMI:

```bash
aws ssm get-parameter --name /aws/service/storagegateway/ami/FILE_S3/latest --query Parameter.Value --output text
```

- [ ] Create SG `saa-sgw-sg` in `saa-vpc`: inbound **HTTP 80 from My IP** (activation only), **NFS 2049 + 111 (TCP/UDP) + 20048** from `10.0.0.0/16`.
- [ ] Launch the AMI in **saa-public-a**, type **m5.xlarge**, public IP on, SG `saa-sgw-sg`, and **add a second EBS volume of 150 GiB gp3** (the cache disk).
- [ ] Open **Storage Gateway → Create gateway**: type **Amazon S3 File Gateway**, platform **Amazon EC2**, endpoint **Publicly accessible**, connection: **IP address** = the instance's public IP → **Activate**.
- [ ] Configure the **cache storage** on the 150 GiB disk.
- [ ] Open **File shares → Create**: bucket `saa-lab37-src-<suffix>`, protocol **NFS**, allowed clients `10.0.0.0/16`.
- [ ] From any instance in `saa-vpc` (e.g. a Session Manager client in public-a with SG allowing outbound), mount the share:

```bash
sudo mkdir -p /mnt/sgw
sudo mount -t nfs -o nolock,hard <gateway-private-ip>:/saa-lab37-src-<suffix> /mnt/sgw
ls /mnt/sgw/share | head
echo "written via NFS" | sudo tee /mnt/sgw/from-gateway.txt
```

- [ ] Check S3: `from-gateway.txt` appears as a **native S3 object**, so other S3 apps (Athena, Lambda) can use it straight away.

> **Note:** objects uploaded *directly* to S3 only show up in the share after a **cache refresh** (`RefreshCache` API, or an automated refresh setting).

### A3. FSx exploration (choose one)

- [ ] **Option 1, read-through (free):** open **FSx → Create file system** and step through each type's options to see deployment types (Single-AZ, Multi-AZ), throughput capacity, and storage type (SSD/HDD), then cancel.
- [ ] **Option 2, hands-on (~$0.20/h):** create **FSx for Lustre**, **Scratch 2**, 1.2 TiB, in **saa-app-a**, with a **data repository association** to `s3://saa-lab37-src-<suffix>/share/`. Mount it from an instance with the `lustre-client` package and `ls`. The S3 objects appear as files, lazily loaded.

---

## Part B — CLI

### B1. DataSync from the CLI

- [ ] Create locations for both buckets (each needs a role DataSync can assume):

```bash
cat > ds-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"datasync.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-datasync-s3 --assume-role-policy-document file://ds-trust.json
aws iam attach-role-policy --role-name saa-datasync-s3 --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
sleep 10
ROLE=arn:aws:iam::$ACCOUNT_ID:role/saa-datasync-s3

SRC_LOC=$(aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::$SRC --subdirectory /share \
  --s3-config BucketAccessRoleArn=$ROLE --query LocationArn --output text)
DST_LOC=$(aws datasync create-location-s3 --region us-west-2 --s3-bucket-arn arn:aws:s3:::$DST --subdirectory /cli-migrated \
  --s3-config BucketAccessRoleArn=$ROLE --query LocationArn --output text)
```

- [ ] Create a task with an exclude filter and start it:

```bash
TASK=$(aws datasync create-task --source-location-arn $SRC_LOC --destination-location-arn $DST_LOC \
  --name saa-cli-sync --excludes FilterType=SIMPLE_PATTERN,Value="/tmp*" \
  --options VerifyMode=ONLY_FILES_TRANSFERRED,TransferMode=CHANGED,OverwriteMode=ALWAYS \
  --query TaskArn --output text)

EXEC=$(aws datasync start-task-execution --task-arn $TASK --query TaskExecutionArn --output text)
sleep 60
aws datasync describe-task-execution --task-execution-arn $EXEC \
  --query '{Status:Status,Files:FilesTransferred,Bytes:BytesTransferred,Verified:Result.VerifyStatus}'
```

> AWS-to-AWS (cross-region) DataSync tasks run in the **destination or source region** depending on the locations. If the call complains about region, create the task in the region of the source location. Run the task again and see that only **changed** files transfer.

### B2. Storage Gateway inspection and cache refresh

- [ ] Look at the gateway and its share, then refresh the cache:

```bash
GW=$(aws storagegateway list-gateways --query 'Gateways[0].GatewayARN' --output text)
aws storagegateway describe-gateway-information --gateway-arn $GW \
  --query '{Name:GatewayName,Type:GatewayType,State:GatewayState,IPs:GatewayNetworkInterfaces[].Ipv4Address}'
SHARE=$(aws storagegateway list-file-shares --gateway-arn $GW --query 'FileShareInfoList[0].FileShareARN' --output text)

aws s3 cp data/file-1.bin s3://$SRC/uploaded-directly.bin
aws storagegateway refresh-cache --file-share-arn $SHARE --folder-list / --recursive
# the new object now shows in /mnt/sgw
```

### B3. Transfer Family SFTP endpoint (optional, ~$0.30/h)

- [ ] Create a service-managed SFTP server:

```bash
SFTP=$(aws transfer create-server --protocols SFTP --identity-provider-type SERVICE_MANAGED \
  --endpoint-type PUBLIC --domain S3 --query ServerId --output text)
echo "sftp endpoint: $SFTP.server.transfer.us-east-1.amazonaws.com"
```

> Add a user with `aws transfer create-user` (an S3 home directory, an IAM role, and an SSH public key), then `sftp -i key user@endpoint`. Partners upload over SFTP and the files land in S3.

### B4. DMS (read-through)

- [ ] Read through a typical **homogeneous** MySQL → Aurora MySQL migration with CDC:

```bash
# 1. Replication instance or DMS Serverless config in the target VPC
# aws dms create-replication-config --replication-config-identifier saa-mysql-to-aurora \
#   --replication-type full-load-and-cdc --compute-config MaxCapacityUnits=4,MinCapacityUnits=1,ReplicationSubnetGroupId=<sg> \
#   --source-endpoint-arn <src> --target-endpoint-arn <tgt> --table-mappings file://mappings.json
# 2. Source endpoint: on-prem MySQL (binlog enabled, binlog_format=ROW)
# 3. Target endpoint: Aurora MySQL
# 4. Start → full load, then continuous CDC until cutover
```

- [ ] Remember the key points:
  - **Heterogeneous** (Oracle → PostgreSQL) needs **schema conversion** first (DMS Schema Conversion / SCT).
  - Large databases plus limited bandwidth: **Snowball Edge** for the initial load, then **DMS CDC** for the delta.

### B5. Application Migration Service (MGN) (read-through)

- [ ] Remember the **rehost** flow: install the **replication agent** on the source servers → continuous block replication into a low-cost **staging area** → **test launch** → **cutover**. MGN is AWS's recommended lift-and-shift tool (CloudEndure Migration is legacy).

---

## Verify

- [ ] DataSync copied the files cross-region, excluding `tmp/`, and verified them.
- [ ] A file written through the NFS share appeared in S3, and a direct S3 upload showed in the share after a cache refresh.
- [ ] You can explain which FSx type and which transfer method fits each scenario in the decision tables.

## Exam tips

- **"Migrate 50 TB from on-prem NAS to EFS/S3 over the network, automated, with integrity checks"** → **DataSync**.
- **"On-prem apps keep using NFS/SMB but store data in S3 durably and cheaply"** → **S3 File Gateway**.
- **"Back up to virtual tapes in the cloud"** → **Tape Gateway**.
- **"Windows shares + AD"** → **FSx for Windows**. **HPC with S3 data** → **FSx for Lustre**.
- **"Migrate a 30 TB Oracle DB to Aurora PostgreSQL with minimal downtime"** → **SCT/Schema Conversion + DMS (full load + CDC)**.
- **"Lift-and-shift hundreds of VMs"** → **Application Migration Service (MGN)**. **Discover and plan** → **Migration Hub / Application Discovery Service / Migration Evaluator**.
- **Rule of thumb:** if the network transfer would take more than about a week, consider **offline transfer**.

## Challenge (optional)

- [ ] Work out how long 100 TB takes over a 1 Gbps link at 80% utilisation: 100×8×10¹² / (0.8×10⁹) ≈ 1,000,000 s ≈ **11.6 days**. Then decide between DataSync over DX and offline transfer.

## Cleanup

- [ ] Delete the DataSync tasks and locations:

```bash
aws datasync delete-task --task-arn $TASK
for L in $(aws datasync list-locations --query 'Locations[].LocationArn' --output text); do aws datasync delete-location --location-arn $L; done
for L in $(aws datasync list-locations --region us-west-2 --query 'Locations[].LocationArn' --output text); do aws datasync delete-location --location-arn $L --region us-west-2; done
# delete the console task saa-s3-migration first if it's still there
```

- [ ] Delete the Storage Gateway share and gateway, then terminate its EC2 instance and delete the cache volume:

```bash
aws storagegateway delete-file-share --file-share-arn $SHARE
aws storagegateway delete-gateway --gateway-arn $GW
# terminate the gateway EC2 instance; the 150 GiB volume is deleted if "Delete on termination" was on
```

- [ ] Delete the Transfer Family server (if created): `aws transfer delete-server --server-id $SFTP`
- [ ] Delete the FSx file system (if created) and the buckets:

```bash
aws s3 rb s3://$SRC --force
aws s3 rb s3://$DST --force --region us-west-2
```

- [ ] Delete the IAM roles (`saa-datasync-s3` and the auto-generated DataSync roles), SG `saa-sgw-sg`, and `rm -rf data ds-trust.json`.

---

## Knowledge check

**Q1.** A company has 200 TB of archive data on-prem, a 500 Mbps internet link, and must move it to S3 within 3 weeks. What should it use?

<details><summary>Answer</summary>

An **offline transfer device** (AWS Snowball Edge or Data Transfer Terminal, depending on availability). At 500 Mbps, the network transfer would take over 37 days.

</details>

**Q2.** On-prem applications write files to an NFS share. The company wants the files stored in S3 for analytics, while keeping low-latency local access to recent files. What should it use?

<details><summary>Answer</summary>

**Amazon S3 File Gateway** (AWS Storage Gateway).

</details>

**Q3.** A company is migrating an on-prem SQL Server database to Amazon RDS for SQL Server and must keep downtime under 15 minutes. What should it use?

<details><summary>Answer</summary>

**AWS DMS** with **full load + change data capture (CDC)**, then cut over when replication lag is near zero. SQL Server to RDS SQL Server is homogeneous, so no schema conversion is needed. Native backup/restore plus log shipping is an alternative.

</details>
