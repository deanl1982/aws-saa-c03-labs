# Lab 19 — EC2 Auto Scaling

| | |
|---|---|
| **Module** | 5 — Compute, Storage Volumes, Load Balancing & Scaling |
| **Exam mapping** | Domain 3 · Task 3.2 (elastic compute) · Domain 2 · Task 2.2 (self-healing, multi-AZ) · Task 2.1 (scalable architectures) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0.15/hour (ALB + 2–6 t3.micro) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-18-elastic-load-balancing.md) · [Back to index](../README.md) · [Next lab ➡](lab-20-ec2-pricing-placement.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Build an **Auto Scaling group (ASG)** from a **launch template** across multiple AZs behind an ALB.
- Configure **target tracking**, **step**, **scheduled**, and **predictive** scaling.
- Explain **health check types** (EC2 vs ELB) and the **health check grace period**.
- Watch **self-healing**, **AZ rebalancing**, and **termination policies** in action.
- Roll out new AMIs or configs with **instance refresh**, and know where **lifecycle hooks** and **warm pools** fit.

## Scaling policy cheat sheet

| Policy | How it works | Use when |
|---|---|---|
| **Target tracking** | Keeps a metric at a target (e.g. CPU 50%, `ALBRequestCountPerTarget` 100) | The default, simplest choice |
| **Step scaling** | CloudWatch alarm thresholds → add/remove N instances per step | Different responses to different breach sizes |
| **Simple scaling** | One alarm → one action, then cooldown | Legacy |
| **Scheduled** | Changes min/max/desired at set times | Predictable patterns (business hours, Monday 9am) |
| **Predictive** | ML forecast from 14 days of history, scales **ahead of** demand | Recurring cyclical load |

---

## Prerequisites

- Lab 18's `saa-alb`, `saa-tg-web`, `saa-alb-sg`, and `saa-target-sg` still exist. If not, recreate them using Lab 18 Part A, steps A1, A3, and A4.

---

## Part A — Console

### A1. Launch template for the ASG

- [ ] Open **EC2 → Launch templates → Create launch template**.
  - Name `saa-asg-lt`, tick **Provide guidance for Auto Scaling**
  - AMI AL2023, `t3.micro`, no key pair, SG `saa-target-sg`, IMDSv2 required
  - **Advanced → Detailed CloudWatch monitoring: Enable** (1-minute metrics give faster scaling)
  - User data (a simplified version of the Lab 18 script):

```bash
#!/bin/bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
mkdir -p /srv/www && echo "<h1>ASG instance $ID ($AZ)</h1>" > /srv/www/index.html
cd /srv/www && nohup python3 -m http.server 80 >/var/log/web.log 2>&1 &
```

### A2. Create the ASG

- [ ] Open **EC2 → Auto Scaling groups → Create**.
  - Name `saa-asg`, launch template `saa-asg-lt` (Latest)
  - VPC `saa-vpc`, subnets **saa-app-a** and **saa-app-b**
  - **Attach to existing load balancer** → target group `saa-tg-web`
  - Health checks: turn on **Elastic Load Balancing health checks**, grace period **60 s**
  - Group size: desired **2**, min **2**, max **6**
  - Scaling: **Target tracking**, metric **Application Load Balancer request count per target**, target **50**, instance warmup **60 s**
  - Add tag `Name=saa-asg-instance`
- [ ] Wait until both instances are **InService** and healthy in `saa-tg-web`.
- [ ] Browse to the ALB DNS name and refresh. You'll see two instances in two AZs.

### A3. Self-healing

- [ ] **Terminate** one ASG instance from the EC2 console.
- [ ] Watch **ASG → Activity**. Within about a minute, the ASG launches a replacement to get back to the desired capacity of 2.

### A4. Trigger scale-out with load

- [ ] From your laptop, send sustained traffic for about 5 minutes:

```bash
ALB_DNS=<your-alb-dns>
end=$((SECONDS+300)); while [ $SECONDS -lt $end ]; do
  seq 1 200 | xargs -P 20 -I{} curl -s -o /dev/null http://$ALB_DNS/
done
```

- [ ] Watch **ASG → Monitoring** and **Activity**. The target-tracking alarm (`TargetTracking-saa-asg-AlarmHigh-...`) fires and the desired capacity goes up.
- [ ] Stop the load. After about 15 minutes, the **AlarmLow** alarm scales back in. Scale-in is deliberately slower than scale-out.

### A5. Scheduled action

- [ ] Open **ASG → Automatic scaling → Scheduled actions → Create**: name `business-hours`, desired 3, min 3, max 6, recurrence `0 8 * * MON-FRI` (time zone of your choice).
- [ ] Add `after-hours`: desired 2, min 2, recurrence `0 18 * * MON-FRI`.

### A6. Instance refresh

- [ ] Create a **new version** of `saa-asg-lt` that changes the user data heading to `<h1>v2 ...`.
- [ ] Choose **ASG → Instance refresh → Start**: minimum healthy **50%**, skip matching off. Watch instances get replaced in a rolling update.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
ASG=saa-asg
```

### B1. Inspect the ASG

- [ ] Look at its configuration and instances:

```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG \
  --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,HC:HealthCheckType,Grace:HealthCheckGracePeriod,AZs:AvailabilityZones,TP:TerminationPolicies}'

