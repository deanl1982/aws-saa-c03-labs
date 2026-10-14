# Lab 30 — Observability & Audit: CloudWatch, CloudTrail & AWS Config

| | |
|---|---|
| **Module** | 8 — Data Protection, Monitoring & Threat Detection |
| **Exam mapping** | Domain 1 · Task 1.2 (monitoring, auditing) · Task 1.3 (compliance) · Domain 2 · Task 2.2 (alarms, automated recovery) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0.50 (custom metrics, Config items and rule evaluations, one t3.micro) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-29-kms-secrets.md) · [Back to index](../README.md) · [Next lab ➡](lab-31-waf-shield-guardduty.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Collect OS-level metrics and logs with the **CloudWatch agent** (memory and disk aren't built-in metrics).
- Build **metric filters**, **alarms**, **composite alarms**, and **dashboards**, and query logs with **Logs Insights**.
- Use **EC2 alarm actions** to recover or stop instances automatically.
- Create a **multi-region CloudTrail trail** with **log file validation** and encryption.
- Assess compliance with **AWS Config rules** and **auto-remediate** noncompliant resources.

## Who does what

| Question | Service |
|---|---|
| "How is it performing?" (metrics, logs, alarms) | **CloudWatch** |
| "**Who** did **what**, **when**, from **where**?" (API calls) | **CloudTrail** |
| "What did this resource **look like** over time, and is it **compliant**?" | **AWS Config** |
| "Is AWS having problems that affect me?" | **AWS Health Dashboard** |
| "Trace a request through microservices" | **AWS X-Ray** / CloudWatch Application Signals |

---

## Prerequisites

- `saa-vpc` and `saa-ssm-ec2-role`. Add the **CloudWatchAgentServerPolicy** managed policy to `saa-ssm-ec2-role`.
- An instance `saa-monitored` in **saa-public-a** (public IP, profile `saa-ssm-ec2-role`):

```bash
source ~/saa-vpc.env
aws iam attach-role-policy --role-name saa-ssm-ec2-role --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
MON=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PUB_A \
  --iam-instance-profile Name=saa-ssm-ec2-role --metadata-options HttpTokens=required \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-monitored}]' \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-status-ok --instance-ids $MON
```

---

## Part A — Console

### A1. Install and configure the CloudWatch agent with SSM

- [ ] Open **Systems Manager → Run Command → Run command**:
  - Document **AWS-ConfigureAWSPackage**, action **Install**, name **AmazonCloudWatchAgent**
  - Target: instance `saa-monitored`
- [ ] Open **Systems Manager → Parameter Store** and create `AmazonCloudWatch-saa-config` (String) with:

```json
{
  "metrics": {
    "namespace": "SAA/EC2",
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": { "collect_list": [
        { "file_path": "/var/log/messages", "log_group_name": "saa-system-logs", "log_stream_name": "{instance_id}" },
        { "file_path": "/var/log/app.log",  "log_group_name": "saa-app-logs",    "log_stream_name": "{instance_id}" }
      ]}
    }
  }
}
```

- [ ] Run Command again: document **AmazonCloudWatch-ManageAgent**, action **configure**, source **ssm**, parameter `AmazonCloudWatch-saa-config`, **restart yes**.
- [ ] After about 5 minutes, open **CloudWatch → Metrics → All metrics → SAA/EC2** and find `mem_used_percent`.

### A2. Metric filter and alarm on application logs

- [ ] On the instance (Session Manager), write some log lines:

```bash
for i in $(seq 1 20); do echo "$(date -Is) INFO request ok" | sudo tee -a /var/log/app.log; done
for i in $(seq 1 6);  do echo "$(date -Is) ERROR payment failed" | sudo tee -a /var/log/app.log; done
```

- [ ] Open **CloudWatch → Log groups → saa-app-logs → Metric filters → Create**:
  - Pattern `ERROR`, metric namespace `SAA/App`, name `ErrorCount`, value `1`, default value `0`
- [ ] Choose **Create alarm** on `ErrorCount`: **Sum ≥ 5 over 1 minute**, action → SNS topic `saa-ops-alerts` (create it and subscribe your email).
- [ ] Write 6 more ERROR lines and watch the alarm go to **In alarm** and the email arrive.

### A3. EC2 auto-recovery alarm

- [ ] Open **EC2 → saa-monitored → Actions → Monitor and troubleshoot → Manage CloudWatch alarms → Create**:
  - Metric **StatusCheckFailed_System ≥ 1** for 2 periods of 1 minute, action **Recover this instance**.

> On a system status failure (underlying hardware), **recover** moves the instance to new hardware, keeping its instance ID, private IP, EIP, and EBS volumes. Simplified automatic recovery is also on by default for supported instance types.

### A4. Dashboard

- [ ] Open **CloudWatch → Dashboards → Create** `saa-ops`, and add a line widget (CPUUtilization, mem_used_percent), a number widget (ErrorCount), and an alarm status widget.

### A5. CloudTrail: event history and a trail

- [ ] Open **CloudTrail → Event history**. Filter by **Event name = RunInstances** and open an event to see who, from which IP, and the request parameters. (Event history covers **90 days** of management events, free.)
- [ ] Open **Trails → Create trail**:
  - Name `saa-org-trail`, **new S3 bucket**, **Log file SSE-KMS encryption**: new key alias `saa-cloudtrail`
  - **Log file validation** ✔, **CloudWatch Logs** ✔ (new log group `saa-cloudtrail-logs`)
  - Events: **Management events** (Read + Write). Optionally **Data events** for S3 on one bucket only (it's high volume).
  - **Insights events** ✘ (extra cost; detects unusual API activity)

### A6. AWS Config

- [ ] Open **AWS Config → Get started (1-click setup)**: record **all resources**, delivery to a new bucket, AWS Config service-linked role.
- [ ] Open **Rules → Add rule** and add these AWS managed rules:
  - `restricted-ssh` (security groups shouldn't allow 0.0.0.0/0 on port 22)
  - `s3-bucket-public-read-prohibited`
  - `encrypted-volumes`
- [ ] Create a test SG `saa-bad-sg` with **SSH 22 from 0.0.0.0/0**. After a few minutes, `restricted-ssh` shows it as **Noncompliant**.
- [ ] Open the rule, choose **Actions → Manage remediation**, and set:
  - Automatic remediation ✔, action **AWS-DisableIncomingSSHOnPort22**, parameter `SecurityGroupIds` = **RESOURCE_ID**, plus an automation role with `ec2:RevokeSecurityGroupIngress`
- [ ] Re-evaluate. The rule is removed automatically and the resource becomes **Compliant**.
- [ ] Open **Resources → saa-bad-sg → Resource timeline** to see the configuration history and CloudTrail events linked to each change.

---

## Part B — CLI

### B1. Publish a custom metric and alarm on it

- [ ] Publish a few data points:

```bash
for v in 3 5 8 12 15; do
  aws cloudwatch put-metric-data --namespace SAA/App --metric-name QueueBacklog --value $v --unit Count \
    --dimensions Service=checkout
  sleep 2
done
```

- [ ] Create an alarm on the metric:

```bash
TOPIC=$(aws sns create-topic --name saa-ops-alerts --query TopicArn --output text)
aws cloudwatch put-metric-alarm --alarm-name saa-backlog-high \
  --namespace SAA/App --metric-name QueueBacklog --dimensions Name=Service,Value=checkout \
  --statistic Maximum --period 60 --evaluation-periods 1 --threshold 10 \
  --comparison-operator GreaterThanThreshold --treat-missing-data notBreaching \
  --alarm-actions $TOPIC
```

### B2. Composite alarm (reduce alarm noise)

- [ ] Fire only when **both** the backlog is high **and** errors are high:

```bash
aws cloudwatch put-composite-alarm --alarm-name saa-checkout-degraded \
  --alarm-rule 'ALARM("saa-backlog-high") AND ALARM("ErrorCount-alarm-name-from-console")' \
  --alarm-actions $TOPIC
```

- [ ] Replace the second alarm name with the exact name of the ErrorCount alarm you created in A2.

### B3. Logs Insights from the CLI

- [ ] Count errors per 5-minute bucket:

```bash
QID=$(aws logs start-query --log-group-name saa-app-logs \
  --start-time $(( $(date +%s) - 3600 )) --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | stats count() as errors by bin(5m)' \
  --query queryId --output text)
sleep 5
aws logs get-query-results --query-id $QID --query results
```

- [ ] Set log retention. The default is **never expire**, which costs money:

```bash
aws logs put-retention-policy --log-group-name saa-app-logs --retention-in-days 14
```

### B4. Export logs to S3, or subscribe them

- [ ] Look at the subscription-filter command (for streaming logs to Kinesis Data Firehose, Lambda, or OpenSearch):

```bash
aws logs put-subscription-filter --help | head -20
```

> **Real-time log processing / centralised log account** → **subscription filters** to Kinesis or Firehose. **Archive cheaply** → export to S3, or Firehose to S3.

### B5. CloudTrail from the CLI

- [ ] Look up recent events:

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --max-results 5 --query 'Events[].{Time:EventTime,User:Username,Event:EventName}' --output table
```

- [ ] Check the trail's status and validate the log files:

```bash
aws cloudtrail get-trail-status --name saa-org-trail --query '{Logging:IsLogging,LatestDelivery:LatestDeliveryTime}'
TRAIL_ARN=$(aws cloudtrail describe-trails --trail-name-list saa-org-trail --query 'trailList[0].TrailARN' --output text)
aws cloudtrail validate-logs --trail-arn $TRAIL_ARN \
  --start-time $(date -u -v-2H +%FT%TZ 2>/dev/null || date -u -d '2 hours ago' +%FT%TZ)
```

> `validate-logs` uses the signed digest files to prove the logs **weren't modified or deleted**. That's what auditors ask for as evidence.

### B6. AWS Config from the CLI

- [ ] Check compliance:

```bash
aws configservice describe-compliance-by-config-rule \
  --query 'ComplianceByConfigRules[].{Rule:ConfigRuleName,Compliance:Compliance.ComplianceType}' --output table

aws configservice get-compliance-details-by-config-rule --config-rule-name restricted-ssh \
  --query 'EvaluationResults[].{Resource:EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,Result:ComplianceType}'
```

- [ ] Add another managed rule:

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName":"ec2-imdsv2-check",
  "Source":{"Owner":"AWS","SourceIdentifier":"EC2_IMDSV2_CHECK"},
  "Scope":{"ComplianceResourceTypes":["AWS::EC2::Instance"]}}'
```

- [ ] Run an advanced query (SQL over resource configuration):

```bash
aws configservice select-resource-config --expression \
  "SELECT resourceId, resourceType, configuration.instanceType WHERE resourceType = 'AWS::EC2::Instance'"
```

---

## Verify

- [ ] Memory metrics and app logs from the CloudWatch agent are arriving.
- [ ] The metric filter alarm fired and you got an email.
- [ ] The CloudTrail trail is logging, and `validate-logs` reports valid files.
- [ ] Config detected and auto-remediated an open SSH security group.

## Exam tips

- **Memory, disk, and swap metrics** aren't in default EC2 metrics. You need the **CloudWatch agent**.
- **"Who deleted the S3 bucket?"** → **CloudTrail**. **"Was this SG compliant last Tuesday, and what changed?"** → **AWS Config**.
- **"Prove log integrity"** → CloudTrail **log file validation**. **"Protect logs from deletion"** → S3 bucket in a separate **log archive account** + **Object Lock** + SCPs.
- **"Org-wide trail"** → **organization trail** from the management or delegated admin account.
- **"Automatically fix noncompliant resources"** → **Config rules + SSM Automation remediation**.
- **"Alert on specific log text"** → **metric filter + alarm**. **"Alert when an API call happens"** → **EventBridge rule on CloudTrail events**.
- **"Query CloudTrail with SQL long-term"** → **CloudTrail Lake** or Athena over the trail's S3 logs.

## Challenge (optional)

- [ ] Deploy the **Operational Best Practices for CIS** Config **conformance pack** and review the compliance score.

## Cleanup

- [ ] Delete the alarms, dashboard, topic, and log groups:

```bash
aws cloudwatch delete-alarms --alarm-names saa-checkout-degraded
aws cloudwatch delete-alarms --alarm-names saa-backlog-high
# delete the console alarms (ErrorCount alarm, recovery alarm) and dashboard saa-ops
aws sns delete-topic --topic-arn $TOPIC
for G in saa-app-logs saa-system-logs saa-cloudtrail-logs; do aws logs delete-log-group --log-group-name $G; done
```

- [ ] Terminate the instance and delete the test SG:

```bash
aws ec2 terminate-instances --instance-ids $MON
aws ec2 wait instance-terminated --instance-ids $MON
aws ec2 delete-security-group --group-id $(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-bad-sg --query 'SecurityGroups[0].GroupId' --output text)
```

- [ ] **Decide** what to keep from CloudTrail and Config:
  - **Keep the CloudTrail trail.** The first copy of management events is free (you pay S3 storage only), and it's best practice.
  - **Stop the Config recorder** to avoid ongoing per-item charges:

```bash
aws configservice stop-configuration-recorder --configuration-recorder-name default
for R in restricted-ssh s3-bucket-public-read-prohibited encrypted-volumes ec2-imdsv2-check; do aws configservice delete-config-rule --config-rule-name $R; done
```

- [ ] Delete the parameter `AmazonCloudWatch-saa-config`, and detach `CloudWatchAgentServerPolicy` from `saa-ssm-ec2-role` if you want.

---

## Knowledge check

**Q1.** Security wants to know which IAM user terminated a production EC2 instance yesterday. Where do they look?

<details><summary>Answer</summary>

**AWS CloudTrail** (event history, or the trail's logs), filtered on `TerminateInstances`.

</details>

**Q2.** Operations needs an alert when EC2 memory use goes over 90%. What's required?

<details><summary>Answer</summary>

Install the **CloudWatch agent** to publish memory metrics, then create a **CloudWatch alarm** on `mem_used_percent`.

</details>

**Q3.** A company must continuously check that all EBS volumes are encrypted and automatically notify the owner of noncompliant volumes. What should it use?

<details><summary>Answer</summary>

The **AWS Config** managed rule `encrypted-volumes`, with an **EventBridge** rule on compliance changes → SNS (or a Config remediation action).

</details>
