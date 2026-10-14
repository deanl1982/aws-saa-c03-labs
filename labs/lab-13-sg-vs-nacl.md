# Lab 13 — Security Groups vs Network ACLs

| | |
|---|---|
| **Module** | 4 — Networking with Amazon VPC |
| **Exam mapping** | Domain 1 · Task 1.2 (network security controls, segmentation) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.05 (two t3.micro instances, plus NAT if still running from Lab 12) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-12-nat-private-access.md) · [Back to index](../README.md) · [Next lab ➡](lab-14-vpc-endpoints-flowlogs.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Explain **stateful** (security group) vs **stateless** (network ACL) filtering.
- Build **tiered security groups** that reference each other, instead of using CIDRs.
- Write NACL rules in the right **order** (lowest number first), including **ephemeral port** ranges.
- Use NACLs to **block a specific IP**, which security groups can't do because they have no deny rules.

## Comparison

| | Security group | Network ACL |
|---|---|---|
| Applies to | ENI / instance | Subnet |
| State | **Stateful**: return traffic allowed automatically | **Stateless**: return traffic needs its own rule |
| Rules | **Allow only** | **Allow and deny** |
| Evaluation | All rules evaluated together | **In order**, lowest number first. The first match wins. |
| Default (custom) | Deny all inbound, allow all outbound | **Deny all** inbound and outbound |
| Default (VPC default NACL) | — | Allow all |
| Can reference | Other SGs, prefix lists, CIDRs | CIDRs only |

**Ephemeral ports:** clients choose a random source port (Linux 32768–60999, Windows 49152–65535, ELB/NAT 1024–65535). With a NACL, the **outbound** rule for responses must allow **1024–65535**.

---

## Architecture

```
Internet ──► [NACL public] ──► web-sg (80 from 0.0.0.0/0) ──► web instance (public-a)
                                         │ 8080
                                         ▼
           [NACL private] ──► app-sg (8080 from web-sg only) ──► app instance (app-a)
```

---

## Prerequisites

- Lab 11 (`saa-vpc`) and Lab 12 (the `saa-ssm-ec2-role` profile, and a NAT gateway in AZ-a so the app instance can install packages).

---

## Part A — Console

### A1. Tiered security groups

- [ ] Open **EC2 → Security groups → Create** in `saa-vpc`:
  - `saa-web-sg`: inbound **HTTP 80** from `0.0.0.0/0`
  - `saa-app-sg`: inbound **Custom TCP 8080**, source **`saa-web-sg`** (choose the SG, not a CIDR)

### A2. Launch the web and app instances

- [ ] Launch `saa-web` (AL2023, t3.micro) in **saa-public-a** with `saa-web-sg`, instance profile `saa-ssm-ec2-role`, and this **user data**:

```bash
#!/bin/bash
dnf -y install httpd
echo "<h1>web tier $(hostname)</h1>" > /var/www/html/index.html
systemctl enable --now httpd
```

- [ ] Launch `saa-app` in **saa-app-a** with `saa-app-sg`, the same profile, and this user data:

```bash
#!/bin/bash
mkdir -p /srv && echo "app tier OK" > /srv/index.html
cd /srv && nohup python3 -m http.server 8080 &
```

- [ ] Browse to `http://<web-public-ip>`. You should see the web page.
- [ ] Open a **Session Manager** session on `saa-web` and run `curl http://<app-private-ip>:8080`. You should get `app tier OK`.
- [ ] From your laptop, try `curl http://<app-private-ip>:8080`. It fails: no route, no public IP, and the SG doesn't allow you.

### A3. Show statefulness

- [ ] Edit `saa-web-sg` and **remove all outbound rules**.
- [ ] Browse to the web page again. **It still works**, because SGs are stateful and responses are allowed automatically.
- [ ] From the web instance, `curl http://<app-private-ip>:8080` now **fails** because new *outbound* connections are blocked.
- [ ] Restore the outbound rule (`All traffic → 0.0.0.0/0`).

### A4. Network ACL: block a single IP

- [ ] Find your public IP (`https://checkip.amazonaws.com`).
- [ ] Open **VPC → Network ACLs → Create network ACL**, name it `saa-public-nacl` in `saa-vpc`, and associate it with **saa-public-a** and **saa-public-b**.
- [ ] Browse to the web page. **It fails**, because a new custom NACL denies everything.
- [ ] Add **inbound** rules:

| Rule # | Type | Port | Source | Allow/Deny |
|---|---|---|---|---|
| 100 | HTTP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | Custom TCP | 1024–65535 | 0.0.0.0/0 | ALLOW (return traffic for the instance's outbound calls) |

- [ ] Add **outbound** rules:

| Rule # | Type | Port | Destination | Allow/Deny |
|---|---|---|---|---|
| 100 | Custom TCP | 1024–65535 | 0.0.0.0/0 | ALLOW (responses to clients) |
| 110 | HTTP | 80 | 0.0.0.0/0 | ALLOW |
| 120 | HTTPS | 443 | 0.0.0.0/0 | ALLOW |
| 130 | Custom TCP | 8080 | 10.0.0.0/16 | ALLOW (web → app) |

- [ ] Browse to the web page. **It works.**
- [ ] Add inbound rule **90**: HTTP 80 from `<your-ip>/32` **DENY**.
- [ ] Browse again. **It's blocked for you**, because rule 90 is evaluated before rule 100.
- [ ] Delete rule 90.

### A5. Prove statelessness

- [ ] Remove outbound rule **100** (the ephemeral ports).
- [ ] Browse to the web page. **It fails.** The request gets in, but the response is dropped on the way out.
- [ ] Restore rule 100.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
MYIP=$(curl -s https://checkip.amazonaws.com)/32
WEB_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-web-sg Name=vpc-id,Values=$VPC_ID --query 'SecurityGroups[0].GroupId' --output text)
APP_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-app-sg Name=vpc-id,Values=$VPC_ID --query 'SecurityGroups[0].GroupId' --output text)
```

### B1. Inspect security groups and their references

- [ ] Look at the rules:

```bash
aws ec2 describe-security-group-rules --filters Name=group-id,Values=$APP_SG \
  --query 'SecurityGroupRules[].{Egress:IsEgress,Proto:IpProtocol,From:FromPort,To:ToPort,CIDR:CidrIpv4,RefSG:ReferencedGroupInfo.GroupId}' \
  --output table
```

### B2. Add a database tier SG with the CLI

- [ ] Create it:

```bash
DB_SG=$(aws ec2 create-security-group --group-name saa-db-sg --description "DB tier" \
  --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $DB_SG \
  --ip-permissions "IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId=$APP_SG,Description='MySQL from app tier'}]"

echo "export WEB_SG=$WEB_SG APP_SG=$APP_SG DB_SG=$DB_SG" >> ~/saa-vpc.env
```

### B3. A NACL for the data subnets

- [ ] Create it and associate it with both data subnets:

```bash
DATA_NACL=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=saa-data-nacl}]' \
  --query NetworkAcl.NetworkAclId --output text)

