# Lab 04 — IAM Roles, STS & Cross-Account Access

| | |
|---|---|
| **Module** | 2 — Identity & Access Management |
| **Exam mapping** | Domain 1 · Task 1.1 (roles, federation, cross-account) · Task 1.2 (secure workloads) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | < $0.05 (one t3.micro for ~20 min) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-03-iam-users-groups-policies.md) · [Back to index](../README.md) · [Next lab ➡](lab-05-advanced-iam-policies.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Explain a role's two halves: the **trust policy** (*who* can assume it) and the **permissions policy** (*what* it can do).
- Assume roles with **AWS STS** and use the temporary credentials you get back.
- Give EC2 access to AWS APIs through an **instance profile** instead of access keys.
- Set up **cross-account access**, including the `ExternalId` safeguard for third parties.
- Recognise federation options: SAML 2.0, OIDC/web identity, **Cognito**, and IAM Identity Center.

## Key concepts

| Concept | Summary |
|---|---|
| Role | An identity with no long-term credentials. Assuming it returns temporary credentials (15 min–12 h). |
| Trust policy | A resource-based policy on the role that names the **principals** allowed to call `sts:AssumeRole`. |
| Instance profile | The container that passes a role to EC2. The console creates it for you automatically, the CLI doesn't. |
| Service-linked role | A predefined role that an AWS service creates and manages for itself. |
| `ExternalId` | A secret shared with a third party that prevents the "confused deputy" problem. |
| `sts:AssumeRoleWithWebIdentity` / `AssumeRoleWithSAML` | Federation from OIDC or SAML identity providers. |

---

## Part A — Console

### A1. Create a role for EC2

- [ ] Open **IAM → Roles → Create role**.
  - Trusted entity: **AWS service → EC2**
  - Permissions: `AmazonS3ReadOnlyAccess` and `AmazonSSMManagedInstanceCore`
  - Name: `saa-ec2-s3-read-role`
- [ ] Open the role's **Trust relationships** tab and note the `"Service": "ec2.amazonaws.com"` principal.

### A2. Launch an instance with the role

- [ ] Open **EC2 → Launch instance**.
  - Name `saa-lab04-instance`, Amazon Linux 2023, `t3.micro`
  - Key pair: **Proceed without a key pair** (you'll use Session Manager)
  - Network: default VPC, public subnet, auto-assign public IP **enabled**
  - **Advanced details → IAM instance profile:** `saa-ec2-s3-read-role`
- [ ] Wait about 2 minutes, then **Connect → Session Manager → Connect**.
- [ ] In the session, run:

```bash
aws sts get-caller-identity          # shows assumed-role/saa-ec2-s3-read-role/i-...
aws s3 ls                            # works
aws ec2 describe-instances --region us-east-1   # AccessDenied (not in role)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/saa-ec2-s3-read-role
```

> The last command shows the **temporary credentials** and their `Expiration`. The SDK refreshes them automatically.

### A3. Create a role a user can assume (simulated cross-account)

> True cross-account access needs two accounts. You'll trust your **own** account ID, which works exactly the same way. If you have a second account, use its ID.

- [ ] Open **IAM → Roles → Create role → AWS account → This account** (or *Another AWS account*).
  - Tick **Require MFA**.
  - Permissions: `ReadOnlyAccess`
  - Name: `saa-readonly-assumable`
- [ ] Copy the **Link to switch roles in console** from the role summary.
- [ ] Go to that link (or use **account menu → Switch role**), enter the account and role, and pick a display colour.
- [ ] Notice the coloured role banner. Try creating something (e.g. an S3 bucket) and confirm it's denied.
- [ ] Use **Switch back**.

---

## Part B — CLI

### B1. Create a role with a trust policy that requires ExternalId

- [ ] Write the trust policy and create the role:

```bash
cat > trust-crossacct.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${ACCOUNT_ID}:root" },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "saa-partner-7731" } }
    }
  ]
}
EOF

aws iam create-role --role-name saa-partner-audit-role \
  --assume-role-policy-document file://trust-crossacct.json \
  --max-session-duration 3600

aws iam attach-role-policy --role-name saa-partner-audit-role \
  --policy-arn arn:aws:iam::aws:policy/SecurityAudit
```

> The principal `arn:aws:iam::<acct>:root` means "any identity in that account **that also has IAM permission to call `sts:AssumeRole`** on this role". It doesn't mean the root user.

### B2. Assume the role

- [ ] Without the ExternalId, the call fails:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/saa-partner-audit-role \
  --role-session-name test
# AccessDenied
```

- [ ] With the ExternalId, it succeeds:

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/saa-partner-audit-role \
  --role-session-name partner-session \
  --external-id saa-partner-7731 \
  --duration-seconds 900)

echo $CREDS | jq '.Credentials.Expiration'
```

- [ ] Export the temporary credentials and call AWS with them:

```bash
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .Credentials.AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .Credentials.SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .Credentials.SessionToken)

aws sts get-caller-identity          # assumed-role/saa-partner-audit-role/partner-session
aws iam get-account-summary --query 'SummaryMap.Roles'   # SecurityAudit allows reads
aws s3 mb s3://should-fail-$SUFFIX   # denied

unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity          # back to your admin user
```

### B3. Let the CLI assume roles through a profile

- [ ] Add a profile that assumes the role:

```bash
cat >> ~/.aws/config <<EOF

[profile saa-partner]
role_arn = arn:aws:iam::${ACCOUNT_ID}:role/saa-partner-audit-role
source_profile = default
external_id = saa-partner-7731
role_session_name = cli-partner
EOF

aws sts get-caller-identity --profile saa-partner
```

### B4. Build an EC2 role and instance profile entirely in the CLI

- [ ] Create the role, profile, and instance:

```bash
cat > trust-ec2.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }]
}
EOF

aws iam create-role --role-name saa-cli-ec2-role --assume-role-policy-document file://trust-ec2.json
aws iam attach-role-policy --role-name saa-cli-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name saa-cli-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# The CLI does NOT auto-create the instance profile
aws iam create-instance-profile --instance-profile-name saa-cli-ec2-profile
aws iam add-role-to-instance-profile --instance-profile-name saa-cli-ec2-profile --role-name saa-cli-ec2-role

sleep 10   # IAM propagation

AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI --instance-type t3.micro \
  --iam-instance-profile Name=saa-cli-ec2-profile \
  --metadata-options HttpTokens=required \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=saa-lab04-cli}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE_ID
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn'
```

- [ ] After about 2 minutes, run a command through SSM (no SSH needed):

```bash
CMD_ID=$(aws ssm send-command --instance-ids $INSTANCE_ID \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["aws sts get-caller-identity","aws s3 ls | head -5"]' \
  --query Command.CommandId --output text)
sleep 5
aws ssm get-command-invocation --command-id $CMD_ID --instance-id $INSTANCE_ID --query StandardOutputContent --output text
```

### B5. Inspect the service-linked roles

- [ ] List them:

```bash
aws iam list-roles --path-prefix /aws-service-role/ --query 'Roles[].RoleName'
```

---

## Verify

- [ ] The EC2 instance can list S3 but can't describe EC2.
- [ ] `saa-partner-audit-role` can only be assumed with the correct ExternalId.
- [ ] The `saa-partner` CLI profile works without exporting credentials by hand.

## Exam tips

- **Third-party SaaS needs access to your account?** Use a cross-account role with **ExternalId**. Never share access keys.
- **Mobile or web app users need AWS access?** Use **Amazon Cognito** identity pools (web identity federation).
- **Corporate Active Directory users need console access?** Use **IAM Identity Center** with AD (or SAML 2.0 federation).
- `iam:PassRole` is the permission that lets a user *give* a role to a service (e.g. launch EC2 with a role). It's a common privilege-escalation path to restrict.
- Roles on a running instance can be swapped (`aws ec2 replace-iam-instance-profile-association`) **without stopping it**.

## Challenge (optional)

- [ ] Add a condition to `saa-readonly-assumable`'s trust policy so it can only be assumed from your public IP (`aws:SourceIp`). Then test switching roles from a different network, such as your phone's hotspot.

## Cleanup

- [ ] Terminate the instances:

```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
# terminate the console instance saa-lab04-instance too
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=saa-lab04-instance" "Name=instance-state-name,Values=running,stopped" \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] Delete the instance profile and roles:

```bash
aws iam remove-role-from-instance-profile --instance-profile-name saa-cli-ec2-profile --role-name saa-cli-ec2-role
aws iam delete-instance-profile --instance-profile-name saa-cli-ec2-profile
for P in AmazonSSMManagedInstanceCore AmazonS3ReadOnlyAccess; do
  aws iam detach-role-policy --role-name saa-cli-ec2-role --policy-arn arn:aws:iam::aws:policy/$P
done
aws iam delete-role --role-name saa-cli-ec2-role

aws iam detach-role-policy --role-name saa-partner-audit-role --policy-arn arn:aws:iam::aws:policy/SecurityAudit
aws iam delete-role --role-name saa-partner-audit-role
```

- [ ] Delete the console roles `saa-ec2-s3-read-role` and `saa-readonly-assumable` (the console removes their instance profiles too).
- [ ] Remove the `[profile saa-partner]` block from `~/.aws/config`, then `rm trust-*.json`.

---

## Knowledge check

**Q1.** A security vendor needs read-only access to 30 of your AWS accounts to run compliance scans. What's the most secure approach?

<details><summary>Answer</summary>

Create a **cross-account IAM role** in each account (e.g. with CloudFormation StackSets) that trusts the vendor's AWS account, requires a unique **ExternalId**, and has least-privilege read-only permissions.

</details>

**Q2.** An application on EC2 uses access keys stored in environment variables. The security team wants long-term credentials removed. What should you do?

<details><summary>Answer</summary>

Create an IAM role with the needed permissions, attach it to the instance as an **instance profile**, and remove the keys. The SDK picks up the role credentials from IMDS automatically.

</details>

**Q3.** A mobile app needs to let users upload photos straight to S3 after they sign in with Google. What should you use?

<details><summary>Answer</summary>

**Amazon Cognito** (user pool for sign-in, plus an identity pool that federates with Google). It exchanges the token for temporary AWS credentials scoped to a per-user S3 prefix.

</details>
