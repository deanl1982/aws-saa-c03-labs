# Lab 31 — Perimeter Security: ACM, WAF, Shield & GuardDuty

| | |
|---|---|
| **Module** | 8 — Data Protection, Monitoring & Threat Detection |
| **Exam mapping** | Domain 1 · Task 1.2 (secure workloads: WAF, Shield, encryption in transit) · Task 1.3 (Macie) · Task 1.1 (threat detection) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0.50 (WAF web ACL at $5/month + $1/rule prorated, ALB, t3.micro). GuardDuty, Inspector, and Macie have 30-day free trials. |

- [ ] **Lab complete**

[⬅ Previous lab](lab-30-cloudwatch-cloudtrail-config.md) · [Back to index](../README.md) · [Next lab ➡](lab-32-route53.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Terminate **TLS** on an ALB with an **ACM** certificate and redirect HTTP → HTTPS.
- Protect a web app with **AWS WAF**: managed rule groups, **rate-based rules**, IP sets, and geo-matching.
- Explain **Shield Standard vs Shield Advanced** for DDoS protection.
- Turn on and interpret **GuardDuty**, **Inspector**, **Macie**, and **Security Hub** findings.

## Security service map

| Service | What it does | Exam trigger |
|---|---|---|
| **ACM** | Free public TLS certificates, auto-renewed, for ALB, CloudFront, and API Gateway | "HTTPS with no certificate management" |
| **WAF** | Layer-7 filtering: SQLi, XSS, bots, rate limits, IP and geo rules | "Block SQL injection / specific countries / too many requests" |
| **Shield Standard** | Free, automatic L3/L4 DDoS protection | Default for every customer |
| **Shield Advanced** | $3k/month. 24/7 DDoS Response Team, cost protection, advanced L7 mitigation. | "Protection against large DDoS + cost protection + expert response" |
| **Firewall Manager** | Central WAF, Shield, SG, and Network Firewall policies across an Org | "Enforce WAF rules on every ALB in every account" |
| **GuardDuty** | ML threat detection from CloudTrail, VPC Flow Logs, DNS, S3, EKS, RDS, and runtime | "Detect crypto-mining, compromised credentials, unusual API calls" |
| **Inspector** | Vulnerability scanning: EC2 (CVEs, network reachability), ECR images, Lambda | "Find software vulnerabilities in instances and containers" |
| **Macie** | Finds **PII / sensitive data in S3** | "Identify buckets that contain credit card numbers" |
| **Security Hub** | Aggregates findings plus checks against CIS / AWS Foundational Best Practices | "Single pane of glass for security posture" |
| **Detective** | Investigates the root cause of findings with graph analysis | "Investigate a GuardDuty finding" |

---

## Prerequisites: a target ALB

- [ ] Create the instance, target group, ALB, and listener. This reuses the patterns from Lab 18:

```bash
source ~/saa-vpc.env
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)

WAF_I=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $APP_A \
  --security-group-ids $TGT_SG --metadata-options HttpTokens=required \
  --user-data '#!/bin/bash
mkdir -p /srv && echo "<h1>Protected app</h1>" > /srv/index.html && cd /srv && nohup python3 -m http.server 80 &' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-waf-target}]' --query 'Instances[0].InstanceId' --output text)

TG=$(aws elbv2 create-target-group --name saa-waf-tg --protocol HTTP --port 80 --vpc-id $VPC_ID \
  --health-check-path / --query 'TargetGroups[0].TargetGroupArn' --output text)
aws ec2 wait instance-running --instance-ids $WAF_I
aws elbv2 register-targets --target-group-arn $TG --targets Id=$WAF_I

WAF_ALB=$(aws elbv2 create-load-balancer --name saa-waf-alb --subnets $PUB_A $PUB_B --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 create-listener --load-balancer-arn $WAF_ALB --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TG
aws elbv2 wait load-balancer-available --load-balancer-arns $WAF_ALB
WAF_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $WAF_ALB --query 'LoadBalancers[0].DNSName' --output text)
sleep 30; curl -s http://$WAF_DNS/
```

---

## Part A — Console

### A1. ACM certificate and HTTPS (needs a domain you control)

> **No domain?** Read through this step and come back after Lab 32 if you register one.

- [ ] Open **ACM → Request certificate → Public**: `app.<yourdomain>`, validation **DNS**. If the domain is in Route 53, choose **Create records in Route 53**.
- [ ] When the status is **Issued**, open `saa-waf-alb` → **Listeners → Add listener**: HTTPS 443 → `saa-waf-tg`, certificate from ACM, security policy `ELBSecurityPolicy-TLS13-1-2-2021-06`.
- [ ] Edit the **HTTP:80** listener's default action to **Redirect to HTTPS 443 (301)**.
- [ ] Create a Route 53 **alias A record** `app.<yourdomain>` → the ALB, then browse to `https://app.<yourdomain>`.

> **For CloudFront, the ACM certificate must be in us-east-1.** For an ALB, it must be in the ALB's region.

### A2. Create a WAF web ACL

- [ ] Open **WAF & Shield → Web ACLs → Create web ACL**.
  - Resource type **Regional (ALB…)**, region us-east-1, name `saa-web-acl`
  - **Associated resources:** `saa-waf-alb`
  - **Add rules → AWS managed rule groups:**
    - **Core rule set** (`AWSManagedRulesCommonRuleSet`)
    - **SQL database** (`AWSManagedRulesSQLiRuleSet`)
    - **Known bad inputs** (`AWSManagedRulesKnownBadInputsRuleSet`)
    - **Amazon IP reputation list**
  - **Add my own rule → Rate-based rule**: `rate-limit-100`, limit **100** requests per **1 minute** per IP, action **Block**
  - Default action **Allow**
  - **CloudWatch metrics** on, **sampled requests** on

### A3. Test the WAF

- [ ] Send a normal request, then attack payloads:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://$WAF_DNS/"                                   # 200
curl -s -o /dev/null -w "%{http_code}\n" "http://$WAF_DNS/?id=1%20OR%201=1--"                 # 403 (SQLi)
curl -s -o /dev/null -w "%{http_code}\n" "http://$WAF_DNS/?q=<script>alert(1)</script>"       # 403 (XSS)
curl -s -o /dev/null -w "%{http_code}\n" -H "User-Agent: " "http://$WAF_DNS/"                 # may be 403 (no UA)
```

- [ ] Trigger the rate limit (wait about 1 minute after this):

```bash
for i in $(seq 1 300); do curl -s -o /dev/null -w "%{http_code}\n" "http://$WAF_DNS/"; done | sort | uniq -c
```

- [ ] Open **Web ACL → Overview → Sampled requests** to see which rule blocked each request.

### A4. IP set and geo-block

- [ ] Open **IP sets → Create** `saa-blocklist` with your public IP `/32`, then add a rule `block-my-ip` (IP set match → Block) at **priority 0**. Your requests now return **403**. Remove the rule afterwards.
- [ ] Add a **geo match** rule that blocks a country you aren't in, and see that it has no effect on you.

### A5. Shield

- [ ] Open **WAF & Shield → Shield → Overview**. Shield Standard is already active. Read the **Shield Advanced** subscription page and note its features (the DDoS Response Team, cost protection, health-based detection). **Don't subscribe** ($3,000/month, 1-year commitment).

### A6. Threat detection services (30-day trials)

- [ ] **GuardDuty → Get started → Enable**. Then open **Settings → Generate sample findings** and browse findings such as `CryptoCurrency:EC2/BitcoinTool.B!DNS` and `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`.
- [ ] **Inspector → Activate** (EC2, ECR, Lambda scanning). After about 15 minutes, look at the **Findings by instance** for `saa-waf-target`.
- [ ] **Macie → Get started → Enable**. Create bucket `saa-lab31-pii-<suffix>`, upload a CSV with fake PII:

```csv
name,email,card
Jane Doe,jane@example.com,4111111111111111
John Roe,john@example.com,5500000000000004
```

  Then run a **one-time sensitive data discovery job** on that bucket and look at the findings (credit card numbers, emails).
- [ ] **Security Hub → Enable** (with the AWS Foundational Security Best Practices standard). After a while, check the **security score** and the aggregated GuardDuty, Inspector, and Macie findings.

---

## Part B — CLI

### B1. Inspect the web ACL and its metrics

- [ ] Look at the ACL's rules and blocked-request counts:

```bash
ACL_ARN=$(aws wafv2 list-web-acls --scope REGIONAL --query "WebACLs[?Name=='saa-web-acl'].ARN" --output text)
ACL_ID=$(aws wafv2 list-web-acls --scope REGIONAL --query "WebACLs[?Name=='saa-web-acl'].Id" --output text)
aws wafv2 get-web-acl --scope REGIONAL --name saa-web-acl --id $ACL_ID \
  --query 'WebACL.Rules[].{Name:Name,Priority:Priority}' --output table

aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests \
  --dimensions Name=WebACL,Value=saa-web-acl Name=Region,Value=us-east-1 Name=Rule,Value=ALL \
  --statistics Sum --period 300 \
  --start-time $(date -u -v-1H +%FT%TZ 2>/dev/null || date -u -d '1 hour ago' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

### B2. Manage an IP set from the CLI (incident response)

- [ ] Create an IP set:

```bash
IPSET=$(aws wafv2 create-ip-set --name saa-cli-blocklist --scope REGIONAL --ip-address-version IPV4 \
  --addresses 198.51.100.0/24 --query 'Summary.ARN' --output text)
```

- [ ] Update it. WAF uses optimistic locking, so you need the current lock token:

```bash
IPSET_ID=$(aws wafv2 list-ip-sets --scope REGIONAL --query "IPSets[?Name=='saa-cli-blocklist'].Id" --output text)
LOCK=$(aws wafv2 get-ip-set --scope REGIONAL --name saa-cli-blocklist --id $IPSET_ID --query LockToken --output text)
aws wafv2 update-ip-set --scope REGIONAL --name saa-cli-blocklist --id $IPSET_ID --lock-token $LOCK \
  --addresses 198.51.100.0/24 203.0.113.66/32
```

### B3. WAF logging

- [ ] Send WAF logs to a CloudWatch log group. The name **must start with `aws-waf-logs-`**:

```bash
aws logs create-log-group --log-group-name aws-waf-logs-saa
aws wafv2 put-logging-configuration --logging-configuration \
  "ResourceArn=$ACL_ARN,LogDestinationConfigs=arn:aws:logs:us-east-1:$ACCOUNT_ID:log-group:aws-waf-logs-saa"
```

### B4. GuardDuty from the CLI

- [ ] Generate sample findings and list them by severity:

```bash
DET=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)
aws guardduty create-sample-findings --detector-id $DET
aws guardduty list-findings --detector-id $DET --max-results 10 --query FindingIds --output text | \
  xargs aws guardduty get-findings --detector-id $DET --finding-ids \
  --query 'Findings[].{Type:Type,Severity:Severity,Resource:Resource.ResourceType}' --output table
```

- [ ] Route **high-severity** findings to an SNS topic with EventBridge:

```bash
aws events put-rule --name saa-guardduty-high --event-pattern '{
  "source":["aws.guardduty"],"detail-type":["GuardDuty Finding"],
  "detail":{"severity":[{"numeric":[">=",7]}]}}'
```

> Add an SNS or Lambda target for automated response, e.g. isolating an instance by swapping its security group.

### B5. Inspector and Security Hub queries

- [ ] List findings:

```bash
aws inspector2 list-findings --max-results 5 \
  --query 'findings[].{Title:title,Severity:severity,Resource:resources[0].id}' --output table

aws securityhub get-findings --max-results 5 \
  --query 'Findings[].{Title:Title,Severity:Severity.Label,Product:ProductName}' --output table
```

---

## Verify

- [ ] WAF returns 403 for SQLi and XSS payloads and for floods over the rate limit.
- [ ] GuardDuty shows sample findings, and Macie found the fake card numbers.
- [ ] *(If you have a domain)* The ALB serves HTTPS with an ACM certificate and redirects HTTP.

## Exam tips

- **WAF attaches to:** ALB, API Gateway (REST), CloudFront, AppSync, Cognito user pools, App Runner, and Verified Access. **Not** to NLB or EC2 directly.
- **"Block an IP at the application layer"** → **WAF IP set**. **"Block an IP at the subnet level"** → **NACL**.
- **"Protect against DDoS at the edge"** → **CloudFront + Shield (+ WAF)**, plus Route 53 (also Shield-protected).
- **"Detect compromised EC2 instances / unusual API activity"** → **GuardDuty**. **"Find CVEs"** → **Inspector**. **"Find PII in S3"** → **Macie**.
- **"Centrally enforce WAF across all accounts"** → **Firewall Manager** (needs Organizations).
- ACM certificates for **CloudFront must be in us-east-1**. ACM public certs are **free**, but you can't export their private keys.

## Challenge (optional)

- [ ] Add **AWS WAF Bot Control** (Common level) and look at its labels in the logs. Note its extra cost.

## Cleanup

- [ ] Detach and delete the WAF resources:

```bash
aws wafv2 delete-logging-configuration --resource-arn $ACL_ARN
aws wafv2 disassociate-web-acl --resource-arn $WAF_ALB
LOCK=$(aws wafv2 get-web-acl --scope REGIONAL --name saa-web-acl --id $ACL_ID --query LockToken --output text)
aws wafv2 delete-web-acl --scope REGIONAL --name saa-web-acl --id $ACL_ID --lock-token $LOCK
for N in saa-cli-blocklist saa-blocklist; do
  ID=$(aws wafv2 list-ip-sets --scope REGIONAL --query "IPSets[?Name=='$N'].Id" --output text)
  L=$(aws wafv2 get-ip-set --scope REGIONAL --name $N --id $ID --query LockToken --output text)
  aws wafv2 delete-ip-set --scope REGIONAL --name $N --id $ID --lock-token $L
done
aws logs delete-log-group --log-group-name aws-waf-logs-saa
```

- [ ] Delete the ALB, target group, and instance:

```bash
aws elbv2 delete-load-balancer --load-balancer-arn $WAF_ALB
sleep 30 && aws elbv2 delete-target-group --target-group-arn $TG
aws ec2 terminate-instances --instance-ids $WAF_I
```

- [ ] Delete the Macie bucket and the EventBridge rule:

```bash
aws s3 rb s3://saa-lab31-pii-<suffix> --force
aws events delete-rule --name saa-guardduty-high
```

- [ ] **Decide** which detection services to keep:
  - **GuardDuty:** recommended to keep on (cheap for small accounts after the trial).
  - **Inspector, Macie, Security Hub:** disable them before the 30-day trial ends unless you want to keep paying (`aws inspector2 disable --resource-types EC2 ECR LAMBDA`, `aws macie2 disable-macie`, `aws securityhub disable-security-hub`).
- [ ] Delete the ACM certificate and the HTTPS listener (if you created them).

---

## Knowledge check

**Q1.** A web application behind an ALB is hit by SQL injection attempts and HTTP floods from a few IPs. What's the most operationally efficient protection?

<details><summary>Answer</summary>

**AWS WAF** on the ALB, with the **AWS managed SQLi rule group** and a **rate-based rule**.

</details>

**Q2.** A company wants DDoS protection with access to AWS DDoS experts during attacks and **credits for scaling charges caused by attacks**. What should it buy?

<details><summary>Answer</summary>

**AWS Shield Advanced.**

</details>

**Q3.** A company needs to find which of its 2,000 S3 buckets contain personal data such as passport numbers. Which service?

<details><summary>Answer</summary>

**Amazon Macie.**

</details>
