# Lab 26 — Serverless APIs with Lambda & API Gateway

| | |
|---|---|
| **Module** | 7 — Decoupling, Serverless & Containers |
| **Exam mapping** | Domain 2 · Task 2.1 (serverless, scalable architectures) · Domain 3 · Task 3.2 (Lambda concurrency and memory) · Domain 1 · Task 1.2 (API security) · Domain 4 · Task 4.2 (pay-per-use) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0 (within the free tier for Lambda, API Gateway, and DynamoDB) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-25-sqs-sns.md) · [Back to index](../README.md) · [Next lab ➡](lab-27-eventbridge-step-functions.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Build a **Lambda** function with a least-privilege **execution role**, environment variables, and the right memory and timeout settings.
- Expose it through **API Gateway** (REST and HTTP APIs) and explain the differences.
- Protect APIs with **throttling, usage plans/API keys**, and authorizers (**Cognito**, Lambda, IAM).
- Control scaling with **reserved** and **provisioned concurrency**.
- Process **SQS** messages with an event source mapping (batching, partial batch failures).

## Lambda facts

| Item | Value |
|---|---|
| Memory | 128 MB – 10,240 MB (**CPU scales with memory**) |
| Timeout | Up to **15 minutes** (anything longer → Step Functions, ECS/Fargate, Batch) |
| Ephemeral `/tmp` | 512 MB – 10 GB |
| Deployment package | 50 MB zipped / 250 MB unzipped, or a **container image up to 10 GB** |
| Default concurrency | 1,000 per region (soft limit), shared by all functions |
| Cold starts | Mitigate with **provisioned concurrency** or **SnapStart** (Java, Python, .NET) |

## API Gateway types

| | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| Cost / latency | Higher | **~70% cheaper, lower latency** | — |
| Features | **API keys and usage plans, caching, request validation, WAF, private endpoints, edge-optimised** | JWT/Cognito auth, Lambda/HTTP integrations | Two-way real-time |
| Exam trigger | "Usage plans", "cache responses", "WAF on the API" | "Simple, cheap proxy to Lambda" | Chat, live dashboards |

---

## Part A — Console

### A1. DynamoDB table

- [ ] Create table `saa-items` with partition key `id` (String), on-demand.

### A2. Lambda function

- [ ] Open **Lambda → Create function → Author from scratch**: name `saa-items-api`, runtime **Python 3.13**, architecture **arm64** (Graviton is cheaper), execution role **Create a new role with basic Lambda permissions**.
- [ ] Replace the code and **Deploy**:

```python
import json, os, uuid, boto3
table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])

def lambda_handler(event, context):
    method = event.get("httpMethod") or event.get("requestContext", {}).get("http", {}).get("method")
    if method == "POST":
        body = json.loads(event.get("body") or "{}")
        item = {"id": str(uuid.uuid4()), "name": body.get("name", "unnamed")}
        table.put_item(Item=item)
        return {"statusCode": 201, "body": json.dumps(item)}
    if method == "GET":
        items = table.scan(Limit=25).get("Items", [])
        return {"statusCode": 200, "body": json.dumps(items)}
    return {"statusCode": 405, "body": "method not allowed"}
```

- [ ] Open **Configuration → Environment variables** and add `TABLE_NAME = saa-items`.
- [ ] Open **Configuration → General**: memory **256 MB**, timeout **10 s**.
- [ ] Open **Configuration → Permissions → role name**, which takes you to IAM. Add an inline policy that allows `dynamodb:PutItem` and `dynamodb:Scan` on **only** the `saa-items` table ARN.

### A3. REST API with Lambda proxy integration

- [ ] Open **API Gateway → Create API → REST API → Build**: name `saa-items-rest`, endpoint **Regional**.
- [ ] Create resource `/items`, then methods **GET** and **POST**, each with integration **Lambda function**, **Lambda proxy integration ✔**, function `saa-items-api`.
- [ ] Choose **Deploy API** to a new stage `dev`, and copy the **Invoke URL**.
- [ ] Test it:

```bash
API=https://xxxx.execute-api.us-east-1.amazonaws.com/dev
curl -s -X POST $API/items -d '{"name":"widget"}'
curl -s $API/items
```

### A4. Throttling, usage plans, and API keys

- [ ] Open **Stages → dev → Stage settings → Throttling**: rate **5 rps**, burst **10**.
- [ ] Open **Usage plans → Create**: `saa-basic`, rate 2 rps, burst 5, quota 100/day. Associate stage `dev`.
- [ ] Open **API keys → Create** `saa-partner-key` and add it to the usage plan.
- [ ] On the **POST** method, turn on **API key required**, then **redeploy**.
- [ ] Test without the key (**403 Forbidden**) and with it:

```bash
curl -s -X POST $API/items -H "x-api-key: <key>" -d '{"name":"keyed"}'
```

### A5. Response caching (REST only)

- [ ] Open **Stages → dev → Edit → Cache settings**: provision cache **0.5 GB**, TTL 60 s. *(This costs ~$0.02/h. Turn it off after testing.)*
- [ ] Call `GET /items` twice and compare the latency in CloudWatch (`CacheHitCount` / `CacheMissCount`).

### A6. Look at monitoring

- [ ] Open **Lambda → Monitor**: invocations, duration, errors, throttles, and concurrent executions. Then choose **View CloudWatch logs**.

---

## Part B — CLI

### B1. Package and create a function

- [ ] Create the execution role:

```bash
cat > lambda-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name saa-cli-lambda-role --assume-role-policy-document file://lambda-trust.json
aws iam attach-role-policy --role-name saa-cli-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name saa-cli-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
```

- [ ] Write a simple handler that reports partial batch failures:

```bash
cat > handler.py <<'EOF'
import json
def lambda_handler(event, context):
    # SQS batch → report partial failures so only bad messages retry
    failures = []
    for r in event.get("Records", []):
        body = r["body"]
        print("processing", body)
        if "fail" in body:
            failures.append({"itemIdentifier": r["messageId"]})
    return {"batchItemFailures": failures}
EOF
zip -q fn.zip handler.py
sleep 10
```

- [ ] Create the function:

```bash
FN_ARN=$(aws lambda create-function --function-name saa-cli-worker \
  --runtime python3.13 --architectures arm64 --handler handler.lambda_handler \
  --role arn:aws:iam::$ACCOUNT_ID:role/saa-cli-lambda-role \
  --zip-file fileb://fn.zip --memory-size 256 --timeout 30 \
  --query FunctionArn --output text)
aws lambda wait function-active-v2 --function-name saa-cli-worker
```

### B2. Invoke synchronously and asynchronously

- [ ] Invoke it both ways:

```bash
aws lambda invoke --function-name saa-cli-worker --cli-binary-format raw-in-base64-out \
  --payload '{"Records":[{"messageId":"1","body":"hello"}]}' out.json && cat out.json

aws lambda invoke --function-name saa-cli-worker --invocation-type Event \
  --cli-binary-format raw-in-base64-out --payload '{}' /dev/null
```

> Async invocations retry twice on failure and can send failures to an **on-failure destination** (SQS, SNS, EventBridge, Lambda) or a DLQ.

### B3. SQS event source mapping

- [ ] Create a queue and connect it to the function:

```bash
Q_URL=$(aws sqs create-queue --queue-name saa-lambda-in --attributes VisibilityTimeout=180 --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

aws lambda create-event-source-mapping --function-name saa-cli-worker --event-source-arn $Q_ARN \
  --batch-size 10 --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures \
  --scaling-config MaximumConcurrency=5
```

- [ ] Send messages and read the logs:

```bash
for m in ok-1 ok-2 fail-3; do aws sqs send-message --queue-url $Q_URL --message-body $m >/dev/null; done
sleep 20
aws logs tail /aws/lambda/saa-cli-worker --since 5m | grep processing
```

> The visibility timeout should be **at least 6× the function timeout** when Lambda consumes the queue.

### B4. Concurrency controls

- [ ] **Reserved concurrency** guarantees capacity and caps the function (it protects downstream databases):

```bash
aws lambda put-function-concurrency --function-name saa-cli-worker --reserved-concurrent-executions 5
```

- [ ] **Provisioned concurrency** keeps environments pre-warmed (no cold starts). It needs a version or alias, and it costs money:

```bash
VER=$(aws lambda publish-version --function-name saa-cli-worker --query Version --output text)
aws lambda create-alias --function-name saa-cli-worker --name live --function-version $VER
aws lambda put-provisioned-concurrency-config --function-name saa-cli-worker --qualifier live \
  --provisioned-concurrent-executions 1
aws lambda get-provisioned-concurrency-config --function-name saa-cli-worker --qualifier live --query Status
```

- [ ] Remove the provisioned concurrency again to stop the charge:

```bash
aws lambda delete-provisioned-concurrency-config --function-name saa-cli-worker --qualifier live
```

### B5. HTTP API in one command (quick create)

- [ ] Create the API pointed at the console function:

```bash
API_FN=arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:saa-items-api
HTTP_API=$(aws apigatewayv2 create-api --name saa-items-http --protocol-type HTTP --target $API_FN \
  --query ApiId --output text)
```

- [ ] Allow API Gateway to invoke the function:

```bash
aws lambda add-permission --function-name saa-items-api --statement-id apigw-http \
  --action lambda:InvokeFunction --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$HTTP_API/*"
```

- [ ] Call it:

```bash
EP=$(aws apigatewayv2 get-api --api-id $HTTP_API --query ApiEndpoint --output text)
curl -s -X POST $EP/items -d '{"name":"from-http-api"}'; echo
curl -s $EP/items; echo
```

### B6. Lambda function URL (simplest HTTPS endpoint)

- [ ] Create a URL with IAM auth and call it:

```bash
aws lambda create-function-url-config --function-name saa-cli-worker --auth-type AWS_IAM
aws lambda get-function-url-config --function-name saa-cli-worker --query FunctionUrl
```

> `AWS_IAM` auth means callers must SigV4-sign their requests. `NONE` makes it public.

### B7. Lambda in a VPC (read-through)

```bash
# aws lambda update-function-configuration --function-name saa-cli-worker \
#   --vpc-config SubnetIds=$APP_A,$APP_B,SecurityGroupIds=$APP_SG
```

- [ ] Remember: a Lambda function in a VPC gets ENIs in your subnets so it can reach RDS or ElastiCache. To reach the internet or AWS APIs it needs a **NAT gateway or VPC endpoints**. **It never gets a public IP**, even in a public subnet.

---

## Verify

- [ ] The REST API's POST and GET work, throttling and API keys are enforced, and the cache was tested.
- [ ] The SQS → Lambda mapping processes messages and retries only the failed one.
- [ ] Reserved and provisioned concurrency were configured, and provisioned concurrency was removed.

## Exam tips

- **"Run code without managing servers, pay per request"** → **Lambda**. **Over 15 minutes** → Fargate, Batch, or Step Functions.
- **"Throttle or meter API clients, monetise APIs"** → API Gateway **usage plans + API keys** (REST).
- **"Authenticate API users with a user directory"** → **Cognito user pool authorizer**.
- **"Lambda overloading RDS with connections"** → **RDS Proxy** and/or **reserved concurrency**.
- **"Eliminate cold starts for a latency-sensitive API"** → **provisioned concurrency**.
- **"Protect the API from SQL injection or common exploits"** → **AWS WAF** on the REST API stage (or on CloudFront).
- **Edge-optimised** endpoints use CloudFront under the hood. **Private** APIs are reachable only through an interface VPC endpoint.

## Challenge (optional)

- [ ] Add a **Cognito user pool** authorizer to the REST API's POST method, sign up a test user, and call the API with the ID token.

## Cleanup

- [ ] Delete the APIs:

```bash
aws apigatewayv2 delete-api --api-id $HTTP_API
aws apigateway delete-rest-api --rest-api-id $(aws apigateway get-rest-apis --query "items[?name=='saa-items-rest'].id" --output text)
```

- [ ] Delete the event source mapping, functions, and queue:

```bash
UUID=$(aws lambda list-event-source-mappings --function-name saa-cli-worker --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid $UUID
aws lambda delete-function --function-name saa-cli-worker
aws lambda delete-function --function-name saa-items-api
aws sqs delete-queue --queue-url $Q_URL
```

- [ ] Delete the usage plan and API key (console), and the DynamoDB table `saa-items`.
- [ ] Delete the IAM roles:

```bash
for P in service-role/AWSLambdaBasicExecutionRole service-role/AWSLambdaSQSQueueExecutionRole; do
  aws iam detach-role-policy --role-name saa-cli-lambda-role --policy-arn arn:aws:iam::aws:policy/$P
done
aws iam delete-role --role-name saa-cli-lambda-role
# also delete the console-created saa-items-api-role-xxxx role and its inline policy
```

- [ ] Delete the `/aws/lambda/*` log groups and the local files: `rm handler.py fn.zip out.json lambda-trust.json`

---

## Knowledge check

**Q1.** A company wants to expose an API to partners with per-partner request quotas and rate limits, and cache frequent GET responses. Which service and features?

<details><summary>Answer</summary>

**API Gateway REST API** with **usage plans + API keys** and **stage caching**.

</details>

**Q2.** A Lambda function processing SQS messages fails on one bad message per batch, causing the whole batch to be retried repeatedly. How do you fix this?

<details><summary>Answer</summary>

Turn on **`ReportBatchItemFailures`** and return only the failed message IDs, and configure a **DLQ** on the source queue for messages that keep failing.

</details>

**Q3.** A latency-sensitive serverless API has slow responses for the first request after periods of idleness. What should you configure?

<details><summary>Answer</summary>

**Provisioned concurrency** on the function alias, or **SnapStart** for supported runtimes.

</details>
