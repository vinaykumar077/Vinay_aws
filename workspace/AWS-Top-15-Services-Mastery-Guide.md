# AWS Top 15 Services — Mastery Guide

> A hands-on, step-by-step playbook to take you from beginner to confident practitioner on the 15 most-used AWS services.
> Every step includes a **Why** note so you understand the reasoning, not just the clicks.

---

## How to Use This Guide

- Work through it in order. Each service builds on concepts from the previous ones (IAM and VPC underpin everything).
- Do every step in a **real AWS account** (free tier). Reading alone will not make you a pro — muscle memory will.
- After each service, do the "Practice Challenge" at the end of its section.
- Keep the **Cleanup** checklist handy so you don't get surprise bills.

### Prerequisites (do this once)

1. **Create an AWS account** at https://aws.amazon.com → *Create an AWS Account*.
   - **Why:** You need a root account to provision everything. Use a strong password + a dedicated email alias.
2. **Secure the root user**: enable MFA (IAM → Security credentials → Assign MFA device).
   - **Why:** The root user can do anything, including closing the account. Compromise here is catastrophic, so it must be locked down and rarely used.
3. **Set a billing alarm** (Billing → Budgets → Create budget → Zero-spend or a $5 budget).
   - **Why:** AWS is pay-as-you-go. A budget alert is your safety net against runaway costs.
4. **Install the AWS CLI** and run `aws configure`.
   - **Why:** The console teaches concepts; the CLI teaches automation and speed. Pros live in both.

---

## The 12-Week Learning Plan

| Week | Focus | Services | Goal |
|------|-------|----------|------|
| 1 | Identity & security foundation | IAM | Users, roles, policies, least privilege |
| 2 | Networking foundation | VPC | Subnets, routing, security groups |
| 3 | Compute | EC2 | Launch, connect, scale instances |
| 4 | Storage | S3 | Buckets, versioning, lifecycle, static hosting |
| 5 | Databases (relational) | RDS | Managed SQL, backups, Multi-AZ |
| 6 | Databases (NoSQL) | DynamoDB | Tables, keys, indexes, capacity |
| 7 | Serverless compute | Lambda | Functions, triggers, layers |
| 8 | API layer | API Gateway | REST/HTTP APIs, integrations, auth |
| 9 | Content delivery & DNS | CloudFront + Route 53 | CDN, caching, domains, routing |
| 10 | Messaging & decoupling | SNS + SQS | Pub/sub, queues, fan-out |
| 11 | Observability | CloudWatch | Metrics, logs, alarms, dashboards |
| 12 | Automation & containers | CloudFormation + ECS | IaC and container orchestration |

**Milestone project (capstone):** Build a serverless web app — Route 53 → CloudFront → S3 (frontend) → API Gateway → Lambda → DynamoDB, with CloudWatch monitoring, all provisioned by CloudFormation, secured by IAM inside a VPC. This single project touches all 15 services.

---

# 1. IAM — Identity and Access Management

**What it is:** The service that controls *who* can do *what* in your AWS account. It is free and it is the foundation of everything.

**Core concepts:**
- **Users** — long-lived identities for humans.
- **Groups** — collections of users sharing permissions.
- **Roles** — temporary identities that services or users *assume*; no long-term credentials.
- **Policies** — JSON documents that grant or deny permissions.

### Step-by-step

1. Go to **IAM console** → **Users** → **Create user**.
   - **Why:** You should never use the root account for daily work; you create scoped users instead.
2. Enter a **user name**, then choose **Provide user access to the AWS Management Console** if it's a human.
   - **Why:** Console access is for people; programmatic-only users (services/scripts) should skip this and use access keys or roles.
3. On **Set permissions**, choose **Add user to group** → **Create group**.
   - **Why:** Attaching policies to groups (not individual users) scales cleanly and keeps permissions consistent.
4. In the group, attach a managed policy (e.g., `PowerUserAccess` for a dev, or a narrower one).
   - **Why:** AWS-managed policies are maintained by AWS and cover common needs without you writing JSON.
5. Explore **Create policy** → **JSON** tab. Write a custom policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": ["s3:GetObject", "s3:PutObject"],
       "Resource": "arn:aws:s3:::my-bucket/*"
     }]
   }
   ```
   - **Why:** Custom policies let you apply **least privilege** — grant only the exact actions and resources needed.
6. Go to **Roles** → **Create role** → **AWS service** → pick **EC2** (or Lambda).
   - **Why:** Roles let AWS services act on your behalf securely, without embedding credentials in code.
7. Set up an **Identity Provider / IAM Identity Center (SSO)** if managing many users.
   - **Why:** Centralized SSO beats managing individual IAM users for teams.
8. Enable **MFA** for every human user (Security credentials → MFA).
   - **Why:** Passwords get phished; MFA blocks the vast majority of account takeovers.
9. Review the **Access Advisor** tab on a role/user.
   - **Why:** It shows which permissions were actually used, so you can trim unused ones and tighten security.

**Key options to explore:** Permission boundaries, service control policies (via Organizations), policy conditions (`aws:SourceIp`, `aws:MultiFactorAuthPresent`), password policy, credential report.

**Practice Challenge:** Create a `read-only-billing` user that can view the billing dashboard and nothing else.

### Exercises (build muscle memory)

1. **Warm-up:** Create a user `dev-alice`, add her to a `Developers` group with `PowerUserAccess`, and log in as her in an incognito window. Confirm she *cannot* create other IAM users.
   - **Verify it works:** As `dev-alice`, try IAM → Create user; you should get an "Access Denied" message.
2. **Core:** Write a custom policy that allows `s3:ListAllMyBuckets` but denies `s3:DeleteBucket`, attach it to a test user, and confirm the deny wins.
   - **Verify it works:** The user can list buckets but gets denied when trying to delete one — proving explicit deny overrides allow.
3. **Roles:** Create an `EC2-S3-ReadOnly` role, launch an EC2 instance with it, and from the instance run `aws s3 ls`.
   - **Verify it works:** The command succeeds with no credentials configured on the box — the role supplied them.
4. **Stretch:** Add a policy condition `aws:MultiFactorAuthPresent: true` so an action only works when the user signed in with MFA.
   - **Verify it works:** Without MFA the action is denied; after re-authenticating with MFA it succeeds.

### Practice Challenge — Solution

Goal: a `read-only-billing` user who can see billing and nothing else.

1. **Enable IAM access to billing first** (this trips people up): sign in as root → account menu (top right) → **Account** → **IAM user and role access to Billing information** → **Edit** → check **Activate IAM Access** → Update.
   - **Why:** Billing is special; even a correct IAM policy won't work until this account-level toggle is on.
2. IAM → **Users** → **Create user** → name `read-only-billing` → enable console access.
3. On permissions, choose **Attach policies directly** → search and attach the AWS managed policy **`Billing`** (view) — or for read-only, create this custom policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       { "Effect": "Allow", "Action": ["aws-portal:View*", "ce:Get*", "cur:Describe*", "budgets:View*"], "Resource": "*" }
     ]
   }
   ```
4. Do **not** attach any other policy — no EC2, S3, etc.
   - **Why:** Least privilege: the only capability granted is viewing billing.
5. **Test:** Log in as the new user in a private browser window → open **Billing dashboard** (works) → try to open **EC2** (Access Denied).
   - **Why this proves success:** Access to billing works, everything else is blocked — exactly the requirement.

### Cleanup after this service

- **Delete practice IAM users** you created (IAM → Users → Delete). Remove their access keys first.
  - **Why:** Unused users with credentials are a standing security risk, even though IAM itself is free.
- **Delete custom policies and groups** no longer attached to anything.
  - **Why:** Keeps the account tidy and avoids confusion over which policies are live.
- **Keep** your locked-down admin user and MFA — you need those going forward.
  - **Why:** These are your day-to-day identity, not throwaway practice resources.
- **Note:** IAM has no direct cost, so cleanup here is about **security hygiene**, not billing.

---

# 2. VPC — Virtual Private Cloud

**What it is:** Your own isolated virtual network in AWS where you launch resources. Everything network-related lives here.

**Core concepts:** CIDR blocks, subnets (public/private), route tables, internet gateway, NAT gateway, security groups (stateful), network ACLs (stateless).

### Step-by-step

1. Go to **VPC console** → **Create VPC** → choose **VPC and more**.
   - **Why:** The "and more" wizard auto-creates subnets, route tables, and gateways — a correct baseline in one click.
2. Set an **IPv4 CIDR block** like `10.0.0.0/16`.
   - **Why:** This defines your private IP address range. A `/16` gives ~65k addresses, plenty of room to subdivide.
3. Configure **2 Availability Zones**, **2 public** and **2 private subnets**.
   - **Why:** Multiple AZs give high availability; public subnets host internet-facing resources, private subnets protect databases and backends.
4. Add an **Internet Gateway** (auto-added by the wizard).
   - **Why:** It's the door between your VPC and the public internet, required for public subnets.
5. Add a **NAT Gateway** (1 per AZ for production).
   - **Why:** It lets private resources make *outbound* internet calls (e.g., OS updates) without being reachable *inbound*.
6. Inspect the **Route Tables**: public subnet routes `0.0.0.0/0` → IGW; private routes `0.0.0.0/0` → NAT.
   - **Why:** Route tables decide where traffic goes. This split is what makes a subnet "public" vs "private."
7. Create a **Security Group** → add inbound rule (e.g., HTTP 80 from `0.0.0.0/0`, SSH 22 from *your IP only*).
   - **Why:** Security groups are stateful firewalls around resources. Restricting SSH to your IP prevents brute-force attacks.
