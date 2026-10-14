# Lab 05 — Advanced Policies: Conditions, Resource Policies & Boundaries

| | |
|---|---|
| **Module** | 2 — Identity & Access Management |
| **Exam mapping** | Domain 1 · Task 1.1 (policy evaluation, least privilege, delegation) · Task 1.3 (access to data) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$0 (S3 storage pennies) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-04-iam-roles-sts.md) · [Back to index](../README.md) · [Next lab ➡](lab-06-organizations-scps.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Walk through **IAM policy evaluation logic** across every policy type.
- Write **condition keys** for IP, MFA, tags (ABAC), region, and VPC endpoint.
- Compare **identity-based** and **resource-based** policies, and know when cross-account access needs both.
- Delegate safely with **permission boundaries**.
- Validate policies with the **IAM Policy Simulator** and **IAM Access Analyzer**.

## Policy evaluation logic (memorise this)

```
                    ┌──────────────────────────────┐
Request ──────────► │ Any explicit DENY anywhere?  │── yes ──► DENY
                    └──────────────┬───────────────┘
                                   no
                    ┌──────────────▼───────────────┐
                    │ SCP / RCP allows? (if in Org)│── no ───► DENY
                    └──────────────┬───────────────┘
                                   yes
                    ┌──────────────▼───────────────┐
                    │ Resource-based policy allows │── yes ──► ALLOW*
                    │ (same account)?              │
                    └──────────────┬───────────────┘
                                   no
                    ┌──────────────▼───────────────┐
                    │ Permission boundary allows?  │── no ───► DENY
                    └──────────────┬───────────────┘
                                   yes
                    ┌──────────────▼───────────────┐
                    │ Session policy allows?       │── no ───► DENY
                    └──────────────┬───────────────┘
                                   yes
                    ┌──────────────▼───────────────┐
                    │ Identity policy allows?      │── no ───► DENY (implicit)
                    └──────────────┬───────────────┘
                                   yes ──► ALLOW
```

\* **Cross-account:** the identity policy in the caller's account **and** the resource policy in the target account must both allow.

**Effective permissions = the intersection of SCPs ∩ boundaries ∩ session policies ∩ identity policies** (with the resource-policy nuance above).

---

## Part A — Console

### A1. Resource-based policy: S3 bucket policy

- [ ] Create a bucket `saa-lab05-<suffix>` (default settings).
- [ ] Open **Permissions → Bucket policy → Edit** and add a policy that **denies** any request not made over HTTPS and **denies** deletes from anyone without MFA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::saa-lab05-<suffix>", "arn:aws:s3:::saa-lab05-<suffix>/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "DenyDeleteWithoutMFA",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::saa-lab05-<suffix>/*",
      "Condition": { "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" } }
    }
  ]
}
```

- [ ] Upload a file, then try to delete it from the console. Whether it works depends on whether your console session used MFA. Think through why.

### A2. Attribute-based access control (ABAC) with tags

- [ ] Create two EC2 instances (t3.micro, no key pair) tagged `Project=blue` and `Project=green`. You'll stop them shortly.
- [ ] Create a role `saa-abac-blue` (trusted entity: this account) with this inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "ec2:Describe*", "Resource": "*" },
    {
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": { "StringEquals": { "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}" } }
    }
  ]
}
```

- [ ] Add the tag `Project=blue` **to the role itself**.
- [ ] Switch to the role. Stopping the **blue** instance should work. Stopping the **green** one should be denied.

> ABAC scales better than RBAC. New projects need new *tags*, not new *policies*.

### A3. Permission boundary: safe delegation

Scenario: let a junior admin create roles for Lambda **without** letting them grant more power than they have themselves.

- [ ] Create a customer-managed policy `saa-boundary-s3-logs`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:*", "logs:*", "cloudwatch:*"], "Resource": "*" }
  ]
}
```

- [ ] Create a user `saa-junior-admin` and give them `IAMFullAccess` + `AmazonS3FullAccess`.
- [ ] Under **Permissions boundary → Set boundary**, pick `saa-boundary-s3-logs`.
- [ ] Open the **IAM Policy Simulator** (`https://policysim.aws.amazon.com`), select `saa-junior-admin`, and simulate:
  - `s3:CreateBucket`: **allowed** (identity ✔ and boundary ✔)
  - `iam:CreateRole`: **denied** (identity ✔ but boundary ✘)
  - `ec2:RunInstances`: **denied** (neither allows it)

### A4. IAM Access Analyzer

- [ ] Open **IAM → Access analyzer → Create analyzer**, choose Type **External access**, and set the zone of trust to **Current account**.
- [ ] Review the findings. Any resources shared publicly or with other accounts show up here.
- [ ] Open **IAM → Policies → Create policy**, paste a deliberately loose policy (`"Action":"*","Resource":"*"`), and look at the **policy validation** warnings from Access Analyzer.

---

## Part B — CLI

### B1. Simulate a custom policy before deploying it

- [ ] Write a policy that allows actions only from a given source IP:

```bash
cat > ip-restricted.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:ListAllMyBuckets",
    "Resource": "*",
    "Condition": { "IpAddress": { "aws:SourceIp": "203.0.113.0/24" } }
  }]
}
EOF
```

- [ ] Simulate it from inside and outside the allowed range:

```bash
aws iam simulate-custom-policy --policy-input-list file://ip-restricted.json \
  --action-names s3:ListAllMyBuckets \
  --context-entries "ContextKeyName=aws:SourceIp,ContextKeyValues=203.0.113.10,ContextKeyType=ip" \
  --query 'EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}'

aws iam simulate-custom-policy --policy-input-list file://ip-restricted.json \
  --action-names s3:ListAllMyBuckets \
  --context-entries "ContextKeyName=aws:SourceIp,ContextKeyValues=198.51.100.5,ContextKeyType=ip" \
  --query 'EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}'
```

> Expected results: `allowed`, then `implicitDeny`.

### B2. Simulate a real principal, including the boundary

- [ ] Run the simulator against the junior admin:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:user/saa-junior-admin \
  --action-names s3:CreateBucket iam:CreateRole ec2:RunInstances \
  --query 'EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision,Boundary:PermissionsBoundaryDecisionDetail.AllowedByPermissionsBoundary}' \
  --output table
```

### B3. Enforce that delegated admins attach the boundary

A common pattern is to let `saa-junior-admin` create roles **only if** they attach the boundary policy to each new role.

- [ ] Write the delegation policy:

```bash
cat > delegate-with-boundary.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRolesOnlyWithBoundary",
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:PutRolePermissionsBoundary"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "iam:PermissionsBoundary": "arn:aws:iam::${ACCOUNT_ID}:policy/saa-boundary-s3-logs" }
      }
    },
    {
      "Sid": "NoBoundaryTampering",
      "Effect": "Deny",
      "Action": ["iam:DeleteUserPermissionsBoundary", "iam:DeleteRolePermissionsBoundary", "iam:CreatePolicyVersion", "iam:SetDefaultPolicyVersion"],
      "Resource": ["arn:aws:iam::${ACCOUNT_ID}:policy/saa-boundary-s3-logs", "arn:aws:iam::${ACCOUNT_ID}:user/saa-junior-admin"]
    }
  ]
}
EOF
```

- [ ] Simulate `iam:CreateRole` with and without the boundary context key:

```bash
aws iam simulate-custom-policy --policy-input-list file://delegate-with-boundary.json \
  --action-names iam:CreateRole \
  --context-entries "ContextKeyName=iam:PermissionsBoundary,ContextKeyValues=arn:aws:iam::${ACCOUNT_ID}:policy/saa-boundary-s3-logs,ContextKeyType=string" \
  --query 'EvaluationResults[0].EvalDecision'

aws iam simulate-custom-policy --policy-input-list file://delegate-with-boundary.json \
  --action-names iam:CreateRole \
  --query 'EvaluationResults[0].EvalDecision'
```

### B4. Validate policies and list Access Analyzer findings

- [ ] Run policy validation:

```bash
aws accessanalyzer validate-policy --policy-type IDENTITY_POLICY \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:*","Resource":"*"},{"Effect":"Allow","Action":"iam:PassRole","Resource":"*"}]}' \
  --query 'findings[].{Type:findingType,Issue:issueCode}' --output table
```

- [ ] List the analyzer's findings:

```bash
ANALYZER_ARN=$(aws accessanalyzer list-analyzers --query 'analyzers[0].arn' --output text)
aws accessanalyzer list-findings --analyzer-arn $ANALYZER_ARN \
  --query 'findings[].{Resource:resource,Type:resourceType,Status:status}' --output table
```

### B5. Generate a least-privilege policy from activity (optional)

> Access Analyzer can **generate** a policy from CloudTrail activity. It needs a trail (Lab 30) and can take several minutes.

```bash
aws accessanalyzer start-policy-generation --help
```

---

## Verify

- [ ] The bucket policy denies plain-HTTP requests (test it: `aws s3 ls s3://saa-lab05-<suffix> --endpoint-url http://s3.amazonaws.com` fails).
- [ ] ABAC lets `saa-abac-blue` manage only blue-tagged instances.
- [ ] The simulator shows the boundary blocking `iam:CreateRole` for the junior admin.

## Exam tips

- **Cross-account S3:** the bucket policy must grant the other account **and** the other account's IAM policy must grant its users. If the uploader owns the objects, use **Object Ownership = Bucket owner enforced** to fix ownership problems.
- **Permission boundaries never grant permissions.** They only cap them.
- Restrict access to "only through our VPC endpoint" with `aws:SourceVpce` in a bucket policy. Restrict to "only our org" with `aws:PrincipalOrgID`.
- **Access Analyzer** = find resources shared outside your zone of trust, and validate and generate policies.

## Challenge (optional)

- [ ] Write a bucket policy that allows access only from principals in your **AWS Organization** (`aws:PrincipalOrgID`) **and** only through a specific VPC endpoint. You'll build the endpoint in Lab 14.

## Cleanup

- [ ] Empty and delete the bucket:

```bash
aws s3 rb s3://saa-lab05-<suffix> --force
```

- [ ] Terminate the blue and green instances:

```bash
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=blue,green" "Name=instance-state-name,Values=running,stopped" \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] Delete the role `saa-abac-blue`, the user `saa-junior-admin` (remove the boundary and policies first), and the policy `saa-boundary-s3-logs`.
- [ ] *(Optional)* Delete the Access Analyzer (keeping it is free).
- [ ] `rm ip-restricted.json delegate-with-boundary.json`

---

## Knowledge check

**Q1.** A user's identity policy grants `s3:*`. Their permission boundary allows only `s3:GetObject`. An SCP allows `s3:*`. Can they run `s3:PutObject`?

<details><summary>Answer</summary>

**No.** Effective permissions are the *intersection*, and the boundary doesn't allow `PutObject`.

</details>

**Q2.** Account A's bucket policy grants Account B's role `s3:GetObject`. Users in Account B still get AccessDenied. What's missing?

<details><summary>Answer</summary>

The role in **Account B** needs an **identity-based policy** that allows `s3:GetObject` on Account A's bucket. Cross-account access needs both sides to allow it.

</details>

**Q3.** A company has hundreds of projects and wants developers to manage only resources belonging to their own project, without writing a policy per project. What should it use?

<details><summary>Answer</summary>

**Attribute-based access control (ABAC):** tag principals and resources with `Project`, then match `aws:PrincipalTag/Project` against `aws:ResourceTag/Project` in one shared policy.

</details>