# Associate: NACL associations are *replaced*, not created
for S in $DATA_A $DATA_B; do
  ASSOC=$(aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=$S \
    --query "NetworkAcls[0].Associations[?SubnetId=='$S'].NetworkAclAssociationId" --output text)
  aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $DATA_NACL
done
```

- [ ] Allow MySQL from the app subnets only, plus return traffic:

```bash
aws ec2 create-network-acl-entry --network-acl-id $DATA_NACL --ingress --rule-number 100 \
  --protocol tcp --port-range From=3306,To=3306 --cidr-block 10.0.11.0/24 --rule-action allow
aws ec2 create-network-acl-entry --network-acl-id $DATA_NACL --ingress --rule-number 110 \
  --protocol tcp --port-range From=3306,To=3306 --cidr-block 10.0.12.0/24 --rule-action allow
aws ec2 create-network-acl-entry --network-acl-id $DATA_NACL --egress --rule-number 100 \
  --protocol tcp --port-range From=1024,To=65535 --cidr-block 10.0.0.0/16 --rule-action allow
```

- [ ] Review the entries in evaluation order:

```bash
aws ec2 describe-network-acls --network-acl-ids $DATA_NACL \
  --query 'NetworkAcls[0].Entries[].{Rule:RuleNumber,Egress:Egress,Action:RuleAction,CIDR:CidrBlock,Ports:PortRange}' \
  --output table