8. Explore **Network ACLs** (subnet-level, stateless).
   - **Why:** NACLs add a second, coarser layer of defense at the subnet boundary.
9. Try a **VPC Endpoint** (Endpoints → Create → `com.amazonaws.<region>.s3`).
   - **Why:** Endpoints let you reach AWS services privately without traversing the internet — more secure and often cheaper.
10. Enable **VPC Flow Logs** (send to CloudWatch).
    - **Why:** They record accepted/rejected traffic for security auditing and troubleshooting.

**Key options to explore:** VPC peering, Transit Gateway, DHCP option sets, IPv6 CIDRs, egress-only internet gateway.

**Practice Challenge:** Launch a "bastion" host in a public subnet and an app server in a private subnet; SSH to the app server only through the bastion.

### Exercises (build muscle memory)

1. **Warm-up:** Create a second VPC with CIDR `10.1.0.0/16` and one public subnet, entirely by hand (not the wizard).
   - **Verify it works:** You created the VPC, subnet, IGW, route table, and association separately and understand what each does.
2. **Core:** Launch an EC2 instance in a public subnet and confirm it gets a public IP and can reach the internet (`ping google.com`).
   - **Verify it works:** Ping succeeds — proving the IGW route and public IP are correct.
3. **Routing:** Launch an instance in a **private** subnet (no public IP). From the public instance, `ping` its private IP.
   - **Verify it works:** The private instance is reachable inside the VPC but has no public IP and can only reach the internet through the NAT.
4. **NACL test:** Add an inbound **Deny** rule (rule number 90) in the NACL blocking your own IP on port 22, then try to SSH.
   - **Verify it works:** Connection is refused — proving NACLs act at the subnet boundary and that lower rule numbers win.
5. **Stretch:** Create the S3 gateway endpoint, then from the private instance run `aws s3 ls` and confirm it works without a NAT route.

### Practice Challenge — Solution

Goal: bastion in public subnet, app server in private subnet, SSH to app only via bastion.

1. **Security groups:**
   - `bastion-sg`: inbound SSH (22) from **My IP**.
   - `app-sg`: inbound SSH (22) with **Source = `bastion-sg`** (type the SG ID in the source box, not a CIDR).
   - **Why:** The app server accepts SSH *only* from the bastion's security group, not from the internet.
2. **Launch the bastion** in a **public** subnet with a public IP and `bastion-sg`.
3. **Launch the app server** in a **private** subnet, no public IP, with `app-sg`.
4. **Copy your key to reach the app server.** Easiest secure method — use SSH agent forwarding so the private key never lands on the bastion:
   ```bash
   ssh-add path/to/key.pem
   ssh -A ec2-user@<bastion-public-ip>
   # now, from the bastion:
   ssh ec2-user@<app-server-private-ip>
   ```
   - **Why `-A` (agent forwarding):** It lets the bastion use your local key to authenticate onward without ever storing it there.
5. **Test the guardrail:** Try to SSH directly to the app server's private IP from your laptop — it fails (no route + no public IP). Only the bastion path works.
   - **Why this proves success:** Direct access is impossible; access flows exclusively through the bastion, which is the whole point.

### Cleanup after this service

- **Delete NAT Gateways first** (VPC → NAT Gateways → Delete).
  - **Why:** NAT Gateways bill **per hour plus per GB** even when idle — this is the single biggest surprise-cost item in a VPC.
- **Release any Elastic IPs** that were allocated for the NAT/bastion (EC2 → Elastic IPs → Release).
  - **Why:** Unattached Elastic IPs incur an hourly charge.
- **Delete VPC Endpoints** (interface endpoints bill hourly).
  - **Why:** Interface endpoints have an ongoing cost; the S3/DynamoDB gateway endpoints are free but tidy them anyway.
- **Delete the VPC** last (VPC → Delete VPC), which removes subnets, route tables, and the internet gateway together.
  - **Why:** AWS blocks VPC deletion until dependent resources (NAT, endpoints, instances) are gone — delete children first.
- **Disable/delete VPC Flow Logs** if you no longer need them.
  - **Why:** They keep writing to CloudWatch Logs, which accrues storage cost.

---

# 3. EC2 — Elastic Compute Cloud

**What it is:** Resizable virtual servers in the cloud.

### Step-by-step

1. **EC2 console** → **Launch instance**.
   - **Why:** This is the core workflow for getting a running server.
2. Choose an **AMI** (e.g., Amazon Linux 2023 or Ubuntu).
   - **Why:** The AMI is the OS + preinstalled software template. Amazon Linux is optimized and free-tier friendly.
3. Choose an **instance type** (e.g., `t2.micro` / `t3.micro`).
   - **Why:** Instance type = CPU/RAM/network capacity. `t2.micro` is free-tier eligible and fine for learning.
4. Create/select a **key pair** (download the `.pem`).
   - **Why:** SSH key pairs are how you securely log in. The private key never leaves your machine.
5. Under **Network settings**, pick your **VPC**, a **public subnet**, enable **Auto-assign public IP**, and attach a **security group**.
   - **Why:** These decide *where* the instance lives and *who* can reach it. Public IP + public subnet make it internet-reachable.
6. Configure **storage** (EBS volume, e.g., 8 GiB gp3).
   - **Why:** EBS is durable block storage that persists independently of the instance. gp3 is cost-effective SSD.
7. Add **User data** (a startup script), e.g., to install a web server.
   - **Why:** User data runs at first boot, letting you automate setup instead of configuring manually.
8. Attach an **IAM role** (instance profile).
   - **Why:** So the instance can call AWS APIs (e.g., read S3) without hardcoded credentials.
9. **Launch**, then connect via **EC2 Instance Connect** or `ssh -i key.pem ec2-user@<public-ip>`.
   - **Why:** Confirms the instance is reachable and your security group/key are correct.
10. Explore **Elastic IP** (allocate + associate).
    - **Why:** A static public IP that survives stop/start, needed for stable endpoints.
11. Create an **Auto Scaling Group** + **Launch Template** + **Application Load Balancer**.
    - **Why:** This is how real workloads scale automatically and stay highly available across AZs.

**Key options to explore:** Spot vs On-Demand vs Reserved vs Savings Plans, placement groups, EBS snapshots, AMI creation, instance metadata service v2 (IMDSv2), termination protection.

**Practice Challenge:** Use user data to deploy a web page, put the instance behind an ALB, and confirm it's reachable through the load balancer DNS name.

### Exercises (build muscle memory)

1. **Warm-up:** Launch a `t2.micro` Amazon Linux instance, connect via EC2 Instance Connect, and run `uname -a`.
   - **Verify it works:** You get a shell and see the kernel info — the instance is live and reachable.
2. **User data:** Relaunch with this user data and confirm a web server auto-installs:
   ```bash
   #!/bin/bash
   dnf install -y httpd
   systemctl enable --now httpd
   echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
   ```
   - **Verify it works:** Browse to `http://<public-ip>` and see the page — no manual setup needed.
3. **Resize:** Stop the instance, change its type from `t2.micro` to `t3.small`, and start it again.
   - **Verify it works:** The instance boots with more resources (check with `free -h` / `nproc`).
4. **Snapshot/AMI:** Create an AMI from the instance, then launch a new instance from that AMI.
   - **Verify it works:** The new instance already has your web server and page baked in.
5. **Stretch:** Attach an Elastic IP, stop/start the instance, and confirm the public IP no longer changes.

### Practice Challenge — Solution

Goal: user-data web page behind an ALB, reachable via the ALB DNS name.

1. **Create a Launch Template** (EC2 → Launch templates → Create) with Amazon Linux, `t3.micro`, a security group allowing HTTP 80, and the user data from Exercise 2 above.
   - **Why a template:** The ALB/target group and Auto Scaling all reference it, and it makes instances reproducible.
2. **Create a Target Group** (EC2 → Target groups → Create) → type **Instances**, protocol **HTTP:80**, health check path `/`.
   - **Why:** The target group is the pool the ALB routes to; the health check confirms an instance can serve `/`.
3. **Launch 2 instances** from the template across two AZs, and register them in the target group.
   - **Why two AZs:** Demonstrates high availability — if one AZ fails, the other still serves.
4. **Create an Application Load Balancer** (EC2 → Load balancers → Create → Application) → internet-facing → select both public subnets → security group allowing HTTP 80 → **listener on 80 forwarding to your target group**.
5. **Wait for targets to become `healthy`** in the target group, then copy the ALB's **DNS name** and open it in a browser.
   - **Why this proves success:** Refreshing shows the page, and because the hostname in the page comes from `$(hostname -f)`, repeated refreshes may show different instance hostnames — proving the ALB is load-balancing across both.

### Cleanup after this service

- **Terminate instances** you're done with (EC2 → Instances → Terminate).
  - **Why:** *Stopping* halts compute charges but you still pay for the attached EBS volume; *terminating* removes both.
- **Delete the Auto Scaling Group first**, then the Launch Template.
  - **Why:** An ASG will relaunch instances you terminate — you must delete it (set desired count to 0) before the instances stay gone.
- **Delete the Application Load Balancer and its target groups.**
  - **Why:** ALBs bill per hour plus per LCU regardless of traffic.
- **Delete unattached EBS volumes and old snapshots** (EC2 → Volumes / Snapshots).
  - **Why:** Volumes and snapshots cost per GB-month even when not in use.
- **Release Elastic IPs** and **deregister custom AMIs** you no longer need.
  - **Why:** Idle Elastic IPs bill hourly; AMIs retain their backing snapshots (storage cost).
- **Keep the key pair** if you'll reuse it; delete it otherwise.
  - **Why:** Key pairs are free but orphaned ones add clutter.

---

# 4. S3 — Simple Storage Service