aws autoscaling describe-auto-scaling-instances \
  --query "AutoScalingInstances[?AutoScalingGroupName=='$ASG'].{Id:InstanceId,AZ:AvailabilityZone,State:LifecycleState,Health:HealthStatus,LT:LaunchTemplate.Version}" --output table
```

### B2. Add a CPU target-tracking policy

- [ ] Add the policy:

```bash
aws autoscaling put-scaling-policy --auto-scaling-group-name $ASG --policy-name cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},
    "TargetValue":50.0,
    "DisableScaleIn":false
  }'
aws autoscaling describe-policies --auto-scaling-group-name $ASG --query 'ScalingPolicies[].{Name:PolicyName,Type:PolicyType}' --output table
```

> If several policies are in place, the ASG picks the one giving the **largest capacity** when scaling out, and scales in only when **every** policy agrees.

### B3. Step scaling with a custom alarm

- [ ] Create the step policy:

```bash
STEP_ARN=$(aws autoscaling put-scaling-policy --auto-scaling-group-name $ASG --policy-name step-out \
  --policy-type StepScaling --adjustment-type ChangeInCapacity --metric-aggregation-type Average \
  --step-adjustments MetricIntervalLowerBound=0,MetricIntervalUpperBound=20,ScalingAdjustment=1 \
                     MetricIntervalLowerBound=20,ScalingAdjustment=3 \
  --query PolicyARN --output text)
```

- [ ] Create the alarm that triggers it:

```bash
aws cloudwatch put-metric-alarm --alarm-name saa-asg-cpu-high \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=$ASG \
  --statistic Average --period 60 --evaluation-periods 2 \
  --threshold 70 --comparison-operator GreaterThanThreshold \
  --alarm-actions $STEP_ARN
```

> With a threshold of 70: CPU between 70 and 90 → +1 instance. CPU of 90 or more → +3 instances.

### B4. Manual capacity changes and standby

- [ ] Change the desired capacity:

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name $ASG --desired-capacity 4
```

- [ ] Put an instance in **Standby** for troubleshooting. It's removed from the load balancer without being terminated:

```bash
IID=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)
aws autoscaling enter-standby --auto-scaling-group-name $ASG --instance-ids $IID --should-decrement-desired-capacity
aws autoscaling exit-standby  --auto-scaling-group-name $ASG --instance-ids $IID
```

### B5. Mark an instance unhealthy (simulate a failure)

- [ ] Flag it and watch the replacement:

```bash
aws autoscaling set-instance-health --instance-id $IID --health-status Unhealthy
aws autoscaling describe-scaling-activities --auto-scaling-group-name $ASG --max-items 3 \
  --query 'Activities[].{Status:StatusCode,Cause:Cause}' --output table
```

### B6. Lifecycle hook

- [ ] Pause new instances in `Pending:Wait` for up to 5 minutes, e.g. to run configuration before they take traffic:

```bash
aws autoscaling put-lifecycle-hook --auto-scaling-group-name $ASG --lifecycle-hook-name launch-hook \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING --heartbeat-timeout 300 --default-result CONTINUE
```

