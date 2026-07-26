# AWS Capstone Project — Serverless Feedback Portal (All 15 Services)

> A complete, buildable project that wires together all 15 core AWS services into one coherent app.
> Follow the phases in order. Each step includes a **Why** so you understand the architecture, not just the commands.

---

## The App: "Product Feedback Portal"

A public website where users submit product feedback. Feedback is stored, processed asynchronously, and monitored end to end.

**User flow:**
1. A visitor loads the site (served globally, fast, over HTTPS).
2. They submit feedback through a form.
3. The API stores it and kicks off background processing (e.g., tagging + email notification).
4. Admins view analytics; the whole system is monitored and audited.

---

## How the 15 Services Map to This Project

| # | Service | Role in this project |
|---|---------|----------------------|
| 1 | **IAM** | Roles/policies for every component (least privilege) |
| 2 | **VPC** | Isolated network with public/private subnets |
| 3 | **EC2** | Bastion host for secure admin access to the private DB |
| 4 | **S3** | Hosts the static frontend + stores CloudTrail logs |
| 5 | **CloudFront** | Global CDN + HTTPS in front of S3 and the API |
| 6 | **Route 53** | Custom domain + DNS routing + health checks |
| 7 | **API Gateway** | Public HTTPS API front door |
| 8 | **Lambda** | Serverless backend (create/read feedback) |
| 9 | **DynamoDB** | Primary NoSQL store for feedback items |
| 10 | **SNS** | Publishes "new feedback" events (pub/sub) |
| 11 | **SQS** | Buffers events for the async worker (decoupling) |
| 12 | **ECS (Fargate)** | Container worker that processes queued feedback |
| 13 | **RDS** | Relational store for aggregated analytics/reporting |
| 14 | **CloudWatch** | Metrics, logs, alarms, dashboard for everything |
| 15 | **CloudFormation** | Provisions the entire stack as code (IaC) |

**Bonus services also used:** **EBS** (ECS/EC2 volumes), **EFS** (shared worker storage), **CloudTrail** (audit log), **ACM** (TLS certs), **ECR** (container registry), **EventBridge** (scheduling).

---

## Architecture Diagram (text)

```
                          Route 53 (DNS)
                               |
                       CloudFront (CDN, HTTPS)
                        /                  \
             S3 (static frontend)     API Gateway (/feedback)
                                            |
                                        Lambda (API handler)
                                        /        \
                                 DynamoDB      SNS topic
                                                   |
                                                 SQS queue
                                                   |
                                        ECS Fargate worker  --> RDS (analytics)
                                                   |              |
                                                 EFS         EC2 bastion (admin)
                                                   
        Everything inside a VPC  |  IAM roles on every component
        CloudWatch monitors all  |  CloudTrail audits all  |  CloudFormation builds all
```

---

## Prerequisites

- AWS account with root MFA enabled and a **billing budget** set (see main guide).
- **AWS CLI v2** installed and configured (`aws configure`).
- **Docker** installed (to build the ECS worker image).
- A **registered domain** (Route 53 or external) — optional but needed for the Route 53 phase.
- Pick a region (examples use `us-east-1`; CloudFront certs **must** be in `us-east-1`).

> **Why start with prerequisites:** Every later phase depends on credentials, a region, and cost guardrails. Getting these right first prevents rework and surprise bills.

---

# Phase 0 — Governance First (IAM + CloudTrail)

Set up security and auditing *before* creating resources, so all activity is captured from the start.

### 0.1 Create a project IAM user/role for deployment

1. IAM → **Roles** → create a role (or use your admin user) with permissions to deploy the stack.
   - **Why:** You deploy through a scoped identity, never the root user.
2. Note the **least-privilege principle**: each component (Lambda, ECS, EC2) gets its **own** role later, scoped to only what it needs.
   - **Why:** If one component is compromised, blast radius is limited to its narrow permissions.

### 0.2 Turn on CloudTrail

1. CloudTrail → **Create trail** → name `feedback-portal-trail`.
2. Create a **new S3 bucket** for logs; enable **log file validation**.
   - **Why:** Immutable, tamper-evident audit logs — you want a record of who created/changed/deleted every resource in this project.