```

> Notice rule `32767` (`*`) DENY at the end. It's the implicit catch-all.

### B4. Block an IP on the public NACL from the CLI

- [ ] Add and test a deny rule:

```bash
PUB_NACL=$(aws ec2 describe-network-acls --filters Name=tag:Name,Values=saa-public-nacl --query 'NetworkAcls[0].NetworkAclId' --output text)
WEB_IP=$(aws ec2 describe-instances --filters Name=tag:Name,Values=saa-web Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

curl -s -m 5 http://$WEB_IP   # works

aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL --ingress --rule-number 90 \
  --protocol tcp --port-range From=80,To=80 --cidr-block $MYIP --rule-action deny
curl -s -m 5 http://$WEB_IP || echo "Blocked by NACL rule 90"

aws ec2 delete-network-acl-entry --network-acl-id $PUB_NACL --ingress --rule-number 90
```

### B5. Managed prefix lists (bonus)

- [ ] Create a prefix list of trusted office CIDRs and reference it from an SG:

```bash
PL=$(aws ec2 create-managed-prefix-list --prefix-list-name saa-office-cidrs --address-family IPv4 --max-entries 5 \
  --entries Cidr=$MYIP,Description=home --query PrefixList.PrefixListId --output text)
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,PrefixListIds=[{PrefixListId=$PL}]"
```

---

## Verify

- [ ] The app tier is reachable **only** from the web tier's SG.
- [ ] You blocked your own IP with a NACL deny rule and then removed it.
- [ ] You saw that removing NACL ephemeral outbound rules breaks responses.

## Exam tips

- **"Block a specific malicious IP"** → **NACL deny** (or AWS WAF for HTTP). SGs can't deny.
- **"Only allow traffic from the load balancer"** → instance SG inbound from the **ALB's SG**.
- **"Instance-level firewall"** → SG. **"Subnet-level firewall"** → NACL.
- A response being blocked while the request gets through almost always means a **NACL missing ephemeral ports**.
- **AWS Network Firewall** is a managed stateful firewall with IDS/IPS and domain filtering for whole VPCs. **Firewall Manager** manages SGs, WAF, and Network Firewall policies centrally across an Organization.

## Challenge (optional)

- [ ] Use **VPC Reachability Analyzer** to find the path from your web instance to the app instance on port 8080 and explain each hop. Then deliberately break the SG and see which component it blames.

## Cleanup

- [ ] Terminate the instances:

```bash
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-web,saa-app Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] Re-associate the public and data subnets with the **default NACL**, then delete `saa-public-nacl` and `saa-data-nacl`:

```bash
DEFAULT_NACL=$(aws ec2 describe-network-acls --filters Name=vpc-id,Values=$VPC_ID Name=default,Values=true --query 'NetworkAcls[0].NetworkAclId' --output text)
for N in $PUB_NACL $DATA_NACL; do
  for A in $(aws ec2 describe-network-acls --network-acl-ids $N --query 'NetworkAcls[0].Associations[].NetworkAclAssociationId' --output text); do
    aws ec2 replace-network-acl-association --association-id $A --network-acl-id $DEFAULT_NACL
  done
  aws ec2 delete-network-acl --network-acl-id $N
done
```

- [ ] Delete the prefix list after removing the SG rule that uses it:

```bash
aws ec2 revoke-security-group-ingress --group-id $WEB_SG \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,PrefixListIds=[{PrefixListId=$PL}]"
aws ec2 delete-managed-prefix-list --prefix-list-id $PL
```

- [ ] **Keep** `saa-web-sg`, `saa-app-sg`, and `saa-db-sg`. Later labs reuse them.
- [ ] Keep the NAT gateway if you're going straight to Lab 14 (you'll replace it with endpoints there). Otherwise follow the Lab 12 NAT cleanup.

---

## Knowledge check

**Q1.** A web server in a public subnet receives requests, but clients never get responses. The SG allows inbound 80. The custom NACL allows inbound 80 and outbound 80. What's wrong?

<details><summary>Answer</summary>

The NACL is **stateless**, so it needs an **outbound rule for ephemeral ports (1024–65535)** to carry the responses back to clients.

</details>

**Q2.** A company is under attack from one IP address hitting its EC2 web servers. What's the fastest network-level way to block it?

<details><summary>Answer</summary>

Add a **NACL inbound DENY** rule for that IP with a lower rule number than the allow rules. If the servers sit behind an ALB or CloudFront, an **AWS WAF IP set** rule also works.

</details>

**Q3.** How do you let database instances accept connections only from the application servers, even as app servers scale in and out?

<details><summary>Answer</summary>

Allow inbound DB traffic on the DB security group **from the app tier's security group ID** instead of from IP addresses.

</details>
