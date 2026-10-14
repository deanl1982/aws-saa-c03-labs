# Lab 11 — Build a Custom VPC from Scratch

| | |
|---|---|
| **Module** | 4 — Networking with Amazon VPC |
| **Exam mapping** | Domain 1 · Task 1.2 (network segmentation) · Domain 3 · Task 3.4 (network architecture) · Domain 2 · Task 2.2 (multi-AZ) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 60 minutes |
| **Estimated cost** | $0 (VPCs, subnets, route tables, and IGWs are free) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-10-s3-replication-advanced.md) · [Back to index](../README.md) · [Next lab ➡](lab-12-nat-private-access.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Plan **CIDR ranges** for a VPC and its subnets, including the 5 IPs AWS reserves in each subnet.
- Build **public**, **private-app**, and **private-data** subnet tiers across **two Availability Zones**.
- Explain what makes a subnet *public*: a route to an **Internet Gateway**, plus public IPs.
- Configure **route tables**, subnet associations, and **DNS** attributes.
- Save the VPC IDs so **Labs 12–39 can reuse this VPC**.

## Target architecture

```
                         VPC saa-vpc 10.0.0.0/16
 ┌──────────────────────────────────┬──────────────────────────────────┐
 │           AZ a (us-east-1a)      │           AZ b (us-east-1b)      │
 │  ┌────────────────────────────┐  │  ┌────────────────────────────┐  │
 │  │ public-a     10.0.1.0/24   │  │  │ public-b     10.0.2.0/24   │  │──► IGW ──► Internet
 │  └────────────────────────────┘  │  └────────────────────────────┘  │
 │  ┌────────────────────────────┐  │  ┌────────────────────────────┐  │
 │  │ app-a        10.0.11.0/24  │  │  │ app-b        10.0.12.0/24  │  │   (private)
 │  └────────────────────────────┘  │  └────────────────────────────┘  │
 │  ┌────────────────────────────┐  │  ┌────────────────────────────┐  │
 │  │ data-a       10.0.21.0/24  │  │  │ data-b       10.0.22.0/24  │  │   (private, isolated)
 │  └────────────────────────────┘  │  └────────────────────────────┘  │
 └──────────────────────────────────┴──────────────────────────────────┘
```

## Key facts

- VPC CIDR: **/16 (65,536 IPs) up to /28 (16 IPs)**. You can add secondary CIDRs later.
- Every subnet loses **5 IPs**: network address, `.1` VPC router, `.2` DNS, `.3` reserved, and broadcast. So a /24 has **251** usable addresses.
- A subnet lives in **exactly one AZ**.
- Each subnet associates with **one** route table, and the **main** route table is used unless you set another.
- Route tables pick the **most specific** route (longest prefix match).
- **IGW:** one per VPC, horizontally scaled, highly available, no bandwidth limit.

---

## Part A — Console (manual build: `saa-console-vpc`)

> The console has a **"VPC and more"** wizard that builds all of this in one click. You'll do it by hand once so you understand every piece, and the wizard is a challenge task at the end.

### A1. Create the VPC

- [ ] Open **VPC → Your VPCs → Create VPC → VPC only**.
  - Name: `saa-console-vpc`, IPv4 CIDR `10.50.0.0/16`, no IPv6, tenancy **Default**
- [ ] Select the VPC and choose **Actions → Edit VPC settings**. Turn on **DNS resolution** and **DNS hostnames**.

### A2. Create subnets

- [ ] Open **Subnets → Create subnet**, select `saa-console-vpc`, and add four subnets:

| Name | AZ | CIDR |
|---|---|---|
| `c-public-a` | us-east-1a | 10.50.1.0/24 |
| `c-public-b` | us-east-1b | 10.50.2.0/24 |
| `c-private-a` | us-east-1a | 10.50.11.0/24 |
| `c-private-b` | us-east-1b | 10.50.12.0/24 |

- [ ] For each **public** subnet, choose **Actions → Edit subnet settings** and turn on **Auto-assign public IPv4 address**.

### A3. Internet gateway

- [ ] Open **Internet gateways → Create**, name it `saa-console-igw`, then **Actions → Attach to VPC** → `saa-console-vpc`.

### A4. Route tables

- [ ] Create a route table `c-public-rt` in `saa-console-vpc`.
  - **Routes → Edit → Add route:** `0.0.0.0/0` → Internet Gateway `saa-console-igw`
  - **Subnet associations:** `c-public-a`, `c-public-b`
- [ ] Create a route table `c-private-rt` with **no** internet route, and associate `c-private-a` and `c-private-b`.
- [ ] Look at the **local** route (`10.50.0.0/16 → local`) in both tables. It can't be removed, and it lets all subnets reach each other.

### A5. Explore the resource map

- [ ] Open **Your VPCs → saa-console-vpc → Resource map** and confirm the subnets, route tables, and IGW are wired the way you expect.

### A6. Test it

- [ ] Launch a `t3.micro` (Amazon Linux 2023) into `c-public-a` with **no key pair**, and a security group allowing nothing inbound.
- [ ] Use **EC2 Instance Connect**. It fails because no SSH rule exists. Add an SG rule: SSH from the `EC2_INSTANCE_CONNECT` prefix list for us-east-1 (or your IP), then connect.
- [ ] From the instance, run `curl -I https://aws.amazon.com`. It works because the subnet routes to the IGW and the instance has a public IP.
- [ ] Terminate the instance.

---

## Part B — CLI (reusable build: `saa-vpc`)

> This VPC is used by **Labs 12–39**. Its IDs are saved to `~/saa-vpc.env` so any later lab can run `source ~/saa-vpc.env`.

### B1. Create the VPC and turn on DNS

- [ ] Create it:

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=saa-vpc}]' \
  --query Vpc.VpcId --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