3. Make it **multi-region** and enable **management events**.
   - **Why:** Captures control-plane actions everywhere so nothing escapes the audit.
4. (Optional) Stream to **CloudWatch Logs** for real-time alerts.
   - **Why:** Enables alarms on sensitive actions (e.g., IAM policy changes).

---

# Phase 1 — Networking Foundation (VPC)

### 1.1 Create the VPC

1. VPC → **Create VPC** → **VPC and more**.
2. CIDR `10.0.0.0/16`, **2 AZs**, **2 public** + **2 private** subnets.
   - **Why:** Two AZs give high availability; public subnets host internet-facing pieces (bastion, NAT, ALB), private subnets protect RDS and the ECS worker.
3. Add **1 NAT Gateway** (per AZ for prod).
   - **Why:** Lets private resources (ECS worker, RDS maintenance) reach the internet outbound without being publicly reachable.
4. Add an **S3 Gateway VPC Endpoint** and **interface endpoints** for SQS/ECR/CloudWatch Logs.
   - **Why:** Private resources talk to AWS services without going over the internet — more secure and avoids NAT data charges.

### 1.2 Security groups

1. `bastion-sg`: inbound SSH (22) **from your IP only**.
2. `ecs-sg`: inbound from ALB SG / API only.
3. `rds-sg`: inbound DB port (5432) **only from `ecs-sg` and `bastion-sg`**.
   - **Why:** Security groups are stateful firewalls. Chaining them (RDS accepts only from ECS/bastion) enforces that the database is never directly exposed.
4. Enable **VPC Flow Logs** to CloudWatch.
   - **Why:** Records accepted/rejected traffic for security forensics and debugging.

---

# Phase 2 — Data Stores (DynamoDB + RDS)

### 2.1 DynamoDB (primary store)

1. DynamoDB → **Create table** `Feedback`.
2. Partition key `feedbackId` (String); consider sort key `createdAt`.
   - **Why:** `feedbackId` distributes items evenly; a sort key enables time-ordered queries per partition.
3. **On-demand** capacity mode.
   - **Why:** Traffic is unpredictable for a new app; on-demand avoids over/under-provisioning.
4. Add a **GSI** on `productId` → to list feedback per product.
   - **Why:** GSIs enable query patterns beyond the primary key (e.g., "all feedback for product X").
5. Enable **Streams** (new + old images) and **PITR**.
   - **Why:** Streams feed downstream processing/analytics; PITR gives 35-day point-in-time recovery.
6. Enable **TTL** on an `expireAt` attribute (optional, for auto-cleanup of drafts).
   - **Why:** DynamoDB deletes expired items for free.

### 2.2 RDS (analytics store)

1. RDS → **Create database** → PostgreSQL, template **Dev/Test**, `db.t3.micro`.
2. Place in **private subnets** with `rds-sg`.
   - **Why:** The analytics DB holds aggregated data and must never be internet-reachable.
3. Enable **Multi-AZ** (prod), **encryption at rest**, **7-day backups**, **Performance Insights**.
   - **Why:** HA failover, compliance, recoverability, and query tuning.
4. Store credentials in **Secrets Manager**.
   - **Why:** No plaintext DB passwords in code; supports automatic rotation.

---

# Phase 3 — Backend Compute (Lambda + API Gateway)

### 3.1 Lambda API handler

1. Lambda → **Create function** `FeedbackApi` (Python or Node.js, arm64).
   - **Why:** arm64 (Graviton) is cheaper per ms.
2. Give it an **execution role** allowing: `dynamodb:PutItem/Query` on the `Feedback` table and `sns:Publish` on the topic (created in Phase 5).
   - **Why:** Least privilege — the function can only touch exactly what it needs.
3. Handler logic:
   - `POST /feedback` → validate input → write to DynamoDB → publish event to SNS.
   - `GET /feedback` → query DynamoDB (via GSI) → return list.
   - **Why:** Keeps the API thin and pushes slow work to async processing (SNS/SQS/ECS).
4. Set **memory** (256 MB start) and **timeout** (10s); add **env vars** (table name, topic ARN).
   - **Why:** Right-sizing memory tunes cost/latency; env vars keep config out of code.
