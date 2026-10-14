# Lab 02 — AWS CLI Setup, Profiles & Querying

| | |
|---|---|
| **Module** | 1 — Getting Started & Account Foundations |
| **Exam mapping** | Domain 1 · Task 1.1 (credentials, temporary vs long-term access) |
| **Difficulty** | ★☆☆ Foundational |
| **Time** | 45 minutes |
| **Estimated cost** | $0 |

- [ ] **Lab complete**

[⬅ Previous lab](lab-01-account-baseline.md) · [Back to index](../README.md) · [Next lab ➡](lab-03-iam-users-groups-policies.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Install and configure **AWS CLI v2** with named profiles.
- Explain the **credential provider chain**: where the CLI and SDKs look for credentials, in order.
- Format and filter output with `--output` and **JMESPath** `--query`.
- Use pagination, `--dry-run`, waiters, and `help` well.
- Set the shell variables that later labs rely on.

## Exam context

The exam doesn't test CLI syntax, but it **does** test credential handling. "Hard-code access keys in the application" is always wrong. The right answer is an **IAM role** (instance profile, task role, Lambda execution role), because roles hand out short-lived credentials automatically.

---

## Prerequisites

- Lab 01 finished (admin user with MFA).

---

## Part A — Console

### A1. Create access keys for local CLI use

> Long-term access keys are fine for a personal lab laptop. In production, use **IAM Identity Center** (`aws configure sso`), covered in Lab 06.

- [ ] Sign in as `admin-<yourname>` and open **IAM → Users → admin-<yourname> → Security credentials**.
- [ ] Under **Access keys**, choose **Create access key**, then **Command Line Interface (CLI)**, and acknowledge the recommendation.
- [ ] Download the `.csv` or copy the keys. **You can't see the secret again.**
- [ ] Note the console's recommendation of alternatives (CloudShell, IAM Identity Center). That recommendation is itself exam material.

### A2. Explore CloudShell

- [ ] Open **CloudShell** and run `aws --version`. CloudShell comes with CLI v2 preinstalled and is already authenticated.
- [ ] Use **Actions → Upload file** and **Download file**. CloudShell keeps 1 GB of `$HOME` storage per region.

---

## Part B — CLI

### B1. Install CLI v2

- [ ] Install for your OS:

```bash
# macOS
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Linux x86_64
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Windows: download and run https://awscli.amazonaws.com/AWSCLIV2.msi

aws --version   # expect aws-cli/2.x
```

- [ ] Install **jq**, which many labs use to parse JSON output:

```bash
brew install jq            # macOS
sudo dnf install -y jq     # Amazon Linux / Fedora
sudo apt-get install -y jq # Debian / Ubuntu
jq --version
```

### B2. Configure a default and a named profile

- [ ] Configure the default profile:

```bash
aws configure
# AWS Access Key ID:     AKIA...
# AWS Secret Access Key: ****
# Default region name:   us-east-1
# Default output format: json
```

- [ ] Add a named profile for a second region:

```bash
aws configure --profile saa-west
# same keys, region us-west-2
```

- [ ] Look at the files the CLI wrote:

```bash
cat ~/.aws/config
cat ~/.aws/credentials   # secrets live here; never commit this file
```

- [ ] Switch between profiles:

```bash
aws sts get-caller-identity
aws sts get-caller-identity --profile saa-west
AWS_PROFILE=saa-west aws configure get region
```

### B3. The credential provider chain

The CLI and SDKs look for credentials in roughly this order and use the **first** set they find:

1. Command-line options (`--profile`, `--region`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_PROFILE`)
3. Web identity token (`AWS_WEB_IDENTITY_TOKEN_FILE`, used by EKS IRSA)
4. Shared credentials file `~/.aws/credentials`, then config file `~/.aws/config` (including SSO and `credential_process`)
5. Container credentials (ECS task role)
6. EC2 instance metadata service (instance profile role)

- [ ] Prove that environment variables beat the config files:

```bash
AWS_REGION=eu-west-1 aws configure list
# the "region" row now shows source "env"
```

### B4. Output formats and JMESPath queries

- [ ] Compare the output formats:

```bash
aws ec2 describe-regions --output table
aws ec2 describe-regions --output text
aws ec2 describe-regions --output yaml
```

- [ ] Filter with `--query`:

```bash
# list only region names
aws ec2 describe-regions --query 'Regions[].RegionName' --output text

# AZ names and IDs in your region
aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[].{Name:ZoneName,Id:ZoneId,State:State}' --output table

# server-side filter (--filters) vs client-side filter (--query)
aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].{Id:ImageId,Name:Name}'
```

> **Note:** an AZ *name* like `us-east-1a` maps to a different physical AZ in each account. The **AZ ID** (`use1-az1`) is the same everywhere. You need it when coordinating across accounts (e.g. shared subnets, PrivateLink).

- [ ] Get the latest Amazon Linux 2023 AMI from **SSM public parameters**, the method later labs use:

```bash
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text
```

### B5. Pagination, dry-run, waiters, and help

- [ ] Try each one:

```bash
# limit results and page manually
aws iam list-policies --scope AWS --max-items 5 --query 'Policies[].PolicyName'

# --dry-run checks permissions without creating anything (EC2 only)
aws ec2 run-instances --dry-run --image-id ami-12345678 --instance-type t3.micro
# "DryRunOperation" means you WOULD be allowed; "UnauthorizedOperation" means you would not

# waiters block until a state is reached
aws ec2 wait --help

# built-in docs
aws s3api put-bucket-versioning help
```

### B6. Set lab-wide shell variables

Later labs assume these are set. Add them to `~/.zshrc` or `~/.bashrc`:

- [ ] Add the variables:

```bash
cat >> ~/.zshrc <<'EOF'
# --- SAA-C03 labs ---
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text 2>/dev/null)
export SUFFIX=<pick-a-short-unique-string-eg-initials-and-4-digits>
EOF
source ~/.zshrc
echo "$AWS_REGION $ACCOUNT_ID $SUFFIX"
```

- [ ] *(Optional)* Turn on auto-prompt to learn commands interactively: `export AWS_CLI_AUTO_PROMPT=on-partial`.

---

## Verify

- [ ] `aws sts get-caller-identity` returns your admin user ARN.
- [ ] `aws configure list-profiles` shows `default` and `saa-west`.
- [ ] `echo $SUFFIX` prints your unique suffix.

## Exam tips

- Apps on EC2, ECS, Lambda, and EKS should get credentials from a **role**, never from embedded keys.
- **Access keys exposed on GitHub?** Deactivate and rotate them, review CloudTrail for misuse, and switch to roles.
- **IAM Access Analyzer**, **IAM credential reports**, and **access advisor** (last-accessed data) help you find and remove unused credentials and permissions.

## Challenge (optional)

- [ ] Write a single `--query` that lists all **IAM users** and the date they were created, sorted oldest first.

<details><summary>Hint</summary>

`aws iam list-users --query 'sort_by(Users,&CreateDate)[].[UserName,CreateDate]' --output table`

</details>

## Cleanup

- [ ] Nothing to remove. Keep the CLI configured.
- [ ] Delete the downloaded access key `.csv` file once the keys are in `~/.aws/credentials`.

---

## Knowledge check

**Q1.** An application on EC2 needs to read from an S3 bucket. A developer suggests storing access keys in the application's config file. What's the best practice?

<details><summary>Answer</summary>

Create an **IAM role** with least-privilege S3 read access and attach it to the instance as an **instance profile**. The SDK fetches temporary credentials from the instance metadata service and rotates them automatically.

</details>

**Q2.** Two teams in different AWS accounts both say "our database is in us-east-1a" but get very different latency results. Why?

<details><summary>Answer</summary>

AZ names map to physical AZs independently in each account. To compare locations across accounts, use **AZ IDs** (e.g. `use1-az2`).

</details>
