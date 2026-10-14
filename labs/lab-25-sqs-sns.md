# Lab 25 — Decoupling with SQS & SNS

| | |
|---|---|
| **Module** | 7 — Decoupling, Serverless & Containers |
| **Exam mapping** | Domain 2 · Task 2.1 (loosely coupled architectures) · Task 2.2 (fault tolerance with DLQs) · Domain 3 · Task 3.2 (queue-based scaling) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0 (the free tier covers 1M SQS requests and 1M SNS publishes) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-24-elasticache.md) · [Back to index](../README.md) · [Next lab ➡](lab-26-lambda-api-gateway.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Decouple producers and consumers with **SQS standard** and **FIFO** queues.
- Tune **visibility timeout**, **long polling**, **delay**, and **retention**.
- Isolate poison messages with **dead-letter queues (DLQs)** and **redrive** them.
- Build an **SNS → SQS fan-out** with **message filtering**.
- Pick between SQS, SNS, EventBridge, Kinesis, and Amazon MQ.

## SQS facts

| Setting | Default | Range / notes |
|---|---|---|
| Message retention | 4 days | 1 min – **14 days** |
| Max message size | 1 MiB | Larger payloads → S3 + Extended Client Library |
| Visibility timeout | 30 s | 0 s – 12 h. Set it **longer than processing time**. |
| Receive wait (long polling) | 0 s | Up to **20 s**. Cuts empty receives and cost. |
| Delivery delay | 0 s | Up to 15 min |
| Standard | Nearly unlimited throughput, **at-least-once**, best-effort ordering | |
| FIFO | **Exactly-once processing**, **strict order per message group**, 300 msg/s (3,000 with batching; much higher in high-throughput mode) | Name must end in `.fifo` |

## Choosing a messaging service

| Need | Service |
|---|---|
| Buffer work between components, consumers pull | **SQS** |
| Push the same message to many subscribers (email, SMS, HTTP, SQS, Lambda) | **SNS** |
| Route events by content to many AWS/SaaS targets, schedules, replay | **EventBridge** (Lab 27) |
| Ordered, replayable, real-time streaming at high throughput, many consumers | **Kinesis Data Streams** (Lab 35) |
| Lift-and-shift apps using JMS / AMQP / MQTT | **Amazon MQ** |

---

## Part A — Console

### A1. Standard queue with a DLQ

- [ ] Open **SQS → Create queue**: **Standard**, name `saa-orders-dlq`, retention **14 days**.
- [ ] Create a second queue, **Standard**, `saa-orders`:
  - Visibility timeout **30 s**, receive message wait time **20 s** (long polling)
  - **Dead-letter queue:** enabled → `saa-orders-dlq`, **Maximum receives 3**
- [ ] Choose **Send and receive messages**, send `{"orderId": 1}`, then **Poll for messages**.
- [ ] Receive the message but **don't delete it**. It disappears for 30 s (visibility timeout), then becomes visible again.
- [ ] Receive it **3 times** without deleting. After the third receive, it moves to `saa-orders-dlq`.
- [ ] Open `saa-orders-dlq` and choose **Start DLQ redrive** to send it back to the source queue.

### A2. FIFO queue

- [ ] Create a **FIFO** queue `saa-payments.fifo` with **content-based deduplication** on.
- [ ] Send 3 messages with **message group ID** `customer-42`: `pay-1`, `pay-2`, `pay-3`.
- [ ] Send `pay-1` **again** within 5 minutes. It's **deduplicated** (not enqueued twice).
- [ ] Poll. The messages arrive **in order**, and only once each.

### A3. SNS fan-out with filtering

- [ ] Open **SNS → Topics → Create topic**: **Standard**, `saa-order-events`.
- [ ] Create two SQS queues, `saa-shipping` and `saa-analytics`.
- [ ] On the topic, choose **Create subscription** twice:
  - Protocol **SQS** → `saa-shipping`, with this **subscription filter policy** (message attributes):

```json
{ "eventType": ["ORDER_PLACED"], "region": ["EU", "US"] }
```

  - Protocol **SQS** → `saa-analytics`, with no filter (receives everything)
- [ ] Add an **Email** subscription and confirm it from your inbox.
- [ ] Publish a message with attributes `eventType=ORDER_PLACED` and `region=US`. **All three subscribers** receive it.
- [ ] Publish with `eventType=ORDER_CANCELLED`. **Only analytics and email** receive it.

> When you subscribe a queue from the SNS console, it updates the SQS access policy for you. From the CLI you do that yourself (B4).

---

## Part B — CLI

### B1. Create queues with redrive policy

- [ ] Create the DLQ:

```bash
DLQ_URL=$(aws sqs create-queue --queue-name saa-cli-dlq --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)
```

- [ ] Create the work queue with long polling and the redrive policy:

```bash
Q_URL=$(aws sqs create-queue --queue-name saa-cli-work --attributes "{
  \"VisibilityTimeout\":\"20\",
  \"ReceiveMessageWaitTimeSeconds\":\"20\",
  \"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"2\\\"}\"
}" --query QueueUrl --output text)
```

### B2. Producer and consumer

- [ ] Producer: send 10 messages in a batch:

```bash
ENTRIES=$(for i in $(seq 1 10); do printf '{"Id":"m%d","MessageBody":"job-%d"},' $i $i; done | sed 's/,$//')
aws sqs send-message-batch --queue-url $Q_URL --entries "[$ENTRIES]" --query 'length(Successful)'
```

- [ ] Consumer: receive up to 10 at once, process, and delete:

```bash
MSGS=$(aws sqs receive-message --queue-url $Q_URL --max-number-of-messages 10 --wait-time-seconds 5)
echo "$MSGS" | jq -r '.Messages[].Body'
echo "$MSGS" | jq -c '[.Messages[] | {Id: .MessageId, ReceiptHandle: .ReceiptHandle}]' > del.json
aws sqs delete-message-batch --queue-url $Q_URL --entries file://del.json
```

### B3. Visibility timeout extension (long-running job)

- [ ] Send a job, receive it, and extend its visibility while "processing":

```bash
aws sqs send-message --queue-url $Q_URL --message-body "long-job"
RH=$(aws sqs receive-message --queue-url $Q_URL --query 'Messages[0].ReceiptHandle' --output text)
aws sqs change-message-visibility --queue-url $Q_URL --receipt-handle "$RH" --visibility-timeout 300
aws sqs get-queue-attributes --queue-url $Q_URL \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

> The message is now **in flight** (`NotVisible`) for 5 minutes. The consumer extends visibility while it's still working.

### B4. SNS fan-out from the CLI

- [ ] Create the topic, then create two queues. For each queue, allow the topic to send to it and subscribe it:

```bash
TOPIC=$(aws sns create-topic --name saa-cli-events --query TopicArn --output text)
for Q in saa-cli-email-svc saa-cli-audit; do
  URL=$(aws sqs create-queue --queue-name $Q --query QueueUrl --output text)
  ARN=$(aws sqs get-queue-attributes --queue-url $URL --attribute-names QueueArn --query Attributes.QueueArn --output text)
  POLICY=$(jq -n --arg q "$ARN" --arg t "$TOPIC" '{Version:"2012-10-17",Statement:[{Effect:"Allow",Principal:{Service:"sns.amazonaws.com"},Action:"sqs:SendMessage",Resource:$q,Condition:{ArnEquals:{"aws:SourceArn":$t}}}]}')
  aws sqs set-queue-attributes --queue-url $URL --attributes "$(jq -n --arg p "$POLICY" '{Policy:$p}')"
  aws sns subscribe --topic-arn $TOPIC --protocol sqs --notification-endpoint $ARN \
    --attributes RawMessageDelivery=true
done
```

- [ ] Add a payload-based filter to the email-service subscription. It filters on the message **body**, not attributes:

```bash
SUB=$(aws sns list-subscriptions-by-topic --topic-arn $TOPIC --query "Subscriptions[?contains(Endpoint,'email-svc')].SubscriptionArn" --output text)
aws sns set-subscription-attributes --subscription-arn $SUB --attribute-name FilterPolicyScope --attribute-value MessageBody
aws sns set-subscription-attributes --subscription-arn $SUB --attribute-name FilterPolicy \
  --attribute-value '{"type":["signup"]}'
```

- [ ] Publish two events and check each queue:

```bash
aws sns publish --topic-arn $TOPIC --message '{"type":"signup","user":"alice"}'
aws sns publish --topic-arn $TOPIC --message '{"type":"login","user":"bob"}'
sleep 3
for Q in saa-cli-email-svc saa-cli-audit; do
  URL=$(aws sqs get-queue-url --queue-name $Q --query QueueUrl --output text)
  echo "== $Q"; aws sqs receive-message --queue-url $URL --max-number-of-messages 10 --query 'Messages[].Body' --output text
done
```

> Expected: `email-svc` only gets the signup, and `audit` gets both.

### B5. FIFO from the CLI

- [ ] Create a FIFO queue and send messages in two groups:

```bash
F_URL=$(aws sqs create-queue --queue-name saa-cli.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true --query QueueUrl --output text)
for i in 1 2 3; do
  aws sqs send-message --queue-url $F_URL --message-body "step-$i" --message-group-id order-777 >/dev/null
done
aws sqs send-message --queue-url $F_URL --message-body "other" --message-group-id order-888 >/dev/null
aws sqs receive-message --queue-url $F_URL --max-number-of-messages 10 --attribute-names MessageGroupId \
  --query 'Messages[].{Body:Body,Group:Attributes.MessageGroupId}' --output table
```

### B6. Queue metrics for scaling

- [ ] Check the queue depth:

```bash
aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=saa-cli-work --statistics Maximum --period 60 \
  --start-time $(date -u -v-30M +%FT%TZ 2>/dev/null || date -u -d '30 min ago' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

> This metric (and **`ApproximateAgeOfOldestMessage`**) drives worker ASG scaling (Lab 19) and alarms.

---

## Verify

- [ ] A message moved to the DLQ after exceeding `maxReceiveCount`, and you redrove it.
- [ ] The FIFO queue kept order within a group and deduplicated a repeat.
- [ ] SNS filters sent different events to different queues.

## Exam tips

- **"Decouple the web tier from processing so spikes don't drop orders"** → **SQS** between them, with an ASG of workers scaled on queue depth.
- **"Messages processed more than once"** → visibility timeout shorter than processing time. Increase it, or make consumers **idempotent**.
- **"Order matters and no duplicates"** → **SQS FIFO**.
- **"One event must trigger several independent processes"** → **SNS fan-out to multiple SQS queues** (or EventBridge).
- **"Reduce SQS cost / empty responses"** → **long polling** (`WaitTimeSeconds` up to 20).
- **"Messages that keep failing block processing"** → **DLQ**.
- **SQS is pull. SNS is push.** SNS doesn't store messages; pair it with SQS for durability.

## Challenge (optional)

- [ ] Encrypt `saa-orders` with **SSE-KMS** using a customer-managed key, and update the SNS topic's KMS permissions so fan-out still works (`kms:GenerateDataKey` and `kms:Decrypt` for `sns.amazonaws.com`).

## Cleanup

- [ ] Delete the queues:

```bash
for Q in saa-cli-dlq saa-cli-work saa-cli-email-svc saa-cli-audit saa-cli.fifo saa-orders saa-orders-dlq saa-payments.fifo saa-shipping saa-analytics; do
  URL=$(aws sqs get-queue-url --queue-name $Q --query QueueUrl --output text 2>/dev/null) && aws sqs delete-queue --queue-url $URL
done
```

- [ ] Delete the topics:

```bash
aws sns delete-topic --topic-arn $TOPIC
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:$ACCOUNT_ID:saa-order-events
```

- [ ] `rm -f del.json`

---

## Knowledge check

**Q1.** An order-processing system sometimes processes the same order twice. Consumers take about 60 s per message, and the queue's visibility timeout is 30 s. What should you do?

<details><summary>Answer</summary>

Increase the **visibility timeout** above the processing time (e.g. 2–3 minutes) or have consumers call `ChangeMessageVisibility`. Making processing idempotent also helps. If strict exactly-once processing is needed, use **FIFO**.

</details>

**Q2.** When an image is uploaded, a company must create a thumbnail, update a search index, and notify an analytics system, all independently and in parallel. What's the most decoupled design?

<details><summary>Answer</summary>

Send the S3 event to an **SNS topic** (or EventBridge) that **fans out to three SQS queues**, each with its own consumer.

</details>

**Q3.** A bank processes transactions per account and order must be strictly kept for each account, while different accounts can be processed in parallel. What should it use?

<details><summary>Answer</summary>

**SQS FIFO** with the **message group ID = account ID**.

</details>
