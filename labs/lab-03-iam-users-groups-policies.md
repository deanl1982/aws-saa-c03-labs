# Lab 03 — IAM Users, Groups & Policies

| | |
|---|---|
| **Module** | 2 — Identity & Access Management |
| **Exam mapping** | Domain 1 · Task 1.1 (secure access to AWS resources) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 60 minutes |
| **Estimated cost** | $0 (IAM is free) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-02-cli-setup.md) · [Back to index](../README.md) · [Next lab ➡](lab-04-iam-roles-sts.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Tell apart IAM **users**, **groups**, **roles**, and **policies**.
- Compare **AWS managed**, **customer managed**, and **inline** policies, and know when to use each.
- Read and write IAM policy JSON (`Effect`, `Action`, `Resource`, `Condition`).
- Apply **least privilege** and enforce **MFA** with a policy condition.
- Use **access advisor** and **credential reports** to audit access.

## Key concepts

| Concept | Summary |
|---|---|
| User | Long-term identity (a person or a legacy app). Has console password and/or access keys. |
| Group | Collection of users. Permissions attach to the group. **Groups cannot be nested** and are **not principals** (they can't appear in a resource policy). |
| Policy | JSON document. Default is **implicit deny**. An explicit `Deny` **always** wins. |
| AWS managed | Maintained by AWS (e.g. `ReadOnlyAccess`). Convenient but broad. |
| Customer managed | Created by you and reusable. Versioned (up to 5 versions). **Preferred for least privilege.** |
| Inline | Embedded in one identity and deleted with it. Use only for strict one-to-one relationships. |

---

## Scenario

You're onboarding a small team:

- **Developers**: full EC2 access in `us-east-1` only, plus read-only S3.
- **Auditors**: read-only across the account.
- Everyone must use **MFA** before doing anything sensitive.

---

## Part A — Console

### A1. Create groups

- [ ] Open **IAM → User groups → Create group**, name it `saa-auditors`, and attach the AWS managed policy **ReadOnlyAccess**.
- [ ] Create group `saa-developers` with **no policies yet**.

### A2. Create a customer-managed policy

- [ ] Open **IAM → Policies → Create policy → JSON** and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2FullInUsEast1",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:RequestedRegion": "us-east-1" }
      }
    },
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": ["s3:Get*", "s3:List*"],
      "Resource": "*"
    }
  ]
}
```

- [ ] Name it `saa-developer-policy` and create it.
- [ ] Attach `saa-developer-policy` to the `saa-developers` group.

### A3. Create an MFA-enforcement policy

- [ ] Create a policy named `saa-require-mfa` that denies everything except MFA self-management until the user signs in with MFA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSelfManageMFA",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice", "iam:EnableMFADevice", "iam:ListMFADevices",
        "iam:ResyncMFADevice", "iam:GetUser", "iam:ChangePassword", "iam:ListVirtualMFADevices"
      ],
      "Resource": [
        "arn:aws:iam::*:user/${aws:username}",
        "arn:aws:iam::*:mfa/*"
      ]
    },
    {
      "Sid": "DenyAllWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice", "iam:EnableMFADevice", "iam:ListMFADevices",
        "iam:ResyncMFADevice", "iam:GetUser", "iam:ChangePassword", "iam:ListVirtualMFADevices",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
```

- [ ] Attach `saa-require-mfa` to **both** groups.

> **Why `BoolIfExists`?** Requests signed with long-term access keys don't include the `aws:MultiFactorAuthPresent` key at all. `IfExists` makes the deny apply to them as well.

### A4. Create users

- [ ] Create user `saa-dev-alice` with console access and a custom password, and add her to `saa-developers`.
- [ ] Create user `saa-audit-bob` with console access and add him to `saa-auditors`.

### A5. Test the permissions

- [ ] Open a **private/incognito window** and sign in as `saa-dev-alice` *without* MFA. Try opening EC2. You should see access-denied errors everywhere.
- [ ] Register an MFA device for Alice (her *Security credentials* page), sign out, and sign back in with MFA.
- [ ] Switch the console to **us-west-2** and open EC2. You should get errors because of the region condition.
- [ ] Switch back to **us-east-1**. EC2 should now work.
- [ ] Try creating an S3 bucket. It should be denied (read-only).

