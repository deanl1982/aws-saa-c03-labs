# Lab 12 — Private Subnets, NAT Gateway & Session Manager

| | |
|---|---|
| **Module** | 4 — Networking with Amazon VPC |
| **Exam mapping** | Domain 1 · Task 1.2 (secure workloads, private access) · Domain 2 · Task 2.2 (HA NAT) · Domain 4 · Task 4.4 (NAT cost) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.10–0.20/hour while NAT gateways run ($0.045/h each + $0.045/GB) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-11-custom-vpc.md) · [Back to index](../README.md) · [Next lab ➡](lab-13-sg-vs-nacl.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Give private-subnet instances **outbound-only** internet through a **NAT Gateway**.
- Design for **NAT high availability**, with one NAT gateway per AZ and per-AZ route tables.
- Compare **NAT Gateway** with **NAT instances** and the **egress-only IGW** (IPv6).
- Administer private instances with **SSM Session Manager**: no bastion, no SSH keys, no inbound ports.

## NAT Gateway vs NAT instance

| | NAT Gateway | NAT instance |
|---|---|---|
| Managed | Yes | You patch and scale it |
| Availability | HA **within one AZ**. Deploy one per AZ. | Single EC2, so you script failover |
| Bandwidth | Scales to 100 Gbps | Limited by instance type |
| Security groups | Can't attach | Can attach |
| Source/dest check | N/A | Must be **disabled** |
| Port forwarding / bastion use | No | Yes |
| Cost | Hourly + per-GB | EC2 price only. Cheaper at small scale. |

---

## Prerequisites

- Lab 11 finished (`saa-vpc` exists and `~/saa-vpc.env` is saved).

---

## Part A — Console

### A1. Create an IAM role for Session Manager (reused in later labs)

- [ ] Open **IAM → Roles → Create role → AWS service → EC2**, attach **AmazonSSMManagedInstanceCore**, and name it `saa-ssm-ec2-role`.

### A2. Create a NAT gateway

- [ ] Open **VPC → NAT gateways → Create NAT gateway**.
  - Name `saa-nat-a`, Subnet **saa-public-a**, Connectivity **Public**, **Allocate Elastic IP**
- [ ] Wait until the state is **Available** (about 2 minutes).
- [ ] Open **Route tables → saa-private-rt-a → Routes → Edit**, add `0.0.0.0/0` → **NAT Gateway** `saa-nat-a`, and save.

### A3. Launch a private instance

- [ ] Open **EC2 → Launch instance**.
  - Name `saa-private-a`, AL2023, `t3.micro`, **no key pair**
  - VPC `saa-vpc`, subnet **saa-app-a**, auto-assign public IP **Disable**
  - New security group `saa-private-sg` with **no inbound rules**
  - Advanced → IAM instance profile `saa-ssm-ec2-role`
- [ ] After about 3 minutes, choose **Connect → Session Manager → Connect**.
- [ ] In the session, run:

```bash
curl -s https://checkip.amazonaws.com    # shows the NAT gateway's Elastic IP
sudo dnf -y update --refresh | tail -2   # outbound works
```

- [ ] Compare the IP from `checkip` with the NAT gateway's Elastic IP in the console. They match.

### A4. Show that inbound from the internet is impossible

- [ ] Note that the instance has **no public IP** and the SG has **no inbound rules**, yet you have a shell. Session Manager connects **outbound** from the SSM agent to the SSM service.

### A5. Session Manager logging

- [ ] Open **Systems Manager → Session Manager → Preferences → Edit** and turn on **CloudWatch logging** (create a log group `saa-ssm-sessions`). Sessions are now audited, and you can enable **KMS encryption** too.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
```

### B1. Create the instance profile (if you didn't do Part A)

- [ ] Create the role and profile:

```bash
cat > ec2-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-ssm-ec2-role --assume-role-policy-document file://ec2-trust.json 2>/dev/null
aws iam attach-role-policy --role-name saa-ssm-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam create-instance-profile --instance-profile-name saa-ssm-ec2-role 2>/dev/null
aws iam add-role-to-instance-profile --instance-profile-name saa-ssm-ec2-role --role-name saa-ssm-ec2-role 2>/dev/null
echo "export SSM_PROFILE=saa-ssm-ec2-role" >> ~/saa-vpc.env
```

### B2. Highly available NAT: one per AZ

- [ ] Create the AZ-b NAT gateway (AZ-a exists from Part A; if you skipped Part A, repeat this block for AZ a):

```bash
EIP_B=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
NAT_B=$(aws ec2 create-nat-gateway --subnet-id $PUB_B --allocation-id $EIP_B \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=saa-nat-b}]' \
  --query NatGateway.NatGatewayId --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_B
aws ec2 create-route --route-table-id $APP_RT_B --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_B
```

- [ ] Look up and save the AZ-a NAT gateway:

```bash
NAT_A=$(aws ec2 describe-nat-gateways --filter Name=tag:Name,Values=saa-nat-a Name=state,Values=available \
  --query 'NatGateways[0].NatGatewayId' --output text)
echo "export NAT_A=$NAT_A NAT_B=$NAT_B" >> ~/saa-vpc.env

aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=$VPC_ID \
  --query 'NatGateways[].{Id:NatGatewayId,Subnet:SubnetId,IP:NatGatewayAddresses[0].PublicIp,State:State}' --output table
