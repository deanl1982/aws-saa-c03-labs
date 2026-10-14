# Lab 06 — AWS Organizations, SCPs & IAM Identity Center

| | |
|---|---|
| **Module** | 2 — Identity & Access Management |
| **Exam mapping** | Domain 1 · Task 1.1 (multi-account strategy, SCPs, centralised identity) · Domain 4 (consolidated billing) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 90 minutes |
| **Estimated cost** | $0 (Organizations and Identity Center are free) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-05-advanced-iam-policies.md) · [Back to index](../README.md) · [Next lab ➡](lab-07-s3-fundamentals.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Design a multi-account structure with **OUs** (Security, Workloads/Prod, Workloads/Dev, Sandbox).
- Apply **Service Control Policies (SCPs)** as guardrails, and explain that they **don't grant** permissions.
- Explain **consolidated billing** benefits: volume discounts and shared RIs/Savings Plans.
- Centralise workforce access with **IAM Identity Center** permission sets.
- Recognise **Control Tower**, **RAM**, tag policies, and delegated administrators.

> ⚠️ **Read first:** this lab turns your account into an Organization **management account** and creates a **member account**. Member accounts need a unique email address, so use plus-addressing (`you+saa-dev@gmail.com`). Closing an account later puts it in a 90-day suspended state. If you're using a work account that already belongs to an Organization, **do this lab in a personal account instead**.

## Key concepts

| Concept | Summary |
|---|---|
| Management account | Pays the bills and owns the org. **SCPs never restrict the management account.** |
| OU | Organizational Unit. Can nest up to 5 levels. Policies are inherited downwards. |
| SCP | Sets the *maximum* permissions for IAM principals in member accounts, including their root user. |
| RCP | Resource Control Policy. Caps the permissions on *resources* (S3, KMS, STS, SQS, Secrets Manager) in member accounts. |
| Identity Center | SSO portal, permission sets (templates that become roles in each account), and an external IdP or AD integration. |
| Control Tower | Sets up a landing zone automatically: account factory, guardrails (SCPs and Config rules), and a log archive. |

---

## Part A — Console

### A1. Create an organization

- [ ] Open **AWS Organizations → Create an organization** (with **all features** enabled).
- [ ] Verify the email address if you're prompted to.

### A2. Create OUs and a member account

- [ ] Under **Root**, create the OUs `Security`, `Workloads`, and `Sandbox`, then create `Prod` and `Dev` inside `Workloads`.
- [ ] Choose **Add an AWS account → Create an AWS account**.
  - Name: `saa-dev`
  - Email: `you+saa-dev@example.com`
  - IAM role name: leave the default, `OrganizationAccountAccessRole`
- [ ] When it's ready, move the account into `Workloads/Dev`.

### A3. Enable and attach SCPs

- [ ] Open **Policies → Service control policies → Enable**.
- [ ] Create the policy `saa-deny-regions-except-us`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "organizations:*", "sts:*", "support:*", "budgets:*",
        "cloudfront:*", "route53:*", "waf:*", "health:*", "account:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "us-west-2"] }
      }
    }
  ]
}
```

- [ ] Create the policy `saa-protect-security-services`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectAuditing",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging", "cloudtrail:DeleteTrail",
        "guardduty:DeleteDetector", "config:StopConfigurationRecorder",
        "organizations:LeaveOrganization"
      ],
      "Resource": "*"
    }
  ]
}
```

- [ ] Attach both policies to the **Workloads** OU.

> Leave the default `FullAWSAccess` SCP attached. SCPs use a deny-list model on top of it.

### A4. Test the SCP from the member account

- [ ] From the management account, **Switch role** into account ID `<saa-dev-id>` with role `OrganizationAccountAccessRole`.
- [ ] Switch region to **eu-west-1** and try to create an S3 bucket or launch EC2. It should be **denied**, even though the role has `AdministratorAccess`.
- [ ] Switch to **us-east-1**. It works.
- [ ] Switch back to the management account.

### A5. Set up IAM Identity Center