### A6. Audit access

- [ ] Open **IAM → Users → saa-dev-alice → Access Advisor** to see which services she has used (last-accessed information).
- [ ] Open **IAM → Credential report → Download report**.

---

## Part B — CLI

> Set up some names first:

```bash
DEV_GROUP=saa-cli-developers
DEV_USER=saa-cli-carol
```

### B1. Create the group and customer-managed policy

- [ ] Write the policy file:

```bash
cat > dev-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2FullInUsEast1",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": "us-east-1" } }
    },
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": ["s3:Get*", "s3:List*"],
      "Resource": "*"
    }
  ]
}
EOF
```

- [ ] Create the group and policy, and attach the policy:

```bash
aws iam create-group --group-name $DEV_GROUP

POLICY_ARN=$(aws iam create-policy \
  --policy-name saa-cli-developer-policy \
  --policy-document file://dev-policy.json \
  --query Policy.Arn --output text)
echo $POLICY_ARN

aws iam attach-group-policy --group-name $DEV_GROUP --policy-arn $POLICY_ARN
```

### B2. Create a user with access keys and add them to the group

- [ ] Create the user:

```bash
aws iam create-user --user-name $DEV_USER --tags Key=team,Value=dev
aws iam add-user-to-group --user-name $DEV_USER --group-name $DEV_GROUP

aws iam create-access-key --user-name $DEV_USER > carol-keys.json
cat carol-keys.json
```

- [ ] Configure a profile with Carol's keys:

```bash
aws configure set aws_access_key_id     "$(jq -r .AccessKey.AccessKeyId carol-keys.json)"     --profile carol
aws configure set aws_secret_access_key "$(jq -r .AccessKey.SecretAccessKey carol-keys.json)" --profile carol
aws configure set region us-east-1 --profile carol
```

### B3. Test the permissions as Carol

- [ ] Run these as Carol:

```bash
aws sts get-caller-identity --profile carol

# allowed: EC2 in us-east-1
aws ec2 describe-vpcs --profile carol --query 'Vpcs[].VpcId'

# denied: EC2 in us-west-2 (region condition)
aws ec2 describe-vpcs --profile carol --region us-west-2

# allowed: list S3
aws s3 ls --profile carol

# denied: create bucket
aws s3 mb s3://saa-carol-test-$SUFFIX --profile carol

# denied: IAM
aws iam list-users --profile carol
```

> New access keys can take a few seconds to propagate. If you get `InvalidClientTokenId`, wait 10 seconds and retry.

### B4. Add an inline policy and compare

- [ ] Add an inline policy to Carol:

```bash
aws iam put-user-policy --user-name $DEV_USER --policy-name allow-list-iam-users \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"iam:ListUsers","Resource":"*"}]}'

sleep 10
aws iam list-users --profile carol --query 'Users[].UserName'   # now allowed

# show every policy that affects Carol
aws iam list-attached-user-policies --user-name $DEV_USER
aws iam list-user-policies --user-name $DEV_USER
aws iam list-groups-for-user --user-name $DEV_USER --query 'Groups[].GroupName'
```

### B5. Policy versions

- [ ] Change the policy to also allow `cloudwatch:Describe*`, creating version 2 as the default:

```bash
jq '.Statement += [{"Sid":"CWRead","Effect":"Allow","Action":"cloudwatch:Describe*","Resource":"*"}]' \
  dev-policy.json > dev-policy-v2.json

aws iam create-policy-version --policy-arn $POLICY_ARN \
  --policy-document file://dev-policy-v2.json --set-as-default

aws iam list-policy-versions --policy-arn $POLICY_ARN \
  --query 'Versions[].{V:VersionId,Default:IsDefaultVersion}'
```

- [ ] Roll back to v1:

```bash
aws iam set-default-policy-version --policy-arn $POLICY_ARN --version-id v1
```

### B6. Access advisor (service last accessed)

