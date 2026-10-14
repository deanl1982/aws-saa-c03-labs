# Lab 01 — Secure Account Baseline & Billing Guardrails

| | |
|---|---|
| **Module** | 1 — Getting Started & Account Foundations |
| **Exam mapping** | Domain 1 · Task 1.1 (secure access) · Domain 4 (cost awareness) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 45 minutes |
| **Estimated cost** | $0 (the first two budgets are free) |

- [ ] **Lab complete**

[⬅ Back to index](../README.md) · [Next lab ➡](lab-02-cli-setup.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Explain the **shared responsibility model**: what AWS secures and what you secure.
- Secure the **root user** with MFA and know which tasks only root can do.
- Create a day-to-day **administrative identity** so you stop using root.
- Set up **billing alerts** and **AWS Budgets** so a forgotten resource doesn't run up a surprise bill.

## Exam context

The exam often asks you to pick the *most secure* option. "Use the root user" is almost never correct. Know these:

- Root-only tasks: change account settings or the root email, close the account, restore IAM admin permissions, change the support plan, register as a seller in the Reserved Instance Marketplace, and enable MFA Delete on an S3 bucket.
- **AWS secures the cloud** (hardware, facilities, hypervisor, managed-service infrastructure). **You secure what's in the cloud** (data, IAM, OS patching on EC2, security groups, encryption choices).

---

## Prerequisites

- An AWS account you can sign into as root.
- An authenticator app (Google Authenticator, Authy, 1Password, etc.) or a FIDO2 security key.

---

## Part A — Console

### A1. Secure the root user

- [ ] Sign in to the [AWS Console](https://console.aws.amazon.com/) as the **root user**.
- [ ] Open the account menu (top right), choose **Security credentials**, then **Assign MFA device**.
- [ ] Choose **Passkey or security key** or **Authenticator app** and finish registration.
- [ ] On the same page, check **Access keys**. If root has any access keys, **delete them**. Root should never have access keys.

### A2. Set an account alias and password policy

- [ ] Open **IAM** and, on the **Dashboard** under *AWS Account*, create an **account alias** (e.g. `yourname-saa-lab`). It gives you an easy-to-remember sign-in URL.
- [ ] Go to **IAM → Account settings → Password policy → Edit** and set:
  - Minimum length 14
  - Require at least one uppercase, lowercase, number, and symbol
  - Password expiry 90 days
  - Prevent reuse of the last 5 passwords

### A3. Create an administrative identity

> AWS's preferred approach is **IAM Identity Center** (Lab 06). For now, create an IAM admin user so you can work through the next labs.

- [ ] Open **IAM → User groups → Create group**, name it `Administrators`, and attach **AdministratorAccess**.
- [ ] Open **IAM → Users → Create user**, name it `admin-<yourname>`, and tick **Provide user access to the AWS Management Console**.
- [ ] Add the user to `Administrators`.
- [ ] Sign out, then sign in as the new user via `https://<alias>.signin.aws.amazon.com/console`.
- [ ] Assign an **MFA device** to this user too (top right → *Security credentials*).
- [ ] Turn on **IAM user/role access to billing information**. As root, go to **Account → IAM user and role access to Billing information → Edit → Activate**.

### A4. Billing alerts and a budget

- [ ] As root (or your admin user once billing access is on), open **Billing and Cost Management → Billing preferences** and turn on **Receive CloudWatch billing alerts** and **Receive Free Tier usage alerts**.
- [ ] Open **Budgets → Create budget → Use a template → Monthly cost budget**.
  - Name: `saa-monthly-budget`
  - Amount: e.g. `$25`
  - Email recipients: your email
- [ ] Create a second budget, **Zero spend budget**, which alerts you once spending goes above $0.01.

### A5. Explore the shared responsibility model

- [ ] Open the **AWS Artifact** console and browse the available compliance reports (SOC, PCI). This is how you show auditors *AWS's* side of the model.
- [ ] Open **AWS Health Dashboard → Your account health** to see where AWS reports service events that affect you.

---

## Part B — CLI

> You haven't installed the CLI locally yet (that's Lab 02), so use **AWS CloudShell**: the `>_` icon in the console toolbar. CloudShell uses the credentials of the user you're signed in as, so sign in as your **admin user**.

### B1. Verify identity

- [ ] Confirm who you are:

```bash
aws sts get-caller-identity
```

> The `Arn` should end with `user/admin-<yourname>`, **not** `root`.

### B2. Account alias and password policy

- [ ] Create or confirm the alias:

```bash
aws iam create-account-alias --account-alias yourname-saa-lab   # skip if already set in Part A
aws iam list-account-aliases
```

- [ ] Apply the password policy:

```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --require-numbers \
  --require-symbols \
  --max-password-age 90 \
  --password-reuse-prevention 5 \
  --allow-users-to-change-password

aws iam get-account-password-policy
```

### B3. Check root account security posture

- [ ] Generate and read the credential report:

```bash
aws iam generate-credential-report
sleep 5
aws iam get-credential-report --query Content --output text | base64 --decode | cut -d, -f1,4,8,9,11 | column -t -s,
```

> Confirm the `<root_account>` row shows `mfa_active = true` and `access_key_1_active = false`.

- [ ] Get an account summary:

```bash
aws iam get-account-summary --query 'SummaryMap.{RootMFA:AccountMFAEnabled,RootKeys:AccountAccessKeysPresent,Users:Users}'
```

### B4. Create a billing alarm (CloudWatch + SNS)

> Billing metrics only exist in **us-east-1**.

- [ ] Create an SNS topic and subscribe your email:

```bash
export AWS_REGION=us-east-1
TOPIC_ARN=$(aws sns create-topic --name saa-billing-alerts --query TopicArn --output text)
aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol email --notification-endpoint you@example.com
```

- [ ] **Confirm the subscription** from the email you receive.
- [ ] Create an alarm that fires when estimated charges go over $10:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name saa-billing-over-10usd \
  --namespace AWS/Billing \
  --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "$TOPIC_ARN"

aws cloudwatch describe-alarms --alarm-names saa-billing-over-10usd \
  --query 'MetricAlarms[0].{Name:AlarmName,State:StateValue,Threshold:Threshold}'
```

### B5. Create a budget with the CLI

- [ ] Create a budget:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > budget.json <<'EOF'
{
  "BudgetName": "saa-cli-budget",
  "BudgetLimit": { "Amount": "20", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
EOF

cat > notifications.json <<'EOF'
[
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [ { "SubscriptionType": "EMAIL", "Address": "you@example.com" } ]
  }
]
EOF

aws budgets create-budget --account-id "$ACCOUNT_ID" \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json

aws budgets describe-budgets --account-id "$ACCOUNT_ID" \
  --query 'Budgets[].{Name:BudgetName,Limit:BudgetLimit.Amount}'
```

---

## Verify

- [ ] The root user has MFA and **no** access keys.
- [ ] You can sign in as `admin-<yourname>` with MFA.
- [ ] You have at least one budget and one billing alarm, and confirmed the SNS email.

## Exam tips

- **Budgets** are for *forecast and actual* spend alerts, and *Budget Actions* can apply SCPs/IAM policies or stop EC2/RDS. **Billing alarms** are plain CloudWatch alarms on `EstimatedCharges`.
- **Cost Explorer** analyses past and forecast spend. **Cost and Usage Report (CUR)** is the most detailed billing data, delivered to S3.
- **AWS Artifact** is where you get compliance reports, so it's the answer to "auditor needs AWS's SOC 2 report".

## Challenge (optional)

- [ ] Add a **Budget Action** to `saa-monthly-budget` that attaches an IAM policy denying `ec2:RunInstances` to your admin group once 100% of the budget is reached. Think about why this is a powerful guardrail, and what makes it risky.

## Cleanup

> Keep the budgets, billing alarm, admin user, and root MFA. They protect you for the rest of the programme.

- [ ] `rm budget.json notifications.json` in CloudShell.
- [ ] *(Optional)* Delete `saa-cli-budget` if you'd rather keep only one budget: `aws budgets delete-budget --account-id "$ACCOUNT_ID" --budget-name saa-cli-budget`

---

## Knowledge check

**Q1.** A company wants to stop its developers from accidentally running up costs. When forecast spend passes 90%, it wants to notify finance **and** automatically stop new EC2 launches. What should you use?

<details><summary>Answer</summary>

**AWS Budgets** with a forecasted alert and a **Budget Action** that applies an IAM policy or SCP denying `ec2:RunInstances`. A CloudWatch billing alarm can only notify; it can't apply policies.

</details>

**Q2.** Under the shared responsibility model, who patches the guest operating system on an EC2 instance? And on an RDS instance?

<details><summary>Answer</summary>

On **EC2**, the customer patches the guest OS. On **RDS**, AWS patches the OS and database engine (inside the maintenance window you choose), because RDS is a managed service.

</details>

**Q3.** An auditor asks for evidence of AWS's ISO 27001 certification. Where do you get it?

<details><summary>Answer</summary>

**AWS Artifact**.

</details>