- [ ] Open **IAM Identity Center → Enable** (organization instance) in `us-east-1`.
- [ ] Under **Users → Add user**, create `saa.learner` with your email.
- [ ] Under **Groups**, create `SAA-Developers` and add the user.
- [ ] Create two permission sets:
  - `SAA-ReadOnly`: predefined `ViewOnlyAccess`, session 4 h
  - `SAA-PowerUser`: predefined `PowerUserAccess`
- [ ] Open **AWS accounts**, select `saa-dev`, choose **Assign users or groups**, select `SAA-Developers`, and assign **both** permission sets.
- [ ] Accept the invitation email, set a password and MFA, and sign in through the **AWS access portal URL**.
- [ ] From the portal, open the `saa-dev` account with `SAA-ReadOnly` and confirm you can only read.

### A6. Look at consolidated billing

- [ ] Open **Billing → Bills** in the management account and note that charges are listed **per member account**.
- [ ] Open **Billing preferences** and look for the **RI and Savings Plans discount sharing** toggle.

---

## Part B — CLI

> Run these from the **management account**.

### B1. Inspect the organization

- [ ] Look around:

```bash
aws organizations describe-organization --query 'Organization.{Id:Id,Master:MasterAccountId,Features:FeatureSet}'
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)
aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query 'OrganizationalUnits[].[Name,Id]' --output table
aws organizations list-accounts --query 'Accounts[].[Name,Id,Status]' --output table
```

### B2. Create an SCP from the CLI

- [ ] Look up the Sandbox OU ID:

```bash
SANDBOX_OU=$(aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID \
  --query "OrganizationalUnits[?Name=='Sandbox'].Id" --output text)
```

- [ ] Write an SCP that denies expensive instance types in the Sandbox:

```bash
cat > deny-big-ec2.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "OnlySmallInstances",
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "StringNotLike": { "ec2:InstanceType": ["t3.*", "t4g.*"] }
    }
  }]
}
EOF
```

- [ ] Create the policy and attach it to the Sandbox OU:

```bash
SCP_ID=$(aws organizations create-policy --name saa-sandbox-small-ec2 \
  --type SERVICE_CONTROL_POLICY \
  --description "Sandbox: only t3/t4g instances" \
  --content file://deny-big-ec2.json \
  --query 'Policy.PolicySummary.Id' --output text)

aws organizations attach-policy --policy-id $SCP_ID --target-id $SANDBOX_OU
aws organizations list-policies-for-target --target-id $SANDBOX_OU --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[].Name'
```

### B3. Move the member account and test with `--dry-run`

- [ ] Move the account into the Sandbox OU:

```bash
DEV_ACCT=$(aws organizations list-accounts --query "Accounts[?Name=='saa-dev'].Id" --output text)
DEV_OU=$(aws organizations list-parents --child-id $DEV_ACCT --query 'Parents[0].Id' --output text)
aws organizations move-account --account-id $DEV_ACCT --source-parent-id $DEV_OU --destination-parent-id $SANDBOX_OU
```

- [ ] Assume the member account's admin role:

```bash
cat >> ~/.aws/config <<EOF

[profile saa-dev]
role_arn = arn:aws:iam::${DEV_ACCT}:role/OrganizationAccountAccessRole
source_profile = default
region = us-east-1
EOF
```

- [ ] Test with `--dry-run`:

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text --profile saa-dev)

aws ec2 run-instances --dry-run --image-id $AMI --instance-type t3.micro  --profile saa-dev   # DryRunOperation (allowed)
aws ec2 run-instances --dry-run --image-id $AMI --instance-type m5.large  --profile saa-dev   # UnauthorizedOperation (SCP)
```

### B4. Identity Center from the CLI

- [ ] Look up the instance, identity store, and permission sets:

```bash
SSO_INSTANCE=$(aws sso-admin list-instances --query 'Instances[0].InstanceArn' --output text)
IDSTORE=$(aws sso-admin list-instances --query 'Instances[0].IdentityStoreId' --output text)

aws sso-admin list-permission-sets --instance-arn $SSO_INSTANCE
aws identitystore list-users --identity-store-id $IDSTORE --query 'Users[].UserName'
```

- [ ] Create a permission set with an inline policy (S3 read in us-east-1 only):

```bash
PS_ARN=$(aws sso-admin create-permission-set --instance-arn $SSO_INSTANCE \
  --name SAA-S3Reader --session-duration PT2H --query 'PermissionSet.PermissionSetArn' --output text)

