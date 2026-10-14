# Lab 28 — Containers: ECR & ECS on Fargate

| | |
|---|---|
| **Module** | 7 — Decoupling, Serverless & Containers |
| **Exam mapping** | Domain 3 · Task 3.2 (elastic compute, containers) · Domain 2 · Task 2.1 (microservices, scalable architectures) · Domain 4 · Task 4.2 (Fargate vs EC2 launch types) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0.10–0.15/hour (2 Fargate tasks at 0.25 vCPU / 0.5 GB, plus an ALB) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-27-eventbridge-step-functions.md) · [Back to index](../README.md) · [Next lab ➡](lab-29-kms-secrets.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Build and push a container image to **Amazon ECR**, with image scanning and lifecycle policies.
- Define an **ECS task definition**, including the **task role** vs the **task execution role**.
- Run an **ECS service on Fargate** behind an **ALB**, with rolling deployments.
- Configure **ECS Service Auto Scaling**.
- Choose between **ECS vs EKS**, **Fargate vs EC2 launch types**, and **App Runner / Lambda**.

## Container service choices

| Need | Choose |
|---|---|
| Run containers without managing servers, AWS-native | **ECS on Fargate** |
| Need GPUs, specific instance types, or the lowest cost at steady high utilisation | **ECS on EC2** (capacity providers, Spot) |
| Already standardised on Kubernetes, or need portability | **EKS** (with Fargate or managed node groups) |
| Simplest web app/API from source or an image, fully managed | **App Runner** |
| Short event-driven functions | **Lambda** (container images up to 10 GB) |
| Run Kubernetes or ECS on-premises | **EKS Anywhere / ECS Anywhere** |

**Task execution role** = what the *ECS agent* needs (pull from ECR, write logs, read secrets).
**Task role** = what *your application code* needs (e.g. read S3). This is exam bait.

---

## Prerequisites

- `saa-vpc` (public subnets used for simplicity) and the ALB SG `saa-alb-sg` from Lab 18.
- **Docker** installed locally (Docker Desktop, Colima, or Finch). No Docker? Skip A1 and use `public.ecr.aws/nginx/nginx:stable` as the image.

---

## Part A — Console

### A1. Build and push an image to ECR

- [ ] Open **ECR → Private registry → Repositories → Create**: `saa-web`, **Scan on push** (basic scanning), tag immutability **Immutable**, encryption AES-256.
- [ ] Choose **View push commands** and follow them, using this app:

```bash
mkdir saa-web && cd saa-web
cat > index.html <<'EOF'
<h1>SAA ECS on Fargate — v1</h1>
EOF
cat > Dockerfile <<'EOF'
FROM public.ecr.aws/nginx/nginx:stable-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker build --platform linux/amd64 -t saa-web:v1 .
docker tag saa-web:v1 $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/saa-web:v1
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/saa-web:v1
```

- [ ] In ECR, open the image and look at its **vulnerability scan findings**.

### A2. Create an ECS cluster

- [ ] Open **ECS → Clusters → Create cluster**: `saa-ecs`, infrastructure **AWS Fargate (serverless)** only.

### A3. Task definition

- [ ] Open **Task definitions → Create new**:
  - Family `saa-web-task`, launch type **Fargate**, OS **Linux/X86_64**, CPU **0.25 vCPU**, memory **0.5 GB**
  - Task execution role: **Create new role** (`ecsTaskExecutionRole`)
  - Container: name `web`, image URI `<acct>.dkr.ecr.us-east-1.amazonaws.com/saa-web:v1`, port **80**
  - Logging: **awslogs** on (default)

### A4. Service behind an ALB

- [ ] Open cluster `saa-ecs` → **Services → Create**.
  - Compute: **Launch type Fargate**, family `saa-web-task`, service name `saa-web-svc`, desired tasks **2**
  - Networking: VPC `saa-vpc`, subnets **saa-public-a**, **saa-public-b**, new SG `saa-ecs-task-sg` (HTTP 80 **from `saa-alb-sg`**), **Public IP: ON** (so tasks can pull from ECR without NAT or endpoints)
  - Load balancing: **Application Load Balancer → Create new** `saa-ecs-alb`, listener HTTP 80, new target group `saa-ecs-tg` (**target type IP**, health check `/`)
  - Service auto scaling: **Use**, min 2, max 4, target tracking **ECSServiceAverageCPUUtilization 50%**
- [ ] When the tasks are **RUNNING** and the targets are **healthy**, browse to the ALB DNS name.

> **Production pattern:** tasks in **private subnets** with **public IP off**, pulling images through **ECR interface endpoints + the S3 gateway endpoint** (or NAT).

### A5. Rolling deployment

- [ ] Change `index.html` to `v2`, then build and push `saa-web:v2`.
- [ ] **Create a new task definition revision** that uses `:v2`.
- [ ] Choose **Update service** → the new revision → **Force new deployment**. In the **Deployments** tab, watch new tasks start, become healthy, and old ones drain.
- [ ] Refresh the browser and see v2.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
CLUSTER=saa-ecs
SVC=saa-web-svc
```

### B1. Inspect the cluster, service, and tasks

- [ ] Look at the service and its tasks:

```bash
aws ecs describe-services --cluster $CLUSTER --services $SVC \
  --query 'services[0].{Desired:desiredCount,Running:runningCount,TaskDef:taskDefinition,LB:loadBalancers[0].targetGroupArn,Deployments:deployments[].{Status:status,Rollout:rolloutState}}'

TASKS=$(aws ecs list-tasks --cluster $CLUSTER --service-name $SVC --query taskArns --output text)
aws ecs describe-tasks --cluster $CLUSTER --tasks $TASKS \
  --query 'tasks[].{Task:taskArn,AZ:availabilityZone,Status:lastStatus,Health:healthStatus,CPU:cpu,Mem:memory}' --output table
```

### B2. Register a task definition from JSON (with a task role)

- [ ] Create a **task role** that lets the app read S3:

```bash
cat > ecs-task-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-ecs-task-role --assume-role-policy-document file://ecs-task-trust.json
aws iam attach-role-policy --role-name saa-ecs-task-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

- [ ] Write a task definition that uses both roles:

```bash
cat > taskdef.json <<EOF
{
  "family": "saa-web-task",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256", "memory": "512",
  "executionRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/saa-ecs-task-role",
  "runtimePlatform": {"cpuArchitecture": "X86_64", "operatingSystemFamily": "LINUX"},
  "containerDefinitions": [{
    "name": "web",
    "image": "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/saa-web:v2",
    "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
    "essential": true,
    "environment": [{"name": "APP_ENV", "value": "lab"}],
    "logConfiguration": {"logDriver": "awslogs", "options": {
      "awslogs-group": "/ecs/saa-web-task", "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "web", "awslogs-create-group": "true"}}
  }]
}
EOF
```

- [ ] Register it and roll the service onto it:

```bash
TD_ARN=$(aws ecs register-task-definition --cli-input-json file://taskdef.json --query taskDefinition.taskDefinitionArn --output text)
aws ecs update-service --cluster $CLUSTER --service $SVC --task-definition $TD_ARN \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},maximumPercent=200,minimumHealthyPercent=100"
aws ecs wait services-stable --cluster $CLUSTER --services $SVC
```

> The **deployment circuit breaker** rolls back automatically if new tasks keep failing health checks.

### B3. Scale manually and check the auto-scaling policy

- [ ] Scale to 3 tasks:

```bash
aws ecs update-service --cluster $CLUSTER --service $SVC --desired-count 3
```

- [ ] Look at the scaling target and policy the console created:

```bash
aws application-autoscaling describe-scalable-targets --service-namespace ecs \
  --query 'ScalableTargets[].{Resource:ResourceId,Min:MinCapacity,Max:MaxCapacity}'
aws application-autoscaling describe-scaling-policies --service-namespace ecs \
  --query 'ScalingPolicies[].{Name:PolicyName,Metric:TargetTrackingScalingPolicyConfiguration.PredefinedMetricSpecification.PredefinedMetricType,Target:TargetTrackingScalingPolicyConfiguration.TargetValue}'
```

### B4. Run a one-off (batch) task

- [ ] Run a task that prints a message and exits:

```bash
TASK_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-ecs-task-sg --query 'SecurityGroups[0].GroupId' --output text)
aws ecs run-task --cluster $CLUSTER --launch-type FARGATE --task-definition saa-web-task \
  --network-configuration "awsvpcConfiguration={subnets=[$PUB_A],securityGroups=[$TASK_SG],assignPublicIp=ENABLED}" \
  --overrides '{"containerOverrides":[{"name":"web","command":["sh","-c","echo batch job ran on Fargate && sleep 5"]}]}' \
  --query 'tasks[0].taskArn' --output text
sleep 60
aws logs tail /ecs/saa-web-task --since 5m | grep "batch job"
```

> **EventBridge Scheduler** or **Step Functions** can launch tasks like this on a schedule or in a workflow, which is useful for jobs longer than Lambda's 15 minutes.

### B5. ECR lifecycle policy

- [ ] Keep only the last 5 images:

```bash
aws ecr put-lifecycle-policy --repository-name saa-web --lifecycle-policy-text '{
  "rules":[{"rulePriority":1,"description":"keep last 5","selection":{"tagStatus":"any","countType":"imageCountMoreThan","countNumber":5},"action":{"type":"expire"}}]}'
aws ecr describe-images --repository-name saa-web --query 'imageDetails[].{Tags:imageTags,Pushed:imagePushedAt,Scan:imageScanStatus.status}'
```

### B6. Fargate Spot capacity provider (cost optimisation)

- [ ] Add the capacity providers and a default strategy:

```bash
aws ecs put-cluster-capacity-providers --cluster $CLUSTER \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,base=1,weight=1 capacityProvider=FARGATE_SPOT,weight=3
```

> New services with no launch type use this strategy: 1 task always on Fargate, then 3:1 Spot to On-Demand. That can save up to ~70% on interruption-tolerant tasks.

---

## Verify

- [ ] The image is in ECR with scan results.
- [ ] The ECS service runs 2+ healthy tasks across two AZs behind the ALB.
- [ ] A rolling deployment moved the service from v1 to v2 with no downtime, and a one-off task ran and logged.

## Exam tips

- **"Run containers without managing EC2 instances"** → **Fargate**.
- **"Containers need AWS permissions"** → **task role** (not the instance role, not the execution role).
- **"Pass DB passwords to containers securely"** → **Secrets Manager or SSM Parameter Store** referenced in the task definition `secrets` (read by the **execution role**).
- **"Persistent shared storage for ECS/EKS tasks"** → **EFS** volumes.
- **"Kubernetes workloads on AWS, managed control plane"** → **EKS**. **"Scale EKS nodes fast and efficiently"** → **Karpenter** or the Cluster Autoscaler.
- ALB **target type IP** is required for Fargate (`awsvpc` network mode gives each task its own ENI).

## Challenge (optional)

- [ ] Turn on **ECS Exec** on the service and open a shell inside a running container with `aws ecs execute-command --interactive --command "/bin/sh"`. You'll need the Session Manager plugin, plus SSM permissions on the task role.

## Cleanup

- [ ] Scale the service to zero and delete it:

```bash
aws ecs update-service --cluster $CLUSTER --service $SVC --desired-count 0
aws ecs delete-service --cluster $CLUSTER --service $SVC --force
aws ecs wait services-inactive --cluster $CLUSTER --services $SVC
```

- [ ] Delete the cluster and the ALB the ECS wizard created:

```bash
aws ecs delete-cluster --cluster $CLUSTER
ALB=$(aws elbv2 describe-load-balancers --names saa-ecs-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 delete-load-balancer --load-balancer-arn $ALB
sleep 30
aws elbv2 delete-target-group --target-group-arn $(aws elbv2 describe-target-groups --names saa-ecs-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
```

- [ ] Deregister the task definitions:

```bash
for TD in $(aws ecs list-task-definitions --family-prefix saa-web-task --query taskDefinitionArns --output text); do aws ecs deregister-task-definition --task-definition $TD >/dev/null; done
```

- [ ] Delete the ECR repository, roles, and log group:

```bash
aws ecr delete-repository --repository-name saa-web --force
aws iam detach-role-policy --role-name saa-ecs-task-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name saa-ecs-task-role
aws logs delete-log-group --log-group-name /ecs/saa-web-task
```

- [ ] Delete SG `saa-ecs-task-sg`, and `rm -rf saa-web taskdef.json ecs-task-trust.json`.
- [ ] *(Optional)* Keep `ecsTaskExecutionRole`. It's commonly reused.

---

## Knowledge check

**Q1.** A containerised app on ECS Fargate needs to write to a DynamoDB table. Where do you grant this permission?

<details><summary>Answer</summary>

On the **ECS task role** (`taskRoleArn`) in the task definition.

</details>

**Q2.** A company runs Kubernetes on-premises and wants to move to AWS with minimal changes to its manifests and tooling. Which service?

<details><summary>Answer</summary>

**Amazon EKS.**

</details>

**Q3.** A batch processing container runs for 2 hours each night, and interruptions are fine because it checkpoints. What's the most cost-effective serverless option?

<details><summary>Answer</summary>

**ECS tasks on Fargate Spot**, triggered by **EventBridge Scheduler** (or AWS Batch on Fargate Spot).

</details>