5. Enable **X-Ray tracing** and **CloudWatch Logs**.
   - **Why:** Trace requests and debug without a server to log into.

### 3.2 API Gateway (HTTP API)

1. API Gateway → **Create API** → **HTTP API**.
2. Routes: `POST /feedback` and `GET /feedback` → integrate with `FeedbackApi` Lambda.
   - **Why:** HTTP APIs are cheaper/simpler and cover this app's needs.
3. Enable **CORS** (allow your site origin).
   - **Why:** Browsers block cross-origin calls from your frontend without CORS headers.
4. Add a **JWT authorizer** (Cognito) on write routes, or an **API key + usage plan**.
   - **Why:** Prevents anonymous abuse and lets you throttle/meter clients.
5. Enable **access logging** to CloudWatch and set **throttling** limits.
   - **Why:** Observability plus protection against traffic spikes hitting the backend.
6. Deploy to a **stage** (`prod`) and note the invoke URL.

---

# Phase 4 — Frontend (S3 + CloudFront + Route 53 + ACM)

### 4.1 S3 static site

1. Create bucket `feedback-portal-frontend`; upload `index.html`, JS, CSS.
2. Keep **Block Public Access ON**.
   - **Why:** The bucket stays private; CloudFront reads it via OAC — no public bucket needed.
3. Enable **versioning**.
   - **Why:** Roll back a bad frontend deploy instantly.

### 4.2 ACM certificate

1. ACM (in **us-east-1**) → request a public cert for `app.yourdomain.com`.
   - **Why:** CloudFront requires the cert in us-east-1; enables HTTPS on your domain.
2. Validate via DNS (Route 53 auto-creates the record).

### 4.3 CloudFront distribution

1. Create a distribution.
2. **Origin 1:** the S3 bucket, using **Origin Access Control (OAC)** — for the site.
3. **Origin 2:** the API Gateway domain — behavior on path `/api/*` → API origin.
   - **Why:** Serving both the site and API through one CloudFront domain avoids CORS issues and gives one HTTPS endpoint.
4. **Viewer protocol policy:** Redirect HTTP → HTTPS. Attach the ACM cert.
   - **Why:** Enforce encryption for all visitors.
5. Set **cache behaviors**: cache static assets long, **disable caching** for `/api/*`.
   - **Why:** Cache what's static for speed; never cache dynamic API responses.
6. Set **default root object** `index.html`; attach **AWS WAF** (optional).
   - **Why:** Serves the homepage at `/`; WAF blocks common attacks at the edge.

### 4.4 Route 53

1. Create/verify a **hosted zone** for your domain.
2. Add an **A (Alias)** record `app.yourdomain.com` → CloudFront distribution.
   - **Why:** Alias records map your domain to AWS resources for free, no IP management.
3. Add a **health check** + **failover routing** (optional).
   - **Why:** Route users to a backup/maintenance page if the primary is unhealthy.

---

# Phase 5 — Async Processing (SNS + SQS + ECS Fargate + EFS)

### 5.1 SNS topic

1. Create SNS topic `new-feedback`.
   - **Why:** Pub/sub decouples the API from processing — the API just publishes and returns fast.
2. Add an **email subscription** (admin alerts) and an **SQS subscription** (worker queue).
   - **Why:** Fan-out: one event notifies an admin *and* queues background work.

### 5.2 SQS queue

1. Create standard queue `feedback-processing`.
2. Set **visibility timeout** > worker processing time; add a **Dead-Letter Queue** (maxReceiveCount 5).
   - **Why:** Prevents duplicate processing; failing messages move to the DLQ instead of blocking the queue.
3. Add an **access policy** allowing SNS to send messages.
   - **Why:** Explicitly authorizes the SNS→SQS delivery.

### 5.3 ECS Fargate worker

1. Build a worker container (reads SQS, enriches data e.g. tags/sentiment, writes summary to **RDS**, saves artifacts to **EFS**).
2. Push the image to **ECR**.
   - **Why:** ECS pulls from a private registry; ECR keeps images secure.