aws sso-admin put-inline-policy-to-permission-set --instance-arn $SSO_INSTANCE --permission-set-arn $PS_ARN \
  --inline-policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:Get*","s3:List*"],"Resource":"*"}]}'
```

- [ ] Assign it to the group in the member account:

```bash
GROUP_ID=$(aws identitystore list-groups --identity-store-id $IDSTORE \
  --query "Groups[?DisplayName=='SAA-Developers'].GroupId" --output text)

aws sso-admin create-account-assignment --instance-arn $SSO_INSTANCE \
  --target-id $DEV_ACCT --target-type AWS_ACCOUNT \
  --permission-set-arn $PS_ARN --principal-type GROUP --principal-id $GROUP_ID
```

### B5. Configure the CLI to sign in with SSO

- [ ] Configure and sign in:

```bash
aws configure sso
# SSO session name: saa
# SSO start URL:    https://d-xxxxxxxxxx.awsapps.com/start   (from Identity Center dashboard)
# SSO region:       us-east-1
# → browser opens, approve, then choose account + role

aws sts get-caller-identity --profile <generated-profile-name>
aws sso login --sso-session saa      # when the token expires
```

---

## Verify

- [ ] SCPs block regions and instance types in member accounts even for admins.
- [ ] You can sign in to the member account through the Identity Center access portal.
- [ ] `aws configure sso` gives you a working short-lived CLI profile.

## Exam tips

- **SCPs never affect the management account.** Keep workloads out of it.
- SCPs **don't grant** anything. The account still needs IAM allows.
- "Stop all accounts from disabling CloudTrail": use an **SCP**. "Detect noncompliance": use **AWS Config**. "Set up a governed multi-account environment quickly": use **Control Tower**.
- Share resources across accounts (subnets, Transit Gateway, License Manager configs, Route 53 Resolver rules) with **AWS RAM**.
- Consolidated billing **combines usage for volume pricing tiers** and shares RI/Savings Plan discounts across accounts.
- Workforce users plus an existing IdP (Okta, Entra ID) means **IAM Identity Center**. Customer-facing app users means **Cognito**.

## Challenge (optional)

- [ ] Create a **tag policy** that enforces `CostCenter` tag values on EC2 instances, and attach it to `Workloads`.
- [ ] Set up a **delegated administrator** for IAM Identity Center or GuardDuty in a member account.

## Cleanup

- [ ] Detach and delete the SCPs:

```bash
aws organizations detach-policy --policy-id $SCP_ID --target-id $SANDBOX_OU
aws organizations delete-policy --policy-id $SCP_ID
# Repeat for saa-deny-regions-except-us and saa-protect-security-services (detach from Workloads first)
```

- [ ] Remove the Identity Center assignments and permission sets, or keep Identity Center for later labs (recommended: **keep it**).
- [ ] **Decide** whether to keep the member account. Keeping it is free, and it's useful for cross-account labs. To close it:

```bash
aws organizations close-account --account-id $DEV_ACCT
```

- [ ] `rm deny-big-ec2.json`

---

## Knowledge check

**Q1.** A company wants to make sure no one in any production account, including account administrators, can disable AWS CloudTrail. What should it do?

<details><summary>Answer</summary>

Attach an **SCP** denying `cloudtrail:StopLogging` / `DeleteTrail` / `UpdateTrail` to the Production OU. SCPs apply to every principal in member accounts, including root and admins.

</details>

**Q2.** An SCP attached to the root of the organization denies `ec2:*` in `ap-southeast-2`. A user in the management account launches an instance there successfully. Why?

<details><summary>Answer</summary>

**SCPs don't apply to the management account.**

</details>

**Q3.** A company has 40 accounts and uses Microsoft Entra ID. Employees should sign in once and reach the accounts their role allows. What's the solution with the least operational overhead?

<details><summary>Answer</summary>

**IAM Identity Center** with Entra ID as the external identity provider (SAML + SCIM provisioning), and **permission sets** assigned to groups per account.

</details>
