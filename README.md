# DevOps Accelerator Platform

> A production-style serverless DevOps project built on AWS using Terraform and GitHub Actions —
> demonstrating Infrastructure as Code, CI/CD automation, event-driven architecture, and cloud monitoring.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Tech Stack](#3-tech-stack)
4. [Complete Request Flow](#4-complete-request-flow)
5. [AWS Services — What & Why](#5-aws-services--what--why)
6. [Repository Structure](#6-repository-structure)
7. [Infrastructure (Terraform)](#7-infrastructure-terraform)
8. [CI/CD Pipeline](#8-cicd-pipeline)
9. [Lambda Functions](#9-lambda-functions)
10. [API Gateway](#10-api-gateway)
11. [S3 Buckets](#11-s3-buckets)
12. [CloudFront CDN](#12-cloudfront-cdn)
13. [SNS Notifications](#13-sns-notifications)
14. [CloudWatch Monitoring](#14-cloudwatch-monitoring)
15. [Deployment Guide](#15-deployment-guide)
16. [GitHub Secrets](#16-github-secrets)
17. [Troubleshooting](#17-troubleshooting)
18. [Future Improvements](#18-future-improvements)
19. [Interview Questions & Answers](#19-interview-questions--answers)
20. [Learning Outcomes](#20-learning-outcomes)

---

## 1. Project Overview

This is a **production-grade, fully serverless platform** built entirely on AWS. Users can upload files through a CloudFront-hosted frontend. Those files are processed automatically using event-driven Lambda functions, and the admin gets notified via SNS — all without managing a single server.

The entire infrastructure is provisioned using **Terraform** (remote state in S3 + DynamoDB locking) and deployed automatically via **GitHub Actions** CI/CD pipelines.

**Why I built this:**  
To gain real, end-to-end DevOps experience — provisioning infra as code, automating deployments, building event-driven pipelines, and setting up monitoring. The kind of full-stack cloud ownership that actually matters on the job.

---

## 2. Architecture Diagram

```
User (Browser)
      │
      ▼
┌─────────────────┐
│   CloudFront    │  ← CDN: HTTPS, caching, DDoS protection
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   S3 Bucket     │  ← Static frontend hosting (HTML/CSS/JS)
│   (Frontend)    │
└────────┬────────┘
         │  User submits file upload form
         ▼
┌─────────────────────┐
│    API Gateway      │  ← REST API endpoint (POST)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────┐
│  Lambda:                │  ← Generates a temporary pre-signed URL
│  generate-presigned-url │
└──────────┬──────────────┘
           │  Returns pre-signed URL to browser
           ▼
┌─────────────────────┐
│   S3 Upload Bucket  │  ← File uploaded directly from browser
└──────────┬──────────┘
           │  S3 Event: ObjectCreated
           ▼
┌─────────────────────────┐
│  Lambda:                │  ← Processes the uploaded file
│  process-uploaded-file  │
└───────────┬─────────────┘
            │
   ┌────────┴────────┐
   ▼                 ▼
┌──────────┐   ┌──────────┐
│CloudWatch│   │   SNS    │  ← Email alert sent to admin
│  Logs    │   │  Topic   │
└──────────┘   └──────────┘
```

---

## 3. Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML/CSS hosted on S3 + CloudFront |
| Backend | AWS Lambda (Python) |
| API Layer | Amazon API Gateway (REST) |
| Infrastructure | Terraform (modular + remote backend) |
| CI/CD | GitHub Actions (3 separate pipelines) |
| Monitoring | CloudWatch Logs, Alarms, Dashboard |
| Notifications | Amazon SNS (email) |
| Security | IAM Roles, Bucket Policies, Pre-signed URLs |
| State Management | S3 (state file) + DynamoDB (locking) |

---

## 4. Complete Request Flow

The end-to-end journey of a single file upload:

```
Step 1  →  User opens the website via the CloudFront URL (HTTPS)

Step 2  →  Browser loads static HTML/CSS/JS from S3 (served through CloudFront edge node)

Step 3  →  User fills out the form and selects a file to upload

Step 4  →  Frontend sends: POST /generate-presigned-url → API Gateway

Step 5  →  API Gateway triggers Lambda: generate-presigned-url

Step 6  →  Lambda generates a temporary pre-signed S3 URL (valid ~5 minutes)
            and returns it to the browser

Step 7  →  Browser uploads the file DIRECTLY to S3 using the pre-signed URL
            (No proxy server involved — S3 handles the upload)

Step 8  →  S3 fires an ObjectCreated event → triggers Lambda: process-uploaded-file

Step 9  →  Lambda processes the file:
            - Logs file metadata (name, size, bucket, timestamp) → CloudWatch
            - Publishes notification message → SNS Topic

Step 10 →  SNS delivers email notification to the subscribed admin address

Step 11 →  Admin verifies execution in CloudWatch Logs dashboard
```

---

## 5. AWS Services — What & Why

### Amazon S3 — 3 Buckets

**What it does:** Object storage for Terraform state, frontend files, and uploaded user files.

**Why I used it:**
- **Terraform state bucket** → Centralized, safe remote storage for the `.tfstate` file; no local file to lose
- **Frontend bucket** → Static site hosting without needing a web server
- **Upload bucket** → Receives files via pre-signed URLs and fires events to Lambda

---

### AWS Lambda

**What it does:** Runs Python code only when triggered — no always-on server.

**Why Lambda over EC2:**
- No server to provision, patch, or babysit
- Scales automatically from 0 to thousands of concurrent executions
- Costs nothing when idle — pay only for execution time
- Naturally event-driven: API Gateway triggers one function; S3 triggers another

---

### Amazon API Gateway

**What it does:** Exposes a REST API endpoint the frontend JavaScript can call.

**Why I used it:**
- Provides a clean HTTP interface (`POST /generate-presigned-url`)
- Handles CORS, throttling, and request/response transformation
- Fully decouples the frontend from Lambda's internal ARN
- Easy to secure later with API keys or Cognito authorizers

---

### CloudFront (CDN)

**What it does:** Serves frontend files globally through AWS edge locations.

**Why CloudFront over direct S3 access:**

| Feature | Direct S3 URL | CloudFront |
|---------|:---:|:---:|
| HTTPS support | ❌ | ✅ |
| Global edge caching | ❌ | ✅ |
| DDoS protection (AWS Shield) | ❌ | ✅ |
| Custom domain support | Complex | Easy |
| Hides actual S3 URL | ❌ | ✅ |

---

### Amazon SNS

**What it does:** Sends email notifications when a file is processed.

**Why SNS over calling an email API directly from Lambda:**
- Decoupled: Lambda just publishes to a topic — it doesn't care who's listening
- Adding more subscribers (Slack, SMS, another email) requires zero Lambda changes
- Follows the open/closed principle for notification systems

---

### CloudWatch

**What it does:** Collects and stores logs and metrics from Lambda automatically.

**Why it matters:**
- No extra setup needed — Lambda ships logs here out of the box
- Central place to debug errors, view execution times, and track file processing
- Can trigger alarms on error rate thresholds (future improvement)

---

### IAM (Roles & Policies)

**What it does:** Defines what each service is allowed to do and nothing more.

**Why least privilege matters:**
- Lambda execution role has only the S3/SNS/CloudWatch permissions it actually needs
- GitHub Actions uses a dedicated IAM user with scoped deployment permissions
- If any component is compromised, the blast radius is minimal

---

### DynamoDB — State Locking

**What it does:** Prevents two Terraform runs from writing state simultaneously.

**Why it matters:**
- Without locking: two concurrent `terraform apply` runs can read the same state, make conflicting changes, and corrupt the `.tfstate` file
- DynamoDB's conditional writes make it ideal for distributed locking — only one process holds the lock at a time

---

## 6. Repository Structure

```
DevOps-Accelerator-Project/
│
├── .github/
│   └── workflows/
│       ├── backend-deploy.yml      # Packages and deploys Lambda functions
│       ├── frontend.yml            # Syncs frontend to S3, invalidates CloudFront cache
│       └── terraform.yml           # Runs: init → validate → plan → apply
│
├── backend/
│   └── lambda/
│       ├── generate-presigned-url/
│       │   ├── main.py             # Generates S3 pre-signed URL, returns to browser
│       │   └── lambda.zip          # Packaged deployment artifact
│       └── process-uploaded-file/
│           ├── main.py             # Logs file metadata, publishes SNS notification
│           └── lambda.zip          # Packaged deployment artifact
│
├── frontend/
│   └── index.html                  # Static upload form (calls API Gateway)
│
├── gigs/                           # Plug-in extension modules
│   ├── project-generator/
│   └── qa-bot/
│
├── infra/
│   └── terraform/
│       ├── main.tf                 # All AWS resource definitions
│       ├── variables.tf            # Input variable declarations
│       ├── terraform.tfvars        # Variable values (bucket names, region, etc.)
│       └── outputs.tf              # Outputs: CloudFront URL, API endpoint, etc.
│
├── .gitignore
└── README.md
```

---

## 7. Infrastructure (Terraform)

### Why Terraform?

- **Declarative:** Describe what you want — Terraform figures out how
- **Version-controlled infra:** Infrastructure changes go through Git like code
- **Reproducible:** Spin up identical environments in different AWS accounts
- **Self-documenting:** The `.tf` files are your infrastructure docs

---

### Remote Backend

Terraform state is stored in S3 — not locally. This is essential for CI/CD and any team workflow.

```hcl
terraform {
  backend "s3" {
    bucket         = "devops-accelerator-platform-tf-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "devops-accelerator-tf-locker"
  }
}
```

| Resource | Purpose |
|----------|---------|
| S3 Bucket | Stores `terraform.tfstate` file |
| DynamoDB Table (key: `LockID`) | State locking to prevent concurrent modifications |

**Why remote state?**  
Local state gets lost if your machine dies and can't be used by CI/CD. Remote state is backed up, accessible from GitHub Actions, and supports team collaboration.

---

### Pre-Setup (Before `terraform init`)

These two resources must be created manually before Terraform can initialize, because Terraform itself needs them to store its own state.

```bash
# Create the S3 bucket for Terraform state
aws s3api create-bucket \
  --bucket devops-accelerator-platform-tf-state \
  --region us-east-1

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name devops-accelerator-tf-locker \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

---

### Terraform Commands

```bash
cd infra/terraform

terraform init      # Download providers, connect to remote backend
terraform validate  # Check syntax and config correctness
terraform plan      # Preview all changes before applying

# ⚠️ Do NOT run terraform apply locally.
# All applies go through the CI/CD pipeline to maintain a single source of truth.
```

---

## 8. CI/CD Pipeline

Three independent GitHub Actions workflows — one per concern.

---

### Pipeline 1 — Terraform (`.github/workflows/terraform.yml`)

**Trigger:** Any push to `main`  
**Purpose:** Provision or update AWS infrastructure

```
Checkout code
    ↓
Configure AWS credentials (via GitHub Secrets)
    ↓
terraform init     (connects to remote S3 backend)
    ↓
terraform validate (fails fast on syntax errors)
    ↓
terraform plan     (shows exactly what will change)
    ↓
terraform apply    (runs only on main branch push)
```

---

### Pipeline 2 — Frontend (`.github/workflows/frontend.yml`)

**Trigger:** Changes inside `frontend/`  
**Purpose:** Deploy updated static files to S3 and refresh CloudFront

```
Checkout code
    ↓
Configure AWS credentials
    ↓
aws s3 sync frontend/ → S3 frontend bucket  (only uploads changed files)
    ↓
CloudFront cache invalidation               (forces edge nodes to fetch fresh content)
```

---

### Pipeline 3 — Backend (`.github/workflows/backend-deploy.yml`)

**Trigger:** Changes inside `backend/`  
**Purpose:** Package and deploy updated Lambda functions

```
Checkout code
    ↓
Configure AWS credentials
    ↓
Zip Lambda function code
    ↓
aws lambda update-function-code → Lambda function
    ↓
Publish new Lambda version
```

---

## 9. Lambda Functions

### `generate-presigned-url`

| Property | Value |
|----------|-------|
| Trigger | API Gateway (HTTP POST) |
| Runtime | Python |
| Purpose | Generate a temporary S3 upload URL |

**How it works:**
1. Receives file metadata (name, type) from the frontend via API Gateway
2. Calls `boto3` to generate a pre-signed S3 `put_object` URL (expires in ~5 min)
3. Returns the URL to the browser
4. Browser uploads directly to S3 using that URL

**Why pre-signed URLs?**  
Without them, you'd either expose the S3 bucket publicly (dangerous) or proxy the file through a server (slow and expensive). Pre-signed URLs are temporary, scoped, and require no server involvement during the actual upload.

---

### `process-uploaded-file`

| Property | Value |
|----------|-------|
| Trigger | S3 ObjectCreated event |
| Runtime | Python |
| Purpose | Process file, log details, send notification |

**How it works:**
1. S3 automatically pushes the upload event payload to Lambda
2. Lambda reads file name, size, bucket, and timestamp from the event
3. Logs all details to CloudWatch
4. Publishes a message to the SNS topic

**Why event-driven?**  
Lambda doesn't poll S3 for new files — S3 pushes the event the instant a file arrives. No wasted compute, no delay, zero cost when idle.

---

## 10. API Gateway

**Type:** REST API  
**Method:** `POST /generate-presigned-url`

- Handles CORS configuration so the browser frontend can call it
- Throttling: Burst limit = 100, Rate limit = 200 req/sec (update after deployment)
- Fully decouples frontend from Lambda's internal function ARN
- Endpoint URL is a Terraform output — update `frontend/index.html` after first deploy

---

## 11. S3 Buckets

| Bucket | Purpose | Public Access |
|--------|---------|:---:|
| `devops-accelerator-platform-tf-state` | Terraform remote state storage | ❌ Blocked |
| `devops-accelerator-frontend-hosting-bucket` | Static HTML/CSS/JS for the frontend | CloudFront only |
| `devops-accelerator-upload-bucket` | Receives user-uploaded files | Pre-signed URL only |

All three buckets have public access blocked at the bucket level. CloudFront and pre-signed URLs provide controlled access without ever exposing raw bucket URLs.

---

## 12. CloudFront CDN

CloudFront sits in front of the S3 frontend bucket. Users never interact with S3 directly.

**How it works:**
- First request: CloudFront fetches the file from S3 origin and caches it at the nearest edge location
- Subsequent requests: served from cache (fast, no S3 cost)
- When frontend is updated: CI/CD triggers a CloudFront cache invalidation to clear stale files

**Custom error handling:** CloudFront can be configured to serve `index.html` on 403/404 — useful for SPAs.

---

## 13. SNS Notifications

**Topic:** `devops-accelerator-upload-notification-topic`  
**Subscription:** Email

After the first `terraform apply`, AWS sends a subscription confirmation email. **You must click Confirm Subscription** before any notifications will arrive.

**Architecture note:** Lambda publishes to SNS — it doesn't send email directly. This keeps Lambda decoupled from the notification mechanism. Add Slack, SMS, or a second email by adding subscribers to the topic — no Lambda changes needed.

---

## 14. CloudWatch Monitoring

Lambda automatically ships logs to CloudWatch. No configuration required.

**Log groups to check:**
- `/aws/lambda/process-uploaded-file`
- `/aws/lambda/generate-presigned-url`

**What to look for in logs:**
- File name and S3 key of the uploaded object
- Total execution duration
- Any Python errors or exceptions

**Recommended alarms to add later:**
- Lambda error rate > 1% over 5 minutes
- Lambda invocation duration approaching timeout limit
- S3 upload bucket size exceeding a threshold

---

## 15. Deployment Guide

### Prerequisites

- AWS account with Administrator Access
- AWS CLI installed and configured (`aws configure`)
- Terraform v1.3+ installed
- Git + GitHub account

---

### Step 1 — AWS IAM Setup

1. Go to IAM Console → Create User → Security credentials → Create Access Key
2. Attach `AdministratorAccess` policy directly
3. Download the CSV (do this before closing the page)
4. Run `aws configure` — paste in Access Key ID and Secret Key

---

### Step 2 — Clone the Repository

```bash
git clone https://github.com/jainilgupta02/devops-capstone-project.git

```

Create your own GitHub repo and point origin there:

```bash
git remote set-url origin git@github.com:your-username/your-repo.git
```

---

### Step 3 — Create Terraform Backend Resources (One-Time Manual Step)

```bash
# S3 state bucket — name must be globally unique, change accordingly
aws s3api create-bucket \
  --bucket your-unique-tf-state-bucket-name \
  --region us-east-1

# DynamoDB lock table
aws dynamodb create-table \
  --table-name devops-accelerator-tf-locker \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

---

### Step 4 — Add GitHub Secrets

Go to: **GitHub Repo → Settings → Secrets → Actions → New repository secret**

| Secret | Value |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | From IAM Access Key CSV |
| `AWS_SECRET_ACCESS_KEY` | From IAM Access Key CSV |
| `AWS_REGION` | `us-east-1` |
| `LAMBDA_FUNCTION_NAME` | `process-uploaded-file` |
| `FRONTEND_BUCKET_NAME` | e.g. `devops-accelerator-frontend-hosting-bucket` |
| `UPLOAD_BUCKET_NAME` | e.g. `devops-accelerator-upload-bucket` |
| `CLOUDFRONT_DIST_ID` | Add this after the first Terraform apply |

> ⚠️ **S3 bucket names are globally unique across all AWS accounts.** Make sure to use your own unique name — not the same one in this repo.

---

### Step 5 — Package Lambda Functions

```bash
cd backend/lambda/process-uploaded-file
zip -r lambda.zip .

cd ../generate-presigned-url
zip -r lambda.zip .
```

> After any change to `main.py`: delete the old `.zip`, re-zip, then push.

---

### Step 6 — Initialize Terraform

```bash
cd infra/terraform
terraform init      # Connects to your remote backend
terraform validate  # Confirms configs are correct
terraform plan      # Review what will be created
```

---

### Step 7 — Push to Main (Triggers CI/CD)

```bash
git add .
git commit -m "Initial deployment"
git push -u origin main
```

GitHub Actions will automatically:
- Run Terraform → provision all AWS infrastructure
- Deploy frontend → S3 + CloudFront
- Deploy Lambda → both functions

---

### Step 8 — Terraform Outputs

After CI/CD succeeds, check the Terraform job logs for output values:

```
Outputs:
cloudfront_url              = "d246o7opnvxl8.cloudfront.net"
frontend_bucket_name        = "devops-accelerator-frontend-hosting-bucket"
lambda_function_name        = "process-uploaded-file"
presigned_url_api_endpoint  = "https://0jwmlx4c0a.execute-api.us-east-1.amazonaws.com"
```

- Add `CLOUDFRONT_DIST_ID` to GitHub Secrets (find it in AWS Console → CloudFront)
- Update the API Gateway endpoint URL in `frontend/index.html`
- Push the updated `index.html` — this triggers the frontend pipeline again

---

### Step 9 — Confirm SNS Subscription

Check your email inbox for a message from AWS SNS. Click **Confirm Subscription** — without this, you won't receive any upload notifications.

---

### Step 10 — Verify in AWS Console

You should now see:

- **3 S3 Buckets** (Terraform state, frontend hosting, file upload)
- **2 Lambda Functions** (`generate-presigned-url`, `process-uploaded-file`)
- **1 API Gateway** with POST method
- **1 SNS Topic** with your email subscribed
- **1 CloudFront Distribution** serving your frontend

> Tip: In AWS Console, click the "Created" column header to sort newest resources to the top.

---

### Step 11 — Final Testing

1. Open `https://<cloudfront_url>` in your browser
2. Upload a file (JPG/PNG/PDF)
3. Check CloudWatch Logs → `/aws/lambda/process-uploaded-file` to confirm execution
4. Check your inbox for the SNS notification email

**If upload fails:** Temporarily increase API Gateway throttling (Burst: 100, Rate: 200), then reset to 0 after testing to avoid unnecessary costs.

---

## 16. GitHub Secrets

| Secret | Used By | What It Does |
|--------|---------|-------------|
| `AWS_ACCESS_KEY_ID` | All three workflows | AWS authentication |
| `AWS_SECRET_ACCESS_KEY` | All three workflows | AWS authentication |
| `AWS_REGION` | All three workflows | Target AWS region |
| `FRONTEND_BUCKET_NAME` | `frontend.yml` | S3 sync destination |
| `CLOUDFRONT_DIST_ID` | `frontend.yml` | Cache invalidation target |
| `LAMBDA_FUNCTION_NAME` | `backend-deploy.yml` | Lambda function to update |
| `UPLOAD_BUCKET_NAME` | `terraform.yml` | Passed as Terraform variable |

---

## 17. Troubleshooting

### `.terraform/` folder is too large to push

```bash
echo ".terraform/" >> .gitignore
git rm -r --cached infra/terraform/.terraform
git commit -m "Remove .terraform from tracking"
git push
```

---

### A file over 100MB is blocking the push

```bash
# Install git-filter-repo
pip3 install git-filter-repo

# Remove the large path from Git history entirely
git filter-repo --force --path infra/terraform/.terraform/ --invert-paths

# Re-add your remote and force push
git remote add origin git@github.com:your-username/your-repo.git
git push --force --set-upstream origin main
```

> Prevent recurrence: always have `.terraform/` in `.gitignore` before committing.

---

### Lambda not triggered after file upload

- Check S3 bucket → Properties → Event notifications (should point to `process-uploaded-file`)
- Verify Lambda execution role has `s3:GetObject` permission
- Open CloudWatch → `/aws/lambda/process-uploaded-file` to see if Lambda was even invoked and what error occurred

---

### Pre-signed URL upload fails (CORS error in browser)

- Check S3 upload bucket has a CORS policy allowing `PUT` from your CloudFront domain
- Check API Gateway → CORS settings are enabled for `POST /generate-presigned-url`

---

### SNS email not arriving

- Check your spam/junk folder first
- Confirm you clicked the subscription confirmation link AWS sent after `terraform apply`
- In AWS Console → SNS → Topics → Subscriptions, verify status shows `Confirmed` (not `Pending`)

---

### API Gateway returning 403 or 502

- 403 → Usually a throttling or authorizer issue. Check throttle limits under API Gateway → Protect → Throttling
- 502 → Lambda threw an unhandled exception. Open CloudWatch logs for the Lambda to see the exact error

---

## 18. Future Improvements

- [ ] Add user authentication using Amazon Cognito before allowing uploads
- [ ] Add file type validation in Lambda (reject anything that isn't JPG/PNG/PDF)
- [ ] Add a CloudWatch Dashboard with custom Lambda metrics
- [ ] Configure CloudWatch Alarms for Lambda error rate and timeout breaches
- [ ] Implement multi-environment setup (dev/staging/prod) using Terraform workspaces
- [ ] Enable CloudTrail for a full audit trail of all API calls
- [ ] Store upload metadata (file name, uploader, timestamp) in DynamoDB
- [ ] Add Slack notifications by chaining SNS → Lambda → Slack webhook
- [ ] Add AWS WAF to CloudFront for application-layer DDoS and bot protection
- [ ] Migrate Lambda to container-based deployment for larger dependencies

---

## 19. Interview Questions & Answers

---

### Why use CloudFront instead of serving directly from S3?

S3 can host static websites but has no native HTTPS on custom domains, no global edge caching, and no DDoS protection. CloudFront provides all three, plus it hides the actual S3 bucket URL so it can't be accessed or abused directly. It's also cheaper at scale since cached content doesn't incur S3 GET request costs.

---

### Why use pre-signed URLs for file uploads?

Pre-signed URLs let users upload files directly from their browser to S3 — without routing the file through a server. They're time-limited (expire after ~5 minutes) and scoped to a specific bucket and key. The alternative — proxying uploads through Lambda — would be slow, expensive, and would hit Lambda's 10MB payload limit for large files.

---

### Why store Terraform state remotely in S3?

Local state is fine solo but breaks in CI/CD and teams. Remote state in S3 means: it's backed up, it's accessible from GitHub Actions runners, multiple people can collaborate, and you never lose it when a laptop dies. It also enables state history via S3 versioning.

---

### What is DynamoDB state locking and why does it matter?

If two Terraform processes run simultaneously (e.g., two CI/CD runs triggered at once), they can read the same state, make conflicting changes, and corrupt the `.tfstate` file. DynamoDB locking prevents this: when `terraform apply` starts, it writes a `LockID` entry to DynamoDB. If another process tries to do the same, it fails until the first one finishes and releases the lock.

---

### Why Lambda instead of EC2?

EC2 is an always-on server you have to provision, patch, monitor, and pay for 24/7. Lambda runs only when triggered — you pay for milliseconds of execution time and it scales automatically from 0 to thousands of concurrent executions. For a workload like "process a file when uploaded" or "generate a URL on request," Lambda is dramatically cheaper and requires zero operational overhead.

---

### What is event-driven architecture and why use it here?

Instead of a server constantly polling S3 every N seconds asking "any new files?", the system reacts to events. When a file is uploaded, S3 pushes an event to Lambda immediately. This is more efficient (no wasted compute), faster (milliseconds vs. polling interval), and cheaper (Lambda costs nothing when idle).

---

### How does the CI/CD pipeline prevent bad deployments?

Terraform runs `validate` before `plan`, and `plan` before `apply`. If either fails, the pipeline stops — nothing gets applied. For Lambda, the zip is only deployed if packaging succeeds. Future hardening would include adding automated tests before deployment stages and requiring a manual approval step before `terraform apply` on production.

---

### How does GitHub Actions authenticate with AWS securely?

Through IAM Access Keys stored as GitHub Secrets. Secrets are encrypted at rest in GitHub and are only injected as environment variables during workflow execution. The IAM user has a scoped policy — only the permissions needed to deploy these specific resources.

---

### What is the difference between SNS and SQS? Why use SNS here?

SNS (Simple Notification Service) is pub/sub — it pushes messages to all subscribers immediately. SQS (Simple Queue Service) is a queue — messages wait until a consumer polls for them. This project uses SNS because the goal is immediate email delivery with no queuing needed. SQS would be appropriate if multiple downstream services needed to independently process each upload event at their own pace.

---

### What would you change to make this truly production-ready?

Authentication via Cognito, WAF on CloudFront, file type and size validation in Lambda, CloudWatch Alarms on error rates, separate environments via Terraform workspaces, CloudTrail audit logging, secrets in AWS Secrets Manager instead of Lambda environment variables, and structured logging (JSON) in Lambda for easier log querying.

---

## 20. Learning Outcomes

This project gave me hands-on experience across the full DevOps stack — not just reading about these tools, but building and debugging a real, working system with them.

**Infrastructure as Code (Terraform)**
- Writing modular Terraform configs with variables and outputs
- Setting up and connecting to a remote S3 backend
- Understanding provider, resource, data source, variable, and output blocks
- Separating infra concerns into `main.tf`, `variables.tf`, and `outputs.tf`

**CI/CD (GitHub Actions)**
- Building multi-trigger, multi-pipeline workflows
- Separating pipelines by concern: infra, frontend, backend
- Securely injecting AWS credentials via GitHub Secrets
- Debugging workflow failures in the Actions tab

**Serverless & Event-Driven Architecture**
- Configuring Lambda triggers: HTTP (API Gateway) and event-driven (S3)
- Understanding Lambda execution roles, cold starts, and timeouts
- Designing loosely coupled systems where components don't depend on each other directly

**AWS Networking & Security**
- IAM least-privilege roles for Lambda functions
- S3 bucket policies vs. ACLs
- Using CloudFront as a security and performance layer
- Pre-signed URLs as a controlled file transfer mechanism

**Observability**
- Reading Lambda logs in CloudWatch log groups and streams
- Understanding log levels and execution metadata
- Setting up SNS for operational alerting

**Debugging Real-World Issues**
- Rewriting Git history with `git-filter-repo` to remove large files
- Diagnosing CORS misconfiguration across API Gateway and S3
- Understanding Terraform state corruption and how locking prevents it
- Tracing errors end-to-end from browser → API Gateway → Lambda → CloudWatch

---

*Built by [Jainil Gupta](https://github.com/jainilgupta02) — aspiring DevOps engineer.*