echo $VPC_ID
```

### B2. Create six subnets across two AZs

- [ ] Create a helper function and the subnets:

```bash
mk_subnet () {  # name cidr az
  aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $2 --availability-zone $3 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$1}]" \
    --query Subnet.SubnetId --output text
}

PUB_A=$(mk_subnet saa-public-a  10.0.1.0/24  us-east-1a)
PUB_B=$(mk_subnet saa-public-b  10.0.2.0/24  us-east-1b)
APP_A=$(mk_subnet saa-app-a     10.0.11.0/24 us-east-1a)
APP_B=$(mk_subnet saa-app-b     10.0.12.0/24 us-east-1b)
DATA_A=$(mk_subnet saa-data-a   10.0.21.0/24 us-east-1a)
DATA_B=$(mk_subnet saa-data-b   10.0.22.0/24 us-east-1b)

for S in $PUB_A $PUB_B; do
  aws ec2 modify-subnet-attribute --subnet-id $S --map-public-ip-on-launch
done

aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[].{Name:Tags[?Key==`Name`]|[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone,Free:AvailableIpAddressCount}' \
  --output table
```

> Notice `Free = 251` on each /24. That's the 5 reserved addresses.

### B3. Internet gateway and public route table

- [ ] Create the IGW and public routing:

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=saa-igw}]' \
  --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=saa-public-rt}]' \
  --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_B
```

### B4. Private route tables (one per AZ, ready for per-AZ NAT in Lab 12)

- [ ] Create them:

```bash
APP_RT_A=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=saa-private-rt-a}]' \
  --query RouteTable.RouteTableId --output text)
APP_RT_B=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=saa-private-rt-b}]' \
  --query RouteTable.RouteTableId --output text)

aws ec2 associate-route-table --route-table-id $APP_RT_A --subnet-id $APP_A
aws ec2 associate-route-table --route-table-id $APP_RT_A --subnet-id $DATA_A
aws ec2 associate-route-table --route-table-id $APP_RT_B --subnet-id $APP_B
aws ec2 associate-route-table --route-table-id $APP_RT_B --subnet-id $DATA_B
```

### B5. Save the IDs for later labs

- [ ] Write the env file:

```bash
cat > ~/saa-vpc.env <<EOF
export VPC_ID=$VPC_ID
export IGW_ID=$IGW_ID
export PUB_A=$PUB_A
export PUB_B=$PUB_B
export APP_A=$APP_A
export APP_B=$APP_B
export DATA_A=$DATA_A
export DATA_B=$DATA_B
export PUB_RT=$PUB_RT
export APP_RT_A=$APP_RT_A
export APP_RT_B=$APP_RT_B
EOF
cat ~/saa-vpc.env
```

### B6. Inspect the routing

- [ ] Look at each route table and its subnets:

```bash
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC_ID \
  --query 'RouteTables[].{Name:Tags[?Key==`Name`]|[0].Value,Routes:Routes[].[DestinationCidrBlock,GatewayId],Subnets:Associations[].SubnetId}'
```

---

## Verify

- [ ] `saa-vpc` has 6 subnets in 2 AZs, 1 IGW, a public route table, and 2 private route tables.
- [ ] Only the public subnets route `0.0.0.0/0` to the IGW.
- [ ] `~/saa-vpc.env` exists and holds every ID.

## Exam tips

- **Public subnet** = route to an IGW. Instances also need a **public or Elastic IP** to be reachable.
- **Multi-AZ** design means at least one subnet per tier per AZ. A single AZ is a single point of failure.
- Plan CIDRs so they **don't overlap** with on-prem or other VPCs, because peering and Transit Gateway need non-overlapping ranges.
- If you run out of IPs, **add a secondary CIDR** to the VPC. You can't resize an existing subnet.
- **Default VPC:** each region has one, with public subnets in every AZ. Fine for experiments, not for production.

## Challenge (optional)

- [ ] Use the console **"VPC and more"** wizard to create the same layout in a few clicks, then compare it with your hand-built VPC in the resource map. Delete it afterwards.
- [ ] Add an **IPv6** CIDR to `saa-console-vpc` and an **egress-only internet gateway** for IPv6 outbound from private subnets.

## Cleanup

- [ ] **Keep `saa-vpc`.** It's free and Labs 12–39 use it.
- [ ] Delete `saa-console-vpc`: terminate its instances, then in **Your VPCs** choose **Actions → Delete VPC**. The console also removes its subnets, route tables, IGW, and security groups.

---

## Knowledge check

**Q1.** How many usable IP addresses does a /28 subnet have in a VPC?

<details><summary>Answer</summary>

16 − 5 reserved = **11**.

</details>

**Q2.** An EC2 instance in a subnet whose route table has `0.0.0.0/0 → igw-…` can't be reached from the internet. What could be missing?

<details><summary>Answer</summary>

A **public IPv4 or Elastic IP**, a **security group** inbound rule, or a **network ACL** allowing the traffic (inbound **and** outbound ephemeral ports).

</details>

**Q3.** A company is planning VPCs that will later connect to its on-premises network (10.0.0.0/8) over VPN. What should it keep in mind when choosing CIDRs?

<details><summary>Answer</summary>

VPC CIDRs **must not overlap** with on-premises ranges or other connected VPCs, or routing won't work. Choose ranges outside 10.0.0.0/8 (e.g. 172.16.0.0/12) or carve out a dedicated, coordinated block.

</details>