**What it is:** Virtually unlimited object storage (files/blobs) with 11 nines of durability.

### Step-by-step

1. **S3 console** → **Create bucket**.
   - **Why:** Buckets are the top-level containers for objects.
2. Enter a **globally unique bucket name** and pick a **region**.
   - **Why:** Names are global across all AWS accounts; region choice affects latency, cost, and compliance.
3. Leave **Block Public Access = ON** (default).
   - **Why:** Most data breaches come from accidentally public buckets. Keep it blocked unless you have a deliberate reason.
4. Enable **Bucket Versioning**.
   - **Why:** Versioning keeps every revision, protecting against accidental deletes/overwrites.
5. Enable **Default encryption** (SSE-S3 or SSE-KMS).
   - **Why:** Encrypts objects at rest automatically. KMS gives you audit trails and key control.
6. Upload an object; explore **Storage class** options at upload (Standard, Intelligent-Tiering, etc.).
   - **Why:** Storage class trades access speed for cost. Intelligent-Tiering auto-optimizes for unpredictable access.
7. Create a **Lifecycle rule** (Management → Lifecycle) to transition objects to **Glacier** after 30 days and expire after 365.
   - **Why:** Automates cost savings by moving cold data to cheaper tiers and deleting stale data.
8. Set up **Static website hosting** (Properties → Static website hosting).
   - **Why:** S3 can serve a website directly — the cheapest way to host static frontends.