- [ ] Scale out by one and watch the state:

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name $ASG --desired-capacity 5
sleep 30
aws autoscaling describe-auto-scaling-instances --query "AutoScalingInstances[?LifecycleState=='Pending:Wait'].InstanceId"
```

- [ ] Complete the hook for that instance:

```bash
NEW=$(aws autoscaling describe-auto-scaling-instances --query "AutoScalingInstances[?LifecycleState=='Pending:Wait'].InstanceId" --output text)
aws autoscaling complete-lifecycle-action --auto-scaling-group-name $ASG --lifecycle-hook-name launch-hook \
  --lifecycle-action-result CONTINUE --instance-id $NEW
aws autoscaling delete-lifecycle-hook --auto-scaling-group-name $ASG --lifecycle-hook-name launch-hook
```

### B7. Warm pool (concept, plus a quick config)

- [ ] Add a warm pool of stopped instances. They're pre-initialised, so scale-out is fast, and stopped instances only cost EBS storage:

```bash
aws autoscaling put-warm-pool --auto-scaling-group-name $ASG --min-size 1 --pool-state Stopped
aws autoscaling describe-warm-pool --auto-scaling-group-name $ASG --query 'Instances[].{Id:InstanceId,State:LifecycleState}'
aws autoscaling delete-warm-pool --auto-scaling-group-name $ASG --force-delete
```

### B8. Turn on predictive scaling in forecast-only mode

- [ ] Add the policy:

```bash
aws autoscaling put-scaling-policy --auto-scaling-group-name $ASG --policy-name predictive \
  --policy-type PredictiveScaling --predictive-scaling-configuration '{
    "MetricSpecifications":[{"TargetValue":50,"PredefinedMetricPairSpecification":{"PredefinedMetricType":"ASGCPUUtilization"}}],
    "Mode":"ForecastOnly"}'
```

---

## Verify

- [ ] The ASG replaced a terminated or unhealthy instance automatically.
- [ ] Load made the ASG scale out, and it scaled in afterwards.
- [ ] Instance refresh rolled out a new launch template version with no downtime.

## Exam tips

- **ELB health checks** on the ASG replace instances whose *application* fails, not just ones whose EC2 status check fails.
- **Default termination policy:** balance AZs first, then terminate the instance with the **oldest launch template/configuration**, then the one closest to the next billing hour.
- **Scale on queue depth:** target tracking on a custom metric like **SQS `ApproximateNumberOfMessagesVisible` / number of instances** (backlog per instance).
- **Predictable spikes** → **scheduled** or **predictive** scaling. **Unpredictable spikes** → target tracking.
- **Slow-booting app** → **warm pools** or a golden AMI.
- To scale stateful workloads cleanly, keep sessions out of the instances.

## Challenge (optional)

- [ ] Create a custom CloudWatch metric `BacklogPerInstance` from an SQS queue and drive the ASG with a target-tracking policy on it (a preview of Lab 25).

## Cleanup

- [ ] Delete the ASG (this terminates its instances):

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name $ASG --force-delete
aws cloudwatch delete-alarms --alarm-names saa-asg-cpu-high
```

- [ ] Delete the ALB, target groups, and log bucket (if they're still there from Lab 18):

```bash
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 30
for TG in saa-tg-web saa-tg-api; do
  aws elbv2 delete-target-group --target-group-arn $(aws elbv2 describe-target-groups --names $TG --query 'TargetGroups[0].TargetGroupArn' --output text)
done
aws s3 rb s3://saa-lab18-alblogs-$SUFFIX --force 2>/dev/null
```

- [ ] **Keep** the launch templates and SGs (free).

---

## Knowledge check

**Q1.** An ASG's instances keep being marked healthy even though the web application returns HTTP 500 errors. What should you change?

<details><summary>Answer</summary>

Turn on **ELB health checks** for the ASG (health check type `ELB`) so instances that fail the target group's health check get replaced.

</details>

**Q2.** Traffic triples every weekday at 9:00 AM and new instances need 10 minutes to become ready. Users see slow responses for the first 10 minutes. What should you do?

<details><summary>Answer</summary>

Add a **scheduled scaling action** (or **predictive scaling**) that scales out before 9:00 AM. Optionally use a **warm pool** to shorten instance readiness.

</details>

**Q3.** Worker instances process jobs from an SQS queue. How should the ASG scale?

<details><summary>Answer</summary>

Use **target tracking on a backlog-per-instance metric** (`ApproximateNumberOfMessagesVisible` ÷ running instances) published as a custom CloudWatch metric.

</details>
