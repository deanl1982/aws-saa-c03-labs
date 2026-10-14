# Lab 16 — EC2 Fundamentals: Launch, User Data & AMIs

| | |
|---|---|
| **Module** | 5 — Compute, Storage Volumes, Load Balancing & Scaling |
| **Exam mapping** | Domain 3 · Task 3.2 (high-performing compute) · Domain 2 · Task 2.2 (recovery with AMIs) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.10 (t3.micro instances, AMI snapshot storage) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-15-peering-transit-gateway.md) · [Back to index](../README.md) · [Next lab ➡](lab-17-ebs-efs-storage.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Decode **instance type names** and pick a family for a given workload.
- Bootstrap instances with **user data**, and read the **instance metadata service (IMDSv2)**.
- Manage the instance lifecycle: stop/start vs **reboot** vs **hibernate** vs terminate, plus **Elastic IPs**.
- Create a **custom AMI** ("golden image"), copy it to another region, and launch from it.
- Use **launch templates** for repeatable launches.

## Instance families

| Family | Prefix examples | Optimised for | Exam triggers |
|---|---|---|---|
| General purpose | **t3/t4g** (burstable), **m7i/m7g** | Balanced | Web servers, small DBs, dev/test |
| Compute optimised | **c7i/c7g** | High CPU | Batch, media transcoding, HPC, gaming servers |
| Memory optimised | **r7i**, **x2**, **z1d** | Large RAM | In-memory DBs, real-time big data |
| Storage optimised | **i4i**, **d3**, **im4gn** | High local I/O (NVMe instance store) | NoSQL, data warehousing, OLTP needing very high IOPS |
| Accelerated | **p5**, **g6**, **inf2**, **trn1** | GPU / ML chips | ML training/inference, graphics |

`m7g.2xlarge` = family **m**, generation **7**, attribute **g** (Graviton/ARM), size **2xlarge**.
Other attributes: `i` Intel, `a` AMD, `d` local NVMe instance store, `n` enhanced networking, `e` extra memory or storage.

---

## Part A — Console

### A1. Launch a web server with user data

- [ ] Open **EC2 → Launch instance**.
  - Name `saa-web-1`, **Amazon Linux 2023**, `t3.micro`
  - Key pair: create one, `saa-key` (RSA, `.pem`), and keep the file safe (`chmod 400 saa-key.pem`)
  - Network: `saa-vpc` / **saa-public-a**, auto-assign public IP **on**, SG `saa-web-sg` (from Lab 13; add **SSH 22 from My IP**)
  - Advanced → IAM profile `saa-ssm-ec2-role`, **Metadata version: V2 only (token required)**
  - **User data:**

```bash
#!/bin/bash
dnf -y install httpd
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>Hello from $ID in $AZ</h1>" > /var/www/html/index.html
systemctl enable --now httpd
```

- [ ] Browse to `http://<public-ip>` and check the page shows the instance ID and AZ.
- [ ] SSH in with `ssh -i saa-key.pem ec2-user@<public-ip>`, then:

```bash
sudo cat /var/log/cloud-init-output.log | tail -20   # user data output
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
curl -s http://169.254.169.254/latest/meta-data/instance-id   # fails: IMDSv1 disabled (401)
```

### A2. Stop/start vs reboot, and Elastic IPs

- [ ] Note the public IP. **Stop** the instance, then **start** it. **The public IP changes.**
- [ ] Open **Elastic IPs → Allocate → Associate** with `saa-web-1`, then stop and start again. **The EIP stays the same.**
- [ ] **Reboot** the instance and note that the public IP and host stay the same.

> **Hibernate** saves RAM to an encrypted EBS root volume, so long-running in-memory apps resume fast. You must turn it on at launch, and RAM is limited to ≤150 GB.

### A3. Create a custom AMI

- [ ] Select `saa-web-1` and choose **Actions → Image and templates → Create image**, name it `saa-web-ami-v1`.
- [ ] Open **AMIs** and wait for **Available**. Look at the backing **EBS snapshot** under **Snapshots**.
- [ ] Launch a new instance **from the AMI** in **saa-public-b** with *no user data*. Browse to it. httpd is already installed, but the page shows **the original instance ID**, because the HTML was baked into the image.

> **Lesson:** bake slow-changing things (packages) into the AMI, and set per-instance values at boot with user data. That's the **golden AMI + user data** pattern, which gives fast scale-out.

### A4. Copy the AMI to another region

- [ ] Select the AMI, choose **Actions → Copy AMI**, destination **us-west-2**, and optionally **encrypt** with a KMS key.
- [ ] Switch to us-west-2 and watch it become available. That's a quick DR building block.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
```

### B1. Explore instance types

- [ ] Compare a few types:

```bash
aws ec2 describe-instance-types --instance-types t3.micro m7g.large c7i.xlarge r7i.large i4i.large \
  --query 'InstanceTypes[].{Type:InstanceType,vCPU:VCpuInfo.DefaultVCpus,MemMiB:MemoryInfo.SizeInMiB,Arch:ProcessorInfo.SupportedArchitectures[0],Net:NetworkInfo.NetworkPerformance,InstStore:InstanceStorageSupported}' \
  --output table
```

- [ ] Find which AZs offer a type:

```bash
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=c7i.xlarge --query 'InstanceTypeOfferings[].Location'
```

### B2. Create a launch template

- [ ] Write the user data and create the template:

```bash
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf -y install httpd
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>LT instance in $AZ</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF

LT_ID=$(aws ec2 create-launch-template --launch-template-name saa-web-lt \
  --version-description v1 \
  --launch-template-data "{
    \"ImageId\":\"$AMI\",
    \"InstanceType\":\"t3.micro\",
    \"IamInstanceProfile\":{\"Name\":\"saa-ssm-ec2-role\"},
    \"SecurityGroupIds\":[\"$WEB_SG\"],
    \"MetadataOptions\":{\"HttpTokens\":\"required\",\"HttpEndpoint\":\"enabled\"},
    \"UserData\":\"$(base64 < userdata.sh | tr -d '\n')\",
    \"TagSpecifications\":[{\"ResourceType\":\"instance\",\"Tags\":[{\"Key\":\"Name\",\"Value\":\"saa-lt-web\"}]}]
  }" --query LaunchTemplate.LaunchTemplateId --output text)
echo "export LT_ID=$LT_ID" >> ~/saa-vpc.env
```

### B3. Launch from the template

- [ ] Launch an instance and fetch its page:

```bash
IID=$(aws ec2 run-instances --launch-template LaunchTemplateId=$LT_ID,Version='$Latest' \
  --subnet-id $PUB_A --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids $IID
PUBIP=$(aws ec2 describe-instances --instance-ids $IID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
sleep 20 && curl -s http://$PUBIP
```

### B4. Create a new template version (e.g. bigger instance)

- [ ] Add version 2 and make it the default:

```bash
aws ec2 create-launch-template-version --launch-template-id $LT_ID --source-version 1 \
  --version-description v2-t3small --launch-template-data '{"InstanceType":"t3.small"}'
aws ec2 modify-launch-template --launch-template-id $LT_ID --default-version 2
aws ec2 describe-launch-template-versions --launch-template-id $LT_ID \
  --query 'LaunchTemplateVersions[].{V:VersionNumber,Desc:VersionDescription,Type:LaunchTemplateData.InstanceType,Default:DefaultVersion}' --output table
```

### B5. Create an AMI and copy it cross-region

- [ ] Create the AMI, then copy it:

```bash
AMI_V2=$(aws ec2 create-image --instance-id $IID --name saa-web-ami-cli-$(date +%Y%m%d%H%M) \
  --description "golden web AMI" --no-reboot --query ImageId --output text)
aws ec2 wait image-available --image-ids $AMI_V2

AMI_WEST=$(aws ec2 copy-image --source-region us-east-1 --source-image-id $AMI_V2 \
  --name saa-web-ami-west --region us-west-2 --query ImageId --output text)
echo "us-west-2 copy: $AMI_WEST"
```

> `--no-reboot` is faster but risks a file-system-inconsistent image. Without it, EC2 reboots the instance to get a clean snapshot.

### B6. Modify a running instance's metadata options and type

- [ ] Change the IMDS hop limit, then resize the instance (resizing needs a stop):

```bash
aws ec2 modify-instance-metadata-options --instance-id $IID --http-tokens required --http-put-response-hop-limit 2

aws ec2 stop-instances --instance-ids $IID && aws ec2 wait instance-stopped --instance-ids $IID
aws ec2 modify-instance-attribute --instance-id $IID --instance-type '{"Value":"t3.small"}'
aws ec2 start-instances --instance-ids $IID
```

> A hop limit of **2** is needed when containers on the instance call IMDS, because the extra network hop counts.

---

## Verify

- [ ] User data installed and configured httpd automatically.
- [ ] IMDSv1 requests fail and IMDSv2 token requests succeed.
- [ ] Your custom AMI launches pre-configured instances, and a copy exists in us-west-2.

## Exam tips

- **IMDSv2** (session tokens) protects against SSRF credential theft. Enforce it with `HttpTokens=required`, or the account-level default.
- **User data** runs **once at first boot** as root, by default. Logs go to `/var/log/cloud-init-output.log`.
- **Faster boot / scale-out** → golden AMI (pre-baked), possibly with **EC2 Image Builder** to automate it.
- **AMIs are regional.** Copy them for multi-region DR. To share with another account, share the AMI **and** the KMS key if it's encrypted.
- **Elastic IP** keeps a static public IPv4. You pay for **all public IPv4 addresses**, including in-use EIPs ($0.005/h).
- **Burstable t3/t4g** accrue CPU credits. **Unlimited mode** avoids throttling but can add cost.

## Challenge (optional)

- [ ] Build a simple **EC2 Image Builder** pipeline that produces a patched AL2023 AMI with httpd preinstalled, on a weekly schedule.

## Cleanup

- [ ] Terminate the instances:

```bash
aws ec2 terminate-instances --instance-ids $IID $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-web-1 Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
# also terminate the console instance launched from the AMI
```

- [ ] Release the Elastic IP (after disassociating it):

```bash
for A in $(aws ec2 describe-addresses --query 'Addresses[].AllocationId' --output text); do aws ec2 release-address --allocation-id $A; done
```

- [ ] Deregister the AMIs and delete their snapshots, in both regions:

```bash
for R in us-east-1 us-west-2; do
  for IMG in $(aws ec2 describe-images --owners self --region $R --filters "Name=name,Values=saa-web-ami*" --query 'Images[].ImageId' --output text); do
    SNAPS=$(aws ec2 describe-images --image-ids $IMG --region $R --query 'Images[0].BlockDeviceMappings[].Ebs.SnapshotId' --output text)
    aws ec2 deregister-image --image-id $IMG --region $R
    for S in $SNAPS; do aws ec2 delete-snapshot --snapshot-id $S --region $R; done
  done
done
```

- [ ] **Keep** the launch template `saa-web-lt` (free) for Lab 19, and keep the key pair `saa-key`.
- [ ] Remove the SSH rule from `saa-web-sg`.

---

## Knowledge check

**Q1.** An application must scale out quickly during traffic spikes, but new instances take 15 minutes to install software through user data. How can you shorten launch time?

<details><summary>Answer</summary>

Pre-install the software in a **custom (golden) AMI** and keep only the small per-instance configuration in user data. EC2 Image Builder can automate the AMI. Warm pools are another option.

</details>

**Q2.** A security audit flags that EC2 instances allow credential theft through SSRF against the metadata endpoint. What should you do?

<details><summary>Answer</summary>

Require **IMDSv2** (`HttpTokens=required`) on every instance, and set it as the account default or enforce it with an SCP or Config rule.

</details>

**Q3.** Which instance family suits an in-memory analytics database that needs 512 GiB of RAM?

<details><summary>Answer</summary>

**Memory optimised** (R or X family).

</details>
