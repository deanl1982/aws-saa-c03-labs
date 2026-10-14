# Lab 18 — Elastic Load Balancing with an Application Load Balancer

| | |
|---|---|
| **Module** | 5 — Compute, Storage Volumes, Load Balancing & Scaling |
| **Exam mapping** | Domain 2 · Task 2.2 (highly available architectures) · Task 2.1 (scalable, loosely coupled) · Domain 3 · Task 3.4 (load balancer choice) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0.10/hour (ALB ~$0.0225/h + LCU, 3 × t3.micro; the optional NLB adds ~$0.0225/h) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-17-ebs-efs-storage.md) · [Back to index](../README.md) · [Next lab ➡](lab-19-auto-scaling.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Build an internet-facing **Application Load Balancer** across two AZs, with targets in **private** subnets.
- Configure **target groups**, **health checks**, and **deregistration delay** (connection draining).
- Route requests with **listener rules**: path-based, host-based, redirects, fixed responses.
- Lock targets down so they **only accept traffic from the ALB's security group**.
- Choose between **ALB, NLB, GWLB**, and the deprecated Classic LB.

## Load balancer comparison

| | ALB | NLB | GWLB |
|---|---|---|---|
| OSI layer | 7 (HTTP/HTTPS/gRPC/WebSocket) | 4 (TCP/UDP/TLS) | 3 (IP, GENEVE) |
| Routing | Path, host, header, query, method, source IP | Port | All traffic to virtual appliances |
| Static IP | No (use Global Accelerator in front) | **Yes, one EIP per AZ** | — |
| Performance | High | **Millions of rps, ultra-low latency** | — |
| Targets | Instances, IPs, **Lambda**, containers | Instances, IPs, **ALB** | Appliance instances or IPs |
| Security groups | Yes | Yes (optional) | No |
| Typical use | Microservices, web apps | Gaming, IoT, TCP/UDP, PrivateLink, static IP whitelisting | Firewalls, IDS/IPS inspection |

---

## Architecture

```
            Internet
               │
     ┌─────────▼─────────┐   alb-sg: 80 from 0.0.0.0/0
     │  ALB (public-a/b) │
     └───┬───────────┬───┘
   /*    │           │ /api/*
 ┌───────▼───┐   ┌───▼───────┐  target-sg: 80 from alb-sg only
 │ tg-web    │   │ tg-api    │
 │ web-1 (a) │   │ api-1 (a) │
 │ web-2 (b) │   └───────────┘
 └───────────┘   (all in private app subnets)
```

---

## Prerequisites

- `saa-vpc` (Lab 11). Targets live in the **private app subnets** and need **no internet access**. They serve pages with Python's built-in HTTP server.

---

## Part A — Console

### A1. Security groups

- [ ] `saa-alb-sg`: inbound **HTTP 80** and **HTTPS 443** from `0.0.0.0/0`.
- [ ] `saa-target-sg`: inbound **HTTP 80** from **`saa-alb-sg`** only.

### A2. Launch the targets

- [ ] Launch **two** instances, `saa-web-1` in **saa-app-a** and `saa-web-2` in **saa-app-b** (AL2023, t3.micro, no public IP, SG `saa-target-sg`, IMDSv2 required), with this user data:

```bash
#!/bin/bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
mkdir -p /srv/www/api
echo "<h1>WEB tier: $ID ($AZ)</h1>" > /srv/www/index.html
echo "{\"tier\":\"api\",\"instance\":\"$ID\",\"az\":\"$AZ\"}" > /srv/www/api/index.html
cd /srv/www && nohup python3 -m http.server 80 >/var/log/web.log 2>&1 &
```

- [ ] Launch a third instance, `saa-api-1`, in **saa-app-a** with the same user data.

### A3. Target groups

- [ ] Open **EC2 → Target groups → Create**.
  - `saa-tg-web`: type **Instances**, HTTP 80, VPC `saa-vpc`, health check path `/`, healthy threshold **2**, interval **10 s**
  - Register `saa-web-1` and `saa-web-2`.
- [ ] Create `saa-tg-api` the same way, with health check path `/api/`, and register `saa-api-1`.

### A4. Create the ALB

- [ ] Open **Load balancers → Create → Application Load Balancer**.
  - Name `saa-alb`, **Internet-facing**, IPv4
  - VPC `saa-vpc`, mappings **us-east-1a → saa-public-a**, **us-east-1b → saa-public-b**
  - SG `saa-alb-sg`
  - Listener HTTP:80 → default action **forward to `saa-tg-web`**
- [ ] When it's **Active**, browse to the ALB DNS name and refresh several times. The response **alternates between web-1 and web-2** across AZs.

### A5. Listener rules

- [ ] Open the **HTTP:80 listener → Rules → Add rule**:
  - Priority 10: **Path** `/api/*` → forward to `saa-tg-api`
  - Priority 20: **Path** `/old/*` → **redirect** to `/` (HTTP 301)
  - Priority 30: **Path** `/health-check-test` → **fixed response** 200 `OK from ALB`
- [ ] Test `/api/`, `/old/x`, and `/health-check-test`.

### A6. Health checks and failover

- [ ] Stop `saa-web-2`. Within about 30 s it shows **unhealthy** (then *unused*) in `saa-tg-web`, and every request goes to web-1.
- [ ] Start it again and watch it return to **healthy**.

### A7. Target group attributes

- [ ] Open `saa-tg-web`, choose **Attributes → Edit**, and look at:
  - **Deregistration delay** (connection draining): set it to 30 s
  - **Stickiness**: turn on load-balancer-generated cookie, 1 minute, and see that refreshes stick to one instance
  - **Load balancing algorithm**: Round robin / Least outstanding requests / Weighted random
- [ ] Turn stickiness off again.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
ALB_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-alb-sg --query 'SecurityGroups[0].GroupId' --output text)
TGT_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-target-sg --query 'SecurityGroups[0].GroupId' --output text)
ALB_ARN=$(aws elbv2 describe-load-balancers --names saa-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text)
ALB_DNS=$(aws elbv2 describe-load-balancers --names saa-alb --query 'LoadBalancers[0].DNSName' --output text)
TG_WEB=$(aws elbv2 describe-target-groups --names saa-tg-web --query 'TargetGroups[0].TargetGroupArn' --output text)
LISTENER=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[0].ListenerArn' --output text)
echo "export ALB_SG=$ALB_SG TGT_SG=$TGT_SG ALB_ARN=$ALB_ARN TG_WEB=$TG_WEB" >> ~/saa-vpc.env
```

### B1. Check target health

- [ ] Look at the targets:

```bash
aws elbv2 describe-target-health --target-group-arn $TG_WEB \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}' --output table
```

### B2. Watch round-robin from the terminal

- [ ] Send a few requests:

```bash
for i in $(seq 1 6); do curl -s http://$ALB_DNS/ ; echo; done
curl -s http://$ALB_DNS/api/ ; echo
```

### B3. Add a header-based rule with the CLI

- [ ] Send requests with header `X-Canary: true` to the API target group:

```bash
TG_API=$(aws elbv2 describe-target-groups --names saa-tg-api --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 create-rule --listener-arn $LISTENER --priority 5 \
  --conditions '[{"Field":"http-header","HttpHeaderConfig":{"HttpHeaderName":"X-Canary","Values":["true"]}}]' \
  --actions "[{\"Type\":\"forward\",\"TargetGroupArn\":\"$TG_API\"}]"

curl -s -H "X-Canary: true" http://$ALB_DNS/ ; echo
```

### B4. Weighted forwarding (blue/green or canary)

- [ ] Change the default action to split traffic 90/10 between the target groups:

```bash
aws elbv2 modify-listener --listener-arn $LISTENER --default-actions "[{
  \"Type\":\"forward\",
  \"ForwardConfig\":{\"TargetGroups\":[
    {\"TargetGroupArn\":\"$TG_WEB\",\"Weight\":90},
    {\"TargetGroupArn\":\"$TG_API\",\"Weight\":10}]}
}]"
for i in $(seq 1 20); do curl -s http://$ALB_DNS/ | head -c 40; echo; done | sort | uniq -c
```

- [ ] Restore the default action:

```bash
aws elbv2 modify-listener --listener-arn $LISTENER --default-actions "[{\"Type\":\"forward\",\"TargetGroupArn\":\"$TG_WEB\"}]"
```

### B5. Network Load Balancer with static IPs (optional, ~$0.03)

- [ ] Allocate an EIP per AZ and create the NLB:

```bash
EIP1=$(aws ec2 allocate-address --query AllocationId --output text)
EIP2=$(aws ec2 allocate-address --query AllocationId --output text)

NLB_ARN=$(aws elbv2 create-load-balancer --name saa-nlb --type network --scheme internet-facing \
  --subnet-mappings SubnetId=$PUB_A,AllocationId=$EIP1 SubnetId=$PUB_B,AllocationId=$EIP2 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
```

- [ ] Create a TCP target group and listener:

```bash
TG_NLB=$(aws elbv2 create-target-group --name saa-tg-nlb --protocol TCP --port 80 --vpc-id $VPC_ID \
  --target-type instance --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 register-targets --target-group-arn $TG_NLB \
  --targets $(aws elbv2 describe-target-health --target-group-arn $TG_WEB --query 'TargetHealthDescriptions[].Target.Id' --output text | xargs -n1 -I{} echo Id={})
aws elbv2 create-listener --load-balancer-arn $NLB_ARN --protocol TCP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_NLB
```

- [ ] Allow the NLB to reach the targets. NLB traffic keeps the **client IP**, so the target SG must allow the clients themselves:

```bash
aws ec2 authorize-security-group-ingress --group-id $TGT_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws elbv2 describe-load-balancers --load-balancer-arns $NLB_ARN --query 'LoadBalancers[0].AvailabilityZones[].LoadBalancerAddresses[0].IpAddress'
```

> The NLB has **fixed IPs**, the answer when a partner must allowlist your IPs.

### B6. ALB access logs to S3

- [ ] Create a bucket that the ELB log-delivery service can write to:

```bash
LOGB=saa-lab18-alblogs-$SUFFIX
aws s3 mb s3://$LOGB
aws s3api put-bucket-policy --bucket $LOGB --policy "{
 \"Version\":\"2012-10-17\",
 \"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"logdelivery.elasticloadbalancing.amazonaws.com\"},
 \"Action\":\"s3:PutObject\",\"Resource\":\"arn:aws:s3:::$LOGB/AWSLogs/$ACCOUNT_ID/*\"}]}"
```

- [ ] Turn on access logs:

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=$LOGB
```

---

## Verify

- [ ] The ALB spreads traffic across two AZs, and `/api/*` goes to the API target group.
- [ ] An unhealthy target is removed from rotation automatically.
- [ ] Targets aren't reachable except through the ALB (they have no public IPs, and the SG only allows the ALB, apart from the optional NLB rule).

## Exam tips

- **"Route /images to one fleet and /api to another"** → **ALB path-based routing**. **"Different domains on one LB"** → **host-based** routing plus **SNI** for multiple certificates.
- **"Static IP / extreme performance / UDP"** → **NLB**. **"Inspect traffic with third-party firewalls"** → **GWLB**.
- **ALB + Lambda targets** = serverless HTTP without API Gateway.
- **The client IP** at targets behind an ALB is in the **`X-Forwarded-For`** header. An NLB preserves the source IP.
- **Cross-zone load balancing:** on by default for ALB (free), off by default for NLB (costs inter-AZ data).
- **Connection draining** = **deregistration delay**. **Sticky sessions** use cookies, but prefer stateless apps with sessions in ElastiCache or DynamoDB.

## Challenge (optional)

- [ ] Add an **HTTPS listener** with an ACM certificate for a domain you own, and redirect HTTP → HTTPS (you'll revisit this in Lab 31).

## Cleanup

> **Going on to Lab 19?** Keep `saa-alb`, `saa-tg-web`, and the SGs. Delete only the targets and the optional NLB.

- [ ] NLB, its target group, and EIPs (if created):

```bash
aws elbv2 delete-load-balancer --load-balancer-arn $NLB_ARN
sleep 30
aws elbv2 delete-target-group --target-group-arn $TG_NLB
aws ec2 release-address --allocation-id $EIP1; aws ec2 release-address --allocation-id $EIP2
aws ec2 revoke-security-group-ingress --group-id $TGT_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
```

- [ ] Instances:

```bash
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-web-1,saa-web-2,saa-api-1 Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] **If you aren't going on to Lab 19**, delete the ALB, the target groups, and the log bucket:

```bash
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 30
aws elbv2 delete-target-group --target-group-arn $TG_WEB
aws elbv2 delete-target-group --target-group-arn $TG_API
aws s3 rb s3://$LOGB --force
```

---

## Knowledge check

**Q1.** A partner must allowlist a fixed set of IP addresses to reach your application. Which load balancer meets this?

<details><summary>Answer</summary>

A **Network Load Balancer** with an **Elastic IP per AZ**, or an ALB fronted by **AWS Global Accelerator**, which gives two static anycast IPs.

</details>

**Q2.** An application behind an ALB needs to log client IP addresses, but the logs only show the ALB's private IPs. What should the developers use?

<details><summary>Answer</summary>

The **`X-Forwarded-For`** HTTP header, which the ALB adds. ALB access logs also record the client IP.

</details>

**Q3.** Users lose their shopping carts when the ALB sends them to a different instance. What's the quickest fix, and what's the better long-term design?

<details><summary>Answer</summary>

Quick fix: turn on **sticky sessions**. Better design: make the app **stateless**, with session data in **ElastiCache** or **DynamoDB**.

</details>