3. Create an **ECS cluster** (Fargate).
   - **Why:** Fargate runs containers without managing EC2 hosts.
4. **Task definition:** image, 0.25 vCPU / 0.5 GB, **task role** (SQS receive/delete, RDS connect, EFS access), **execution role** (ECR pull, CloudWatch logs).
   - **Why:** Two roles separate "what ECS needs to run the task" from "what the app needs at runtime."
5. Mount an **EFS** volume in the task definition.
   - **Why:** Shared, persistent file storage across worker tasks (e.g., report files).
6. Create a **Service** (desired count 1) in **private subnets** with `ecs-sg`.
   - **Why:** The worker needs no inbound internet access; it pulls work from SQS.
7. Add **Service Auto Scaling** on SQS queue depth (target tracking).
   - **Why:** Scale workers up when the backlog grows, down when idle — cost-efficient.
8. Logs via **awslogs** driver → CloudWatch.
   - **Why:** Centralized worker logs for debugging.

---

# Phase 6 — Admin Access (EC2 Bastion + EBS)

1. Launch a small **EC2** bastion (`t3.micro`, Amazon Linux) in a **public subnet** with `bastion-sg`.
   - **Why:** A hardened jump host is the only door into the private network for DB admin.
2. Attach an **Elastic IP**; use a **key pair**; enable **IMDSv2**.
   - **Why:** Stable IP for SSH; IMDSv2 blocks SSRF-based credential theft.
3. The root volume is an **EBS** gp3; take a **snapshot** before changes.
   - **Why:** EBS is durable block storage; snapshots are your instance backups.
4. From the bastion, connect to **RDS** in the private subnet to run analytics queries.
   - **Why:** Demonstrates the secure path: your IP → bastion → private DB, never exposing RDS.
5. Prefer **SSM Session Manager** over SSH where possible.
   - **Why:** No open SSH port, no key management, full audit in CloudTrail.

---

# Phase 7 — Observability (CloudWatch)

1. **Alarms:**
   - Lambda `Errors` > 0 and `Duration` p99 high.
   - API Gateway `5XXError` rate high.
   - SQS `ApproximateAgeOfOldestMessage` too high (backlog).
   - DynamoDB throttling; RDS CPU/storage.
   - **Why:** Each alarm catches a distinct failure mode before users complain.
2. Route alarms to an **SNS `ops-alerts`** topic (email/SMS).
   - **Why:** Humans get paged automatically on breach.
3. **Logs Insights** queries across Lambda + ECS + API logs.
   - **Why:** Correlate a failed request across all tiers quickly.
4. Build a **Dashboard** with widgets for requests, errors, queue depth, DB health.
   - **Why:** One pane of glass for system health.
5. Add a **Synthetics canary** hitting `GET /feedback`.
   - **Why:** Detect user-facing outages proactively on a schedule.
6. **EventBridge** scheduled rule → nightly Lambda that aggregates DynamoDB → RDS.
   - **Why:** Scheduled batch analytics complements real-time processing.

---

# Phase 8 — Provision Everything as Code (CloudFormation)

Rebuild the whole project declaratively so it's reproducible and reviewable. Split into nested stacks.

### 8.1 Stack layout

1. **network.yaml** — VPC, subnets, NAT, endpoints, security groups.
2. **data.yaml** — DynamoDB table, RDS instance, Secrets Manager secret.
3. **backend.yaml** — Lambda, API Gateway, IAM roles, SNS, SQS.
4. **frontend.yaml** — S3 bucket, CloudFront, ACM reference, Route 53 records.
5. **worker.yaml** — ECR reference, ECS cluster/service/task, EFS.
6. **observability.yaml** — CloudWatch alarms, dashboard, log groups.
7. **root.yaml** — nested stack referencing all of the above.
   - **Why:** Nested stacks keep each concern modular and let you update one layer without touching others.

### 8.2 Minimal starter template (DynamoDB + Lambda + API)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Feedback Portal - backend core
Parameters:
  TableName:
    Type: String
    Default: Feedback
