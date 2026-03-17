# OpenClaw AWS Security Assistant

> Use OpenClaw as an AI-powered cloud security auditor to automatically detect vulnerabilities across your AWS account — with zero write access.

---

## Demo

<!-- Screenshot: OpenClaw running an audit and returning a findings table -->
<img width="1322" height="2408" alt="image" src="https://github.com/user-attachments/assets/97312b19-ea18-47ea-8fda-8a2b68350dd3" />


---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1: Create the IAM User](#step-1-create-the-iam-user)
- [Step 2: Create the IAM Role](#step-2-create-the-iam-role)
- [Step 3: Install AWS CLI](#step-3-install-aws-cli)
- [Step 4: Configure OpenClaw](#step-4-configure-openclaw)
- [Step 5: Test the Connection](#step-5-test-the-connection)
- [Step 6: Run Security Audits](#step-6-run-security-audits)
- [Audit Prompts](#audit-prompts)
- [Automated Reporting](#automated-reporting)
- [Safety Guardrails](#safety-guardrails)
- [Recommended Audit Schedule](#recommended-audit-schedule)
- [Security Best Practices](#security-best-practices)

---

## Overview

This guide walks you through setting up OpenClaw as a read-only AWS security assistant capable of auditing:

- **IAM** — users, roles, policies, MFA, access keys
- **S3** — public buckets, encryption, versioning, logging
- **EC2 & Networking** — security groups, open ports, VPC flow logs
- **RDS** — public databases, encryption, backups

OpenClaw is given **read-only** access only — it can never modify, delete, or create resources in your account.

---

## Architecture

```
You ──► OpenClaw Agent ──► Read-Only IAM Role ──► AWS APIs
                      │
                      ▼
               Security Report
```

---

## Prerequisites

- An AWS account with admin access (to create the IAM role)
- A Linux/macOS machine with OpenClaw installed
- AWS CLI v2

---

## Step 1: Create the IAM User

Before creating the role, you need a dedicated IAM user that OpenClaw will authenticate as.

### 1a. Create the User

1. Go to **IAM → Users → Create user**
2. Username: `openclaw-user`
3. **Do not** enable console access — this is a programmatic-only user
4. Click **Next**

---

### 1b. Attach an Inline Policy to the User

The user itself needs only one permission — the ability to assume the audit role. Everything else is scoped to the role.

1. Click **Attach policies directly → Create inline policy → JSON tab**
2. Paste the following, replacing `YOUR_ACCOUNT_ID`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/OpenClawAuditRole"
  }]
}
```

3. Policy name: `openclaw-assume-role`
4. Click **Create user**

---

### 1c. Generate Access Keys

1. Click into `openclaw-user`
2. Go to **Security credentials → Access keys → Create access key**
3. Use case: **Command Line Interface (CLI)**
4. Copy the **Access Key ID** and **Secret Access Key**

Save them to your AWS credentials file:

```bash
# ~/.aws/credentials
[openclaw]
aws_access_key_id = AKIA...YOUR_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```

```bash
chmod 600 ~/.aws/credentials
```

---

## Step 2: Create the IAM Role

### 2a. Open IAM in the AWS Console

1. Log into the [AWS Console](https://console.aws.amazon.com)
2. Search for **IAM** in the top search bar
3. Click **Roles** in the left sidebar
4. Click **Create role**




---

### 1b. Set the Trust Policy

1. Select **Custom trust policy**
2. Delete the default JSON and paste the following — replacing `YOUR_ACCOUNT_ID` with your 12-digit AWS account ID (found top-right in the console):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:root" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "openclaw-security-audit" }
    }
  }]
}
```

3. Click **Next**

<!-- Screenshot: Trust policy JSON pasted into the AWS console editor -->
<img width="2102" height="786" alt="image" src="https://github.com/user-attachments/assets/107efcdf-6662-4557-9a5f-55c0374b932c" />


---

### 1c. Attach the Permission Policy

1. Click **Create inline policy** at the bottom of the page
2. Switch to the **JSON tab**
3. Paste the following permission policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OpenClawSecurityAudit",
      "Effect": "Allow",
      "Action": [
        "iam:Get*", "iam:List*", "iam:GenerateCredentialReport",
        "s3:GetBucketAcl", "s3:GetBucketPolicy", "s3:GetBucketPublicAccessBlock",
        "s3:GetBucketLogging", "s3:GetBucketVersioning", "s3:GetEncryptionConfiguration",
        "s3:ListAllMyBuckets",
        "ec2:Describe*",
        "rds:Describe*", "rds:ListTagsForResource",
        "securityhub:Get*", "securityhub:List*",
        "config:Get*", "config:List*", "config:Describe*",
        "cloudtrail:GetTrailStatus", "cloudtrail:DescribeTrails",
        "guardduty:Get*", "guardduty:List*"
      ],
      "Resource": "*"
    }
  ]
}
```

4. Click **Next**

---

### 1d. Name and Create the Role

1. Role name: `OpenClawAuditRole`
2. Description: `Read-only audit role for OpenClaw security assistant`
3. Click **Create role**

<!-- Screenshot: IAM Console with "Create role" button highlighted -->
<img width="2940" height="1536" alt="image" src="https://github.com/user-attachments/assets/32fa3518-2ac0-4d83-8a0a-494eac8e6bc5" />

---

### 1e. Copy the Role ARN

1. Click into the newly created `OpenClawAuditRole`
2. Copy the **ARN** at the top — it looks like:

```
arn:aws:iam::123456789012:role/OpenClawAuditRole
```

> ⚠️ **Keep your ARN private.** It contains your AWS Account ID. Never share it publicly or paste it into chat prompts.

---

## Step 2: Install AWS CLI

The AWS CLI is required for OpenClaw to communicate with AWS APIs.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify the installation:

```bash
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x Linux/x86_64
```

---

## Step 3: Configure OpenClaw

### ⚠️ Never put credentials in a chat prompt

Credentials passed into prompts are logged in conversation history and can be exposed via prompt injection. Always use environment variables or config files.

### Option A: Environment Variables (Recommended)

Set these in your terminal **before** launching OpenClaw:

```bash
export AWS_ROLE_ARN="arn:aws:iam::YOUR_ACCOUNT_ID:role/OpenClawAuditRole"
export AWS_ROLE_SESSION_NAME="openclaw-audit"
export AWS_EXTERNAL_ID="openclaw-security-audit"

# Then launch OpenClaw
openclaw start
```

### Option B: OpenClaw Config File

Save to `~/.openclaw/config.json` and lock down the permissions:

```json
{
  "env": {
    "AWS_ROLE_ARN": "arn:aws:iam::YOUR_ACCOUNT_ID:role/OpenClawAuditRole",
    "AWS_ROLE_SESSION_NAME": "openclaw-audit",
    "AWS_EXTERNAL_ID": "openclaw-security-audit"
  }
}
```

```bash
chmod 600 ~/.openclaw/config.json
```

### Option C: AWS Credentials File

```ini
# ~/.aws/credentials
[openclaw]
role_arn = arn:aws:iam::YOUR_ACCOUNT_ID:role/OpenClawAuditRole
source_profile = default
external_id = openclaw-security-audit
```

```bash
export AWS_PROFILE=openclaw
```

---

### OpenClaw System Prompt

Configure OpenClaw with this system prompt to set its behavior as a security auditor:

```
You are a cloud security expert. When asked to audit AWS, use the AWS CLI
tools available to you. Always use the read-only role configured in your
environment. Never attempt write, delete, or modify operations. Report all
findings with severity ratings: CRITICAL, HIGH, MEDIUM, or LOW.
```

---

## Step 4: Test the Connection

Verify OpenClaw can assume the role successfully:

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::YOUR_ACCOUNT_ID:role/OpenClawAuditRole" \
  --role-session-name "openclaw-audit" \
  --external-id "openclaw-security-audit"
```

A successful response returns temporary credentials with `AccessKeyId`, `SecretAccessKey`, and `SessionToken`. You're ready to audit.

---

## Step 5: Run Security Audits

## Audit Prompts

Use these prompts directly in OpenClaw:

### 🔑 IAM Audit

```
Audit my AWS IAM configuration. Check for:
- Users with console access but no MFA enabled
- Access keys older than 90 days
- Overly permissive policies (AdministratorAccess attached to users)
- Root account usage in the last 30 days
- Roles with wildcard (*) permissions
Output a findings table with severity ratings.
```

### 🪣 S3 Audit

```
Audit all S3 buckets for:
- Public access via ACL or bucket policy
- Missing server-side encryption
- Disabled versioning
- Missing access logging
- Buckets without lifecycle policies
Flag any bucket exposed to the internet as CRITICAL.
```

### 🌐 EC2 & Networking Audit

```
Audit EC2 and networking for:
- Security groups with 0.0.0.0/0 on SSH (port 22) or RDP (port 3389)
- Instances with unnecessary public IPs
- Unattached EBS volumes (data exposure risk)
- VPCs using default security groups
- Missing VPC Flow Logs
```

### 🗄️ RDS Audit

```
Audit RDS databases for:
- Publicly accessible instances
- Unencrypted storage
- Disabled automated backups
- Default master usernames (admin/postgres/root)
- Minor version auto-upgrade disabled
- Missing deletion protection
```

### 🔎 Full Audit

```
Run a full security audit across IAM, S3, EC2, and RDS.
Compile all findings into a report grouped by severity: CRITICAL, HIGH, MEDIUM, LOW.
For each finding include: resource name, issue description, and recommended remediation.
```

---

## Automated Reporting

Use this prompt to generate a structured JSON report:

```
Run a full security audit across IAM, S3, EC2, and RDS.
Compile all findings into a JSON report grouped by severity.
Save it as aws-security-report-YYYY-MM-DD.json
```

Example output structure:

```json
{
  "timestamp": "2026-03-16T10:00:00Z",
  "account": "123456789012",
  "critical": [
    { "resource": "my-public-bucket", "issue": "S3 bucket is publicly accessible", "service": "S3" }
  ],
  "high": [],
  "medium": [],
  "low": []
}
```

---

## Safety Guardrails

Add these as hard rules in OpenClaw's system prompt to prevent any accidental modifications:

```
HARD RULES — never violate these:
1. Never call any AWS API that is not a Describe*, Get*, List*, or Generate* action
2. If asked to fix or remediate something, explain the fix but do NOT execute it
3. Always confirm the role ARN before running any audit
4. Never log or store AWS credentials or account IDs in responses
5. If a CRITICAL finding is detected, surface it immediately before completing the full audit
```

---

## Recommended Audit Schedule

| Frequency | Scope |
|-----------|-------|
| Daily | IAM credential report, GuardDuty findings |
| Weekly | S3 public access, Security Group changes |
| Monthly | Full audit across all 4 domains |
| On-demand | After any infrastructure change |

---

## Security Best Practices

- ✅ Use a **read-only IAM role** — never give OpenClaw write permissions
- ✅ Always set an **ExternalId** on the trust policy to prevent confused deputy attacks
- ✅ Store credentials in **environment variables or config files** — never in chat prompts
- ✅ Set **`chmod 600`** on any config file containing credentials
- ✅ **Rotate** the session name periodically and monitor CloudTrail for audit session activity
- ✅ **Delete conversations** after setup if your ARN or Account ID was mentioned in chat
- ✅ Review OpenClaw's **installed skills** regularly — malicious skills can steal credentials
