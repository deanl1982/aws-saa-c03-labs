# Lab 27 — Event-Driven Orchestration: EventBridge & Step Functions

| | |
|---|---|
| **Module** | 7 — Decoupling, Serverless & Containers |
| **Exam mapping** | Domain 2 · Task 2.1 (event-driven, loosely coupled architectures; workflow orchestration) · Task 2.2 (retries and error handling) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0 (free tier: 4,000 Step Functions state transitions; EventBridge custom events are $1/million) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-26-lambda-api-gateway.md) · [Back to index](../README.md) · [Next lab ➡](lab-28-ecs-fargate.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Build a **Step Functions** workflow with **Choice**, **Wait**, **Task** (direct SDK integrations), **Retry**, and **Catch**.
- Compare **Standard** and **Express** workflows.
- Route events with **EventBridge** custom buses and content-based **event patterns**.
- **Archive and replay** events, and run jobs with **EventBridge Scheduler**.
- Choose between choreography (EventBridge/SNS) and orchestration (Step Functions).

## Standard vs Express workflows

| | Standard | Express |
|---|---|---|
| Max duration | **1 year** | 5 minutes |
| Execution semantics | Exactly-once | At-least-once (async) / at-most-once (sync) |
| Pricing | Per state transition | Per request + duration (cheaper at high volume) |
| History | 90 days in the console | CloudWatch Logs |
| Use | Long-running business processes, human approval | High-volume event processing, IoT, streaming |

## Workflow you'll build

```
 Start → Validate (Choice: amount > 0?) ──no──► Rejected (Fail)
              │ yes
              ▼
        SaveOrder (DynamoDB PutItem, Retry ×3, Catch → SaveFailed)
              ▼
        HighValue? (Choice: amount > 1000?) ──yes──► ReviewWait (Wait 10 s)
              │ no                                         │
              ▼                                            ▼
        Notify (SNS Publish) ◄─────────────────────────────┘ → End
```

---

## Prerequisites (CLI)

- [ ] Create the DynamoDB table and SNS topic the workflow uses:

```bash
aws dynamodb create-table --table-name saa-sfn-orders \
  --attribute-definitions AttributeName=orderId,AttributeType=S \
  --key-schema AttributeName=orderId,KeyType=HASH --billing-mode PAY_PER_REQUEST
TOPIC=$(aws sns create-topic --name saa-sfn-notify --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC --protocol email --notification-endpoint you@example.com
```

- [ ] Confirm the SNS email subscription.

---

## Part A — Console: Step Functions

### A1. Create the state machine

- [ ] Open **Step Functions → State machines → Create state machine → Blank**, type **Standard**.
- [ ] Switch to the **Code** view and paste the definition below, replacing `<TOPIC_ARN>`:

```json
{
  "Comment": "SAA order workflow",
  "StartAt": "Validate",
  "States": {
    "Validate": {
      "Type": "Choice",
      "Choices": [{ "Variable": "$.amount", "NumericGreaterThan": 0, "Next": "SaveOrder" }],
      "Default": "Rejected"
    },
    "SaveOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "saa-sfn-orders",
        "Item": {
          "orderId": { "S.$": "$.orderId" },
          "amount":  { "N.$": "States.Format('{}', $.amount)" }
        }
      },
      "ResultPath": null,
      "Retry": [{
        "ErrorEquals": ["DynamoDB.ProvisionedThroughputExceededException", "States.TaskFailed"],
        "IntervalSeconds": 2, "MaxAttempts": 3, "BackoffRate": 2.0
      }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "ResultPath": "$.error", "Next": "SaveFailed" }],
      "Next": "HighValue?"
    },
    "HighValue?": {
      "Type": "Choice",
      "Choices": [{ "Variable": "$.amount", "NumericGreaterThan": 1000, "Next": "ReviewWait" }],
      "Default": "Notify"
    },
    "ReviewWait": { "Type": "Wait", "Seconds": 10, "Next": "Notify" },
    "Notify": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "<TOPIC_ARN>",
        "Message.$": "States.Format('Order {} for {} confirmed', $.orderId, $.amount)"
      },
      "End": true
    },
    "Rejected":   { "Type": "Fail", "Error": "InvalidAmount", "Cause": "amount must be > 0" },
    "SaveFailed": { "Type": "Fail", "Error": "SaveFailed",    "Cause": "DynamoDB write failed" }
  }
}
```

- [ ] Switch to the **Design** view to see the graph.
- [ ] **Config:** name `saa-order-workflow`, execution role **Create new role** (the console adds DynamoDB and SNS permissions from the definition), logging **ALL** to a new log group.
- [ ] Create it.

### A2. Run executions

- [ ] **Start execution** with `{"orderId": "o-1", "amount": 50}`. Watch it go through `Validate → SaveOrder → HighValue? → Notify`. You get an email.
- [ ] Start one with `{"orderId": "o-2", "amount": 5000}`. It takes the **ReviewWait** path.
- [ ] Start one with `{"orderId": "o-3", "amount": -1}`. It ends in **Rejected** (red).
- [ ] Open the **Events** table of an execution and look at the input and output of each state.

### A3. Show Retry and Catch

- [ ] Edit the state machine, change `TableName` to `does-not-exist`, and save.
- [ ] Run `{"orderId":"o-4","amount":10}`. `SaveOrder` **retries 3 times** with backoff, then the **Catch** sends it to `SaveFailed`. Look at `$.error` in the state input.
- [ ] Change the table name back.

---

## Part B — CLI: EventBridge routing, archive, replay, and Scheduler

```bash
SFN_ARN=$(aws stepfunctions list-state-machines --query "stateMachines[?name=='saa-order-workflow'].stateMachineArn" --output text)
```

### B1. Start an execution from the CLI

- [ ] Start one and check the result:

```bash
EXEC=$(aws stepfunctions start-execution --state-machine-arn $SFN_ARN \
  --input '{"orderId":"cli-1","amount":250}' --query executionArn --output text)
sleep 5
aws stepfunctions describe-execution --execution-arn $EXEC --query '{Status:status,Output:output}'
```

### B2. Custom event bus and rule → Step Functions

- [ ] Create the bus:

```bash
aws events create-event-bus --name saa-orders-bus
```

- [ ] Create a role EventBridge can use to start executions:

```bash
cat > eb-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"events.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-eb-to-sfn --assume-role-policy-document file://eb-trust.json
aws iam put-role-policy --role-name saa-eb-to-sfn --policy-name start --policy-document \
  "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"states:StartExecution\",\"Resource\":\"$SFN_ARN\"}]}"
```

- [ ] Create a rule that matches only orders of 100 or more from the shop, using content filtering:

```bash
aws events put-rule --name saa-big-orders --event-bus-name saa-orders-bus --event-pattern '{
  "source": ["com.saa.shop"],
  "detail-type": ["OrderPlaced"],
  "detail": { "amount": [{ "numeric": [">=", 100] }] }
}'
```

- [ ] Target the state machine, passing only `detail` as input:

```bash
sleep 10
aws events put-targets --rule saa-big-orders --event-bus-name saa-orders-bus --targets "[{
  \"Id\":\"sfn\",\"Arn\":\"$SFN_ARN\",
  \"RoleArn\":\"arn:aws:iam::$ACCOUNT_ID:role/saa-eb-to-sfn\",
  \"InputPath\":\"$.detail\"}]"
```

### B3. Publish events

- [ ] Send one small and one big order:

```bash
aws events put-events --entries '[
  {"Source":"com.saa.shop","DetailType":"OrderPlaced","EventBusName":"saa-orders-bus","Detail":"{\"orderId\":\"eb-small\",\"amount\":20}"},
  {"Source":"com.saa.shop","DetailType":"OrderPlaced","EventBusName":"saa-orders-bus","Detail":"{\"orderId\":\"eb-big\",\"amount\":2500}"}
]'
sleep 15
aws stepfunctions list-executions --state-machine-arn $SFN_ARN --max-results 5 \
  --query 'executions[].{Name:name,Status:status,Start:startDate}' --output table
```

> Only `eb-big` started an execution. The pattern filtered out the $20 order.

### B4. Test an event pattern without sending events

- [ ] Check a sample event against the pattern:

```bash
aws events test-event-pattern --event-pattern '{"detail":{"amount":[{"numeric":[">=",100]}]}}' \
  --event '{"id":"1","account":"111122223333","source":"x","time":"2026-01-01T00:00:00Z","region":"us-east-1","resources":[],"detail-type":"t","detail":{"amount":150}}'
```

### B5. Archive and replay

- [ ] Create an archive of the bus:

```bash
BUS_ARN=$(aws events describe-event-bus --name saa-orders-bus --query Arn --output text)
aws events create-archive --archive-name saa-orders-archive --event-source-arn $BUS_ARN --retention-days 7
```

- [ ] Send an event that will be archived:

```bash
aws events put-events --entries '[{"Source":"com.saa.shop","DetailType":"OrderPlaced","EventBusName":"saa-orders-bus","Detail":"{\"orderId\":\"replay-me\",\"amount\":999}"}]'
```

- [ ] After a few minutes (archiving isn't instant), replay the last hour back onto the bus:

```bash
ARCHIVE_ARN=$(aws events describe-archive --archive-name saa-orders-archive --query ArchiveArn --output text)
aws events start-replay --replay-name saa-replay-$(date +%s) --event-source-arn $ARCHIVE_ARN \
  --event-start-time $(date -u -v-1H +%FT%TZ 2>/dev/null || date -u -d '1 hour ago' +%FT%TZ) \
  --event-end-time $(date -u +%FT%TZ) \
  --destination "{\"Arn\":\"$BUS_ARN\"}"
```

> Replay lets you **reprocess events** after fixing a bug in a consumer.

### B6. EventBridge Scheduler (the cron replacement)

- [ ] Create a role the scheduler can use:

```bash
cat > sched-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"scheduler.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-scheduler-role --assume-role-policy-document file://sched-trust.json
aws iam put-role-policy --role-name saa-scheduler-role --policy-name start --policy-document \
  "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"states:StartExecution\",\"Resource\":\"$SFN_ARN\"}]}"
sleep 10
```

- [ ] Run the workflow every 5 minutes with a fixed input:

```bash
aws scheduler create-schedule --name saa-every-5-min \
  --schedule-expression "rate(5 minutes)" --flexible-time-window Mode=OFF \
  --target "{\"Arn\":\"$SFN_ARN\",\"RoleArn\":\"arn:aws:iam::$ACCOUNT_ID:role/saa-scheduler-role\",\"Input\":\"{\\\"orderId\\\":\\\"scheduled\\\",\\\"amount\\\":1}\"}"
```

> Scheduler supports one-time schedules (`at(2026-12-31T23:59:00)`), cron with **time zones**, and 270+ AWS API targets.

### B7. S3 → EventBridge (default bus)

- [ ] If you turned on **EventBridge notifications** on a bucket in Lab 10, create a rule on the **default** bus for:

```json
{ "source": ["aws.s3"], "detail-type": ["Object Created"], "detail": { "bucket": { "name": ["<bucket>"] }, "object": { "key": [{ "suffix": ".csv" }] } } }
```

---

## Verify

- [ ] Executions took the expected paths: success, high value, rejected, and caught error.
- [ ] EventBridge content filtering started executions only for orders of 100 or more.
- [ ] The archive and replay re-delivered events, and the schedule triggers every 5 minutes.

## Exam tips

- **"Coordinate several steps with retries, branching, and human approval, minimal code"** → **Step Functions** (Standard). **High-volume, short** → **Express**.
- **"Route events from SaaS partners (Zendesk, Datadog…) or AWS services by content"** → **EventBridge** (partner event buses).
- **"Run a task on a schedule without a server"** → **EventBridge Scheduler** (or an EventBridge rule with a schedule).
- **"Reprocess past events after fixing a bug"** → **EventBridge archive and replay**.
- **"React to AWS API calls"** (e.g. a security group changed) → EventBridge rule on **CloudTrail** events.
- **Step Functions direct SDK integrations** (DynamoDB, SNS, SQS, ECS, Glue, SageMaker, Bedrock…) remove the need for "glue" Lambda functions.
- **EventBridge Pipes:** point-to-point source → filter → enrich → target (e.g. SQS → Step Functions).

## Challenge (optional)

- [ ] Add a **`.waitForTaskToken`** human-approval step that sends a token through SNS email and waits for `SendTaskSuccess`.
- [ ] Add a **Map** state that processes an array of order items in parallel.

## Cleanup

- [ ] Delete the schedule, rule, archive, and bus:

```bash
aws scheduler delete-schedule --name saa-every-5-min
aws events remove-targets --rule saa-big-orders --event-bus-name saa-orders-bus --ids sfn
aws events delete-rule --name saa-big-orders --event-bus-name saa-orders-bus
aws events delete-archive --archive-name saa-orders-archive
aws events delete-event-bus --name saa-orders-bus
```

- [ ] Delete the state machine, table, and topic:

```bash
aws stepfunctions delete-state-machine --state-machine-arn $SFN_ARN
aws dynamodb delete-table --table-name saa-sfn-orders
aws sns delete-topic --topic-arn $TOPIC
```

- [ ] Delete the IAM roles:

```bash
aws iam delete-role-policy --role-name saa-eb-to-sfn --policy-name start && aws iam delete-role --role-name saa-eb-to-sfn
aws iam delete-role-policy --role-name saa-scheduler-role --policy-name start && aws iam delete-role --role-name saa-scheduler-role
```

- [ ] Delete the console-created Step Functions role and log group, and `rm eb-trust.json sched-trust.json`.

---

## Knowledge check

**Q1.** An order process has 6 steps, including payment, inventory, and shipping. Some steps need retries with exponential backoff, and one needs a manager's approval that can take up to 3 days. What should you use?

<details><summary>Answer</summary>

An **AWS Step Functions Standard** workflow with **Retry/Catch**, using a **`.waitForTaskToken`** callback for the approval.

</details>

**Q2.** A company wants to trigger different Lambda functions depending on fields inside events from its SaaS helpdesk platform, without writing polling code. What should it use?

<details><summary>Answer</summary>

An **Amazon EventBridge partner event source / event bus** with **rules that use content-based event patterns** to route to each Lambda function.

</details>

**Q3.** A bug in a consumer meant 6 hours of events were processed incorrectly. How can the team reprocess them after deploying a fix?

<details><summary>Answer</summary>

**EventBridge archive and replay**, replaying the archived events for that time window to the bus.

</details>