Resources:
  FeedbackTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
    Properties:
      TableName: !Ref TableName
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - { AttributeName: feedbackId, AttributeType: S }
      KeySchema:
        - { AttributeName: feedbackId, KeyType: HASH }
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true

  ApiLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: FeedbackAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: [ dynamodb:PutItem, dynamodb:Query, dynamodb:GetItem ]
                Resource: !GetAtt FeedbackTable.Arn

  FeedbackApiFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt ApiLambdaRole.Arn
      Architectures: [ arm64 ]
      Timeout: 10
      Environment:
        Variables:
          TABLE_NAME: !Ref TableName
      Code:
        ZipFile: |
          import os, json, uuid, boto3
          ddb = boto3.resource("dynamodb")
          table = ddb.Table(os.environ["TABLE_NAME"])
          def handler(event, context):
              item = {"feedbackId": str(uuid.uuid4()), "body": event.get("body","")}
              table.put_item(Item=item)
              return {"statusCode": 200, "body": json.dumps(item)}

  HttpApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: FeedbackHttpApi
      ProtocolType: HTTP

Outputs:
  ApiId:
    Value: !Ref HttpApi
  TableArn:
    Value: !GetAtt FeedbackTable.Arn
    Export:
      Name: FeedbackTableArn
```

- **Why this template:** It shows the key CloudFormation patterns — `DeletionPolicy: Retain` protects data, an inline IAM role scoped to one table demonstrates least privilege, `!GetAtt`/`!Ref` wire resources together, and `Outputs`/`Export` share values with other stacks.

### 8.3 Deploy

```bash
aws cloudformation deploy \
  --template-file backend.yaml \
  --stack-name feedback-backend \
  --capabilities CAPABILITY_IAM
```

- **Why `--capabilities CAPABILITY_IAM`:** CloudFormation requires explicit acknowledgment when a stack creates IAM roles.

### 8.4 Safe updates

1. Create a **change set** before applying updates.
   - **Why:** Preview exactly what will change/replace/delete before it happens.
2. Add **termination protection** to the root stack.
   - **Why:** Prevents accidental teardown of the whole environment.

---

# Phase 9 — End-to-End Test

1. Open `https://app.yourdomain.com` → submit feedback.
2. Confirm the item appears in **DynamoDB**.
3. Confirm **SNS** sent the admin email and **SQS** received a message.
4. Confirm the **ECS** worker processed it and wrote a summary to **RDS** (check via bastion).
5. Confirm **CloudWatch** shows metrics/logs; trip an alarm to verify the alert.
6. Confirm **CloudTrail** recorded all the API calls you made.
   - **Why:** A capstone isn't "done" until you've verified the full path across every service.

---

## Cost Control & Teardown

Because CloudFormation built it, teardown is mostly one command per stack:

```bash
aws cloudformation delete-stack --stack-name feedback-observability
aws cloudformation delete-stack --stack-name feedback-worker
aws cloudformation delete-stack --stack-name feedback-frontend
aws cloudformation delete-stack --stack-name feedback-backend
aws cloudformation delete-stack --stack-name feedback-data
aws cloudformation delete-stack --stack-name feedback-network
```

Manual cleanup checklist for things stacks may retain:
- Empty and delete S3 buckets (frontend + CloudTrail logs).
- Delete DynamoDB/RDS if you set `DeletionPolicy: Retain`.
- Release **Elastic IPs**, delete **NAT Gateways** (they bill hourly), remove **EFS**.
- Delete **ECR** images and old **EBS snapshots**.
- **Why:** NAT gateways, RDS, EIPs, and load balancers are the usual sources of surprise bills.

---

## What You'll Have Proven

By finishing this, you've hands-on used all 15 services together the way real systems do: a secured (**IAM**), networked (**VPC**), monitored (**CloudWatch**), audited (**CloudTrail**), fully code-provisioned (**CloudFormation**) serverless app spanning frontend delivery (**S3/CloudFront/Route 53**), an API tier (**API Gateway/Lambda**), data (**DynamoDB/RDS**), async processing (**SNS/SQS/ECS**), and admin access (**EC2**) — with **EBS/EFS/ACM/ECR/EventBridge** filling in the supporting roles.

That's a portfolio-grade project. Build it, break it, fix it — that's what makes you a pro.