9. Add a **Bucket policy** for controlled access:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": "*",
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::my-bucket/*"
     }]
   }
   ```
   - **Why:** Resource-based policies control access at the bucket level (needed for public web content or cross-account sharing).
10. Enable **S3 Event Notifications** → trigger a Lambda on upload.
    - **Why:** Event-driven processing (e.g., generate thumbnails when an image lands).

**Key options to explore:** Presigned URLs, Cross-Region Replication, Object Lock (WORM), Requester Pays, Access Points, S3 Transfer Acceleration, storage lens.

**Practice Challenge:** Host a static website in S3, then serve it globally via CloudFront (see Section 9).

### Exercises (build muscle memory)

1. **Warm-up:** Create a bucket, upload a file via the console and again via CLI: `aws s3 cp test.txt s3://your-bucket/`.
   - **Verify it works:** `aws s3 ls s3://your-bucket/` lists the object both ways.
2. **Versioning:** Enable versioning, upload `test.txt`, edit it, upload again, then use **Show versions** to view and restore the first version.
   - **Verify it works:** You can see multiple versions and roll back to the original content.
3. **Presigned URL:** Generate a temporary link: `aws s3 presign s3://your-bucket/test.txt --expires-in 300`.
   - **Verify it works:** The link opens the file in a browser and stops working after 5 minutes — proving time-limited access to a private object.
4. **Lifecycle:** Create a lifecycle rule to expire objects after 1 day and observe the rule in the Management tab.
   - **Verify it works:** The rule is listed and shows the transition/expiration timeline.
5. **Stretch:** Turn on **Static website hosting**, add a bucket policy for public read, and load the website endpoint URL.

### Practice Challenge — Solution

Goal: host a static site in S3, then serve it globally over CloudFront. (This is the combined S3 + CloudFront flow; CloudFront details are in Section 9.)

1. **Create the bucket** (keep Block Public Access **ON** — with CloudFront + OAC you don't need a public bucket).
2. **Upload site files** (`index.html`, CSS, JS). Set `index.html` as the intended entry page.
3. **Enable static website hosting** (Properties → Static website hosting → Enable → index document `index.html`).
   - **Why:** Defines the default document so `/` serves your homepage.
4. **Create a CloudFront distribution** (see Section 9): Origin = this S3 bucket, **Origin Access Control (OAC)** enabled, Viewer protocol policy = **Redirect HTTP to HTTPS**, Default root object = `index.html`.
5. **Apply the OAC bucket policy** CloudFront generates (it prompts you to copy/update it) so only CloudFront can read the bucket.
   - **Why:** The bucket stays private; only the CDN can fetch objects — secure and best practice.
6. **Wait for the distribution to deploy**, then open the **CloudFront domain name** (`dxxxx.cloudfront.net`).
   - **Why this proves success:** Your site loads over HTTPS from a global edge network, while the bucket itself remains private (a direct S3 URL returns Access Denied).

### Cleanup after this service

- **Empty the bucket before deleting it** (S3 → bucket → Empty), then delete the bucket.
  - **Why:** AWS won't delete a non-empty bucket. With versioning on, you must also delete *all object versions and delete markers*.
- **Delete old object versions** if versioning is enabled (Show versions → delete).
  - **Why:** Every version consumes storage you're billed for, even hidden ones.
- **Remove lifecycle-transitioned objects in Glacier** if no longer needed.
  - **Why:** Archived data still costs per GB, plus retrieval fees.
- **Disable/delete S3 Event Notifications** pointing at Lambda.
  - **Why:** Prevents orphaned triggers from firing against functions you later delete.
- **Note:** Keep any bucket you're using for CloudTrail logs or CloudFront until those sections are done.
  - **Why:** Deleting a bucket other services depend on will break them.

---

# 5. RDS — Relational Database Service

**What it is:** Managed relational databases (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora). AWS handles patching, backups, and failover.

### Step-by-step

1. **RDS console** → **Create database** → **Standard create**.
   - **Why:** Standard create exposes all options so you learn what each does (Easy create hides them).
2. Choose an **engine** (start with PostgreSQL or MySQL).
   - **Why:** Pick what your app needs; Aurora is AWS's high-performance MySQL/PostgreSQL-compatible option.
3. Choose a **template**: Free tier / Dev / Production.
   - **Why:** Templates preset sensible defaults; Free tier avoids charges while learning.
4. Set **DB instance identifier**, **master username**, and **password** (or use Secrets Manager).
   - **Why:** Credentials for the admin account. Secrets Manager rotation is best practice for production.
5. Pick **instance class** (`db.t3.micro`) and **storage** (gp3, enable autoscaling).
   - **Why:** Sizing controls performance/cost; storage autoscaling prevents "disk full" outages.
6. Enable **Multi-AZ deployment** (for production).
   - **Why:** Multi-AZ maintains a synchronous standby in another AZ and fails over automatically — high availability.
7. Under **Connectivity**, place it in your **VPC**, in **private subnets**, with a security group allowing port 5432/3306 only from your app tier.
   - **Why:** Databases should never be publicly reachable; restrict access to the app's security group.
8. Set **Backup retention** (e.g., 7 days) and **backup window**.
   - **Why:** Automated backups + point-in-time recovery protect against data loss.
9. Enable **Encryption at rest** and **Performance Insights**.
   - **Why:** Encryption meets compliance; Performance Insights helps you find slow queries.
10. Create the DB, then connect from your app/EC2 using the **endpoint**.
    - **Why:** The endpoint is the stable DNS name your app uses; it survives failovers.

**Key options to explore:** Read replicas, Aurora Serverless v2, parameter groups, option groups, RDS Proxy, blue/green deployments, snapshots and restore.

**Practice Challenge:** Create a read replica and observe how read traffic can be offloaded to it.

### Exercises (build muscle memory)

1. **Warm-up:** Create a free-tier PostgreSQL `db.t3.micro`, then connect from an EC2 instance in the same VPC: `psql -h <endpoint> -U postgres -d postgres`.
   - **Verify it works:** You get a `postgres=#` prompt — network path and security group are correct.
2. **Core:** Create a table and insert rows:
   ```sql
   CREATE TABLE feedback (id serial PRIMARY KEY, msg text);
   INSERT INTO feedback (msg) VALUES ('hello'), ('world');
   SELECT * FROM feedback;
   ```
   - **Verify it works:** The `SELECT` returns your two rows.
3. **Backup/restore:** Take a manual snapshot, then restore it as a new instance.
   - **Verify it works:** The restored instance contains the same `feedback` table and rows.
4. **Security check:** Try connecting from your laptop (outside the VPC).
   - **Verify it works:** It fails/times out — proving the DB is correctly locked to the VPC/security group and not public.
5. **Stretch:** Enable Performance Insights and run a slow query to see it appear in the dashboard.

### Practice Challenge — Solution

Goal: create a read replica and offload reads to it.

1. **Start from your primary instance** (must have automated backups enabled — a prerequisite for replicas).
   - **Why:** RDS builds replicas from the backup/transaction stream, so backup retention must be > 0.
2. RDS → **Databases** → select the primary → **Actions** → **Create read replica**.
3. Set a **replica identifier** (e.g., `mydb-read-1`), choose an instance class, optionally a different AZ/region.
   - **Why:** A replica in another AZ/region adds resilience and can serve nearer users.
4. **Create** and wait until it's **Available**. Note its **separate endpoint**.
   - **Why:** The replica has its own read-only endpoint distinct from the primary.
5. **Test the offload:** On the **primary** endpoint, `INSERT` a new row. Then query that row on the **replica** endpoint.
   ```sql
   -- on replica endpoint:
   SELECT * FROM feedback;      -- shows rows replicated from primary
   INSERT INTO feedback ...     -- FAILS: read-only
   ```
   - **Why this proves success:** The row you inserted on the primary appears on the replica (replication works), and writes to the replica are rejected (it's read-only) — so you can safely route read queries there to relieve the primary.

### Cleanup after this service

- **Delete read replicas first**, then the primary DB instance (RDS → Databases → Delete).
  - **Why:** Replicas depend on the primary; deleting the primary while replicas exist can fail or promote them unexpectedly.
- **Decide on a final snapshot** at deletion time.
  - **Why:** Take one if you want the data later; skip it to avoid snapshot storage cost. Either way, the running instance charge stops.
- **Delete manual snapshots** you no longer need (RDS → Snapshots).
  - **Why:** Manual snapshots persist and bill per GB until you delete them; automated ones vanish with the instance.
- **Delete the DB subnet group and custom parameter/option groups** if unused.
  - **Why:** They're free but clutter the account; removing them keeps things clean.
- **Remove the RDS secret in Secrets Manager** if you created one.
  - **Why:** Stored secrets have a small monthly charge and are a credential to clean up.

---

# 6. DynamoDB — Managed NoSQL

**What it is:** Serverless key-value and document database with single-digit-millisecond latency at any scale.

### Step-by-step

1. **DynamoDB console** → **Create table**.
   - **Why:** Tables are the core container; unlike RDS there are no servers to manage.
2. Set a **Partition key** (e.g., `userId`) and optionally a **Sort key** (e.g., `orderId`).
   - **Why:** The partition key determines data distribution; the sort key enables range queries within a partition. Good key design is *the* skill in DynamoDB.
3. Choose **Capacity mode**: **On-demand** vs **Provisioned**.
   - **Why:** On-demand auto-scales and is great for unpredictable traffic; provisioned is cheaper for steady, predictable load.
4. (Provisioned) Set **Read/Write Capacity Units** and enable **auto scaling**.
   - **Why:** RCUs/WCUs cap throughput; auto scaling adjusts them to demand to balance cost and performance.
5. Add a **Global Secondary Index (GSI)**.
   - **Why:** GSIs let you query by attributes other than the primary key — essential for flexible access patterns.
6. Enable **Point-in-time recovery (PITR)**.
   - **Why:** Continuous backups let you restore to any second in the last 35 days.
7. Enable **DynamoDB Streams**.
   - **Why:** Streams emit change events you can process with Lambda (e.g., replicate, audit, notify).
8. Enable **TTL** on a timestamp attribute.
   - **Why:** DynamoDB auto-deletes expired items (like sessions) at no cost.
9. Insert/query items in the **Explore items** UI or via CLI.
   - **Why:** Confirms your key design supports the queries your app needs.
10. Consider **Global Tables** for multi-region.
    - **Why:** Active-active replication across regions for global low latency and disaster recovery.

**Key options to explore:** DAX (in-memory cache), PartiQL queries, conditional writes, transactions, on-demand backups, export to S3.

**Practice Challenge:** Model a simple to-do app: design the keys so you can fetch "all tasks for a user" efficiently.

### Exercises (build muscle memory)

1. **Warm-up:** Create an on-demand table `Users` with partition key `userId`. Add items in the console's **Explore items** view.
   - **Verify it works:** Items appear in the table view with the attributes you entered.
2. **CLI writes/reads:**
   ```bash
   aws dynamodb put-item --table-name Users \
     --item '{"userId":{"S":"u1"},"name":{"S":"Alice"}}'
   aws dynamodb get-item --table-name Users --key '{"userId":{"S":"u1"}}'
   ```
   - **Verify it works:** `get-item` returns the exact item you put.
3. **Query vs Scan:** Add several items, then `query` by a single `userId` and separately `scan` the whole table.
   - **Verify it works:** Query returns one partition instantly; scan reads everything — you feel why query is preferred.
4. **GSI:** Add a GSI on an attribute like `email`, then query by email.
   - **Verify it works:** You can now look up users by email, not just `userId`.
5. **Stretch:** Enable TTL on an `expireAt` attribute set a few minutes out, and confirm the item disappears later.

### Practice Challenge — Solution

Goal: model a to-do app so "all tasks for a user" is a single efficient query.

1. **Use a composite primary key** (single-table design):
   - **Partition key** `PK` = `USER#<userId>`
   - **Sort key** `SK` = `TASK#<taskId>`
   - **Why:** All of one user's tasks share the same partition key, so they're physically grouped and retrievable in one query.
2. **Create the table** with those keys (String type for both).
3. **Insert sample tasks:**
   ```bash
   aws dynamodb put-item --table-name Todo --item '{"PK":{"S":"USER#u1"},"SK":{"S":"TASK#t1"},"title":{"S":"Buy milk"},"done":{"BOOL":false}}'
   aws dynamodb put-item --table-name Todo --item '{"PK":{"S":"USER#u1"},"SK":{"S":"TASK#t2"},"title":{"S":"Walk dog"},"done":{"BOOL":false}}'
   ```
4. **Fetch all tasks for a user with a single Query:**
   ```bash
   aws dynamodb query --table-name Todo \
     --key-condition-expression "PK = :u" \
     --expression-attribute-values '{":u":{"S":"USER#u1"}}'
   ```
   - **Why this proves success:** One `Query` returns every task for `u1` with no table scan — this is the efficient, scalable access pattern. To fetch only incomplete tasks, add a `begins_with(SK, "TASK#")` and filter on `done`, or model a GSI.
5. **Bonus:** To list tasks by status across users, add a GSI with `PK = STATUS#<done>` — showing how DynamoDB access patterns drive key design.

### Cleanup after this service

- **Delete practice tables** (DynamoDB → Tables → Delete). This also removes their GSIs and streams.
  - **Why:** On-demand tables cost nothing at rest, but provisioned-capacity tables bill for RCUs/WCUs continuously.
- **Turn off/verify provisioned capacity** before relying on "free" — or switch to on-demand.
  - **Why:** Provisioned capacity + auto scaling can keep billing even with zero traffic.
- **Delete on-demand backups and S3 exports** you created.
  - **Why:** Backups and exported data sit in storage you're billed for.
- **Note:** PITR and streams stop billing automatically when the table is deleted.
  - **Why:** They're table-scoped features, so no separate cleanup is needed.

---

# 7. Lambda — Serverless Functions

**What it is:** Run code without provisioning servers. You pay per request and per millisecond of compute.

### Step-by-step

1. **Lambda console** → **Create function** → **Author from scratch**.
   - **Why:** Starting from scratch teaches the fundamentals before using blueprints.
2. Set a **name**, **runtime** (Python/Node.js), and **architecture** (arm64 is cheaper).
   - **Why:** Runtime = language; arm64 (Graviton) gives better price/performance.
3. Under **Permissions**, let it create a new **execution role** or attach an existing one.
   - **Why:** The execution role defines what AWS resources the function may access (least privilege).
4. Write code in the inline editor; click **Deploy**, then **Test** with a sample event.
   - **Why:** Testing confirms your handler processes the event shape correctly.
5. Configure **Memory** and **Timeout** (Configuration → General).
   - **Why:** Memory also scales CPU; timeout caps runaway executions. Tuning memory often *lowers* cost by finishing faster.
6. Add a **Trigger** (API Gateway, S3, EventBridge, SQS, DynamoDB Streams).
   - **Why:** Triggers are the event sources that invoke your function — the heart of event-driven architecture.
7. Add **Environment variables**.
   - **Why:** Keep config (table names, endpoints) out of code; encrypt sensitive values with KMS.
8. Create/attach a **Layer**.
   - **Why:** Layers share libraries/dependencies across functions, keeping deployment packages small.
9. Set **Concurrency** (reserved/provisioned).
   - **Why:** Reserved concurrency protects downstream systems; provisioned concurrency eliminates cold starts for latency-sensitive apps.
10. View logs in **CloudWatch Logs** and traces in **X-Ray**.
    - **Why:** Observability is how you debug serverless, where there's no server to SSH into.

**Key options to explore:** Function URLs, VPC access, dead-letter queues, destinations, versions and aliases, Lambda SnapStart, container image deployment.

**Practice Challenge:** Build an S3 → Lambda pipeline that logs the name of every uploaded file.

### Exercises (build muscle memory)

1. **Warm-up:** Create a Python function that returns `{"statusCode": 200, "body": "hello"}` and run a console **Test**.
   - **Verify it works:** The test result shows the 200 response and execution logs.
2. **Input handling:** Modify it to read `event["name"]` and return `"Hello <name>"`; test with `{"name":"Alice"}`.
   - **Verify it works:** The response echoes the name you passed — proving you understand the event object.
3. **Env vars:** Add an environment variable `GREETING` and use it in the response.
   - **Verify it works:** Changing the variable changes the output without editing code.
4. **Logs:** Add `print("processing", event)` and view the output in CloudWatch Logs.
   - **Verify it works:** Your log line appears under the function's `/aws/lambda/...` log group.
5. **Stretch:** Give the function DynamoDB write permissions and have it `put_item` on each invocation.

### Practice Challenge — Solution

Goal: uploading a file to S3 automatically triggers a Lambda that logs the file name.

1. **Create the trigger bucket** (S3) — e.g., `tf-lambda-trigger-<suffix>`.
2. **Create the Lambda** `LogUploads` (Python 3.12, arm64). Its execution role needs the default logging permissions (auto-created) — no S3 read needed just to log the name.
3. **Function code:**
   ```python
   def handler(event, context):
       for record in event["Records"]:
           bucket = record["s3"]["bucket"]["name"]
           key = record["s3"]["object"]["key"]
           print(f"New file uploaded: s3://{bucket}/{key}")
       return {"statusCode": 200}
   ```
   - **Why the loop:** S3 can batch multiple object events into one invocation, so you iterate `Records`.
4. **Add the trigger:** In the Lambda console → **Add trigger** → **S3** → select your bucket → event type **All object create events** → Add.
   - **Why here:** This creates the S3 event notification and grants S3 permission to invoke the function automatically.
5. **Test it:** Upload any file to the bucket, then open the function's **CloudWatch Logs**.
   - **Why this proves success:** A new log stream shows `New file uploaded: s3://.../yourfile` within seconds — the event-driven pipeline fired end to end with zero servers.

### Cleanup after this service

- **Delete practice functions** (Lambda → Functions → Delete).
  - **Why:** Lambda has no idle cost, but leftover functions with triggers can fire unexpectedly and clutter the account.
- **Remove triggers/event source mappings** (S3 notifications, SQS/DynamoDB mappings) pointing at the function.
  - **Why:** Orphaned event sources cause errors and confusing logs after the function is gone.
- **Delete the auto-created CloudWatch Log groups** (`/aws/lambda/<name>`).
  - **Why:** Log data persists after the function is deleted and bills for storage.
- **Delete unused Lambda Layers.**
  - **Why:** Layer versions consume storage until removed.
- **Remove the execution role** if it was created only for this function.
  - **Why:** Security hygiene — drop identities that no longer serve a purpose.

---

# 8. API Gateway

**What it is:** A fully managed front door for APIs that routes requests to Lambda, HTTP endpoints, or AWS services.

### Step-by-step

1. **API Gateway console** → **Create API** → choose **HTTP API** (simpler/cheaper) or **REST API** (more features).
   - **Why:** HTTP APIs cover most needs at lower cost; REST APIs add features like request validation and API keys.
2. Define a **route** (e.g., `GET /items`).
   - **Why:** Routes map incoming HTTP requests to backend integrations.
3. Add an **integration** → **Lambda function**.
   - **Why:** This connects the route to your compute; API Gateway handles the HTTP-to-event translation.
4. Configure a **stage** (e.g., `dev`, `prod`) with **auto-deploy** or manual deploy.
   - **Why:** Stages let you run multiple environments from one API and control releases.
5. Set up an **Authorizer** (JWT/Cognito, Lambda authorizer, or IAM).
   - **Why:** Authorizers enforce authentication so only permitted callers reach your backend.
6. Enable **CORS**.
   - **Why:** Browsers block cross-origin calls without CORS headers; required for web frontends calling your API.
7. Enable **Throttling** and **usage plans / API keys** (REST).
   - **Why:** Protects your backend from spikes and lets you meter/limit per-client usage.
8. Enable **Access logging** and **execution logging** to CloudWatch.
   - **Why:** Essential for debugging request/response issues and auditing.
9. Add **Request/response mapping** or validation.
   - **Why:** Validate input early to reject bad requests before they hit Lambda, saving cost and improving security.
10. Test with the built-in **Test** feature or `curl` the invoke URL.
    - **Why:** Confirms the full path (route → auth → integration) works end to end.

**Key options to explore:** Custom domain names, WebSocket APIs, VPC links, caching, WAF integration, canary deployments.

**Practice Challenge:** Expose your Lambda from Section 7 as a public `GET /files` endpoint secured with an API key.

### Exercises (build muscle memory)

1. **Warm-up:** Create an HTTP API with one route `GET /hello` integrated to a Lambda that returns `"hi"`. Open the invoke URL in a browser.
   - **Verify it works:** The browser shows `hi` — the full route→integration path works.
2. **Path params:** Add `GET /hello/{name}` and return the name from `event["pathParameters"]["name"]`.
   - **Verify it works:** Visiting `/hello/Alice` returns `Alice`.
3. **CORS:** Enable CORS for your site origin and call the API from a browser `fetch()`.
   - **Verify it works:** The browser call succeeds instead of throwing a CORS error.
4. **Stages:** Create `dev` and `prod` stages and deploy to each.
   - **Verify it works:** Two distinct invoke URLs exist, letting you test before promoting.
5. **Stretch:** Add a `POST` route and send JSON with `curl -X POST ... -d '{"x":1}'`, echoing the body back.

### Practice Challenge — Solution

Goal: expose a Lambda as `GET /files` protected by an API key. (API keys require a **REST API**, since HTTP APIs don't support usage-plan API keys.)

1. **Create a REST API** (API Gateway → APIs → Build → REST API → New API).
2. **Create a resource** `/files` (Actions → Create Resource), then **Create Method** `GET` on it → integration type **Lambda Function** → select your function → check **Lambda Proxy integration**.
   - **Why proxy integration:** It passes the whole request to Lambda and takes the function's response as-is, keeping wiring simple.
3. **Require an API key on the method:** select the `GET` method → **Method Request** → set **API Key Required = true**.
4. **Deploy to a stage** (Actions → Deploy API → new stage `prod`). Note the invoke URL.
5. **Create an API key** (API Gateway → API Keys → Create) and a **Usage Plan** (throttle/quota); associate the plan with the `prod` stage and add the API key to the plan.
   - **Why the usage plan:** An API key only works when it's linked to a stage through a usage plan.
6. **Test:**
   ```bash
   curl https://<api-id>.execute-api.<region>.amazonaws.com/prod/files          # 403 Forbidden
   curl -H "x-api-key: <your-key>" https://<...>/prod/files                      # 200 + response
   ```
   - **Why this proves success:** The call without the key is rejected (403), and the same call *with* the `x-api-key` header succeeds — the endpoint is public but gated by the key.

### Cleanup after this service

- **Delete the API** (API Gateway → your API → Delete). This removes its routes, integrations, and stages.
  - **Why:** HTTP/REST APIs bill per request, so an idle API costs nothing — but leaving public endpoints open is a security concern.
- **Delete API keys and usage plans** you created (REST APIs).
  - **Why:** Removes live credentials and keeps the account clean.
- **Delete the access/execution CloudWatch Log groups** for the API.
  - **Why:** Retained logs accrue storage charges.
- **Remove custom domain names and their base path mappings** if configured.
  - **Why:** Frees the domain and avoids dangling DNS/cert associations.

---

# 9. CloudFront + Route 53 (Delivery & DNS)

## CloudFront — Content Delivery Network

**What it is:** A global CDN that caches content at edge locations close to users.

### Step-by-step

1. **CloudFront console** → **Create distribution**.
   - **Why:** A distribution is the CDN configuration tying an origin to edge locations.
2. Set the **Origin** (your S3 bucket or ALB/custom domain).
   - **Why:** The origin is the source of truth CloudFront pulls from and caches.
3. For S3 origins, use **Origin Access Control (OAC)**.
   - **Why:** OAC keeps the bucket private while still letting CloudFront read it — no public bucket needed.
4. Set **Viewer protocol policy** → **Redirect HTTP to HTTPS**.
   - **Why:** Forces encrypted connections for all visitors.
5. Configure **Cache behaviors** and **TTLs**.
   - **Why:** Caching rules control how long content is stored at the edge — the core lever for performance and origin offload.
6. Attach an **ACM certificate** for your custom domain.
   - **Why:** Enables HTTPS on your own domain name (certs must be in us-east-1 for CloudFront).
7. Set a **default root object** (`index.html`).
   - **Why:** So `example.com/` serves your homepage instead of an error.
8. Enable **WAF** and **logging** (optional).
   - **Why:** WAF blocks malicious traffic at the edge; logs support analytics and security review.

## Route 53 — DNS

1. **Route 53 console** → **Create hosted zone** for your domain.
   - **Why:** A hosted zone holds the DNS records that map your domain to AWS resources.
2. Create an **A record (Alias)** pointing to the CloudFront distribution.
   - **Why:** Alias records map your apex/subdomain to AWS resources for free, with no IP to manage.
3. Explore **routing policies**: Simple, Weighted, Latency, Failover, Geolocation.
   - **Why:** These enable blue/green deploys, disaster recovery, and geo-based routing.
4. Set up a **health check** + **failover routing**.
   - **Why:** Automatically routes users away from an unhealthy endpoint to a backup.

**Practice Challenge:** Serve your S3 static site through CloudFront on a custom domain via Route 53 with HTTPS.

### Exercises (build muscle memory)

1. **Warm-up:** Put your S3 static site behind a CloudFront distribution (OAC) and load the `*.cloudfront.net` URL.
   - **Verify it works:** The site loads over HTTPS from CloudFront while the bucket stays private.
2. **Caching:** Change a file in S3, reload the CloudFront URL (old version served), then create an **invalidation** for `/*`.
   - **Verify it works:** After the invalidation completes, the new content appears — proving how edge caching and invalidation work.
3. **Route 53 basics:** In your hosted zone, create a simple A/Alias record to the distribution.
   - **Verify it works:** `dig your-subdomain` (or nslookup) resolves to CloudFront.
4. **Routing policy:** Create a **weighted** record set (two records, 50/50) pointing at two targets.
   - **Verify it works:** Repeated DNS lookups return different answers, showing traffic splitting.
5. **Stretch:** Add a Route 53 **health check** and a failover record pair (primary + secondary).

### Practice Challenge — Solution

Goal: custom domain → Route 53 → CloudFront → S3, over HTTPS. (You need a domain you control.)

1. **Request an ACM certificate in `us-east-1`** (ACM must be us-east-1 for CloudFront) for `app.yourdomain.com` → validate via DNS (ACM can auto-create the record in Route 53).
   - **Why us-east-1:** CloudFront only reads certificates from that region.
2. **Create/confirm the CloudFront distribution** with your S3 origin + OAC (from the S3 challenge).
3. **Add the alternate domain name (CNAME):** in the distribution's settings, set **Alternate domain name** = `app.yourdomain.com` and **Custom SSL certificate** = your ACM cert.
   - **Why:** This tells CloudFront to answer for your domain and serve HTTPS with your cert.
4. **Create the Route 53 record:** hosted zone → **Create record** → name `app` → type **A** → **Alias = Yes** → route to **CloudFront distribution** → select yours.
   - **Why an alias:** Alias records map a name to a CloudFront distribution for free with no IP.
5. **Wait for DNS + distribution deploy**, then browse `https://app.yourdomain.com`.
   - **Why this proves success:** Your custom domain serves the site over HTTPS through the CDN — you'll see the padlock (valid cert), and the origin bucket remains private.

### Cleanup after this service

- **Disable the CloudFront distribution first, wait for it to finish, then delete it** (CloudFront → Distributions → Disable → Delete).
  - **Why:** CloudFront won't let you delete an enabled distribution; disabling can take several minutes to propagate globally.
- **Delete Route 53 records** you added (A/Alias, health checks).
  - **Why:** Health checks bill per check; dangling alias records point to a now-deleted distribution.
- **Consider the hosted zone**: keep it if you still own/use the domain, delete it otherwise.
  - **Why:** A hosted zone costs ~$0.50/month per zone — small but ongoing.
- **Leave the ACM certificate** (it's free) unless you want to tidy up.
  - **Why:** Public ACM certs have no charge, so there's no cost reason to delete them.
- **Note:** Domain *registration* fees are non-refundable and separate from the hosted zone.
  - **Why:** Deleting DNS records does not cancel a registered domain.

---

# 10. SNS + SQS (Messaging & Decoupling)

## SQS — Simple Queue Service

**What it is:** Managed message queues that decouple components so they can fail and scale independently.

### Step-by-step

1. **SQS console** → **Create queue** → **Standard** or **FIFO**.
   - **Why:** Standard = high throughput, at-least-once, best-effort ordering. FIFO = exactly-once, strict ordering. Pick based on needs.
2. Set **Visibility timeout**.
   - **Why:** How long a message is hidden after a consumer picks it up, preventing double-processing while it's being handled.
3. Set **Message retention** (up to 14 days).
   - **Why:** How long unprocessed messages survive if consumers are down.
4. Configure a **Dead-Letter Queue (DLQ)** with a `maxReceiveCount`.
   - **Why:** Messages that repeatedly fail get moved aside for inspection instead of blocking the queue forever.
5. Enable **encryption** and set an **access policy**.
   - **Why:** Protects message contents and controls which principals can send/receive.
6. Trigger a **Lambda** from the queue.
   - **Why:** Serverless consumers scale automatically with queue depth.

## SNS — Simple Notification Service

1. **SNS console** → **Create topic** → **Standard** or **FIFO**.
   - **Why:** Topics are pub/sub channels; publishers send once, all subscribers receive.
2. Create **Subscriptions** (email, SMS, Lambda, SQS, HTTP).
   - **Why:** Subscribers are the endpoints that receive published messages.
3. Set up **fan-out**: SNS topic → multiple SQS queues.
   - **Why:** One event triggers many parallel workflows — a classic decoupled pattern.
4. Add a **subscription filter policy**.
   - **Why:** Subscribers receive only messages matching their filter, avoiding unnecessary processing.

**Practice Challenge:** Build fan-out: publish an "order placed" event to SNS that lands in two SQS queues (one for billing, one for shipping), each drained by its own Lambda.

### Exercises (build muscle memory)

1. **Warm-up (SQS):** Create a standard queue, send a message from the console, then **Poll for messages** to receive it.
   - **Verify it works:** The message you sent appears when you poll.
2. **Visibility timeout:** Receive a message but don't delete it; watch it reappear after the visibility timeout.
   - **Verify it works:** The message becomes visible again — showing how at-least-once delivery + timeouts work.
3. **DLQ:** Configure a DLQ with `maxReceiveCount=2`, then receive-without-deleting 3 times.
   - **Verify it works:** After the threshold, the message moves to the DLQ.
4. **Warm-up (SNS):** Create a topic, add an **email** subscription (confirm via the email link), and publish a test message.
   - **Verify it works:** The message arrives in your inbox.
5. **Stretch:** Add a subscription **filter policy** so a subscriber only receives messages with a specific attribute.

### Practice Challenge — Solution

Goal: one SNS publish fans out to a billing queue and a shipping queue, each with its own Lambda consumer.

1. **Create two SQS queues:** `billing-queue` and `shipping-queue`.
2. **Create an SNS topic** `order-placed`.
3. **Subscribe both queues to the topic:** SNS → topic → **Create subscription** → protocol **Amazon SQS** → select each queue (do this twice). Enable **raw message delivery** if you want the plain payload.
   - **Why:** Fan-out means the single topic pushes a copy of every message to *both* queues.
4. **Add SQS access policies** allowing the SNS topic to send messages (the console usually offers to add these automatically when you subscribe).
   - **Why:** Without the policy, SNS can't deliver into the queues.
5. **Create two Lambdas** (`BillingConsumer`, `ShippingConsumer`), each with an **SQS trigger** on its respective queue; each logs the message.
6. **Test:** SNS → topic → **Publish message** with a body like `{"orderId":"123","total":50}`.
   - **Why this proves success:** Both Lambdas fire from their own queues and both log the order — one publish triggered two independent workflows, which is exactly the decoupled fan-out pattern. If one consumer is down, its messages wait safely in its queue without affecting the other.

### Cleanup after this service

- **Delete SQS queues** including any Dead-Letter Queues (SQS → Delete).
  - **Why:** Queues bill per request; idle queues are near-free but leftover ones can keep triggering consumers.
- **Delete SNS topics and their subscriptions** (SNS → Topics → Delete).
  - **Why:** Deleting the topic removes subscriptions; otherwise email/SMS subscriptions may linger.
- **Confirm no Lambda is still polling a deleted queue** (remove the event source mapping).
  - **Why:** A mapping to a missing queue produces continuous errors in logs.
- **Note:** SMS subscriptions can incur real per-message charges — remove them promptly.
  - **Why:** SMS is one of the few messaging features with a non-trivial per-use cost.

---

# 11. CloudWatch — Observability

**What it is:** The monitoring backbone — metrics, logs, alarms, dashboards, and events.

### Step-by-step

1. **CloudWatch → Metrics** → browse namespaces (EC2, Lambda, RDS...).
   - **Why:** Nearly every service publishes metrics automatically; this is where you watch system health.
2. Create an **Alarm** on a metric (e.g., EC2 CPU > 80% for 5 min).
   - **Why:** Alarms notify you (via SNS) or trigger auto scaling when thresholds are breached.
3. Explore **Logs → Log groups**; run a **Logs Insights** query.
   - **Why:** Centralized logs + a query language let you debug across many resources fast.
4. Create a **Metric filter** on a log group.
   - **Why:** Turn log patterns (e.g., "ERROR") into metrics you can alarm on.
5. Build a **Dashboard** with widgets.
   - **Why:** A single pane of glass for the metrics that matter to your team.
6. Set up **EventBridge / CloudWatch Events** rules.
   - **Why:** React to events (schedules, state changes) to trigger automation like Lambda.
7. Enable **Container/Application Insights** where relevant.
   - **Why:** Deeper, curated observability for ECS/EKS and applications.
8. Use **Synthetics Canaries** for endpoint monitoring.
   - **Why:** Proactively test user-facing endpoints on a schedule before customers notice outages.

**Practice Challenge:** Alarm on Lambda errors and get an email via SNS when your function throws.

### Exercises (build muscle memory)

1. **Warm-up:** Open **Metrics** and chart an EC2 or Lambda metric over the last hour.
   - **Verify it works:** You see a live graph — confirming the service publishes metrics automatically.
2. **Logs Insights:** Run a query on a Lambda log group:
   ```
   fields @timestamp, @message | sort @timestamp desc | limit 20
   ```
   - **Verify it works:** Recent log lines return — you can now search logs, not just scroll them.
3. **Metric filter:** Create a metric filter on a log group matching the pattern `ERROR` and graph the resulting metric.
   - **Verify it works:** Emitting an `ERROR` log increments the custom metric.
4. **Dashboard:** Build a dashboard with 3 widgets (an EC2 metric, a Lambda metric, a log widget).
   - **Verify it works:** The dashboard shows all three at once.
5. **Stretch:** Create a **composite alarm** that only alerts when two underlying alarms are both in ALARM.

### Practice Challenge — Solution

Goal: get an email when a Lambda function throws errors.

1. **Create an SNS topic** `lambda-alerts` and add an **email subscription**; click the confirmation link in your inbox.
   - **Why first:** The alarm needs a notification target that's already confirmed.
2. **Create the alarm:** CloudWatch → **Alarms** → **Create alarm** → **Select metric** → **Lambda** → **By Function Name** → pick your function's **Errors** metric.
3. **Set the condition:** Statistic **Sum**, period **1 minute**, threshold **Greater than 0**.
   - **Why Sum > 0:** Any error in the period should trigger the alert.
4. **Set the action:** "In alarm" → **Send notification to** → SNS topic `lambda-alerts`. Name the alarm and create it.
5. **Trigger it:** Invoke the function with input that makes it throw (e.g., `raise Exception("boom")` in the code, then run a Test).
   - **Why this proves success:** Within a couple of minutes the alarm flips to **In alarm** and you receive an email — you've closed the loop from failure → metric → alarm → notification, which is the core of observability.

### Cleanup after this service

- **Delete alarms and dashboards** you created (CloudWatch → Alarms / Dashboards).
  - **Why:** Each alarm and each dashboard beyond the free tier has a small monthly charge.
- **Delete or set retention on Log groups** (Logs → Log groups → Actions → Edit retention).
  - **Why:** Logs default to **never expire** and silently accumulate storage cost — set a retention (e.g., 7–30 days) instead of keeping them forever.
- **Delete Synthetics canaries** (they run on a schedule and bill per run) and their artifact S3 bucket.
  - **Why:** Canaries keep executing and charging until deleted.
- **Delete EventBridge rules** you created for scheduling.
  - **Why:** Removes scheduled triggers that would otherwise keep invoking targets.

---

# 12. CloudFormation — Infrastructure as Code

**What it is:** Define your entire infrastructure in templates (YAML/JSON) and deploy it repeatably as a "stack."

### Step-by-step

1. Write a template. Minimal example:
   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Resources:
     MyBucket:
       Type: AWS::S3::Bucket
       Properties:
         VersioningConfiguration:
           Status: Enabled
   ```
   - **Why:** Declarative templates make infrastructure versionable, reviewable, and reproducible.
2. **CloudFormation console** → **Create stack** → upload the template.
   - **Why:** A stack is a managed collection of resources created/updated/deleted together.
3. Provide **Parameters**.
   - **Why:** Parameters make templates reusable across environments (dev/prod) without editing them.
4. Review the **Change set** before updating.
   - **Why:** Change sets show exactly what will be added/modified/deleted — preventing surprise disruptions.
5. Use **Outputs** and **Exports**.
   - **Why:** Share values (like a bucket name or VPC ID) between stacks.
6. Add **Conditions**, **Mappings**, and **intrinsic functions** (`!Ref`, `!GetAtt`, `!Sub`).
   - **Why:** These make templates dynamic and environment-aware.
7. Enable **Termination protection** and **DeletionPolicy: Retain** on stateful resources.
   - **Why:** Prevents accidental deletion of databases/buckets when a stack is torn down.
8. Explore **Nested stacks** and **StackSets**.
   - **Why:** Break large infra into modules and deploy across multiple accounts/regions.

**Key alternatives to know:** AWS SAM (serverless), AWS CDK (define infra in real programming languages), Terraform (third-party, multi-cloud).

**Practice Challenge:** Recreate your S3 + Lambda pipeline entirely in a CloudFormation template.

### Exercises (build muscle memory)

1. **Warm-up:** Deploy the minimal one-bucket template from the steps above and confirm the bucket appears.
   - **Verify it works:** The stack reaches `CREATE_COMPLETE` and the bucket exists.
2. **Parameters:** Add a `BucketName` parameter and pass it at deploy time.
   - **Verify it works:** The bucket is named from your parameter — the template is now reusable.
3. **Update + change set:** Add versioning to the bucket, create a **change set**, review it, then execute.
   - **Verify it works:** The change set previews exactly the versioning change before you apply it.
4. **Outputs:** Add an `Outputs` section exporting the bucket ARN and view it on the stack's **Outputs** tab.
   - **Verify it works:** The ARN shows up after deploy.
5. **Stretch:** Add `DeletionPolicy: Retain` to the bucket, delete the stack, and confirm the bucket survives.

### Practice Challenge — Solution

Goal: recreate the S3 → Lambda "log uploads" pipeline entirely as one CloudFormation template.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: S3 to Lambda log-uploads pipeline
Resources:
  UploadBucket:
    Type: AWS::S3::Bucket
    Properties:
      NotificationConfiguration:
        LambdaConfigurations:
          - Event: s3:ObjectCreated:*
            Function: !GetAtt LogFn.Arn

  LogFnRole:
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

  LogFn:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt LogFnRole.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              for r in event["Records"]:
                  print("New file:", r["s3"]["object"]["key"])
              return {"statusCode": 200}

  AllowS3Invoke:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LogFn
      Action: lambda:InvokeFunction
      Principal: s3.amazonaws.com
      SourceArn: !GetAtt UploadBucket.Arn

Outputs:
  BucketName:
    Value: !Ref UploadBucket
```

Steps:
1. Save as `pipeline.yaml`, then deploy:
   ```bash
   aws cloudformation deploy --template-file pipeline.yaml \
     --stack-name s3-lambda-pipeline --capabilities CAPABILITY_IAM
   ```
   - **Why `CAPABILITY_IAM`:** The template creates an IAM role, which requires explicit acknowledgment.
2. **Note the `AllowS3Invoke` permission:** S3 can't trigger the function without it — a very common omission.
3. **Test:** upload a file to the created bucket and check the Lambda's CloudWatch logs.
   - **Why this proves success:** The entire pipeline (bucket + role + function + trigger + permission) came from one declarative file and works end to end — and `delete-stack` will remove it all cleanly.

### Cleanup after this service

- **Delete the stack** (CloudFormation → Stacks → Delete) — this is the *whole point* of IaC cleanup.
  - **Why:** One delete removes every resource the stack created, in dependency order, with no orphans. This is the cleanest teardown method in all of AWS.
- **Check the stack events if deletion fails**, and resolve the blocker (e.g., a non-empty S3 bucket).
  - **Why:** Stacks can get stuck on resources with `DeletionPolicy: Retain` or buckets that still hold objects.
- **Manually delete any `Retain`-policy resources** (databases, buckets) the stack left behind.
  - **Why:** You explicitly told CloudFormation to keep them, so you must remove them yourself.
- **Delete leftover CloudFormation-managed IAM roles** only if truly unused.
  - **Why:** Avoids breaking other stacks that might share a role.

---

# 13. ECS — Elastic Container Service

**What it is:** Run and orchestrate Docker containers, either on EC2 or serverless with Fargate.

### Step-by-step

1. Push a container image to **ECR** (Elastic Container Registry) → **Create repository**.
   - **Why:** ECR is the private registry ECS pulls images from; keeping images private is standard practice.
2. **ECS console** → **Create cluster** (Fargate).
   - **Why:** A cluster is the logical grouping; Fargate removes the need to manage EC2 hosts.
3. Create a **Task Definition** (image, CPU/memory, ports, env vars, IAM task role).
   - **Why:** The task definition is the blueprint for how a container runs — the container equivalent of a launch template.
4. Assign a **Task execution role** and **Task role**.
   - **Why:** Execution role lets ECS pull images/write logs; task role gives your app permission to call AWS APIs.
5. Create a **Service** from the task definition; set **desired count**.
   - **Why:** A service keeps N copies of your task running and replaces failed ones automatically.
6. Attach an **Application Load Balancer** + **target group**.
   - **Why:** Distributes traffic across container replicas and enables rolling deploys.
7. Configure **Service Auto Scaling** on CPU/memory or request count.
   - **Why:** Scales containers up/down with demand for cost efficiency and availability.
8. Send logs via the **awslogs** driver to CloudWatch.
   - **Why:** Centralized container logs for debugging.

**Know the difference:** ECS (AWS-native, simpler) vs EKS (managed Kubernetes, portable but more complex). Start with ECS + Fargate.

**Practice Challenge:** Containerize a simple web app, push to ECR, and run it on Fargate behind an ALB.

### Exercises (build muscle memory)

1. **Warm-up:** Build a tiny Docker image locally and run it: `docker run -p 8080:80 nginx`, browse `localhost:8080`.
   - **Verify it works:** nginx's page loads locally — your Docker setup is good.
2. **ECR push:** Create an ECR repo, then authenticate and push:
   ```bash
   aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
   docker tag nginx <acct>.dkr.ecr.<region>.amazonaws.com/myrepo:latest
   docker push <acct>.dkr.ecr.<region>.amazonaws.com/myrepo:latest
   ```
   - **Verify it works:** The image appears in the ECR repo in the console.
3. **Task definition:** Create a Fargate task definition referencing your ECR image, port 80.
   - **Verify it works:** The task definition saves with your image and port mapping.
4. **Run task:** Run a single task in your cluster and check it reaches `RUNNING`.
   - **Verify it works:** The task shows `RUNNING` and its logs appear in CloudWatch.
5. **Stretch:** Scale the service to 3 tasks and watch them spread across subnets/AZs.

### Practice Challenge — Solution

Goal: containerize a web app, push to ECR, run on Fargate behind an ALB.

1. **Write the app + Dockerfile:**
   ```dockerfile
   FROM public.ecr.aws/nginx/nginx:latest
   RUN echo "<h1>Hello from Fargate</h1>" > /usr/share/nginx/html/index.html
   ```
2. **Create an ECR repo** and **push** the image (see Exercise 2 above).
3. **Create an ECS cluster** (Networking only / Fargate) in your VPC.
4. **Create a task definition** (Fargate, 0.25 vCPU / 0.5 GB): container = your ECR image, **container port 80**, log driver **awslogs**, and attach a **task execution role** (`AmazonECSTaskExecutionRolePolicy`) so ECS can pull the image and write logs.
5. **Create an ALB + target group** (target type **IP**, since Fargate uses awsvpc networking), listener on port 80.
6. **Create an ECS service** from the task definition: desired count 2, place in your subnets, security group allowing HTTP 80 from the ALB, and register it with the target group.
   - **Why target type IP:** Fargate tasks get their own ENI/IP, so the target group tracks IPs, not instances.
7. **Wait for tasks `RUNNING` and targets `healthy`**, then open the **ALB DNS name**.
   - **Why this proves success:** The page loads through the ALB, served by containers with no EC2 hosts to manage — and if you kill a task, the service relaunches it automatically.

### Cleanup after this service

- **Set the service desired count to 0, then delete the service**, then the cluster.
  - **Why:** Running Fargate tasks bill per vCPU/GB-second; the service relaunches killed tasks, so scale to 0 before deleting.
- **Delete the Application Load Balancer and target groups.**
  - **Why:** ALBs bill hourly plus per LCU regardless of traffic.
- **Delete ECR repositories (and their images).**
  - **Why:** Stored images cost per GB-month; empty the repo before deleting.
- **Deregister task definitions** (optional) and **delete the awslogs Log groups**.
  - **Why:** Task definitions are free but clutter the console; log groups keep charging for storage.
- **Delete Service Auto Scaling targets/policies** if created separately.
  - **Why:** Leftover scaling policies can interfere with future services of the same name.

---

# 14. EBS + EFS (Block & File Storage)

**What it is:** Persistent storage for compute. **EBS** = block volumes attached to a single EC2 instance; **EFS** = shared network file system for many instances.

### Step-by-step (EBS)

1. **EC2 → Volumes → Create volume** (type `gp3`, set size/IOPS/throughput).
   - **Why:** gp3 lets you tune IOPS and throughput independently of size, unlike gp2.
2. **Attach** to a running instance, then format and mount it.
   - **Why:** A raw volume must be formatted with a filesystem before use.
3. Create a **Snapshot**.
   - **Why:** Snapshots are incremental backups stored in S3; the basis for backups and creating AMIs.
4. Enable **encryption**.
   - **Why:** Encrypts data at rest and all snapshots derived from the volume.

### Step-by-step (EFS)

1. **EFS console → Create file system** in your VPC.
   - **Why:** EFS is regional and can be mounted from multiple AZs simultaneously.
2. Set **performance/throughput mode** and **lifecycle** (move to Infrequent Access).
   - **Why:** Matches performance to workload and auto-tiers cold files to cut cost.
3. Mount from multiple EC2 instances via the **NFS** client.
   - **Why:** Shared storage lets a fleet of servers read/write the same files — great for web content or CMS.

**Practice Challenge:** Mount one EFS file system on two EC2 instances and confirm a file written on one appears on the other.

### Exercises (build muscle memory)

1. **Warm-up (EBS):** Create a 1 GiB gp3 volume, attach it to a running instance, then `lsblk` to see it.
   - **Verify it works:** The new block device (e.g., `/dev/xvdf`) appears unformatted.
2. **Format + mount:** `mkfs -t xfs /dev/xvdf`, `mkdir /data`, `mount /dev/xvdf /data`, write a file.
   - **Verify it works:** The file persists in `/data`; `df -h` shows the mounted volume.
3. **Snapshot:** Create a snapshot of the volume, then create a *new* volume from that snapshot and attach it.
   - **Verify it works:** The new volume already contains your file — proving snapshot restore.
4. **Persistence test:** Stop and start the instance; confirm the EBS data survives (unlike instance store).
   - **Verify it works:** Your file is still in `/data` after reboot.
5. **Stretch:** Enable encryption on a new volume and confirm it shows as encrypted.

### Practice Challenge — Solution

Goal: mount one EFS file system on two EC2 instances and see a file written on one appear on the other.

1. **Create the EFS file system** in your VPC (EFS → Create file system). Ensure it has **mount targets** in the subnets/AZs where your instances live.
2. **Create/adjust a security group for EFS** (`efs-sg`) allowing inbound **NFS (TCP 2049)** from your instances' security group.
   - **Why:** NFS traffic to EFS uses port 2049; without this rule the mount hangs.
3. **Launch two EC2 instances** in the same VPC (ideally different AZs) with the EFS client available.
4. **On each instance, mount EFS:**
   ```bash
   sudo dnf install -y amazon-efs-utils
   sudo mkdir /mnt/shared
   sudo mount -t efs <file-system-id>:/ /mnt/shared
   ```
   - **Why amazon-efs-utils:** It handles TLS and simplifies the mount command versus raw NFS.
5. **Test the shared behavior:**
   ```bash
   # on instance A:
   echo "written from A" | sudo tee /mnt/shared/hello.txt
   # on instance B:
   cat /mnt/shared/hello.txt
   ```
   - **Why this proves success:** Instance B immediately reads the file instance A wrote — confirming EFS is a single shared file system mounted concurrently across instances and AZs (something EBS cannot do).

### Cleanup after this service

- **Detach and delete EBS volumes** you created (EC2 → Volumes → Delete).
  - **Why:** Unattached volumes still bill per GB-month — a very common forgotten cost.
- **Delete EBS snapshots** you no longer need.
  - **Why:** Snapshots are incremental but still accumulate storage charges over time.
- **Unmount EFS from all instances, then delete the file system** (EFS → Delete).
  - **Why:** EFS bills per GB of stored data; you must remove mount targets/access points as part of deletion.
- **Delete EFS access points and mount targets** if not auto-removed.
  - **Why:** Dangling mount targets keep the file system from deleting cleanly.

---

# 15. CloudTrail — Governance & Audit

**What it is:** Records every API call made in your account — the audit log of *who did what, when, and from where*.

### Step-by-step

1. **CloudTrail console** → **Create trail**.
   - **Why:** A trail persists events to S3 for long-term retention (the default event history is only 90 days).
2. Send trail to a dedicated, **locked-down S3 bucket** (enable log file validation).
   - **Why:** Tamper-evident, immutable audit logs are essential for security and compliance.
3. Enable it as a **multi-region trail**.
   - **Why:** Captures activity everywhere, so attackers can't hide by operating in an unused region.
4. Enable **management events** and, selectively, **data events** (e.g., S3 object-level).
   - **Why:** Management events cover control-plane actions; data events add fine-grained (but higher-volume) resource access logging.
5. Send events to **CloudWatch Logs**.
   - **Why:** Enables real-time alerting on suspicious activity (e.g., "root login" or "policy changed").
6. Explore **Event history** to trace a specific action.
   - **Why:** Your first stop when investigating "who deleted that resource?"

**Practice Challenge:** Find the exact API call and identity behind a resource you created earlier today using Event history.

### Exercises (build muscle memory)

1. **Warm-up:** Open **Event history** and filter by **Event name = `RunInstances`** to find EC2 launches.
   - **Verify it works:** You see each launch with who did it and when.
2. **Trace an identity:** Filter by **User name** to see everything one identity did.
   - **Verify it works:** The list shows that user's actions across services.
3. **Create a trail:** Create a multi-region trail writing to a dedicated S3 bucket with log file validation.
   - **Verify it works:** Log files begin appearing in the bucket within ~15 minutes.
4. **Read a log:** Download a CloudTrail JSON log file from S3 and find a `eventName`/`sourceIPAddress` pair.
   - **Verify it works:** You can read the raw event and identify the caller's IP.
5. **Stretch:** Send the trail to CloudWatch Logs and create a metric filter + alarm on `ConsoleLogin` by root.

### Practice Challenge — Solution

Goal: find the exact API call and identity behind a resource you created earlier today.

1. **Open CloudTrail → Event history.** (This is available even without a trail, covering the last 90 days.)
2. **Set the time range** to today.
3. **Filter by resource or event:** e.g., choose **Resource name** and enter your bucket/instance name, or **Event name** like `CreateBucket` / `RunInstances`.
   - **Why:** Narrows thousands of events down to the one action you care about.
4. **Open the matching event** and expand it. Read these fields:
   - `eventName` — the exact API call (e.g., `CreateBucket`).
   - `userIdentity` — *who* (IAM user/role/root, ARN, access key).
   - `sourceIPAddress` — *from where*.
   - `eventTime` — *when*.
5. **View the full JSON** (there's a "View event" / "View record" button).
   - **Why this proves success:** You can now answer "who created this, when, and from what IP" for any resource — the core audit/forensics skill CloudTrail exists to provide.

### Cleanup after this service

- **Delete extra trails** (CloudTrail → Trails → Delete), keeping only what you need.
  - **Why:** The first copy of management events is free, but additional trails/copies and data events incur charges.
- **Empty and delete the CloudTrail log S3 bucket** once you no longer need the audit history.
  - **Why:** Trail logs pile up in S3 and bill for storage; disable the trail before deleting its bucket.
- **Remove the CloudWatch Logs integration** if you set one up.
  - **Why:** Streaming events to CloudWatch Logs adds ongoing log-storage cost.
- **Caution:** In a real/shared account, keep an org-wide trail running.
  - **Why:** Audit logging is a security baseline; only delete trails in a personal learning account.

---

## Cost Control & Cleanup (do this every session)

- **Stop or terminate** EC2 instances you're done with (terminate deletes; stop keeps EBS charges).
- **Delete** NAT Gateways when idle — they bill hourly even with no traffic.
- **Empty and delete** unused S3 buckets and old snapshots.
- **Delete** RDS instances or take a final snapshot if you need the data.
- **Delete CloudFormation stacks** to remove everything a template created in one action.
- Check the **Cost Explorer** and **Billing dashboard** weekly.
- **Why all this matters:** The most common beginner mistake is leaving billable resources (NAT gateways, RDS, Elastic IPs, load balancers) running. Cleanup discipline is a pro habit.

---

## Certification Path (optional, to validate your skills)

1. **AWS Certified Cloud Practitioner (CLF-C02)** — foundational, great after Weeks 1–4.
2. **AWS Certified Solutions Architect – Associate (SAA-C03)** — the industry-standard proof; covers most services here.
3. **AWS Certified Developer – Associate (DVA-C02)** — deeper on Lambda, DynamoDB, API Gateway.

---

## Quick Reference: What Each Service Solves

| Service | Solves |
|---------|--------|
| IAM | Who can do what (security foundation) |
| VPC | Isolated networking |
| EC2 | Virtual servers |
| S3 | Object storage |
| RDS | Managed relational databases |
| DynamoDB | Managed NoSQL at scale |
| Lambda | Run code without servers |
| API Gateway | Managed API front door |
| CloudFront | Global content delivery / caching |
| Route 53 | DNS and traffic routing |
| SNS | Pub/sub notifications |
| SQS | Decoupling via queues |
| CloudWatch | Monitoring, logs, alarms |
| CloudFormation | Infrastructure as code |
| ECS | Container orchestration |

---

*Tip: You become a pro by shipping the capstone project, breaking it, and fixing it — not by finishing this document. Build, observe, iterate.*
