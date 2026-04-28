# IAM Automation Lab  CloudFormation + GitSync

> **Course Lab** | AWS IAM | CloudFormation GitSync Deployment

---

## Overview

This lab provisions AWS IAM resources automatically using a CloudFormation template synced from GitHub. Any push to the `main` branch triggers an automatic stack update no manual deployments needed.

---

## What This Template Creates

### 1. Secrets Manager  One-Time Password
- Auto-generated secure password stored at `iam-lab/temporary-password`
- Shared across all three IAM users
- All users are forced to change it on first login

### 2. IAM Groups & Policies

| Group | Policy | Permissions |
|---|---|---|
| `iam-lab-s3-group` | `iam-lab-s3-group-policy` | List S3 buckets |
| `iam-lab-ec2-group` | `iam-lab-ec2-group-policy` | List/Describe EC2 instances |

### 3. IAM Users

| User | Group | Can List S3 | Can List EC2 | Can Create EC2 |
|---|---|---|---|---|
| `s3-user` | S3 Group | ✅ | ❌ | ❌ |
| `ec2-user1` | EC2 Group | ❌ | ✅ | ❌ |
| `ec2-user2` | EC2 Group | ❌ | ✅ | ✅ |

> `ec2-user2` gets the `iam-lab-ec2-create-policy` attached **directly** — not through the group — so `ec2-user1` does not inherit it.

### 4. Supporting Policies

| Policy | Purpose |
|---|---|
| `iam-lab-ec2-create-policy` | Grants `ec2:RunInstances` — attached only to `ec2-user2` |
| `iam-lab-self-password-policy` | Grants `iam:ChangePassword` — allows first-login password reset |

---

## Repository Structure

```
cloudformation-template-moodle10/
├── iam-lab.yaml                  # Main CloudFormation template
└── deployment-config.yaml        # GitSync deployment config (parameters + tags)
```

---

## GitSync Setup  Step by Step

### Prerequisites
- AWS Account with root or admin access
- GitHub account with the template pushed to `main` branch
- GitHub connection created in AWS (`iam-cloudformation-lab`)

### Step 1 — Create the IAM Role for GitSync

1. Go to **IAM → Roles → Create role**
2. Trusted entity: **AWS Service → CloudFormation**
3. Attach these policies:
   - `IAMFullAccess`
   - `SecretsManagerFullAccess`
   - `AWSCloudFormationFullAccess`
   - `AmazonEventBridgeFullAccess`
4. Name the role: `iam-lab1-role`
5. After creation, go to **Trust relationships → Edit trust policy** and replace with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "cloudformation.amazonaws.com",
          "cloudformation.sync.codeconnections.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Step 2  Create the Stack

1. Go to **CloudFormation → Stacks → Create stack**
2. Select **"Choose an existing template"**
3. Select **"Sync from Git"**
4. Click **Next**

### Step 3  Specify Stack Details

| Field | Value |
|---|---|
| Stack name | `iam-lab-moodle-stack` |
| Repository provider | GitHub |
| Connection | `iam-cloudformation-lab` |
| Repository | `cloudformation-template-moodle10` |
| Branch | `main` |
| Template file path | `iam-lab.yaml` |
| Deployment file path | `deployment-config.yaml` |
| IAM role | `iam-lab1-role` |

### Step 4  Configure Stack Options

- Under **Permissions → IAM role** → click **Remove** (leave blank)
- Leave all other settings as default
- Click **Next → Submit**

---

## Deployment Config File

`deployment-config.yaml` tells GitSync where the template is and what parameters to apply:

```yaml
template-file-path: ./iam-lab.yaml

parameters:
  - parameter-key: PasswordLength
    parameter-value: "16"

tags:
  - key: Project
    value: IAM-Automation-Lab
  - key: ManagedBy
    value: CloudFormation-GitSync
  - key: Environment
    value: Lab
```

---

## How GitSync Works

```
You push to GitHub (main branch)
         ↓
EventBridge detects the change
         ↓
CloudFormation GitSync triggers
         ↓
Stack updates automatically ✅
```

---

## Testing the Lab (Task 2)

### Step 1  Get the Temporary Password

1. Go to **Secrets Manager → iam-lab/temporary-password**
2. Click **Retrieve secret value**
3. Copy the password

### Step 2  Get Your Console Login URL

```
https://<YOUR_ACCOUNT_ID>.signin.aws.amazon.com/console
```

Find your Account ID in the top-right corner of the AWS Console.

### Step 3  Login & Test Each User

Open an **Incognito window** for each user so your root session stays active.

#### s3-user
| Service | Expected Result |
|---|---|
| S3 → List buckets | ✅ Success |
| EC2 → List instances | ❌ Access Denied |

#### ec2-user1
| Service | Expected Result |
|---|---|
| S3 → List buckets | ❌ Access Denied |
| EC2 → List instances | ✅ Success |
| EC2 → Launch instance | ❌ Access Denied |

#### ec2-user2
| Service | Expected Result |
|---|---|
| S3 → List buckets | ❌ Access Denied |
| EC2 → List instances | ✅ Success |
| EC2 → Launch instance | ✅ Success |

### Step 4  Test GitSync Auto-Update

1. Open `iam-lab.yaml` in GitHub
2. Change the password length parameter from `16` to `20`
3. Commit and push to `main`
4. Go to **CloudFormation → iam-lab-moodle-stack**
5. Watch status change: `UPDATE_IN_PROGRESS` → `UPDATE_COMPLETE` ✅

---

## Deliverables Checklist

- [ ] GitHub repo link shared
- [ ] Stack status: `CREATE_COMPLETE`
- [ ] Screenshot: `s3-user` accessing S3 (success)
- [ ] Screenshot: `s3-user` accessing EC2 (denied)
- [ ] Screenshot: `ec2-user1` accessing EC2 (success)
- [ ] Screenshot: `ec2-user1` launching EC2 (denied)
- [ ] Screenshot: `ec2-user2` accessing EC2 (success)
- [ ] Screenshot: `ec2-user2` launching EC2 (success)
- [ ] GitSync auto-update verified

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Role cannot be assumed` | Wrong trust policy | Add `cloudformation.sync.codeconnections.amazonaws.com` to trust policy |
| `Missing Events:PutRule` | Missing EventBridge permissions | Attach `AmazonEventBridgeFullAccess` to the role |
| `Password does not comply` | Missing `iam:ChangePassword` permission | Attach `iam-lab-self-password-policy` to the user |
| Role not showing in dropdown | Role just created | Click the 🔄 refresh button next to the dropdown |

---

## Author

**Alphonse Shema**  
AWS IAM Automation Lab  
Region: `eu-north-1` (Stockholm)
