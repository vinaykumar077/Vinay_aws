# AWS S3 — Complete Deep Dive (Every Feature Covered)

> **Target Audience:** 5+ year DevOps engineer preparing for job switch  
> **Goal:** Master every S3 feature with hands-on practicals, real-world scenarios, interview-ready knowledge, and resume-worthy bullet points.

---

## Table of Contents

1. [What is S3 — Core Concepts](#1-what-is-s3--core-concepts)
2. [Bucket Creation & Configuration](#2-bucket-creation--configuration)
3. [S3 Storage Classes](#3-s3-storage-classes)
4. [Versioning](#4-versioning)
5. [Bucket Policies & ACLs (Access Control)](#5-bucket-policies--acls-access-control)
6. [S3 Block Public Access](#6-s3-block-public-access)
7. [S3 Encryption (SSE-S3, SSE-KMS, SSE-C, Client-Side)](#7-s3-encryption)
8. [S3 Lifecycle Policies](#8-s3-lifecycle-policies)
9. [S3 Replication (CRR & SRR)](#9-s3-replication-crr--srr)
10. [S3 Event Notifications](#10-s3-event-notifications)
11. [S3 Static Website Hosting](#11-s3-static-website-hosting)
12. [S3 Transfer Acceleration](#12-s3-transfer-acceleration)
13. [S3 Presigned URLs](#13-s3-presigned-urls)
14. [S3 Multipart Upload](#14-s3-multipart-upload)
15. [S3 Object Lock & Governance/Compliance Mode](#15-s3-object-lock)
16. [S3 Access Points](#16-s3-access-points)
17. [S3 Inventory](#17-s3-inventory)
18. [S3 Analytics & Storage Class Analysis](#18-s3-analytics--storage-class-analysis)
19. [S3 Metrics & CloudWatch Integration](#19-s3-metrics--cloudwatch-integration)
20. [S3 Server Access Logging](#20-s3-server-access-logging)
21. [S3 Object Lambda](#21-s3-object-lambda)
22. [S3 Batch Operations](#22-s3-batch-operations)
23. [S3 Select & Glacier Select](#23-s3-select--glacier-select)
24. [S3 CORS (Cross-Origin Resource Sharing)](#24-s3-cors)
25. [S3 Requester Pays](#25-s3-requester-pays)
26. [S3 Tags (Object & Bucket Tagging)](#26-s3-tags)
27. [S3 MFA Delete](#27-s3-mfa-delete)
28. [S3 VPC Endpoints (Gateway & Interface)](#28-s3-vpc-endpoints)
29. [S3 Intelligent-Tiering Archive Configurations](#29-s3-intelligent-tiering-archive-configurations)
30. [S3 Storage Lens](#30-s3-storage-lens)
31. [Interview Questions (50+)](#interview-questions)
32. [Resume Points for DevOps Engineers](#resume-points)

---

# 1. What is S3 — Core Concepts

## What is Amazon S3?

Amazon Simple Storage Service (S3) is an **object storage service** offering industry-leading scalability, data availability, security, and performance. Unlike block storage (EBS) or file storage (EFS), S3 stores data as **objects** inside **buckets**.

### Key Terminology

| Term | Meaning |
|------|---------|
| **Bucket** | A container for objects. Globally unique name. Like a top-level folder. |
| **Object** | A file + its metadata. Each object has a **key** (full path), **value** (file data), **version ID**, and **metadata**. |
| **Key** | The full "path" to the object inside the bucket (e.g., `logs/2024/jan/app.log`). S3 is flat — there are no real folders, just key prefixes. |
| **Region** | Buckets are created in a specific AWS Region. Data stays in that region unless you replicate it. |
| **Object Size** | Min: 0 bytes, Max: **5 TB**. Single PUT limit: **5 GB** (use multipart for larger). |
| **Durability** | **99.999999999% (11 nines)** — designed to sustain the loss of 2 facilities simultaneously. |
| **Availability** | Varies by storage class — **99.99%** for S3 Standard. |

### How S3 Differs From Other Storage

| Feature | S3 (Object) | EBS (Block) | EFS (File) |
|---------|-------------|-------------|------------|
| Access | HTTP/HTTPS API | Attached to EC2 | NFS mount |
| Structure | Flat (key-value) | Block-level | Hierarchical |
| Max Size | 5 TB per object, unlimited total | 64 TB per volume | Petabytes |
| Multi-AZ | Built-in (≥ 3 AZs) | Single AZ | Multi-AZ |
| Use Case | Backups, static hosting, data lake | Boot volumes, databases | Shared file systems |

---

# 2. Bucket Creation & Configuration

## Feature Explanation

Every S3 interaction starts with a **bucket**. Bucket names are **globally unique** across all AWS accounts worldwide. Once created, the name cannot be changed.

### Bucket Naming Rules
- 3–63 characters, lowercase letters, numbers, hyphens
- Must start with a letter or number
- Cannot be formatted as an IP address (e.g., `192.168.1.1`)
- Globally unique across ALL AWS accounts

### Global Namespace vs Account-Regional Namespace

This is one of the most misunderstood concepts in S3 and a **common interview question**.

#### S3 Uses a GLOBAL Namespace (for bucket names)

S3 bucket names exist in a **single, flat, global namespace** shared across **ALL AWS accounts worldwide**. This means:

- If someone in any AWS account, in any region, has already created a bucket named `my-app-logs`, **no one else in the world** can use that name.
- Bucket names are like **domain names** — first come, first served, globally.
- This is why you often see naming patterns like `{company}-{team}-{env}-{purpose}` to avoid collisions.

```
Account A (India)  creates:  "my-app-logs"     ← ✅ Success
Account B (USA)    creates:  "my-app-logs"     ← ❌ FAILS — name already taken globally
Account B (USA)    creates:  "my-app-logs-usa"  ← ✅ Success (different name)
```

**BUT** — the **data** inside the bucket is stored in the **region you select**. The name is global, the storage is regional.

```
Bucket name "vinay-logs" → Global (unique worldwide)
Bucket data             → Stored in ap-south-1 (Mumbai) only, unless you replicate
```

#### Most Other AWS Services Use Account-Regional Namespace

Services like EC2, Lambda, DynamoDB, SQS, etc. use an **account + region scoped** namespace:

- The same name can exist in **different accounts** and **different regions** without conflict.
- Resources are isolated by `Account ID + Region`.

```
Account A, us-east-1:   Lambda "process-orders"   ← ✅
Account A, ap-south-1:  Lambda "process-orders"   ← ✅ (different region, no conflict)
Account B, us-east-1:   Lambda "process-orders"   ← ✅ (different account, no conflict)
```

#### Comparison Table

| Aspect | S3 (Global Namespace) | Most AWS Services (Account-Regional) |
|--------|----------------------|--------------------------------------|
| **Scope** | Bucket name unique **worldwide** | Resource name unique within **account + region** |
| **Collision** | Two AWS accounts CANNOT have the same bucket name | Two accounts CAN have the same Lambda/DynamoDB name |
| **Data Location** | Data stored in the chosen **region** | Data stored in the chosen **region** |
| **Why?** | S3 bucket names map to DNS hostnames (`bucket.s3.amazonaws.com`) — DNS is global | Other services are accessed via Account ID + Region in the ARN |
| **Examples** | S3 | EC2, Lambda, DynamoDB, SQS, SNS, RDS, ECS, IAM* |

> *IAM is also **global** (not regional) — IAM users, roles, and policies are account-wide but unique only within your account, not worldwide.

#### Why Is S3 Global?

The technical reason is **DNS**. Every S3 bucket gets a DNS hostname:

```
https://vinay-devops-s3-demo-2024.s3.amazonaws.com          ← Virtual-hosted style
https://s3.ap-south-1.amazonaws.com/vinay-devops-s3-demo-2024  ← Path style (deprecated)
```

Since DNS is a global system, the bucket name must be globally unique to resolve to a single endpoint. Two buckets with the same name would create a DNS conflict.

#### Interview Tip

> **Q:** "Is S3 a global service or a regional service?"  
> **A:** S3 is a **regional service** with a **global namespace**. Your data is stored in the region you choose (and stays there unless replicated). But the bucket name must be unique across all AWS accounts worldwide because bucket names map to DNS hostnames. The S3 console shows all your buckets globally (not filtered by region), which often confuses people into thinking S3 is global — but the data is regional.

#### Real-Time Scenario
> **Scenario:** Your company acquires another company. Both companies have a bucket named `prod-backups` in different AWS accounts.  
> **Reality check:** This is actually **impossible** — the second company could never have created `prod-backups` if the first company already had it. This is why enterprise naming conventions like `{company}-{account-id}-{region}-{purpose}` are critical from day one.  
> **Migration tip:** During M&A, you'll need to rename/consolidate buckets, which means creating new buckets + copying data + updating all references (IAM policies, application configs, CI/CD pipelines).

### Console Practical

**Step 1:** Go to **S3 Console** → Click **"Create bucket"**

**Step 2:** Configure:
- **Bucket name:** `vinay-devops-s3-demo-2024` (must be globally unique)
- **AWS Region:** `ap-south-1` (Mumbai) — pick closest to your users
- **Object Ownership:** "ACLs disabled (recommended)" — we'll control access via policies instead
- **Block Public Access:** Keep ALL checked (default) — we'll modify this later when learning that feature

> **Why:** We pick a region close to our users to minimize latency. ACLs are legacy; bucket policies are the modern, recommended approach. Blocking public access by default follows the principle of least privilege.

**Step 3:** Leave versioning **disabled** for now (we'll enable it in the Versioning section).

**Step 4:** Encryption — Leave as **SSE-S3** (default, free).

**Step 5:** Click **Create bucket**.

### Real-Time Scenario
> **Scenario:** Your company has a microservices architecture. Each team deploys its own services. You need a naming convention for S3 buckets so they don't clash.  
> **Solution:** Use a convention like `{company}-{team}-{env}-{purpose}` → e.g., `acme-payments-prod-logs`, `acme-orders-staging-artifacts`. Enforce this via an **SCP (Service Control Policy)** that denies `s3:CreateBucket` unless the name matches a pattern.

### Console Steps — Enforcing S3 Naming Convention via SCP

#### Prerequisites
- **AWS Organizations** must be enabled with **All features** (not just consolidated billing).
- You must be logged into the **Management Account** (root account of the organization).
- SCPs must be enabled for the organization (they are NOT enabled by default with consolidated billing only).

---

#### Step 1: Enable SCPs in AWS Organizations (if not already enabled)

1. Go to **AWS Organizations Console** → `https://console.aws.amazon.com/organizations/`
2. In the left sidebar, click **Policies**.
3. Click **Service control policies**.
4. If SCPs are not enabled, click **Enable service control policies** → Confirm.

> **Why:** SCPs are the ONLY mechanism that can restrict what API calls member accounts can make, regardless of their IAM permissions. Even an admin in a member account cannot bypass an SCP.

---

#### Step 2: Create the SCP Policy

1. In the **Service control policies** page, click **Create policy**.
2. **Policy name:** `EnforceS3BucketNamingConvention`
3. **Description:** `Denies s3:CreateBucket unless the bucket name starts with the company prefix (acme-)`
4. In the **Policy editor**, paste the following JSON:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyNonCompliantS3BucketNames",
            "Effect": "Deny",
            "Action": "s3:CreateBucket",
            "Resource": "arn:aws:s3:::*",
            "Condition": {
                "StringNotLike": {
                    "s3:prefix": "acme-*"
                }
            }
        }
    ]
}
```

> **⚠️ Important:** The `s3:prefix` condition key does NOT work for `CreateBucket`. Instead, use the **resource ARN** approach below, which is the correct and tested method:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyNonCompliantS3BucketNames",
            "Effect": "Deny",
            "Action": "s3:CreateBucket",
            "Resource": "*",
            "Condition": {
                "StringNotLikeIfExists": {
                    "aws:ResourceTag/ManagedBy": "*"
                }
            }
        },
        {
            "Sid": "AllowOnlyCompanyPrefixedBuckets",
            "Effect": "Deny",
            "Action": "s3:CreateBucket",
            "NotResource": [
                "arn:aws:s3:::acme-*"
            ]
        }
    ]
}
```

> **Why this works:** The `NotResource` element matches any S3 bucket ARN that does NOT start with `acme-`. Combined with `Effect: Deny`, this means any `CreateBucket` call for a bucket name not starting with `acme-` is denied. The second statement is the key one — it uses `NotResource` to enforce the naming pattern at the ARN level.

hi

**Simpler, single-statement version (recommended):**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EnforceS3NamingConvention",
            "Effect": "Deny",
            "Action": "s3:CreateBucket",
            "NotResource": [
                "arn:aws:s3:::acme-*"
            ]
        }
    ]
}
```

> **Reading this policy:** "Deny `s3:CreateBucket` on any resource that is NOT `arn:aws:s3:::acme-*`." In other words — you can ONLY create buckets whose name starts with `acme-`.

5. Click **Create policy**.

---

#### Step 3: Attach the SCP to an Organizational Unit (OU) or Account

1. In **AWS Organizations**, go to **AWS accounts** (left sidebar) to see your OU structure.
2. Click on the **target OU** (e.g., `Workloads`, `Production`, or the root OU to apply to ALL accounts).
3. Go to the **Policies** tab → Click **Attach** → Select `EnforceS3BucketNamingConvention` → **Attach policy**.

> **Why attach to an OU, not individual accounts?** OUs let you manage policies at scale. When a new account is moved into the OU, it automatically inherits the SCP. Attaching to individual accounts doesn't scale.

> **Tip:** NEVER remove the default `FullAWSAccess` SCP from the Root OU unless you know exactly what you're doing — it can lock out all accounts.

---

#### Step 4: Test the SCP

**Test 1 — Non-compliant name (should be DENIED):**

1. Log into a **member account** (not the management account — SCPs don't affect the management account).
2. Go to **S3 Console** → Click **Create bucket**.
3. Enter a name that violates the convention: `my-random-bucket-test`
4. Click **Create bucket** → ❌ You should see an **Access Denied** error.

**Test 2 — Compliant name (should SUCCEED):**

1. In the same member account, click **Create bucket** again.
2. Enter a compliant name: `acme-payments-dev-logs`
3. Click **Create bucket** → ✅ Bucket should be created successfully.

**Test 3 — Via CLI (from the member account):**

```bash
# This should FAIL
aws s3 mb s3://random-bucket-name-12345

# This should SUCCEED
aws s3 mb s3://acme-analytics-staging-data
```

> **Why test from CLI too?** SCPs enforce restrictions regardless of the access method (Console, CLI, SDK, CloudFormation). Testing via CLI confirms there's no console-only behavior.

---

#### Step 5: Enforce a More Specific Naming Pattern (Optional)

If you want to enforce the full `{company}-{team}-{env}-{purpose}` pattern (e.g., require at least 4 segments separated by hyphens), you can use a more granular `NotResource` with multiple wildcard patterns:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EnforceFullNamingConvention",
            "Effect": "Deny",
            "Action": "s3:CreateBucket",
            "NotResource": [
                "arn:aws:s3:::acme-*-prod-*",
                "arn:aws:s3:::acme-*-staging-*",
                "arn:aws:s3:::acme-*-dev-*",
                "arn:aws:s3:::acme-*-test-*"
            ]
        }
    ]
}
```

> **Reading this policy:** "Deny CreateBucket unless the name matches one of: `acme-{team}-prod-{purpose}`, `acme-{team}-staging-{purpose}`, `acme-{team}-dev-{purpose}`, or `acme-{team}-test-{purpose}`."

> **Why enumerate environments?** S3 ARN wildcards don't support regex — you can only use `*` and `?`. By listing the allowed environments explicitly, you prevent someone from creating `acme-payments-yolo-logs`.

---

#### Step 6: Monitor SCP Denials with CloudTrail

1. Go to **CloudTrail Console** → **Event history**.
2. Filter by:
   - **Event name:** `CreateBucket`
   - **Error code:** `AccessDenied`
3. You'll see entries like:
   ```
   Event: CreateBucket
   Error: AccessDenied
   User: arn:aws:iam::111111111111:user/developer
   Bucket: my-random-bucket (non-compliant name)
   ```

> **Why:** This gives you visibility into who is trying to violate the naming convention. In production, set up a **CloudWatch Alarm** on these events to alert your platform team.

---

#### Summary of What Happens

```
Developer tries:  s3:CreateBucket("random-logs")
                        ↓
        SCP evaluates:  Is "random-logs" in NotResource "arn:aws:s3:::acme-*"?
                        ↓
                    YES — it doesn't match "acme-*"
                        ↓
                    Effect: DENY → ❌ Access Denied
                        
Developer tries:  s3:CreateBucket("acme-payments-prod-logs")
                        ↓
        SCP evaluates:  Is "acme-payments-prod-logs" in NotResource "arn:aws:s3:::acme-*"?
                        ↓
                    NO — it DOES match "acme-*", so NotResource doesn't apply
                        ↓
                    SCP has no opinion → Falls through to IAM evaluation → ✅ Allowed
```

> **Key Insight:** SCPs are **guardrails**, not permissions. They don't GRANT access — they set the maximum boundary. The developer still needs IAM permissions to create buckets. The SCP just ensures they can't create non-compliant ones.

---

# 3. S3 Storage Classes

## Feature Explanation

S3 offers **multiple storage classes** optimized for different access patterns and cost profiles. Choosing the right class can **cut storage costs by 50–95%**.

| Storage Class | Use Case | Availability | Min Storage Duration | Retrieval Fee | Cost (approx.) |
|---------------|----------|-------------|---------------------|---------------|-----------------|
| **S3 Standard** | Frequently accessed data | 99.99% | None | None | $0.023/GB |
| **S3 Standard-IA** | Infrequently accessed, but needs fast retrieval | 99.9% | 30 days | Per-GB fee | $0.0125/GB |
| **S3 One Zone-IA** | Infrequent access, re-creatable data (one AZ only) | 99.5% | 30 days | Per-GB fee | $0.01/GB |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | 99.9% | None | No retrieval fee; monitoring fee | $0.0025/1000 objects |
| **S3 Glacier Instant Retrieval** | Archive, millisecond retrieval | 99.9% | 90 days | Per-GB fee | $0.004/GB |
| **S3 Glacier Flexible Retrieval** | Archive, minutes–hours retrieval | 99.99% | 90 days | Per-GB + per-request | $0.0036/GB |
| **S3 Glacier Deep Archive** | Long-term archive (7–10 years) | 99.99% | 180 days | 12–48 hours retrieval | $0.00099/GB |

### Important Details

- **Minimum storage duration charge:** If you store an object in S3 Standard-IA for only 5 days and then delete it, you're **still billed for 30 days**.
- **Minimum object size charge:** IA classes charge a minimum of **128 KB per object** (even if the object is 1 KB).
- **S3 Intelligent-Tiering** automatically moves objects between tiers (Frequent → Infrequent → Archive) based on access patterns — **no retrieval fees**.

### Console Practical

**Step 1:** Upload a file to your bucket → Click **"Upload"** → **"Add files"** → select a file.

**Step 2:** Before clicking Upload, expand **"Properties"** → **"Storage class"** → Select **"S3 Standard-IA"**.

> **Why:** This shows you how to set the storage class at upload time. In production, you'd rarely do this manually — instead, you use Lifecycle Policies (covered later) to automatically transition objects.

**Step 3:** After upload, click on the object → check the **"Storage class"** field in the properties tab.

**Step 4:** To change the class after upload: Select the object → **Actions** → **"Edit storage class"** → choose **S3 Glacier Instant Retrieval** → Save.

> **Why:** Changing the storage class creates a **new copy** of the object in the target class and deletes the old one. S3 handles this transparently. This is useful when you discover data is accessed less than expected.

### Real-Time Scenario
> **Scenario:** Your company stores application logs in S3. Logs from the last 30 days are queried daily (CloudWatch Logs Insights, Athena). Logs from 30–90 days are accessed weekly for incident investigations. Logs older than 90 days are needed only for annual compliance audits.  
> **Solution:**  
> - Days 0–30: **S3 Standard**  
> - Days 30–90: **S3 Standard-IA** (lifecycle policy transition)  
> - Days 90–365: **S3 Glacier Flexible Retrieval**  
> - After 365 days: **S3 Glacier Deep Archive** (or delete)  
> This can reduce log storage costs by **70–80%**.

---

# 4. Versioning

## Feature Explanation

S3 Versioning keeps **multiple variants** of an object in the same bucket. When enabled:
- Every overwrite creates a **new version** (old versions are preserved)
- Deleting an object doesn't remove it — it places a **delete marker** (a special version that hides the object)
- You can restore any previous version

### Key Points
- Versioning is set at the **bucket level** (not per-object)
- Once enabled, it can be **suspended** but **never fully disabled**
- Each version takes up storage (and you pay for all versions)
- Versioning is a **prerequisite** for Cross-Region Replication and MFA Delete

### Console Practical

**Step 1:** Go to your bucket → **Properties** tab → **Bucket Versioning** → Click **Edit** → Enable → Save.

> **Why:** Versioning protects against accidental overwrites and deletes. It's the first line of defense for data protection.

**Step 2:** Upload a file, e.g., `config.json`.

**Step 3:** Modify `config.json` locally and upload it again with the same name.

**Step 4:** In the **Objects** tab, toggle **"Show versions"** ON.

> **Why:** You now see both versions with different **Version IDs**. The latest version is served by default when someone requests `config.json`.

**Step 5:** Delete `config.json` (without showing versions). You'll see it "disappears."

**Step 6:** Toggle **"Show versions"** ON again. You'll see a **Delete Marker** at the top.

> **Why:** The delete marker is a zero-byte "tombstone" version. The actual data is still there. To restore, simply **delete the delete marker**.

**Step 7:** Select the delete marker → **Delete** → confirm. The previous version of `config.json` reappears.

> **Why:** This is how you recover accidentally deleted files. In production, combine versioning with Lifecycle Policies to auto-expire old versions after N days (to control costs).

### Real-Time Scenario
> **Scenario:** A junior engineer accidentally overwrites the Terraform state file (`terraform.tfstate`) stored in S3.  
> **Solution:** Because versioning is enabled, you go to the S3 bucket, toggle "Show versions," find the previous version of the state file, and download it. Crisis averted.  
> **Prevention:** Enable **MFA Delete** (covered later) so that deleting versions requires MFA authentication.

---

# 5. Bucket Policies & ACLs (Access Control)

## Feature Explanation

S3 access control has **two main mechanisms**:

### 1. Bucket Policies (Recommended)
- **JSON-based** policies attached to the bucket
- Can grant/deny access to **IAM users, roles, other AWS accounts, or even the public**
- **Resource-based policy** (attached to the resource, not the identity)

### 2. Access Control Lists (ACLs) — Legacy
- **Per-object or per-bucket** access grants
- Predefined groups: `AllUsers`, `AuthenticatedUsers`, `LogDelivery`
- AWS now recommends **disabling ACLs** and using bucket policies exclusively

### Policy Evaluation Logic
```
Explicit DENY  →  wins always
Explicit ALLOW →  grants access (if no deny)
No statement   →  implicit DENY
```

If an IAM policy says ALLOW but a bucket policy says DENY → **DENY wins**.

### Console Practical

**Step 1:** Go to your bucket → **Permissions** tab.

**Step 2:** Scroll to **"Bucket policy"** → Click **Edit**.

**Step 3:** Add a policy that allows a specific IAM role to read objects:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReadFromSpecificRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ACCOUNT-ID:role/MyAppRole"
            },
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::vinay-devops-s3-demo-2024",
                "arn:aws:s3:::vinay-devops-s3-demo-2024/*"
            ]
        }
    ]
}
```

> **Why:** Two resources are specified — `arn:...bucket` (for `ListBucket` — listing operations are bucket-level) and `arn:...bucket/*` (for `GetObject` — reading objects is object-level). This is a very common mistake in interviews and real work.

**Step 4:** Add a **DENY** statement to block uploads of unencrypted objects:

```json
{
    "Sid": "DenyUnencryptedUploads",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::vinay-devops-s3-demo-2024/*",
    "Condition": {
        "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "aws:kms"
        }
    }
}
```

> **Why:** This enforces that every object uploaded must use KMS encryption. Any upload without the correct encryption header is denied. This is a very common compliance requirement in enterprises.

### Real-Time Scenario
> **Scenario:** Your company uses a centralized logging bucket that receives logs from 10 different AWS accounts. Each account should only be able to write to its own prefix (e.g., account `111111111111` writes to `logs/111111111111/`).  
> **Solution:** Use a bucket policy with `Condition` keys:
> ```json
> "Condition": {
>     "StringLike": {
>         "s3:prefix": "${aws:PrincipalAccount}/*"
>     }
> }
> ```
> This ensures each account can only write to its own "folder" — a pattern called **prefix-based isolation**.

---

# 6. S3 Block Public Access

## Feature Explanation

Block Public Access is a **safety net** that overrides bucket policies and ACLs that would make data public. It has **four independent settings**:

| Setting | What It Blocks |
|---------|---------------|
| **BlockPublicAcls** | Blocks PUT requests that include a public ACL |
| **IgnorePublicAcls** | Ignores existing public ACLs on objects/buckets |
| **BlockPublicPolicy** | Blocks PUT of bucket policies that would make the bucket public |
| **RestrictPublicBuckets** | Restricts access to the bucket to only AWS service principals and authorized users |

These can be set at:
1. **Account level** (applies to ALL buckets in the account)
2. **Bucket level** (per-bucket override)

**Account-level settings override bucket-level settings.** If you block public access at the account level, no bucket policy can make a bucket public.

### Console Practical

**Step 1:** Go to **S3 Console** → Click **"Block Public Access settings for this account"** (left sidebar).

**Step 2:** Verify all four options are **checked/enabled**.

> **Why:** This is the first thing a security auditor checks. In enterprises, this is enforced via SCPs. Even if someone writes a `"Principal": "*"` bucket policy, Block Public Access prevents the bucket from becoming publicly accessible.

**Step 3:** Go to your specific bucket → **Permissions** → **Block Public Access** → Try turning off "Block all public access."

> **Why:** You're testing what happens when you relax this. In production, you would only do this for specific use cases like static website hosting (and even then, you'd use CloudFront OAC instead).

**Step 4:** Re-enable all four settings immediately after testing.

### Real-Time Scenario
> **Scenario:** Capital One's infamous 2019 data breach involved a misconfigured S3 bucket. AWS later introduced Block Public Access as a default.  
> **Your action:** In your DevOps role, write a **Config Rule** (`s3-bucket-public-read-prohibited`) that automatically flags any bucket that becomes public, and an **auto-remediation** that re-enables Block Public Access.

---

# 7. S3 Encryption

## Feature Explanation

S3 supports **four encryption methods**:

### Server-Side Encryption (SSE)

| Method | Key Management | When to Use |
|--------|---------------|-------------|
| **SSE-S3 (AES-256)** | AWS manages keys entirely | Default, simplest, no extra cost |
| **SSE-KMS** | AWS KMS manages keys; you control key policy | Compliance, audit trail (CloudTrail logs every key use) |
| **SSE-KMS with Bucket Key** | Same as SSE-KMS but reduces KMS API calls by 99% | Cost optimization when using KMS at scale |
| **SSE-C** | You provide the key with every request; AWS never stores it | You must manage keys yourself |

### Client-Side Encryption
- You encrypt data **before** uploading to S3
- AWS never sees your plaintext data
- You manage the entire encryption lifecycle

### Default Encryption
Since **January 2023**, ALL new objects are encrypted by default with **SSE-S3**. You can change the default to SSE-KMS.

### Console Practical

**Step 1:** Go to your bucket → **Properties** → scroll to **"Default encryption"** → **Edit**.

**Step 2:** Change from "SSE-S3" to **"SSE-KMS"** → Select **"AWS managed key (aws/s3)"** → Save.

> **Why:** SSE-KMS gives you an audit trail in CloudTrail — every time an object is read/written, a KMS `Decrypt`/`GenerateDataKey` event is logged. This is required for PCI-DSS, HIPAA, and many compliance frameworks.

**Step 3:** Enable **"Bucket Key"** → Save.

> **Why:** Without Bucket Key, every object operation makes a separate KMS API call ($0.03 per 10,000 requests). With Bucket Key enabled, S3 generates a short-lived bucket-level key and uses that for multiple objects — reduces KMS costs by up to **99%**.

**Step 4:** Upload an object → After upload, click on it → Check **"Server-side encryption settings"** — it should show `aws:kms`.

**Step 5:** Try uploading via CLI without encryption header (after adding a deny-unencrypted policy):
```bash
aws s3 cp test.txt s3://vinay-devops-s3-demo-2024/ --no-sign-request
```
This should fail → proving your policy enforcement works.

### Real-Time Scenario
> **Scenario:** Your company handles healthcare data (HIPAA). The compliance team mandates that all S3 data must be encrypted with a **customer-managed KMS key** (CMK), and every access must be auditable.  
> **Solution:**  
> 1. Create a CMK in KMS with a key policy granting your app roles `kms:Decrypt` and `kms:GenerateDataKey`.  
> 2. Set bucket default encryption to SSE-KMS with that CMK.  
> 3. Add a bucket policy denying any `PutObject` without the correct KMS key.  
> 4. Enable CloudTrail to capture all KMS events.  
> This gives you end-to-end encryption + audit trail.

---

# 8. S3 Lifecycle Policies

## Feature Explanation

Lifecycle policies **automate** the movement of objects between storage classes and their eventual deletion. Rules can be based on:
- **Object age** (days since creation)
- **Object version** (current vs. noncurrent versions)
- **Key prefix** or **tags**

### What You Can Do With Lifecycle Rules
1. **Transition** objects to cheaper storage classes
2. **Expire** (delete) objects after N days
3. **Delete incomplete multipart uploads** after N days
4. **Expire noncurrent versions** (when versioning is enabled)
5. **Permanently delete expired delete markers**

### Transition Flow (Waterfall)
```
S3 Standard → Standard-IA → Intelligent-Tiering → One Zone-IA
                                                   ↓
                              Glacier Instant → Glacier Flexible → Glacier Deep Archive
```
**Rules:**
- You cannot transition from a lower-cost class to a higher-cost class via lifecycle
- Minimum 30 days before transitioning from Standard to Standard-IA
- Minimum 30 days between each IA/Glacier transition

### Console Practical

**Step 1:** Go to your bucket → **Management** tab → **Lifecycle rules** → **Create lifecycle rule**.

**Step 2:** Configure:
- **Rule name:** `log-archival-policy`
- **Prefix filter:** `logs/` (apply only to objects under the logs/ prefix)
- OR **Tag filter:** `environment=production`

> **Why:** You don't want to archive ALL objects — only specific data like logs. Prefix/tag filters scope the rule precisely.

**Step 3:** Add transitions:
- Move **current versions** to **S3 Standard-IA** after **30 days**
- Move to **S3 Glacier Flexible Retrieval** after **90 days**
- Move to **S3 Glacier Deep Archive** after **180 days**

> **Why:** This implements the classic log retention pattern. Recent logs stay accessible; old logs move to progressively cheaper storage automatically.

**Step 4:** Add expiration:
- **Expire current versions** after **365 days**
- **Permanently delete noncurrent versions** after **30 days**
- **Delete expired object delete markers** → Yes
- **Delete incomplete multipart uploads** after **7 days**

> **Why:** Without the incomplete multipart cleanup, failed large uploads leave behind invisible parts that keep costing you money. This is a commonly missed cost optimization.

**Step 5:** Review and **Create rule**.

### Real-Time Scenario
> **Scenario:** Your S3 bill jumped 40% month-over-month. Investigation reveals 500 GB of incomplete multipart upload parts from a failed ETL pipeline, plus 2 TB of noncurrent object versions from months of config file updates.  
> **Solution:**  
> 1. Add a lifecycle rule to **abort incomplete multipart uploads after 7 days**.  
> 2. Add a rule to **delete noncurrent versions after 30 days**.  
> 3. Use **S3 Storage Lens** (covered later) to identify the top cost-contributing buckets/prefixes.  
> This immediately recovers storage costs.

---

# 9. S3 Replication (CRR & SRR)

## Feature Explanation

S3 Replication asynchronously copies objects from a **source bucket** to a **destination bucket**.

| Type | Full Name | Source → Destination | Use Case |
|------|-----------|---------------------|----------|
| **CRR** | Cross-Region Replication | Bucket in Region A → Bucket in Region B | DR, compliance (data sovereignty), latency reduction |
| **SRR** | Same-Region Replication | Bucket in Region A → Different bucket in same Region A | Log aggregation, prod-to-test data copy, compliance copies |

### Key Points
- **Versioning must be enabled** on both source and destination buckets
- Replication is **asynchronous** (usually seconds to minutes)
- **Existing objects** are NOT replicated (only new objects after rule creation). Use **S3 Batch Replication** for existing objects.
- **Delete markers** are NOT replicated by default (can be enabled)
- You can replicate to a **different AWS account** (cross-account replication)
- **Replication Time Control (RTC):** Guarantees 99.99% of objects replicated within **15 minutes** (costs extra)

### Console Practical

**Step 1:** Create a **second bucket** in a different region, e.g., `vinay-devops-s3-replica-us-east-1` in `us-east-1`. Enable **versioning** on it.

**Step 2:** Go to your **source bucket** → **Management** tab → **Replication rules** → **Create replication rule**.

**Step 3:** Configure:
- **Rule name:** `dr-replication-to-us-east-1`
- **Source:** Entire bucket (or filter by prefix/tags)
- **Destination:** Choose the replica bucket
- **IAM Role:** Let S3 create a new role (it creates a role allowing `s3:GetObject` on source and `s3:ReplicateObject` on destination)
- **Storage class for replicated objects:** Keep same (or choose a cheaper class like S3 Standard-IA)

> **Why:** Replicating to a cheaper storage class at the destination saves money for DR copies that are rarely accessed.

**Step 4:** **Do NOT** check "Replicate existing objects" yet (that triggers a Batch Replication job).

**Step 5:** Upload a new file to the **source bucket**. Wait 30–60 seconds. Check the **destination bucket** — the file should appear.

> **Why:** This validates that real-time replication is working. In production, you'd set up a CloudWatch metric `ReplicationLatency` to monitor this.

**Step 6:** Delete the file in the source bucket. Check the destination — the delete marker is NOT replicated by default.

> **Why:** This is intentional — it prevents accidental bulk deletes from propagating to your DR copy. You can enable delete marker replication if you want synchronized deletes.

### Real-Time Scenario
> **Scenario:** Your company has a regulatory requirement that all customer data must have a copy in a different geographic region. Additionally, EU data must stay in EU regions.  
> **Solution:**  
> - Set up **CRR** from `eu-west-1` (Ireland) to `eu-central-1` (Frankfurt) — both EU regions for GDPR compliance.  
> - Use **Replication Time Control (RTC)** + **S3 Replication Metrics** to prove the 15-minute SLA to auditors.  
> - For the analytics team in `us-east-1`, create a separate replication rule with **anonymized/aggregated** data only.

---

# 10. S3 Event Notifications

## Feature Explanation

S3 can send notifications when specific events occur on objects. This enables **event-driven architectures**.

### Supported Events
- `s3:ObjectCreated:*` — any upload (PUT, POST, COPY, CompleteMultipartUpload)
- `s3:ObjectRemoved:*` — any delete
- `s3:ObjectRestore:*` — Glacier restore initiated/completed
- `s3:ReducedRedundancyLostObject` — data loss on RRS (legacy)
- `s3:Replication:*` — replication failures
- `s3:LifecycleTransition` — lifecycle actions
- `s3:IntelligentTiering` — tier changes
- `s3:ObjectTagging:*` — tag changes
- `s3:ObjectAcl:Put` — ACL changes

### Supported Destinations
1. **Amazon SNS** — fan-out to multiple subscribers
2. **Amazon SQS** — queue for processing
3. **AWS Lambda** — run code in response
4. **Amazon EventBridge** — advanced routing, filtering, and integration with 100+ AWS services (recommended for new setups)

### Console Practical

**Step 1:** Go to your bucket → **Properties** → scroll to **"Event notifications"**.

**Step 2:** Click **"Create event notification"**:
- **Name:** `image-upload-trigger`
- **Prefix:** `uploads/images/`
- **Suffix:** `.jpg`
- **Event types:** `s3:ObjectCreated:*`
- **Destination:** Lambda function (or SQS/SNS)

> **Why:** The prefix + suffix filter ensures only `.jpg` images in the `uploads/images/` prefix trigger the event. This avoids unnecessary Lambda invocations and costs.

**Step 3:** Alternatively, enable **Amazon EventBridge** integration:
- Scroll to **"Amazon EventBridge"** → **Edit** → **Enable** → Save.

> **Why:** EventBridge gives you content-based filtering, multiple targets, replay/archive capabilities, and schema discovery. It's the more powerful and flexible option. AWS recommends EventBridge for all new event-driven architectures.

### Real-Time Scenario
> **Scenario:** Users upload profile pictures to `uploads/images/`. You need to automatically generate thumbnails (100x100, 300x300) and store them in `processed/thumbnails/`.  
> **Solution:**  
> - S3 Event Notification on `s3:ObjectCreated:*` for prefix `uploads/images/` → triggers a **Lambda function** → Lambda uses the `sharp` library (Node.js) to resize the image → uploads thumbnails back to S3 under `processed/thumbnails/`.  
> **Important:** Ensure the Lambda writes to a DIFFERENT prefix than the trigger, or you'll create an **infinite loop**.

---

# 11. S3 Static Website Hosting

## Feature Explanation

S3 can host a static website (HTML, CSS, JS, images) directly. The bucket serves content over HTTP (not HTTPS natively — you need CloudFront for HTTPS).

### Key Points
- Provides a **website endpoint**: `http://<bucket-name>.s3-website-<region>.amazonaws.com`
- You specify an **index document** (e.g., `index.html`) and an **error document** (e.g., `error.html`)
- Block Public Access must be **relaxed** for the website to be publicly accessible (or use CloudFront OAC)
- S3 website endpoints do **not** support HTTPS — use CloudFront in front for production

### Console Practical

**Step 1:** Create a new bucket for this demo, e.g., `vinay-static-site-demo`.

**Step 2:** Go to **Properties** → **Static website hosting** → **Edit** → **Enable** → Set:
- **Index document:** `index.html`
- **Error document:** `error.html`

> **Why:** When someone visits the root URL, S3 serves `index.html`. When a 404 occurs, it serves `error.html` instead of an XML error — better user experience.

**Step 3:** Note the **Bucket website endpoint** URL.

**Step 4:** Upload `index.html` and `error.html`:

```html
<!-- index.html -->
<!DOCTYPE html>
<html><body><h1>Hello from S3!</h1></body></html>
```

**Step 5:** Go to **Permissions** → Turn OFF "Block public access" → Confirm.

**Step 6:** Add a bucket policy for public read:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::vinay-static-site-demo/*"
        }
    ]
}
```

> **Why:** The bucket policy grants public read access to all objects. Combined with Block Public Access being disabled, this makes the website accessible to anyone.

**Step 7:** Visit the website endpoint URL in your browser.

> **Why (Production):** In real projects, you would NEVER do this. Instead, you'd keep Block Public Access ON and put CloudFront with OAC (Origin Access Control) in front. CloudFront gives you HTTPS, caching, WAF, and geographic restrictions.

### Real-Time Scenario
> **Scenario:** Your company hosts a React/Angular SPA on S3 + CloudFront. Users get a 403 error when refreshing on routes like `/dashboard` or `/profile`.  
> **Solution:** In CloudFront, create a **custom error response** that maps 403/404 errors to `/index.html` with a 200 status code. This lets the client-side router handle the route. In S3 static website hosting, you can set the error document to `index.html` as well.

---

# 12. S3 Transfer Acceleration

## Feature Explanation

S3 Transfer Acceleration uses **CloudFront's edge locations** to speed up uploads to S3 over long distances. Instead of uploading directly to the S3 region, you upload to the **nearest edge location**, and AWS routes the data over its optimized private backbone network.

### Key Points
- Uses a special endpoint: `<bucket-name>.s3-accelerate.amazonaws.com`
- Costs **extra** ($0.04/GB on top of transfer costs)
- Only benefits **long-distance transfers** (cross-continent)
- Automatically disables if it's not faster than a normal transfer (no extra charge)
- Not compatible with buckets with periods in the name

### Console Practical

**Step 1:** Go to your bucket → **Properties** → **Transfer acceleration** → **Edit** → **Enable**.

> **Why:** This enables the accelerated endpoint. Your application can now upload to `vinay-devops-s3-demo-2024.s3-accelerate.amazonaws.com` instead of the regional endpoint.

**Step 2:** Use the **Speed Comparison tool** to test:
- Navigate to: `https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html`
- This tool uploads small payloads to multiple regions with and without acceleration and shows the speed difference.

> **Why:** If you're in India uploading to `us-east-1`, you'll see 2–4x speed improvement. If you're uploading to `ap-south-1` (same region), acceleration may not help (and won't be charged).

**Step 3:** Test upload via CLI:
```bash
aws s3 cp large-file.zip s3://vinay-devops-s3-demo-2024/ --region us-east-1 --endpoint-url https://vinay-devops-s3-demo-2024.s3-accelerate.amazonaws.com
```

### Real-Time Scenario
> **Scenario:** Your global media company has content creators in India, Brazil, and Japan uploading 4K video files (5–50 GB) to a centralized S3 bucket in `us-east-1` for processing.  
> **Solution:**  
> - Enable Transfer Acceleration on the bucket.  
> - Update the upload SDK/CLI to use the accelerate endpoint.  
> - Combine with **Multipart Upload** for reliability.  
> - Result: 60% faster uploads from India, 40% from Brazil.

---

# 13. S3 Presigned URLs

## Feature Explanation

Presigned URLs grant **temporary access** to a private S3 object (or upload) without making the bucket public or sharing AWS credentials.

### How It Works
1. You (an IAM user/role) generate a presigned URL using your credentials
2. The URL contains an **embedded signature** and **expiration time**
3. Anyone with the URL can perform the allowed action (GET/PUT) until it expires
4. No AWS credentials needed on the client side

### Key Points
- Default expiration: **3600 seconds (1 hour)**
- Max expiration: **7 days** (when generated with IAM user credentials), **12 hours** with IAM role (e.g., from Lambda)
- The URL is only valid if the **signer's credentials are still valid** (if the IAM user's access key is deactivated, the URL stops working even before expiry)

### Console Practical

**Step 1:** Upload a private file to your bucket, e.g., `reports/financial-q1-2024.pdf`.

**Step 2:** Click on the object → Click **"Object actions"** → **"Share with a presigned URL"** → Set duration (e.g., 1 hour) → **Create presigned URL**.

> **Why:** This generates a GET presigned URL. You can share this with someone who doesn't have AWS access — they can download the file using just this URL, until it expires.

**Step 3:** Copy the URL and open it in a browser or incognito window — the file downloads.

**Step 4:** Wait for the URL to expire (or set a 1-minute expiry for testing), then try again — you'll get `AccessDenied`.

**Step 5:** Generate via CLI:
```bash
aws s3 presign s3://vinay-devops-s3-demo-2024/reports/financial-q1-2024.pdf --expires-in 300
```

> **Why:** The CLI is how you'd generate presigned URLs programmatically (or from a Lambda function). The `--expires-in` is in seconds.

### Real-Time Scenario
> **Scenario:** Your mobile app lets users upload profile photos. You don't want to expose your AWS credentials in the mobile app, and you don't want all uploads going through your backend (bandwidth cost + latency).  
> **Solution:**  
> 1. Mobile app calls your backend API: "I want to upload a photo."  
> 2. Backend (Lambda) generates a **PUT presigned URL** for `uploads/{userId}/{uuid}.jpg` with a 10-minute expiry.  
> 3. Mobile app uploads directly to S3 using the presigned URL.  
> 4. S3 Event Notification triggers processing.  
> This is called **client-side upload** and is the standard pattern for mobile/web apps.

---

# 14. S3 Multipart Upload

## Feature Explanation

Multipart upload allows you to upload a single object as a set of **parts** independently, in parallel, and in any order.

### When to Use
- **Required** for objects > **5 GB** (single PUT limit is 5 GB)
- **Recommended** for objects > **100 MB**
- Each part: **5 MB – 5 GB** (except the last part, which can be smaller)
- Maximum parts: **10,000**
- Maximum object size: **5 TB**

### How It Works
1. **Initiate** the multipart upload → S3 returns an **upload ID**
2. **Upload parts** (each gets an ETag) — can be parallelized
3. **Complete** the upload (send the part list + ETags) → S3 assembles the object
4. If something fails, **abort** the upload (or let lifecycle clean it up)

### Console Practical

**Step 1:** The S3 console **automatically uses multipart upload** for files > 16 MB. Upload a file > 100 MB through the console and observe the progress bar showing parallel uploads.

> **Why:** The console handles multipart transparently, but understanding the mechanics is crucial for CLI/SDK work and debugging.

**Step 2:** Via CLI — the `aws s3 cp` command automatically uses multipart for large files. Configure thresholds:
```bash
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
aws s3 cp large-video.mp4 s3://vinay-devops-s3-demo-2024/
```

> **Why:** Tuning chunk size affects parallelism and retry granularity. Smaller chunks = more parallel streams but more HTTP overhead. For high-bandwidth connections, larger chunks are better.

**Step 3:** List incomplete multipart uploads:
```bash
aws s3api list-multipart-uploads --bucket vinay-devops-s3-demo-2024
```

> **Why:** Incomplete uploads are invisible in the console but cost you storage. This is a critical housekeeping command. Use lifecycle policies to auto-abort them after 7 days.

### Real-Time Scenario
> **Scenario:** Your CI/CD pipeline uploads a 2 GB Docker image layer to S3 as part of artifact storage. The upload fails at 1.8 GB due to a network glitch.  
> **Without multipart:** Start the entire 2 GB upload from scratch.  
> **With multipart:** Only the failed part (16 MB) needs to be retried. The other 112 parts are already uploaded.

---

# 15. S3 Object Lock

## Feature Explanation

Object Lock enforces **WORM (Write Once Read Many)** on objects — once written, objects cannot be deleted or overwritten for a fixed retention period.

### Two Retention Modes

| Mode | Who Can Delete Before Retention Expires? |
|------|------------------------------------------|
| **Governance Mode** | Only users with special IAM permission (`s3:BypassGovernanceRetention`) |
| **Compliance Mode** | **Nobody** — not even the root account. The object is truly immutable. |

### Legal Hold
- An **independent flag** (separate from retention)
- When active, the object cannot be deleted regardless of retention settings
- Can be toggled on/off by users with `s3:PutObjectLegalHold` permission
- Used during investigations or litigation

### Key Points
- Object Lock requires **versioning** to be enabled
- Can only be enabled at **bucket creation time**
- Retention is set **per-object** (or as a default on the bucket)
- Compliance mode locks are **irreversible** — even AWS Support cannot remove them

### Console Practical

**Step 1:** Create a **new bucket** with Object Lock enabled:
- Create bucket → scroll to **"Advanced settings"** → check **"Enable" under Object Lock**
- Acknowledge the warning → Create.

> **Why:** Object Lock can only be enabled at creation time. You cannot retrofit it onto an existing bucket.

**Step 2:** Set default retention:
- Go to bucket → **Properties** → **Object Lock** → **Edit**
- **Default retention mode:** Governance
- **Default retention period:** 30 days

> **Why:** Governance mode lets authorized admins override the lock — useful for testing. In production compliance scenarios (SEC 17a-4, HIPAA), you'd use Compliance mode.

**Step 3:** Upload an object. Try to delete it.

> **Why:** The delete will fail with an `AccessDenied` error because the object is locked. This proves the retention is working.

**Step 4:** Add a **Legal Hold** to an object:
- Select the object → **Properties** → **Object Lock Legal Hold** → Enable.

> **Why:** Even after the retention period expires, the Legal Hold keeps the object protected. This is used when your legal team says "preserve everything related to case XYZ."

### Real-Time Scenario
> **Scenario:** Your financial services company must retain trading records for 7 years per SEC Rule 17a-4(f). Records must be non-rewritable and non-erasable.  
> **Solution:**  
> - Create a bucket with Object Lock enabled  
> - Set default retention: **Compliance mode**, 2555 days (7 years)  
> - **Even the root account cannot delete these objects** before 7 years  
> - This satisfies SEC 17a-4 requirements

---

# 16. S3 Access Points

## Feature Explanation

Access Points are **named network endpoints** attached to a bucket that simplify managing data access at scale. Instead of one massive bucket policy, you create multiple access points, each with its own:
- **Access point policy** (simpler, scoped)
- **Network origin controls** (VPC-only or internet)
- **Block Public Access settings** (per-access-point)

### Key Points
- Each access point gets its own **ARN** and **hostname**
- Up to **10,000** access points per account per region
- Can restrict access to a **specific VPC** (VPC access point)
- Great for multi-tenant architectures

### Console Practical

**Step 1:** Go to **S3** → **Access Points** (left sidebar) → **Create access point**.

**Step 2:** Configure:
- **Name:** `analytics-team-access`
- **Bucket:** your bucket
- **Network origin:** Internet (or VPC for private access)
- **Block Public Access:** Enable all

> **Why:** The analytics team gets their own endpoint with its own policy. They don't need to modify the bucket policy (which could affect other teams).

**Step 3:** Add an **Access Point Policy**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AnalyticsReadOnly",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ACCOUNT-ID:role/AnalyticsTeamRole"
            },
            "Action": ["s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:us-east-1:ACCOUNT-ID:accesspoint/analytics-team-access",
                "arn:aws:s3:us-east-1:ACCOUNT-ID:accesspoint/analytics-team-access/object/*"
            ]
        }
    ]
}
```

> **Why:** This policy is self-contained and doesn't interfere with other access points. Much cleaner than adding dozens of statements to a single bucket policy.

### Real-Time Scenario
> **Scenario:** A shared data lake bucket is used by 5 teams (ML, Analytics, DevOps, Security, Finance). Each team needs different access levels to different prefixes.  
> **Solution:** Create 5 access points, each with its own policy scoped to the team's prefix. The ML team's access point allows read/write to `ml-models/`, the Security team's allows read-only to `audit-logs/`, etc.

---

# 17. S3 Inventory

## Feature Explanation

S3 Inventory provides **scheduled reports** (CSV, ORC, or Parquet) listing objects in a bucket along with their metadata (size, last modified, storage class, encryption status, replication status, etc.).

### Why Not Just ListObjects?
- `ListObjects` is an API call that costs money and is slow for billions of objects
- Inventory runs in the background and delivers a flat file — much cheaper and faster for analytics

### Key Points
- Delivered **daily** or **weekly**
- Output to a **destination S3 bucket** (can be the same or different)
- Available in **CSV, ORC, or Apache Parquet** format
- Can be queried with **Athena, Redshift Spectrum, or S3 Select**

### Console Practical

**Step 1:** Go to your bucket → **Management** tab → **Inventory configurations** → **Create inventory configuration**.

**Step 2:** Configure:
- **Name:** `weekly-full-inventory`
- **Scope:** Entire bucket
- **Destination bucket:** Same bucket, prefix `inventory-reports/`
- **Frequency:** Weekly
- **Format:** Apache Parquet
- **Additional fields:** Select all (size, last modified, storage class, encryption, ETag, replication status, Object Lock)

> **Why:** Parquet format is compressed and columnar — ideal for querying with Athena. Selecting all fields gives you maximum visibility into your bucket's contents.

**Step 3:** Wait for the first inventory report (takes up to 48 hours for the first delivery).

> **Why:** Unlike real-time APIs, inventory is a batch process. The first report takes longer; subsequent ones arrive on schedule.

### Real-Time Scenario
> **Scenario:** You have a bucket with 500 million objects. Management asks: "How much data is in Glacier vs Standard? What percentage is encrypted with KMS?"  
> **Solution:**  
> 1. Enable S3 Inventory with all metadata fields, output as Parquet.  
> 2. Point Athena at the inventory output.  
> 3. Run SQL: `SELECT storage_class, encryption_status, COUNT(*), SUM(size) FROM inventory GROUP BY 1, 2`  
> 4. Get the answer in seconds instead of hours of ListObject API calls.

---

# 18. S3 Analytics & Storage Class Analysis

## Feature Explanation

S3 Analytics observes access patterns of your objects over 30+ days and recommends whether to transition them to a different storage class (specifically, Standard-IA).

### Key Points
- Takes **24–48 hours** to start showing data
- Needs **30+ days** of data for meaningful recommendations
- Can be filtered by **prefix** or **tag**
- Exports results as a CSV to another S3 bucket
- Only recommends transitions to **Standard-IA** (not Glacier)

### Console Practical

**Step 1:** Go to your bucket → **Metrics** tab → **Analytics configurations** → **Create analytics configuration**.

**Step 2:** Configure:
- **Name:** `analyze-logs-prefix`
- **Filter:** Prefix `logs/`
- **Export:** Enable, destination bucket + prefix

> **Why:** By filtering on `logs/`, you analyze only your log data access patterns. After 30 days, S3 shows you charts: what percentage of data was accessed frequently vs. infrequently, and whether Standard-IA would save money.

### Real-Time Scenario
> **Scenario:** Your team is debating whether to move application logs to S3 Standard-IA. Some engineers say logs are rarely accessed; others say they're accessed daily for debugging.  
> **Solution:** Enable S3 Analytics on the `logs/` prefix. After 30 days, the data shows that only 5% of logs are accessed after the first 7 days → recommendation: transition to Standard-IA after 7 days (saves 40% on storage).

---

# 19. S3 Metrics & CloudWatch Integration

## Feature Explanation

S3 publishes two types of metrics to CloudWatch:

### 1. Storage Metrics (Free, daily)
- `BucketSizeBytes` — total size of the bucket
- `NumberOfObjects` — total object count
- Published once per day

### 2. Request Metrics (Paid, per-minute)
- `AllRequests`, `GetRequests`, `PutRequests`
- `4xxErrors`, `5xxErrors`
- `FirstByteLatency`, `TotalRequestLatency`
- `BytesDownloaded`, `BytesUploaded`
- Can be filtered by prefix or tag

### Console Practical

**Step 1:** Go to your bucket → **Metrics** tab → observe the **Bucket metrics** section (storage metrics are already enabled by default).

> **Why:** These daily metrics tell you bucket growth trends. If `BucketSizeBytes` is growing faster than expected, investigate.

**Step 2:** Create a **request metrics filter**:
- Click **"View additional charts for the request metrics"** → **"Create filter"**
- **Name:** `api-uploads-metrics`
- **Prefix:** `uploads/`

> **Why:** Per-minute request metrics let you set CloudWatch alarms — e.g., alert if `5xxErrors` exceeds a threshold, indicating S3 throttling or issues.

**Step 3:** Go to **CloudWatch** → **Metrics** → **S3** → view your metrics.

**Step 4:** Create an alarm:
- Metric: `5xxErrors` for your bucket
- Threshold: > 10 in 5 minutes
- Action: SNS notification

> **Why:** 5xx errors from S3 indicate server-side issues or throttling. An alarm lets you respond before users are affected.

### Real-Time Scenario
> **Scenario:** Your application experiences periodic slowdowns. Investigation shows `FirstByteLatency` spikes to 500ms (normally 50ms) every day at 2 AM.  
> **Solution:** Correlate with CloudWatch — turns out an ETL job at 2 AM reads millions of objects from the same prefix, causing S3 throttling (S3 has per-prefix limits of 5,500 GET/s). Fix: spread objects across multiple prefixes using a hash prefix.

---

# 20. S3 Server Access Logging

## Feature Explanation

Server Access Logging records every request made to your S3 bucket in a detailed log format. Each log record includes:
- **Requester** (IAM user, anonymous, AWS service)
- **Request time, action** (GET, PUT, DELETE)
- **HTTP status code**
- **Bytes transferred**
- **Error codes**

### Key Points
- Logs are delivered on a **best-effort** basis (some delay, not real-time)
- Target bucket must be in the **same region** as the source bucket
- **Never log to the same bucket** — creates an infinite loop of log generation!
- Use a **separate dedicated logging bucket**
- Log records are in **space-delimited** format (not JSON)

### Console Practical

**Step 1:** Create a **separate bucket** for logs, e.g., `vinay-s3-access-logs`.

**Step 2:** Go to your main bucket → **Properties** → **Server access logging** → **Edit** → **Enable**.

**Step 3:** Set target bucket: `vinay-s3-access-logs`, target prefix: `s3-logs/main-bucket/`.

> **Why:** The prefix organizes logs by source bucket — essential when multiple buckets log to the same logging bucket.

**Step 4:** Perform some actions on your main bucket (upload, download, list objects).

**Step 5:** After a few minutes, check the logging bucket — you'll see log files appearing.

> **Why:** Each log file contains multiple log records. You can query these with Athena for security analysis (e.g., "who accessed this file?") or cost analysis (e.g., "which prefix gets the most GET requests?").

### Real-Time Scenario
> **Scenario:** A sensitive file was leaked. Security team asks: "Who accessed `confidential/financials.xlsx` in the last 30 days?"  
> **Solution:**  
> 1. S3 Access Logs are stored in the logging bucket.  
> 2. Create an Athena table on the log data.  
> 3. Query: `SELECT requester, request_time, operation FROM s3_logs WHERE key = 'confidential/financials.xlsx' AND request_time > current_date - interval '30' day`  
> 4. Identify all accessors with timestamps.

---

# 21. S3 Object Lambda

## Feature Explanation

S3 Object Lambda lets you add your **own code** to process data as it is being retrieved from S3. You create an **Object Lambda Access Point** that sits between the requester and the bucket. When a GET request comes in, your Lambda function transforms the data before returning it.

### Use Cases
- **Redact** PII (personally identifiable information) before returning to unauthorized apps
- **Resize** images on the fly based on the requester
- **Decompress** files on read
- **Convert** data formats (CSV to JSON)
- **Watermark** images dynamically

### Console Practical

**Step 1:** Create a standard **S3 Access Point** first (prerequisite for Object Lambda).

**Step 2:** Go to **S3** → **Object Lambda Access Points** → **Create**.

**Step 3:** Configure:
- **Name:** `redact-pii-access-point`
- **Supporting Access Point:** the one from Step 1
- **Lambda function:** select or create a Lambda that redacts email addresses from CSV files
- **Payload:** configure the transformation settings

> **Why:** The data stored in S3 remains unchanged. Different teams access the same data through different Object Lambda Access Points with different transformations. The ML team gets full data; the marketing team gets PII-redacted data.

### Real-Time Scenario
> **Scenario:** Your data lake stores customer records with PII (names, emails, SSNs). The analytics team needs access but must not see PII (GDPR/CCPA).  
> **Solution:**  
> - Store the original data in S3 normally.  
> - Create an Object Lambda Access Point that runs a Lambda function to mask/hash PII fields.  
> - Analytics team accesses data through this access point — they see hashed values instead of real PII.  
> - No need to maintain a separate redacted copy of the dataset.

---

# 22. S3 Batch Operations

## Feature Explanation

S3 Batch Operations lets you perform **bulk operations** on billions of objects with a single API call. You provide a **manifest** (list of objects) and specify an **operation**.

### Supported Operations
- **Copy** objects (cross-bucket, cross-account, change storage class)
- **Invoke Lambda** function on each object
- **Replace all tags** / **Delete all tags**
- **Replace ACLs**
- **Restore from Glacier**
- **Object Lock retention** changes
- **Replicate existing objects** (Batch Replication)

### Console Practical

**Step 1:** Go to **S3** → **Batch Operations** (left sidebar) → **Create job**.

**Step 2:** Choose manifest source:
- Use **S3 Inventory report** (recommended for large buckets)
- Or a **CSV file** you create listing bucket + key per line

**Step 3:** Choose operation, e.g., **"Replace all object tags"**:
- Tag key: `compliance-reviewed`, Value: `true`

**Step 4:** Set IAM role with permissions to perform the operation.

**Step 5:** **Create job** → S3 validates the manifest → **Run job** when ready.

> **Why:** Tagging 100 million objects one-by-one via the API would take days and cost hundreds in API calls. Batch Operations does it efficiently with built-in retry logic, reporting, and progress tracking.

### Real-Time Scenario
> **Scenario:** After enabling S3 Replication, you realize 50 million existing objects were NOT replicated (replication only applies to new objects).  
> **Solution:** Use **S3 Batch Replication** — create a Batch Operations job using your inventory as the manifest and the "Replicate" operation. S3 replicates all existing objects to the destination bucket.

---

# 23. S3 Select & Glacier Select

## Feature Explanation

S3 Select lets you use **SQL expressions** to retrieve only a **subset** of data from an object, rather than downloading the entire object.

### Supported Formats
- CSV, JSON, Apache Parquet
- Can work with GZIP or BZIP2 compressed CSV/JSON

### Key Points
- You send a SQL query with the GET request
- S3 processes the query server-side and returns only matching data
- **Saves up to 80% on data transfer costs** and **up to 400% faster**
- Limited SQL: `SELECT`, `WHERE`, `LIMIT` — no `JOIN`, `GROUP BY`, `ORDER BY`
- For complex queries, use **Athena** (which uses S3 Select under the hood)

### Console Practical

**Step 1:** Upload a CSV file to your bucket, e.g., `data/sales-2024.csv`:
```csv
id,product,amount,region
1,Widget A,150,US
2,Widget B,300,EU
3,Widget A,200,US
4,Widget C,100,APAC
```

**Step 2:** Select the object → **Actions** → **"Query with S3 Select"**.

**Step 3:** Configure:
- **Format:** CSV
- **Delimiter:** Comma
- **First line:** Column headers
- **SQL:** `SELECT * FROM s3object s WHERE s.region = 'US'`

> **Why:** Instead of downloading the entire 5 GB CSV file and filtering client-side, S3 processes the filter server-side and returns only the matching rows. This saves bandwidth, time, and compute costs.

**Step 4:** Run → see only US records returned.

### Real-Time Scenario
> **Scenario:** Your IoT system writes daily CSVs (5 GB each) to S3 with sensor data. A dashboard needs the average temperature for sensor ID "temp-42" from today's file.  
> **Solution:**  
> - Use S3 Select: `SELECT AVG(CAST(temperature AS FLOAT)) FROM s3object s WHERE s.sensor_id = 'temp-42'`  
> - Instead of downloading 5 GB, you retrieve a few bytes of the result.  
> - For more complex analytics across multiple files, use Athena.

---

# 24. S3 CORS

## Feature Explanation

CORS (Cross-Origin Resource Sharing) controls which **web domains** are allowed to make requests to your S3 bucket from a browser. Without CORS configured, a website on `app.example.com` cannot load images/files from your S3 bucket via JavaScript.

### Key Points
- CORS is a **browser-enforced** security mechanism
- Only relevant for **browser-based access** (not CLI/SDK)
- Configured as an XML or JSON document on the bucket
- You specify **allowed origins**, **allowed methods**, **allowed headers**, and **max age** (cache duration)

### Console Practical

**Step 1:** Go to your bucket → **Permissions** → scroll to **"Cross-origin resource sharing (CORS)"** → **Edit**.

**Step 2:** Add CORS configuration:
```json
[
    {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["GET", "PUT", "POST"],
        "AllowedOrigins": ["https://app.example.com"],
        "ExposeHeaders": ["ETag", "x-amz-request-id"],
        "MaxAgeSeconds": 3600
    }
]
```

> **Why:** This allows `app.example.com` to make GET/PUT/POST requests to the bucket from a browser. `AllowedHeaders: ["*"]` permits any request headers. `MaxAgeSeconds: 3600` caches the preflight response for 1 hour (reducing OPTIONS calls).

### Real-Time Scenario
> **Scenario:** Your React app at `app.example.com` uploads files directly to S3 using presigned URLs. The upload fails in the browser with a CORS error, but works fine from Postman.  
> **Solution:** Postman doesn't enforce CORS — only browsers do. Add a CORS rule allowing `https://app.example.com` with `PUT` method. Also ensure `AllowedHeaders` includes `Content-Type` and any custom headers your app sends.

---

# 25. S3 Requester Pays

## Feature Explanation

Normally, the **bucket owner** pays for all storage and data transfer costs. With **Requester Pays**, the **requester** (the person downloading data) pays for the data transfer and request costs. The bucket owner still pays for storage.

### Key Points
- Anonymous access is **not allowed** — requesters must be authenticated AWS users
- Requesters add `x-amz-request-payer: requester` header to their requests
- Common for **public datasets** shared with external parties

### Console Practical

**Step 1:** Go to your bucket → **Properties** → **Requester pays** → **Edit** → **Enable**.

> **Why:** If you host a large public dataset (e.g., genomics data, satellite imagery), you don't want to pay for everyone's downloads. Requester Pays shifts the transfer cost to the data consumer.

**Step 2:** Try accessing an object without the header — it will fail.

**Step 3:** Access with the header:
```bash
aws s3 cp s3://vinay-devops-s3-demo-2024/public-dataset/data.csv ./data.csv --request-payer requester
```

### Real-Time Scenario
> **Scenario:** Your genomics company publishes a 50 TB reference genome dataset for researchers worldwide. Without Requester Pays, you'd pay $4,500/month in transfer costs.  
> **Solution:** Enable Requester Pays. Each researcher who downloads the data pays their own transfer costs. This is exactly how AWS Open Data programs work.

---

# 26. S3 Tags (Object & Bucket Tagging)

## Feature Explanation

Tags are **key-value pairs** attached to buckets or individual objects. They enable:
- **Cost allocation** — track S3 costs by project, team, environment
- **Access control** — bucket policies can use tag-based conditions
- **Lifecycle management** — lifecycle rules can target tagged objects
- **Analytics** — S3 Analytics can filter by tags

### Key Points
- **Bucket tags:** Up to 50 tags per bucket
- **Object tags:** Up to 10 tags per object
- Tags are **case-sensitive**
- Tag changes are **not versioned** (changing a tag on a versioned object doesn't create a new version)

### Console Practical

**Step 1:** Go to your bucket → **Properties** → **Tags** → **Edit** → Add tags:
- `Project`: `feedback-portal`
- `Environment`: `production`
- `Team`: `devops`
- `CostCenter`: `CC-1234`

> **Why:** These tags appear in your AWS Cost Explorer, letting you filter S3 costs by project/team. This is critical for chargeback in multi-team organizations.

**Step 2:** Upload an object and add object-level tags:
- Select object → **Properties** → **Tags** → **Edit**
- `DataClassification`: `confidential`
- `RetentionPolicy`: `7-years`

> **Why:** Object tags can drive lifecycle policies (e.g., "transition all objects tagged `RetentionPolicy=7-years` to Glacier after 90 days") and bucket policies (e.g., "deny delete on objects tagged `DataClassification=confidential`").

### Real-Time Scenario
> **Scenario:** Your company needs to know how much S3 storage each of 5 product teams is using for cost allocation.  
> **Solution:**  
> 1. Tag each bucket with `Team` = team name.  
> 2. Activate the `Team` tag as a **cost allocation tag** in the Billing console.  
> 3. Cost Explorer now shows S3 costs broken down by team.

---

# 27. S3 MFA Delete

## Feature Explanation

MFA Delete adds an **extra layer of protection** by requiring **Multi-Factor Authentication** to:
1. **Delete an object version** permanently
2. **Change the versioning state** of the bucket (suspend versioning)

### Key Points
- Can only be enabled by the **root account** via the **CLI/API** (not console)
- Versioning must be enabled first
- Only the **root account** can enable/disable MFA Delete
- Normal deletes (which create delete markers) are NOT affected — only permanent deletes of specific versions

### Console Practical

**Note:** MFA Delete **cannot be enabled from the console** — this is intentional for security.

**Step 1:** Use the root account's CLI credentials:
```bash
aws s3api put-bucket-versioning \
  --bucket vinay-devops-s3-demo-2024 \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::ACCOUNT-ID:mfa/root-account-mfa-device TOTP-CODE"
```

> **Why:** Restricting MFA Delete to root + CLI prevents even a compromised admin IAM user from disabling it. This is the highest level of delete protection S3 offers.

**Step 2:** Try deleting a specific object version — it will require MFA.

### Real-Time Scenario
> **Scenario:** A disgruntled ex-employee still has IAM credentials (due to delayed offboarding). They attempt to permanently delete objects from the backup bucket.  
> **Solution:** With MFA Delete enabled, they cannot permanently delete any object version without the root account's MFA device. Combined with aggressive IAM key rotation and CloudTrail alerts, this provides defense in depth.

---

# 28. S3 VPC Endpoints

## Feature Explanation

VPC Endpoints allow your resources in a VPC (EC2, Lambda, ECS) to access S3 **privately** without going through the public internet, NAT Gateway, or internet gateway.

### Two Types

| Type | How It Works | Cost |
|------|-------------|------|
| **Gateway Endpoint** | Route table entry; traffic stays on AWS private network | **Free** |
| **Interface Endpoint (PrivateLink)** | ENI in your subnet with a private IP | Per-hour + per-GB charge |

### Key Points
- **Gateway Endpoints** are the standard for S3 — free and simple
- Gateway Endpoints support **endpoint policies** (restrict which buckets are accessible)
- Gateway Endpoints work only within the VPC (not across VPC peering from the other side)
- Interface Endpoints are needed for **on-premises access** via Direct Connect/VPN

### Console Practical

**Step 1:** Go to **VPC Console** → **Endpoints** → **Create endpoint**.

**Step 2:** Configure:
- **Name:** `s3-gateway-endpoint`
- **Service:** `com.amazonaws.ap-south-1.s3` (type: Gateway)
- **VPC:** Select your VPC
- **Route tables:** Select the route tables for your private subnets

> **Why:** This adds a route to your private subnet route tables that directs S3 traffic through the Gateway Endpoint. Your EC2/ECS in private subnets can now access S3 without a NAT Gateway — saving you $0.045/GB in NAT data processing charges.

**Step 3:** Add an **endpoint policy**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSpecificBuckets",
            "Effect": "Allow",
            "Principal": "*",
            "Action": ["s3:GetObject", "s3:PutObject"],
            "Resource": [
                "arn:aws:s3:::vinay-devops-s3-demo-2024/*",
                "arn:aws:s3:::company-shared-artifacts/*"
            ]
        }
    ]
}
```

> **Why:** The endpoint policy restricts which buckets are accessible through this endpoint. This prevents data exfiltration — even if an EC2 instance is compromised, it can only access approved buckets.

### Real-Time Scenario
> **Scenario:** Your ECS tasks in a private subnet transfer 500 GB/day to S3 through a NAT Gateway, costing $675/month (500 × 30 × $0.045).  
> **Solution:** Create an S3 Gateway Endpoint (free). Route S3 traffic through the endpoint instead of NAT. **Monthly savings: $675/month → $0/month**.

---

# 29. S3 Intelligent-Tiering Archive Configurations

## Feature Explanation

S3 Intelligent-Tiering automatically moves objects between access tiers based on usage patterns. You can extend it with **archive tiers**:

| Tier | Access Pattern | Retrieval Time | Auto |
|------|---------------|----------------|------|
| **Frequent Access** | Regularly accessed | Instant | Default |
| **Infrequent Access** | Not accessed for 30 days | Instant | Automatic |
| **Archive Instant Access** | Not accessed for 90 days | Milliseconds | Automatic |
| **Archive Access** | Not accessed for 90+ days | 3–5 hours | Opt-in |
| **Deep Archive Access** | Not accessed for 180+ days | 12–48 hours | Opt-in |

### Key Points
- **No retrieval fees** (unlike Standard-IA or Glacier)
- Small **monitoring fee** per object ($0.0025 per 1,000 objects/month)
- Objects < **128 KB** are always stored in Frequent Access (not monitored or tiered)
- Archive Access and Deep Archive must be **explicitly enabled** via configuration

### Console Practical

**Step 1:** Go to your bucket → **Properties** → **Intelligent-Tiering Archive configurations** → **Create configuration**.

**Step 2:** Configure:
- **Name:** `auto-archive-old-data`
- **Filter:** Prefix `data/` (or apply to entire bucket)
- **Archive Access tier:** Enable, 90 days
- **Deep Archive Access tier:** Enable, 180 days

> **Why:** This extends Intelligent-Tiering to automatically archive data you stop accessing. Unlike lifecycle rules, there's no risk of archiving data you still need — Intelligent-Tiering monitors actual access patterns.

### Real-Time Scenario
> **Scenario:** You have a data lake with millions of objects. Access patterns are unpredictable — some datasets are queried daily, others not touched for months. You can't create lifecycle rules because you don't know which data will be accessed when.  
> **Solution:** Use Intelligent-Tiering with archive configurations. S3 automatically promotes/demotes objects based on actual usage. No data is ever stranded in the wrong tier.

---

# 30. S3 Storage Lens

## Feature Explanation

S3 Storage Lens provides **organization-wide visibility** into S3 usage, activity, and cost optimization opportunities. It's like a dashboard for all your S3 buckets across all accounts and regions.

### Key Points
- **Free tier:** 28 usage metrics, 14-day data retention
- **Advanced tier ($):** 35+ metrics, 15-month retention, prefix-level aggregation, CloudWatch publishing
- Can aggregate across **AWS Organizations** (all accounts)
- Provides **actionable recommendations** (e.g., "enable lifecycle rules on bucket X")

### Console Practical

**Step 1:** Go to **S3** → **Storage Lens** (left sidebar) → **Dashboards**.

**Step 2:** You'll see the **default dashboard** (automatically created). Click on it.

**Step 3:** Explore:
- **Summary:** Total storage, object count, request count
- **Cost optimization:** Incomplete multipart uploads, non-current versions, buckets without lifecycle rules
- **Data protection:** Buckets without versioning, replication, or encryption
- **Activity:** Top buckets by requests, downloads, uploads

> **Why:** Storage Lens gives you a single view of all S3 usage. The cost optimization section alone can save thousands by identifying "low-hanging fruit" like orphaned multipart uploads or buckets without lifecycle policies.

**Step 4:** Create a **custom dashboard** → enable **Advanced metrics** → enable **prefix aggregation**.

> **Why:** Prefix aggregation lets you see which "folders" within a bucket are consuming the most storage — essential for chargeback and identifying growth hotspots.

### Real-Time Scenario
> **Scenario:** Your AWS bill shows S3 costs doubled in 3 months. Finance wants to know why.  
> **Solution:**  
> 1. Open S3 Storage Lens → sort buckets by storage growth.  
> 2. Identify the top 3 buckets driving growth.  
> 3. Enable prefix aggregation → find the exact prefix responsible.  
> 4. Discovery: The CI/CD artifact bucket has no lifecycle rule → 6 months of build artifacts consuming 10 TB.  
> 5. Fix: Add a lifecycle rule to expire artifacts after 30 days.

---

# Interview Questions

## Basic Level (1–2 years experience)

**Q1: What is S3 and what are its key characteristics?**  
**A:** S3 is an object storage service offering 99.999999999% (11 nines) durability and 99.99% availability. It stores unlimited data as objects in buckets. Objects can be 0 bytes to 5 TB. Data is automatically replicated across at least 3 AZs (except One Zone-IA).

**Q2: What's the difference between S3 and EBS?**  
**A:** S3 is object storage (accessed via HTTP API, unlimited scale, multi-AZ). EBS is block storage (attached to EC2, fixed size, single AZ). S3 is for backups, static sites, data lakes. EBS is for boot volumes and databases.

**Q3: What is versioning and why would you enable it?**  
**A:** Versioning keeps multiple variants of an object. It protects against accidental deletions (via delete markers) and overwrites. It's required for replication and MFA Delete. Once enabled, it can be suspended but not disabled.

**Q4: What is the maximum file size you can upload in a single PUT?**  
**A:** 5 GB. For larger files (up to 5 TB), use multipart upload. AWS recommends multipart for files over 100 MB.

**Q5: What are S3 storage classes? Name at least 5.**  
**A:** S3 Standard, Standard-IA, One Zone-IA, Intelligent-Tiering, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive. Each optimized for different access patterns and cost profiles.

**Q6: How do you make an S3 bucket public?**  
**A:** 1) Disable Block Public Access. 2) Add a bucket policy with `"Principal": "*"` and `"Effect": "Allow"` on `s3:GetObject`. However, in production, you should NOT make buckets public — use CloudFront with OAC instead.

**Q7: What is a presigned URL?**  
**A:** A time-limited URL that grants temporary access to a private S3 object. Generated using your AWS credentials. The recipient doesn't need AWS credentials. Expires after a configurable duration (default 1 hour, max 7 days).

---

## Intermediate Level (3–5 years experience)

**Q8: Explain the difference between S3 bucket policies and IAM policies for access control.**  
**A:**  
- **Bucket policies** are resource-based (attached to the bucket), can grant cross-account access, and can reference any principal.  
- **IAM policies** are identity-based (attached to users/roles), apply only within the same account, and scope what the identity can do.  
- If an IAM policy ALLOWs but a bucket policy DENYs → DENY wins. Explicit DENY always takes priority.

**Q9: Your S3 bucket has versioning enabled. You delete an object. What happens internally?**  
**A:** S3 creates a **delete marker** — a zero-byte placeholder version that "hides" the object. The actual data versions are preserved. To restore, delete the delete marker. To permanently delete, you must delete the specific version ID.

**Q10: How would you enforce encryption on all uploads to an S3 bucket?**  
**A:** Two approaches:  
1. Set **default bucket encryption** (SSE-S3 or SSE-KMS).  
2. Add a **bucket policy** that denies `s3:PutObject` when the `s3:x-amz-server-side-encryption` header is missing or not set to the required value.

**Q11: What's the difference between CRR and SRR?**  
**A:** CRR (Cross-Region Replication) copies objects to a bucket in a different region (for DR, compliance, latency). SRR (Same-Region Replication) copies within the same region (for log aggregation, compliance copies, prod-to-test). Both require versioning on source and destination.

**Q12: How do S3 lifecycle policies work? Give a practical example.**  
**A:** Lifecycle rules automate transitions and expirations based on object age or version status. Example: Move logs to Standard-IA after 30 days, to Glacier after 90 days, to Deep Archive after 180 days, delete after 365 days. Also clean up incomplete multipart uploads after 7 days and expire noncurrent versions after 30 days.

**Q13: What is an S3 VPC Gateway Endpoint and why is it important?**  
**A:** A Gateway Endpoint allows private resources (EC2, ECS in private subnets) to access S3 without traversing the public internet or NAT Gateway. It's free, more secure (traffic stays on AWS backbone), and saves NAT data processing costs ($0.045/GB). It's added to VPC route tables and supports endpoint policies for bucket-level access control.

**Q14: What is S3 Transfer Acceleration?**  
**A:** It uses CloudFront edge locations to speed up uploads over long distances. Users upload to the nearest edge location, then AWS routes the data over its optimized backbone. Uses endpoint `bucket.s3-accelerate.amazonaws.com`. Beneficial for cross-continent transfers; automatically disabled (no charge) if not faster.

**Q15: What happens if your S3 application suddenly gets 10,000 PUT requests/second?**  
**A:** S3 supports 3,500 PUT/s and 5,500 GET/s **per prefix**. If all requests target the same prefix, you'll see throttling (503 errors). Solution: distribute objects across multiple prefixes using a hash strategy (e.g., `a1b2c3/image.jpg` instead of `images/image.jpg`). S3 also auto-scales partitions, so sustained high traffic is eventually handled.

---

## Advanced Level (5+ years / Senior DevOps)

**Q16: Your company has 500 S3 buckets across 10 AWS accounts. How do you ensure security compliance across all of them?**  
**A:**  
1. **SCPs** at the Organizations level to enforce Block Public Access and deny unencrypted uploads.  
2. **AWS Config rules** (`s3-bucket-public-read-prohibited`, `s3-bucket-ssl-requests-only`, `s3-bucket-server-side-encryption-enabled`) with auto-remediation.  
3. **S3 Storage Lens** across the organization for visibility.  
4. **CloudTrail** (organization trail) for audit.  
5. **AWS Config Aggregator** for centralized compliance dashboard.  
6. **Guardrails** via Control Tower if using Landing Zone.

**Q17: Explain S3 Object Lock compliance mode vs. governance mode. When would you use each?**  
**A:**  
- **Compliance mode:** Absolutely immutable. Not even the root account can delete before retention expires. Used for regulatory requirements (SEC 17a-4, HIPAA). Irreversible once set.  
- **Governance mode:** Can be overridden by users with `s3:BypassGovernanceRetention` permission. Used for data protection where admins may need emergency override. Testing/staging environments.

**Q18: A client reports slow S3 downloads. How do you troubleshoot?**  
**A:**  
1. Check **S3 request metrics** (CloudWatch) for `FirstByteLatency` and `TotalRequestLatency`.  
2. Check for **throttling** (503 SlowDown errors).  
3. Verify the **Region** — is the client far from the bucket region? → Use CloudFront or Transfer Acceleration.  
4. Check if the **storage class** requires restoration (Glacier) before access.  
5. Check if large objects are being downloaded without **byte-range fetching** (parallel partial downloads).  
6. Verify the client's **network bandwidth** and check for **VPC endpoint** saturation if applicable.  
7. Check if **S3 Select** could reduce the data transferred.

**Q19: How would you design a cost-optimized S3 architecture for a data lake processing 10 TB/day?**  
**A:**  
1. **Ingestion:** Multipart upload with Transfer Acceleration for remote sources.  
2. **Hot data (0-30 days):** S3 Standard.  
3. **Warm data (30-90 days):** Lifecycle transition to Standard-IA.  
4. **Cold data (90+ days):** Intelligent-Tiering with Archive configurations.  
5. **Compliance data:** S3 Glacier Deep Archive with Object Lock.  
6. **Query:** S3 Select for simple queries, Athena for complex analytics (avoid full downloads).  
7. **Networking:** VPC Gateway Endpoint (free) instead of NAT Gateway.  
8. **Monitoring:** Storage Lens for cost visibility, lifecycle rule to abort multipart uploads.  
9. **Format:** Use Parquet/ORC instead of CSV/JSON (columnar, compressed — 80% less storage and faster queries).

**Q20: What is S3 Bucket Key and how does it reduce costs?**  
**A:** When using SSE-KMS, every object operation (read/write) calls KMS for encrypt/decrypt. At scale (millions of objects), KMS API costs add up ($0.03/10K requests). S3 Bucket Key generates a short-lived data key at the bucket level and uses it for multiple objects within a time window, reducing KMS calls by up to 99%. Downside: CloudTrail shows the bucket ARN instead of the object ARN in KMS logs (slightly less granular audit trail).

**Q21: How does S3 achieve 11 nines of durability?**  
**A:** S3 stores data redundantly across **a minimum of 3 Availability Zones** within a region. Each AZ is a physically separate data center (or group) with independent power, cooling, and networking. S3 uses checksums (MD5/SHA-256) to detect and automatically repair data corruption. It's designed to sustain the concurrent loss of 2 facilities.

**Q22: Explain S3 consistency model.**  
**A:** Since December 2020, S3 provides **strong read-after-write consistency** for all operations (PUT, DELETE, LIST). This means: after a successful PUT, any subsequent GET returns the latest version. After a DELETE, subsequent GETs return 404. LIST operations immediately reflect new/deleted objects. This is a significant improvement over the eventual consistency model used before.

**Q23: What is S3 Object Lambda? Design a real-world use case.**  
**A:** S3 Object Lambda intercepts GET requests and runs a Lambda function to transform data before returning it. Use case: A data lake stores customer records with PII. Create two Object Lambda Access Points — one for the ML team (returns full data) and one for the marketing team (Lambda masks PII fields). Same data, different views, no data duplication.

**Q24: How would you implement cross-account S3 access?**  
**A:** Three approaches:  
1. **Bucket policy:** Add a statement allowing the external account's ARN. Simplest for simple use cases.  
2. **IAM role in the bucket account** + cross-account `AssumeRole` from the other account. Better for complex permissions.  
3. **S3 Access Points:** Create an access point with a policy granting cross-account access. Best for multi-tenant scenarios.  
4. **AWS RAM (Resource Access Manager):** Share access points across accounts in an Organization.

**Q25: You notice S3 costs increasing despite stable data volume. What could be the cause?**  
**A:** Common culprits:  
1. **Noncurrent versions** accumulating (versioning enabled without lifecycle expiration).  
2. **Incomplete multipart uploads** (failed large uploads leaving orphaned parts).  
3. **Request costs** — high volume of LIST/GET/PUT operations.  
4. **Data transfer costs** — egress to the internet or across regions.  
5. **Lifecycle transition costs** — moving data to cheaper classes has a per-request transition fee.  
6. **KMS API costs** — if using SSE-KMS without Bucket Key.  
Diagnose with: S3 Storage Lens, Cost Explorer with S3-specific filters, S3 request metrics.

**Q26: What's the difference between S3 server access logging and CloudTrail data events for S3?**  
**A:**  

| Aspect | Server Access Logging | CloudTrail Data Events |
|--------|----------------------|----------------------|
| **Delivery** | Best-effort, delayed | Near real-time, reliable |
| **Format** | Space-delimited log files | JSON (structured) |
| **Cost** | Free (you pay for log storage) | $0.10 per 100,000 events |
| **Scope** | Single bucket | All S3 buckets in the account |
| **Integration** | Query with Athena | CloudWatch, EventBridge, Lambda |
| **Compliance** | Less reliable for audit | Preferred for compliance/audit |

Use access logging for cost analysis and debugging. Use CloudTrail for security auditing and compliance.

**Q27: How do you prevent S3 data exfiltration in an enterprise environment?**  
**A:**  
1. **VPC Endpoint policies** — restrict which buckets are accessible from each VPC.  
2. **SCPs** — deny `s3:PutBucketPolicy` with `"Principal": "*"`.  
3. **Block Public Access** — enforce at the organization level.  
4. **S3 Access Analyzer** — alerts on public or cross-account access.  
5. **Macie** — scans for sensitive data (PII, credentials) in buckets.  
6. **Condition keys** in policies: `aws:SourceVpc`, `aws:SourceVpce`, `aws:PrincipalOrgID`.  
7. **GuardDuty S3 Protection** — detects anomalous data access patterns.

**Q28: Design a disaster recovery strategy for critical S3 data with RPO < 15 minutes and RTO < 1 hour.**  
**A:**  
1. **Versioning** enabled (protect against accidental deletes).  
2. **CRR with Replication Time Control (RTC)** — guarantees 99.99% of objects replicated within 15 minutes.  
3. **S3 Replication Metrics** — CloudWatch alarms on `ReplicationLatency` and `OperationsFailedReplication`.  
4. **Object Lock** in Governance mode on the DR bucket (prevent malicious deletion).  
5. **Route 53 health checks** — failover DNS to serve from the DR region.  
6. **Regular DR drills** — test restore from the DR bucket quarterly.  
RPO < 15 min ✓ (RTC), RTO < 1 hour ✓ (DNS failover + data already replicated).

---

## Scenario-Based Questions

**Q29: Your CI/CD pipeline pushes 500 build artifacts/day to S3. After 6 months, storage costs are $5,000/month. How do you optimize?**  
**A:** Lifecycle policy: expire artifacts > 30 days (or keep only last 5 versions). Move to Standard-IA after 7 days. Abort incomplete multipart uploads after 1 day. Tag artifacts with `BuildId` for traceability. Expected savings: 80%+.

**Q30: Two applications need to write to the same S3 bucket but must not read each other's data. How?**  
**A:** Create two S3 Access Points (or use prefix-based IAM policies). Each app's IAM role allows `PutObject` and `GetObject` only to its own prefix (e.g., `app-a/*`, `app-b/*`). Bucket policy delegates access to the access points.

**Q31: How would you migrate 100 TB from on-premises to S3?**  
**A:**  
- **Online:** AWS DataSync (managed, supports incremental), or `aws s3 sync` with multipart and Transfer Acceleration.  
- **Offline:** AWS Snowball Edge (50 TB each, order 2). Or AWS Snowmobile for PB-scale.  
- **Hybrid:** Use DataSync for initial bulk + ongoing sync, Snowball for the initial seed if bandwidth is limited.

**Q32: You have a Lambda function triggered by S3 events. It's being invoked twice for the same object. Why?**  
**A:** S3 event notifications guarantee **at-least-once delivery** (not exactly-once). Your Lambda must be idempotent. Use DynamoDB conditional writes (check if the object was already processed) or SQS FIFO with deduplication as a buffer between S3 and Lambda.

---

# Resume Points

## For a Senior DevOps Engineer (S3-Focused)

Use these bullet points in your resume, adapted to your actual experience:

### Infrastructure & Cloud (S3-specific)

- **Architected and managed enterprise S3 storage** spanning 50+ buckets across 10 AWS accounts with centralized governance using S3 Storage Lens, AWS Config rules, and Organization-level Block Public Access.

- **Reduced S3 storage costs by 65%** ($120K annual savings) by implementing tiered lifecycle policies — transitioning 200 TB of logs/artifacts through Standard → Standard-IA → Glacier → Deep Archive with automated expiration.

- **Designed cross-region disaster recovery** for mission-critical data using S3 CRR with Replication Time Control (RTC), achieving RPO < 15 minutes and validated through quarterly DR drills.

- **Implemented S3 security hardening** across the organization including SSE-KMS encryption with customer-managed CMKs, bucket policies enforcing TLS-only access, VPC endpoint policies for data exfiltration prevention, and MFA Delete on compliance-critical buckets.

- **Built event-driven file processing pipelines** using S3 Event Notifications → SQS → Lambda for automated image resizing, document processing, and ETL ingestion serving 500K+ daily uploads.

- **Optimized data transfer performance** for globally distributed teams by implementing S3 Transfer Acceleration and multipart upload, reducing upload times by 60% for cross-continent content creators.

- **Automated S3 infrastructure provisioning** using CloudFormation/Terraform with versioning, encryption, lifecycle policies, replication, and access logging configured as code, ensuring consistent security posture across all environments.

- **Designed multi-tenant data lake architecture** using S3 Access Points with per-team access policies and prefix-based isolation, serving 5 internal teams with differentiated access levels to shared datasets.

- **Implemented S3 compliance controls** meeting HIPAA/SOC2 requirements — Object Lock (Compliance mode) for immutable audit logs, CloudTrail data events for access auditing, and Macie for PII scanning.

- **Reduced NAT Gateway data transfer costs by $8,000/month** by implementing S3 VPC Gateway Endpoints for all private subnet workloads, routing 500 GB/day of S3 traffic through the free endpoint.

- **Set up comprehensive S3 monitoring** using CloudWatch request metrics, custom dashboards, and proactive alerting on 5xx errors, throttling, and replication lag, reducing incident detection time from hours to minutes.

- **Managed S3 static website hosting** with CloudFront CDN, ACM TLS certificates, and Route 53 DNS — serving 10M+ monthly page views with sub-100ms latency globally.

### Tips for Using These

1. **Quantify everything** — replace placeholder numbers with your actual metrics
2. **Use the STAR format** in interviews — Situation, Task, Action, Result
3. **Tailor** to the job description — emphasize security for security-focused roles, cost optimization for FinOps-focused roles
4. **Be ready to deep-dive** — interviewers will ask follow-up questions on every bullet point

---

## Quick Reference Cheat Sheet

| Feature | Key Command/Setting | When to Use |
|---------|-------------------|-------------|
| Versioning | Properties → Bucket Versioning | Always for important data |
| Encryption | Properties → Default Encryption → SSE-KMS | Compliance requirements |
| Lifecycle | Management → Lifecycle Rules | Cost optimization |
| Replication | Management → Replication Rules | DR, compliance |
| Access Logging | Properties → Server Access Logging | Security audit |
| Events | Properties → Event Notifications | Event-driven architecture |
| Block Public Access | Permissions → Block Public Access | Always ON (default) |
| Presigned URL | Object Actions → Share | Temporary secure sharing |
| Transfer Acceleration | Properties → Transfer Acceleration | Global uploads |
| Object Lock | Properties → Object Lock | WORM compliance |
| Storage Lens | S3 → Storage Lens | Organization-wide visibility |
| CORS | Permissions → CORS | Browser-based access |
| Inventory | Management → Inventory | Large bucket audits |
| VPC Endpoint | VPC → Endpoints | Private network access |
| Tags | Properties → Tags | Cost allocation |
| Intelligent-Tiering | Properties → IT Archive Config | Unpredictable access patterns |

---

> **Final tip for your job switch:** S3 questions appear in almost every AWS/DevOps interview. Focus on: encryption enforcement, lifecycle cost optimization, replication for DR, event-driven architectures, and security hardening (Block Public Access + VPC Endpoints + bucket policies). These are the topics where interviewers test depth versus surface knowledge.

**Good luck with your job switch, Vinay! 🚀**
