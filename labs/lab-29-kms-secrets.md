# Lab 29 — Encryption & Secrets: KMS, Secrets Manager & Parameter Store

| | |
|---|---|
| **Module** | 8 — Data Protection, Monitoring & Threat Detection |
| **Exam mapping** | Domain 1 · Task 1.3 (data security controls: encryption, key management, rotation) · Task 1.2 (secrets for workloads) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 75 minutes |
| **Estimated cost** | ~$1 (customer-managed KMS keys at $1/month prorated, secrets at $0.40/month prorated) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-28-ecs-fargate.md) · [Back to index](../README.md) · [Next lab ➡](lab-30-cloudwatch-cloudtrail-config.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Compare **AWS owned**, **AWS managed**, and **customer managed** KMS keys, and **symmetric vs asymmetric** keys.
- Write **key policies** that separate key administrators from key users, and use **grants**.
- Do **envelope encryption** with data keys.
- Configure **automatic key rotation**, **multi-Region keys**, and scheduled deletion.
- Choose between **Secrets Manager** and **SSM Parameter Store** for configuration and credentials.

## KMS key types

| | AWS owned | AWS managed (`aws/s3`, `aws/rds`…) | Customer managed |
|---|---|---|---|
| Visible in your account | No | Yes | Yes |
| You control the key policy | No | No | **Yes** |
| Rotation | AWS | Automatic yearly (can't change) | Optional, **90–2,560 days**, plus on-demand |
| Cross-account use | No | **No** | **Yes** (key policy) |
| Cost | Free | Free (pay per API call) | **$1/month** + API calls |

## Secrets Manager vs Parameter Store

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Purpose | Credentials, API keys | Config values, plus secrets as `SecureString` |
| **Automatic rotation** | **Yes** (built-in for RDS, Redshift, DocumentDB; Lambda for others) | No (DIY) |
| Cross-region replication | **Yes** | No |
| Cost | $0.40/secret/month + API calls | **Standard tier free**, Advanced $0.05/month |
| Size | 64 KB | 4 KB (standard), 8 KB (advanced) |
| Hierarchy / paths | — | **Yes** (`/app/prod/db/url`) |

---

## Part A — Console

### A1. Create a customer-managed key with separated duties

- [ ] Open **KMS → Customer managed keys → Create key**: **Symmetric**, **Encrypt and decrypt**.
  - Advanced: key material origin **KMS**, **Single-Region**
  - Alias `saa-app-key`, description `SAA lab key`
  - **Key administrators:** your admin user (can manage the key but not use it to encrypt or decrypt data)
  - **Key users:** role `saa-ssm-ec2-role` (from Lab 12)
- [ ] Open the key's **Key policy** tab and switch to the **policy view**. Identify the three statements: account root delegation, key administrators, and key users.
- [ ] Open **Key rotation** and turn on automatic rotation with a period of **365 days**.

### A2. Use the key with S3 and EBS

- [ ] Create bucket `saa-lab29-<suffix>` with default encryption **SSE-KMS → `saa-app-key`** and **Bucket Key** on.
- [ ] Upload a file. Then open **CloudTrail → Event history**, filter **Event source = kms.amazonaws.com**, and find the `GenerateDataKey` event made on your behalf by S3.

### A3. Secrets Manager

- [ ] Open **Secrets Manager → Store a new secret → Other type of secret**:
  - Key/value: `username = appuser`, `password = S3cr3t!Initial`, `host = db.example.internal`
  - Encryption key `saa-app-key`, name `saa/app/db-credentials`
  - Rotation: look at the options (it needs a Lambda rotation function for "other" secret types; for RDS secrets it's one click). **Don't turn it on** for this generic secret.
- [ ] Choose **Replicate secret** → **us-west-2**, with encryption key **aws/secretsmanager** (`saa-app-key` is a single-Region key, so it can't be used in us-west-2). Replication is for multi-region apps and DR.
- [ ] Open **Overview → Retrieve secret value**.

### A4. Parameter Store

- [ ] Open **Systems Manager → Parameter Store → Create parameter** and create:
  - `/saa/app/prod/log_level`: **String**, `INFO`
  - `/saa/app/prod/api_key`: **SecureString**, KMS key `saa-app-key`, value `abc123-secret`
  - `/saa/app/prod/allowed_ips`: **StringList**, `10.0.0.0/16,192.168.0.0/24`
- [ ] Open `api_key` and toggle **Show decrypted value**.

---

## Part B — CLI

```bash
KEY_ID=$(aws kms describe-key --key-id alias/saa-app-key --query KeyMetadata.KeyId --output text)
```

### B1. Direct encrypt and decrypt (up to 4 KB)

- [ ] Encrypt a small secret:

```bash
echo -n "my small secret" > plain.txt
aws kms encrypt --key-id alias/saa-app-key --plaintext fileb://plain.txt \
  --encryption-context app=saa,env=lab --query CiphertextBlob --output text | base64 --decode > cipher.bin
```

- [ ] Decrypt it. The encryption context **must match**:

```bash
aws kms decrypt --ciphertext-blob fileb://cipher.bin --encryption-context app=saa,env=lab \
  --query Plaintext --output text | base64 --decode; echo
```

- [ ] Try a mismatched context. It fails with `InvalidCiphertextException`:

```bash
aws kms decrypt --ciphertext-blob fileb://cipher.bin --encryption-context app=other 2>&1 | tail -1
```

> Encryption context is **additional authenticated data**, and it's logged in CloudTrail, which makes it useful for auditing and for policy conditions (`kms:EncryptionContext:app`).

### B2. Envelope encryption

- [ ] Get a data key: a plaintext copy plus a copy encrypted under the KMS key:

```bash
aws kms generate-data-key --key-id alias/saa-app-key --key-spec AES_256 > datakey.json
jq -r .Plaintext datakey.json | base64 --decode > dk.bin
jq -r .CiphertextBlob datakey.json | base64 --decode > dk.enc
```

- [ ] Encrypt a large file locally with the plaintext data key, then destroy the plaintext key:

```bash
dd if=/dev/urandom of=big.dat bs=1M count=20 2>/dev/null
openssl enc -aes-256-cbc -pbkdf2 -in big.dat -out big.dat.enc -pass file:dk.bin
rm dk.bin datakey.json
```

- [ ] Later, ask KMS to decrypt the data key, then decrypt the file locally:

```bash
aws kms decrypt --ciphertext-blob fileb://dk.enc --query Plaintext --output text | base64 --decode > dk.bin
openssl enc -d -aes-256-cbc -pbkdf2 -in big.dat.enc -out big.dat.dec -pass file:dk.bin
cmp big.dat big.dat.dec && echo "Envelope decryption OK"
```

> That's exactly how S3, EBS, and RDS use KMS. Only the small data key goes to KMS, never your bulk data.

### B3. Grants (temporary, programmatic permission)

- [ ] Grant a role permission to decrypt, then revoke it:

```bash
GRANT_ID=$(aws kms create-grant --key-id $KEY_ID \
  --grantee-principal arn:aws:iam::$ACCOUNT_ID:role/saa-ssm-ec2-role \
  --operations Decrypt --query GrantId --output text)
aws kms list-grants --key-id $KEY_ID --query 'Grants[].{Id:GrantId,Ops:Operations,Grantee:GranteePrincipal}'
aws kms revoke-grant --key-id $KEY_ID --grant-id $GRANT_ID
```

### B4. Rotation details and on-demand rotation

- [ ] Check the rotation settings and rotate now:

```bash
aws kms get-key-rotation-status --key-id $KEY_ID
aws kms rotate-key-on-demand --key-id $KEY_ID
aws kms list-key-rotations --key-id $KEY_ID
```

> Rotation keeps the **same key ID and ARN**. Old key material is kept so existing ciphertext still decrypts. Imported key material (BYOK) has to be rotated by hand.

### B5. Multi-Region key

- [ ] Create a primary key and replicate it:

```bash
MRK=$(aws kms create-key --multi-region --description "SAA multi-region key" --query KeyMetadata.KeyId --output text)
aws kms replicate-key --key-id $MRK --replica-region us-west-2
```

- [ ] Encrypt in us-east-1 and decrypt in us-west-2 with the replica:

```bash
aws kms encrypt --key-id $MRK --plaintext fileb://plain.txt --query CiphertextBlob --output text | base64 --decode > mrk.bin
aws kms decrypt --region us-west-2 --key-id $MRK --ciphertext-blob fileb://mrk.bin --query Plaintext --output text | base64 --decode; echo
```

> **Multi-Region keys** share key material across regions, so you can encrypt in one region and decrypt in another without re-encrypting. Useful for global DynamoDB tables or client-side encryption in DR.

### B6. Secrets Manager from the CLI

- [ ] Read the secret:

```bash
aws secretsmanager get-secret-value --secret-id saa/app/db-credentials --query SecretString --output text | jq .
```

- [ ] Put a new version, which moves the `AWSCURRENT` label:

```bash
aws secretsmanager put-secret-value --secret-id saa/app/db-credentials \
  --secret-string '{"username":"appuser","password":"N3w!Rotated","host":"db.example.internal"}'
aws secretsmanager list-secret-version-ids --secret-id saa/app/db-credentials \
  --query 'Versions[].{Id:VersionId,Stages:VersionStages}' --output table
```

- [ ] Add a resource policy that denies access except from one role:

```bash
aws secretsmanager put-resource-policy --secret-id saa/app/db-credentials --block-public-policy --resource-policy "{
 \"Version\":\"2012-10-17\",
 \"Statement\":[{\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"*\",
 \"Condition\":{\"StringNotLike\":{\"aws:PrincipalArn\":[\"arn:aws:iam::$ACCOUNT_ID:role/saa-ssm-ec2-role\",\"arn:aws:iam::$ACCOUNT_ID:user/admin-*\"]}}}]}"
```

### B7. Parameter Store from the CLI

- [ ] Write a parameter, then read the whole path:

```bash
aws ssm put-parameter --name /saa/app/prod/db_url --type String --value "jdbc:mysql://db:3306/app" --overwrite
aws ssm get-parameters-by-path --path /saa/app/prod --recursive --with-decryption \
  --query 'Parameters[].{Name:Name,Type:Type,Value:Value}' --output table
```

- [ ] Look at the version history:

```bash
aws ssm get-parameter-history --name /saa/app/prod/log_level --query 'Parameters[].{V:Version,Value:Value}'
```

- [ ] Reference a secret **through** Parameter Store, which is handy for apps that only speak SSM:

```bash
aws ssm get-parameter --name /aws/reference/secretsmanager/saa/app/db-credentials --with-decryption --query Parameter.Value --output text
```

---

## Verify

- [ ] Encrypt and decrypt with encryption context works, and a mismatched context fails.
- [ ] Envelope encryption round-tripped a 20 MB file.
- [ ] The secret has several versions, and you read Parameter Store by path with decryption.

## Exam tips

- **"Automatically rotate database credentials"** → **Secrets Manager** (native RDS rotation).
- **"Store configuration values hierarchically, free"** → **Parameter Store** (standard).
- **"Control and audit the encryption key, rotate it, use it across accounts"** → **customer managed KMS key**. AWS managed keys **can't be shared across accounts**.
- **"FIPS 140-3 Level 3, single-tenant HSM, full control of keys"** → **CloudHSM** (or a KMS custom key store backed by CloudHSM).
- **"Keys must stay outside AWS"** → **KMS External Key Store (XKS)**.
- **"Share an encrypted EBS snapshot or AMI with another account"** → use a **customer managed key** and grant the other account access in the key policy.
- Deleting a key needs a **7–30 day waiting period**. **Disable** the key first to see what breaks.

## Challenge (optional)

- [ ] Write a key-policy condition with `kms:ViaService = s3.us-east-1.amazonaws.com` so the key can **only** be used through S3, not by direct `kms:Decrypt` calls.

## Cleanup

- [ ] Delete the secret (and its replica) and the parameters:

```bash
aws secretsmanager remove-regions-from-replication --secret-id saa/app/db-credentials --remove-replica-regions us-west-2
aws secretsmanager delete-secret --secret-id saa/app/db-credentials --force-delete-without-recovery
aws ssm delete-parameters --names /saa/app/prod/log_level /saa/app/prod/api_key /saa/app/prod/allowed_ips /saa/app/prod/db_url
```

- [ ] Delete the bucket:

```bash
aws s3 rb s3://saa-lab29-<suffix> --force
```

- [ ] Schedule deletion of the keys (7 days minimum; delete replicas first):

```bash
aws kms schedule-key-deletion --region us-west-2 --key-id $MRK --pending-window-in-days 7
aws kms schedule-key-deletion --key-id $MRK --pending-window-in-days 7
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
```

- [ ] `rm -f plain.txt cipher.bin dk.bin dk.enc big.dat* mrk.bin`

---

## Knowledge check

**Q1.** A company must encrypt data in S3 and RDS, rotate keys every 180 days, and let a partner account decrypt specific objects. Which key type?

<details><summary>Answer</summary>

A **customer managed KMS key** with a 180-day rotation period and a key policy that grants the partner account `kms:Decrypt`.

</details>

**Q2.** An app needs a database password that's rotated automatically every 30 days without app downtime. What should you use?

<details><summary>Answer</summary>

**AWS Secrets Manager** with automatic rotation. The app retrieves the secret at runtime, ideally with client-side caching.

</details>

**Q3.** Why does KMS use envelope encryption instead of encrypting large data directly?

<details><summary>Answer</summary>

KMS `Encrypt` is limited to **4 KB**, and sending bulk data to KMS would be slow and costly. A **data key** encrypts the data locally, and only the small data key is encrypted by the KMS key.

</details>
