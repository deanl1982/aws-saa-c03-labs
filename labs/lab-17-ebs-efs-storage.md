# Lab 17 — Block & File Storage: EBS, Snapshots, EFS & Instance Store

| | |
|---|---|
| **Module** | 5 — Compute, Storage Volumes, Load Balancing & Scaling |
| **Exam mapping** | Domain 3 · Task 3.1 (high-performing storage) · Domain 4 · Task 4.1 (cost-optimised storage) · Domain 2 · Task 2.2 (snapshots, multi-AZ file storage) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.20 (small EBS volumes, snapshots, and EFS, prorated) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-16-ec2-fundamentals.md) · [Back to index](../README.md) · [Next lab ➡](lab-18-elastic-load-balancing.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Pick an **EBS volume type** from IOPS, throughput, and cost needs.
- Attach, format, mount, and **resize a volume live** (Elastic Volumes).
- Use **snapshots** to back up, copy, and move data across AZs and regions, and automate them with **Data Lifecycle Manager**.
- Mount one **EFS** file system on instances in **different AZs** at the same time.
- Know when **instance store** is the right answer, despite being ephemeral.

## EBS volume types

| Type | Kind | Max IOPS | Max throughput | Notes / triggers |
|---|---|---|---|---|
| **gp3** | SSD | 16,000 (3,000 baseline) | 1,000 MiB/s | **Default.** IOPS and throughput set **independently of size**. ~20% cheaper than gp2. |
| **gp2** | SSD | 16,000 (3 IOPS/GiB, burst 3,000) | 250 MiB/s | Legacy. IOPS tied to size. |
| **io2 Block Express** | SSD | **256,000** | 4,000 MiB/s | Critical DBs, sub-ms latency, 99.999% durability, **Multi-Attach** |
| **io1** | SSD | 64,000 | 1,000 MiB/s | Legacy provisioned IOPS, Multi-Attach |
| **st1** | HDD | 500 | 500 MiB/s | Big sequential throughput: logs, ETL, Kafka. **Can't be a boot volume.** |
| **sc1** | HDD | 250 | 250 MiB/s | Cold, infrequently accessed, lowest cost. **Can't be a boot volume.** |

**EBS facts:** locked to **one AZ**. Moving it to another AZ means **snapshot → restore**. Snapshots are incremental and stored in S3 (regionally). **Encryption** happens at the volume level with KMS, and snapshots of encrypted volumes are encrypted.

## File storage comparison

| | EBS | EFS | FSx for Windows | FSx for Lustre | Instance store |
|---|---|---|---|---|---|
| Protocol | Block | **NFSv4** (Linux) | **SMB**, AD | Lustre (HPC) | Block (local NVMe) |
| Shared | Single instance (Multi-Attach io1/io2 in one AZ) | **Thousands of instances, multi-AZ** | Yes | Yes | No |
| Persistence | Persistent | Persistent | Persistent | Scratch or persistent | **Lost on stop/terminate** |

---

## Prerequisites

- `saa-vpc`, and the `saa-ssm-ec2-role` profile. Two instances, one in **saa-public-a** and one in **saa-public-b** (launch them from `saa-web-lt` in Lab 16, or by hand).

```bash
source ~/saa-vpc.env
I_A=$(aws ec2 run-instances --launch-template LaunchTemplateId=$LT_ID --subnet-id $PUB_A --query 'Instances[0].InstanceId' --output text)
I_B=$(aws ec2 run-instances --launch-template LaunchTemplateId=$LT_ID --subnet-id $PUB_B --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids $I_A $I_B
```

---

## Part A — Console

### A1. Create, attach, and mount an EBS volume

- [ ] Open **EC2 → Volumes → Create volume**: **gp3**, 10 GiB, 3000 IOPS, 125 MiB/s, AZ **us-east-1a** (must match instance A), **Encrypted** (default key), tag Name `saa-data-vol`.
- [ ] Choose **Actions → Attach volume** → instance A, device `/dev/sdf`.
- [ ] Start a Session Manager session on instance A and run:

```bash
lsblk                                   # new NVMe device, e.g. nvme1n1
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data && sudo mount /dev/nvme1n1 /data
echo "important data" | sudo tee /data/file.txt
df -h /data
```

### A2. Resize live (Elastic Volumes)

- [ ] Select the volume and choose **Actions → Modify volume**: size **20 GiB**, IOPS **4000**.
- [ ] Watch the state go *modifying → optimizing → completed*. The instance keeps running throughout.
- [ ] In the session, grow the file system:

```bash
lsblk                       # device shows 20G
sudo xfs_growfs -d /data
df -h /data                 # file system now 20G
```

### A3. Snapshot and restore into another AZ

- [ ] Choose **Actions → Create snapshot**, description `saa-data-snap`.
- [ ] When it completes, choose **Snapshots → Actions → Create volume from snapshot** in **us-east-1b**.
- [ ] Attach it to instance B and mount it (**don't** run `mkfs`), then confirm `/data/file.txt` exists.

> This is the **only** way to move an EBS volume to another AZ.

### A4. Automate snapshots with Data Lifecycle Manager

- [ ] Open **EC2 → Lifecycle Manager → Create lifecycle policy → EBS snapshot policy**.
  - Target: volumes tagged `Backup=daily`
  - Schedule: every 24 h at 03:00 UTC, **retain 7**
  - Cross-region copy: us-west-2, retain 3 *(optional)*
- [ ] Tag `saa-data-vol` with `Backup=daily`.

### A5. Recycle Bin for snapshots

- [ ] Open **Recycle Bin → Create retention rule**: resource type **EBS snapshots**, all resources, retain **1 day**.
- [ ] Delete `saa-data-snap`, then recover it from **Recycle Bin**.

### A6. EFS shared across AZs

- [ ] Create SG `saa-efs-sg` in `saa-vpc` with inbound **NFS 2049** from `saa-web-sg`.
- [ ] Open **EFS → Create file system → Customize**.
  - Name `saa-efs`, **Regional** (multi-AZ), **Elastic throughput**, lifecycle: transition to **IA after 30 days**, to **Archive after 90 days**
  - Network: `saa-vpc`, mount targets in **us-east-1a** and **us-east-1b**, SG `saa-efs-sg`
- [ ] Choose **Attach** and copy the **EFS mount helper** command.
- [ ] On **both** instances:

```bash
sudo dnf -y install amazon-efs-utils
sudo mkdir /efs
sudo mount -t efs -o tls <fs-id>:/ /efs
```

- [ ] On instance A: `echo "from A" | sudo tee /efs/shared.txt`. On instance B: `cat /efs/shared.txt`. **The same file is visible from both AZs.**

---

## Part B — CLI

### B1. Create a gp3 volume with custom IOPS and throughput

- [ ] Create it and attach it:

```bash
VOL=$(aws ec2 create-volume --availability-zone us-east-1a --volume-type gp3 --size 50 \
  --iops 6000 --throughput 250 --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=saa-cli-gp3}]' \
  --query VolumeId --output text)
aws ec2 wait volume-available --volume-ids $VOL
aws ec2 attach-volume --volume-id $VOL --instance-id $I_A --device /dev/sdg
```

### B2. Migrate gp2 to gp3 live

- [ ] Create a gp2 volume and inspect it:

```bash
GP2=$(aws ec2 create-volume --availability-zone us-east-1a --volume-type gp2 --size 100 --query VolumeId --output text)
aws ec2 wait volume-available --volume-ids $GP2
aws ec2 describe-volumes --volume-ids $GP2 --query 'Volumes[0].{Type:VolumeType,Iops:Iops}'   # gp2 100GiB = 300 IOPS
```

- [ ] Convert it to gp3:

```bash
aws ec2 modify-volume --volume-id $GP2 --volume-type gp3 --iops 3000 --throughput 125
aws ec2 describe-volumes-modifications --volume-ids $GP2 \
  --query 'VolumesModifications[0].{State:ModificationState,From:OriginalVolumeType,To:TargetVolumeType,Progress:Progress}'
```

> Same price class, **10× the baseline IOPS** of this gp2 volume, and cheaper per GB. That's a classic cost-optimisation question.

### B3. Snapshots and cross-region copy

- [ ] Snapshot the volume, then copy it to us-west-2 with encryption:

```bash
SNAP=$(aws ec2 create-snapshot --volume-id $VOL --description "saa cli snap" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=saa-cli-snap}]' --query SnapshotId --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP

SNAP_WEST=$(aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id $SNAP \
  --region us-west-2 --encrypted --description "DR copy" --query SnapshotId --output text)
```

- [ ] Look at the **Snapshot Archive** tier, which cuts snapshot storage cost by ~75% for snapshots kept 90+ days:

```bash
aws ec2 modify-snapshot-tier --snapshot-id $SNAP --storage-tier archive --dry-run 2>&1 | head -2
```

### B4. Turn on EBS encryption by default (account-level, per region)

- [ ] Enable it:

```bash
aws ec2 enable-ebs-encryption-by-default
aws ec2 get-ebs-encryption-by-default
aws ec2 get-ebs-default-kms-key-id
```

> To encrypt an **existing unencrypted** volume: snapshot → **copy the snapshot with encryption** → create a volume from it → swap it in.

### B5. EFS from the CLI

- [ ] Create the file system:

```bash
EFS_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-efs-sg --query 'SecurityGroups[0].GroupId' --output text)

FS=$(aws efs create-file-system --performance-mode generalPurpose --throughput-mode elastic \
  --encrypted --tags Key=Name,Value=saa-efs-cli --query FileSystemId --output text)
sleep 10
```

- [ ] Add a mount target in each AZ and a lifecycle policy:

```bash
aws efs create-mount-target --file-system-id $FS --subnet-id $PUB_A --security-groups $EFS_SG
aws efs create-mount-target --file-system-id $FS --subnet-id $PUB_B --security-groups $EFS_SG

aws efs put-lifecycle-configuration --file-system-id $FS --lifecycle-policies \
  TransitionToIA=AFTER_30_DAYS TransitionToPrimaryStorageClass=AFTER_1_ACCESS
```

- [ ] Create an **access point**, which enforces a POSIX user and root directory per application:

```bash
aws efs create-access-point --file-system-id $FS \
  --posix-user Uid=1001,Gid=1001 \
  --root-directory 'Path=/app1,CreationInfo={OwnerUid=1001,OwnerGid=1001,Permissions=750}'
aws efs describe-mount-targets --file-system-id $FS --query 'MountTargets[].{AZ:AvailabilityZoneName,IP:IpAddress,State:LifeCycleState}' --output table
```

### B6. Instance store (read-through, or try it, ~$0.11/h)

- [ ] Optionally launch an instance with instance store:

```bash
# Launch an instance type with local NVMe (the "d" suffix)
# aws ec2 run-instances --image-id <ami> --instance-type m6id.large --subnet-id $PUB_A ...
# lsblk → shows the instance store NVMe device, which must be formatted at every boot.
```

- [ ] Remember the rule: **instance store data survives reboot but is LOST on stop, hibernate, or terminate.** Use it for caches, buffers, scratch space, or replicated data (Cassandra, Kafka).

---

## Verify

- [ ] You resized a mounted volume online.
- [ ] You restored a snapshot in a different AZ, and the data was intact.
- [ ] Both instances in two AZs read and write the same EFS file.

## Exam tips

- **"Shared file system for Linux instances across AZs"** → **EFS**. **"Windows SMB with AD"** → **FSx for Windows**. **"HPC with S3 integration"** → **FSx for Lustre**. **"NetApp ONTAP features / multi-protocol"** → **FSx for NetApp ONTAP**.
- **"Highest IOPS for a single-instance database"** → **io2 Block Express**. Even more (millions) of IOPS that can be lost → **instance store**.
- **"Throughput-heavy sequential workload at low cost"** → **st1**.
- **"Cheapest for infrequently accessed EBS data"** → **sc1**.
- **gp2 → gp3** = lower cost and independent performance tuning, with no downtime.
- RAID 0 across EBS volumes increases performance. RAID 1 is rarely the answer, because EBS already replicates within its AZ.
- **EFS One Zone** is cheaper, at the cost of resilience. **EFS-IA / Archive** lifecycle saves up to ~90%.

## Challenge (optional)

- [ ] Turn on **Fast Snapshot Restore** for a snapshot in one AZ, and explain when the extra cost is worth it (volumes that need full performance immediately after restore).

## Cleanup

- [ ] Delete the EFS mount targets and file systems:

```bash
for M in $(aws efs describe-mount-targets --file-system-id $FS --query 'MountTargets[].MountTargetId' --output text); do aws efs delete-mount-target --mount-target-id $M; done
sleep 30 && aws efs delete-file-system --file-system-id $FS
# repeat for the console file system saa-efs
```

- [ ] Terminate the instances, then delete the leftover volumes once they're detached:

```bash
aws ec2 terminate-instances --instance-ids $I_A $I_B
aws ec2 wait instance-terminated --instance-ids $I_A $I_B
for V in $(aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].VolumeId' --output text); do aws ec2 delete-volume --volume-id $V; done
```

- [ ] Delete the snapshots, in both regions:

```bash
aws ec2 delete-snapshot --snapshot-id $SNAP
aws ec2 delete-snapshot --snapshot-id $SNAP_WEST --region us-west-2
# plus the console snapshot(s); empty Recycle Bin or delete the retention rule
```

- [ ] Delete the DLM policy and the `saa-efs-sg` security group.
- [ ] *(Optional)* Keep EBS encryption-by-default on. It's a good security baseline.

---

## Knowledge check

**Q1.** A database on EC2 needs 80,000 IOPS with sub-millisecond latency on persistent storage. Which volume type?

<details><summary>Answer</summary>

**io2 Block Express** (gp3 tops out at 16,000 IOPS).

</details>

**Q2.** A company must move an EBS volume from us-east-1a to us-east-1c. How?

<details><summary>Answer</summary>

Create a **snapshot** and **create a new volume from it in us-east-1c**.

</details>

**Q3.** Hundreds of Linux instances in an Auto Scaling group across three AZs need concurrent read/write access to shared content. Which storage?

<details><summary>Answer</summary>

**Amazon EFS** (Regional), with mount targets in each AZ.

</details>