- [ ] Check which services Carol has used:

```bash
JOB=$(aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::$ACCOUNT_ID:user/$DEV_USER --query JobId --output text)
sleep 5
aws iam get-service-last-accessed-details --job-id $JOB \
  --query 'ServicesLastAccessed[?TotalAuthenticatedEntities>`0`].[ServiceName,LastAuthenticated]' --output table
```

---

## Verify

- [ ] Alice/Carol can use EC2 only in us-east-1.
- [ ] Alice can't do anything without MFA.
- [ ] Bob can read everything but change nothing.

## Exam tips

- **Explicit deny > explicit allow > implicit deny.**
- Policy variables like `${aws:username}` let one policy work for many users (e.g. per-user S3 home folders).
- **Groups are not principals.** You can't name a group in an S3 bucket policy `Principal`.
- Need to give a team a standard set of permissions? Use a **group with customer-managed policies**, not per-user inline policies.
- For many people, **IAM Identity Center** (Lab 06) beats individual IAM users.

## Challenge (optional)

- [ ] Write a policy that lets users manage objects only under `s3://<bucket>/home/${aws:username}/*` and list only their own prefix (the `s3:prefix` condition on `ListBucket`).

## Cleanup

- [ ] Console users and groups:

```bash
# Console-created users (delete login profile + MFA first, or delete via console)
for U in saa-dev-alice saa-audit-bob; do
  aws iam delete-login-profile --user-name $U 2>/dev/null
  for M in $(aws iam list-mfa-devices --user-name $U --query 'MFADevices[].SerialNumber' --output text); do
    aws iam deactivate-mfa-device --user-name $U --serial-number $M
    aws iam delete-virtual-mfa-device --serial-number $M
  done
  for G in $(aws iam list-groups-for-user --user-name $U --query 'Groups[].GroupName' --output text); do
    aws iam remove-user-from-group --user-name $U --group-name $G
  done
  aws iam delete-user --user-name $U
done
```

- [ ] CLI user:

```bash
aws iam delete-user-policy --user-name $DEV_USER --policy-name allow-list-iam-users
aws iam delete-access-key --user-name $DEV_USER --access-key-id "$(jq -r .AccessKey.AccessKeyId carol-keys.json)"
aws iam remove-user-from-group --user-name $DEV_USER --group-name $DEV_GROUP
aws iam delete-user --user-name $DEV_USER
```

- [ ] Groups and policies:

```bash
aws iam detach-group-policy --group-name $DEV_GROUP --policy-arn $POLICY_ARN
aws iam delete-group --group-name $DEV_GROUP
aws iam delete-policy-version --policy-arn $POLICY_ARN --version-id v2
aws iam delete-policy --policy-arn $POLICY_ARN
```

- [ ] Delete the console groups `saa-developers` and `saa-auditors` and the policies `saa-developer-policy` and `saa-require-mfa` (in the console, detach them first).
- [ ] Local files and profile: `rm dev-policy*.json carol-keys.json`, then remove the `[carol]` profile from `~/.aws/credentials` and `~/.aws/config`.

---

## Knowledge check

**Q1.** A user is in two groups. Group A's policy allows `s3:*` on all buckets. Group B's policy explicitly denies `s3:DeleteBucket`. Can the user delete a bucket?

<details><summary>Answer</summary>

**No.** An explicit deny always overrides an allow, whichever policy it comes from.

</details>

**Q2.** A company wants 50 developers to have the same permissions, updated in one place. Which approach needs the least operational effort?

<details><summary>Answer</summary>

Put the developers in an **IAM group** (or an Identity Center permission set) with a **customer-managed policy** attached. Updating the policy updates everyone.

</details>

**Q3.** You attach a policy that denies all actions when `aws:MultiFactorAuthPresent` is `false`, using `Bool`. Users calling the CLI with long-term keys can still act without MFA. Why?

<details><summary>Answer</summary>

Long-term key requests don't include the `aws:MultiFactorAuthPresent` key at all, so a `Bool` condition doesn't match. Use **`BoolIfExists`** so missing keys count as "no MFA".

</details>