```

### B3. Launch a private instance in AZ b

- [ ] Create a security group and launch the instance:

```bash
SG_PRIV=$(aws ec2 create-security-group --group-name saa-private-cli-sg \
  --description "no inbound" --vpc-id $VPC_ID --query GroupId --output text)

AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)

PRIV_B=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $APP_B --security-group-ids $SG_PRIV \
  --iam-instance-profile Name=saa-ssm-ec2-role \
  --metadata-options HttpTokens=required \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-private-b}]' \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids $PRIV_B
```

### B4. Connect with Session Manager from your laptop

- [ ] Install the **Session Manager plugin**:

```bash
# macOS (Apple silicon or Intel)
brew install --cask session-manager-plugin
session-manager-plugin --version
```

- [ ] Confirm the instance is registered with SSM, then start a session:

```bash
aws ssm describe-instance-information --query 'InstanceInformationList[].{Id:InstanceId,Ping:PingStatus}' --output table

aws ssm start-session --target $PRIV_B
# inside: curl -s https://checkip.amazonaws.com ; exit
```

### B5. Port forwarding through Session Manager (the bastion replacement)

- [ ] Forward a local port to the private instance's port 22, or to a database behind it:

```bash
aws ssm start-session --target $PRIV_B \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["22"],"localPortNumber":["2222"]}'
```

> Later labs use `AWS-StartPortForwardingSessionToRemoteHost` to reach RDS endpoints through a private instance, with no bastion and no public DB.

### B6. Show the per-AZ egress paths

- [ ] Compare the egress IP from each AZ:

```bash
PRIV_A=$(aws ec2 describe-instances --filters Name=tag:Name,Values=saa-private-a Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

for I in $PRIV_A $PRIV_B; do
  echo "== $I"
  CMD=$(aws ssm send-command --instance-ids $I --document-name AWS-RunShellScript \
    --parameters 'commands=["curl -s https://checkip.amazonaws.com"]' --query Command.CommandId --output text)
  sleep 4
  aws ssm get-command-invocation --command-id $CMD --instance-id $I --query StandardOutputContent --output text
done
```

> Each instance leaves through its own AZ's NAT gateway IP. If AZ a fails, AZ b's egress isn't affected. That's why you deploy a NAT gateway **per AZ**.

---

## Verify

- [ ] Private instances have no public IP but can reach the internet.
- [ ] You reached both instances through Session Manager with no inbound SG rules.
- [ ] Each private route table points to the NAT gateway in **its own AZ**.

## Exam tips

- **"Private instances need to download patches"** → NAT Gateway in a public subnet, and private route `0.0.0.0/0 → NAT`.
- **"Highly available NAT"** → one NAT gateway **per AZ**, each AZ's private route table using its local NAT.
- **"Reduce NAT cost for S3/DynamoDB traffic"** → **gateway VPC endpoint** (free, Lab 14).
- **"Securely administer instances without opening port 22 or managing bastions"** → **SSM Session Manager**.
- **IPv6 outbound only** → **egress-only internet gateway** (NAT isn't used for IPv6).
- NAT gateways also come in a **private** connectivity type, for routing to other VPCs or on-prem with overlapping CIDRs.

## Challenge (optional)

- [ ] Look at the NAT gateway's CloudWatch metrics (`BytesOutToDestination`, `ErrorPortAllocation`) and explain what `ErrorPortAllocation` would tell you.

## Cleanup

> **NAT gateways cost money every hour.** If you're going straight to **Lab 13**, keep everything. Otherwise:

- [ ] Terminate the instances:

```bash
aws ec2 terminate-instances --instance-ids $PRIV_B $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-private-a Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] Delete the NAT routes and gateways, then release the Elastic IPs:

```bash
aws ec2 delete-route --route-table-id $APP_RT_A --destination-cidr-block 0.0.0.0/0
aws ec2 delete-route --route-table-id $APP_RT_B --destination-cidr-block 0.0.0.0/0
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_A
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_B
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_A $NAT_B

for A in $(aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].AllocationId' --output text); do
  aws ec2 release-address --allocation-id $A
done
```

- [ ] Delete the security groups `saa-private-sg` and `saa-private-cli-sg` once the instances have terminated.
- [ ] **Keep** the `saa-ssm-ec2-role` instance profile and `saa-vpc`.

---

## Knowledge check

**Q1.** A company has one NAT gateway in AZ-a serving private subnets in AZ-a and AZ-b. During an AZ-a outage, instances in AZ-b lose internet access. How do you fix this?

<details><summary>Answer</summary>

Deploy a **NAT gateway in each AZ** and give each AZ's private subnets a route table that points to their own AZ's NAT gateway.

</details>

**Q2.** Security wants to remove all bastion hosts and SSH keys but still have audited shell access to private EC2 instances. What should you use?

<details><summary>Answer</summary>

**AWS Systems Manager Session Manager**, with session logging to CloudWatch Logs or S3 and IAM-controlled access.

</details>

**Q3.** Private instances download large datasets from S3 through a NAT gateway, and the NAT data-processing charges are high. What's the cheapest fix?

<details><summary>Answer</summary>

Add an **S3 gateway VPC endpoint** to the private route tables. Gateway endpoints are free, and S3 traffic then skips the NAT.

</details>
