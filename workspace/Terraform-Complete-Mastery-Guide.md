# Terraform — Complete Mastery Guide (Beginner to Advanced)

> A single, no-gaps roadmap to learn Terraform from zero to expert, with detailed steps and hands-on labs at every stage.
> Every concept includes a **Why** note so you understand the reasoning, not just the syntax.

---

## How to Use This Guide

- Work top to bottom. Each section builds on the last.
- Type every lab yourself in a real terminal — do not copy-paste blindly. Muscle memory is the goal.
- Use a **sandbox cloud account** (AWS free tier works well) and always run the **Cleanup** step so you don't get billed.
- Keep the mental model front and center: **Terraform turns declarative config into real infrastructure and tracks it in state.**

### What Terraform Is (and Why It Exists)

Terraform is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp. You declare the *desired end state* of your infrastructure in configuration files, and Terraform figures out the API calls needed to reach that state.

- **Declarative, not imperative:** You describe *what* you want, not the step-by-step *how*.
  - **Why it matters:** You never write "create this, then that." Terraform computes the diff between reality and your config and applies only what's needed.
- **Provider-agnostic:** One workflow for AWS, Azure, GCP, Kubernetes, GitHub, Datadog, and 3000+ providers.
  - **Why it matters:** One tool and one mental model across your entire stack.
- **State-driven:** Terraform records what it created in a **state file** to map config to real resources.
  - **Why it matters:** State is how Terraform knows what already exists and what to change. Understanding state is the single biggest thing that separates beginners from pros.

---

## The Learning Roadmap

| Stage | Topics | Outcome |
|-------|--------|---------|
| **1. Foundations** | IaC concepts, install, first `apply` | Provision one real resource |
| **2. Core Language** | HCL, resources, providers, state | Read/write any basic config |
| **3. Inputs & Outputs** | Variables, outputs, locals, data sources | Parameterize configs |
| **4. Meta-arguments** | count, for_each, depends_on, lifecycle | Control resource behavior |
| **5. Expressions & Functions** | Loops, conditionals, built-in functions, dynamic blocks | Write dynamic configs |
| **6. Modules** | Authoring, calling, versioning, registry | Reusable, composable infra |
| **7. State Management** | Remote backends, locking, workspaces, import | Team-safe, scalable state |
| **8. Provisioners & Lifecycle** | Provisioners, null_resource, time | Handle edge cases |
| **9. Testing & Quality** | fmt, validate, tflint, terraform test, Terratest | Trustworthy code |
| **10. Collaboration & CI/CD** | Git workflow, pipelines, Terraform Cloud | Team delivery |
| **11. Security & Governance** | Secrets, policy-as-code (Sentinel/OPA), drift | Safe at scale |
| **12. Advanced & Ecosystem** | Custom providers, CDKTF, monorepos, patterns | Expert-level |

**Capstone:** Build a multi-environment (dev/staging/prod) VPC + app stack using reusable modules, remote state with locking, a CI/CD pipeline that plans on PR and applies on merge, and policy checks. That single project proves mastery.

**Certification target:** HashiCorp Certified: Terraform Associate (003).

---

# Stage 1 — Foundations

### 1.1 Install Terraform

**Windows (PowerShell, via Chocolatey):**
```powershell
choco install terraform
```
**macOS (Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```
**Verify:**
```bash
terraform -version
```
- **Why:** Confirms the binary is on your PATH. Terraform is a single static binary — no runtime to install.

> **Tip:** Use **tfenv** (or `tfswitch`) to manage multiple Terraform versions per project.
> **Why:** Different projects pin different versions; a version manager prevents "works on my machine" drift.

### 1.2 Set up your editor

1. Install the **HashiCorp Terraform** extension for VS Code.
   - **Why:** Syntax highlighting, autocomplete, formatting, and inline docs dramatically reduce errors.
2. Enable format-on-save.
   - **Why:** Keeps code canonical (`terraform fmt`) automatically.

### 1.3 Your first configuration

Create a folder `tf-lab-01` and a file `main.tf`:
```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello, Terraform!"
}
```
- **Why `terraform` block:** It pins the Terraform version and declares which providers (and versions) the config needs, so builds are reproducible.
- **Why `local` provider:** It creates files locally — zero cloud cost, perfect for learning the workflow first.

### 1.4 The core workflow (memorize this)

```bash
terraform init      # download providers, set up the working dir
terraform plan      # preview what will change
terraform apply     # make the changes (creates hello.txt)
terraform destroy   # tear everything down
```
- **Why `init`:** Downloads provider plugins and initializes the backend. Run it once per new config or when providers/backends change.
- **Why `plan`:** Shows an execution plan — a dry run — *before* touching anything. Always read the plan.
- **Why `apply`:** Executes the plan. It asks for confirmation unless you pass `-auto-approve`.
- **Why `destroy`:** Cleanly removes everything Terraform manages — your cost-control lifeline.

**Lab 1:** Run the full cycle. Open `hello.txt`, change the `content`, run `plan` (see the diff), `apply` (see it update in place), then `destroy`.

---

# Stage 2 — Core Language (HCL) + First Cloud Resource

### 2.1 HCL building blocks

- **Blocks:** `resource`, `provider`, `variable`, `output`, `module`, `data`, `locals`, `terraform`.
- **Arguments:** `key = value` pairs inside blocks.
- **Expressions:** values, references, functions.
  - **Why:** HCL (HashiCorp Configuration Language) is declarative and JSON-compatible but far more readable. Knowing the block types is like knowing the parts of speech.

### 2.2 Configure a real provider (AWS)

`providers.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```
- **Why the version constraint `~> 5.0`:** Allows 5.x updates but blocks a breaking 6.0. Pinning prevents surprise breakage.
- **Why not hardcode credentials:** Terraform reads them from the AWS CLI config / environment variables / IAM roles. Never put secrets in `.tf` files.

### 2.3 Create your first cloud resource

```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "tf-demo-<your-unique-suffix>"
}
```
- **Why S3:** Cheap, simple, and instantly shows the create/read/update/delete cycle on a real cloud API.

**Lab 2:** `init` → `plan` → `apply`. Confirm the bucket exists in the AWS console. Then change a tag and re-apply to see an in-place update.

### 2.4 Understand state (critical concept)

After apply, look at `terraform.tfstate`.
- It maps each resource in your config to a real cloud resource ID.
- **Why it exists:** Terraform compares (desired config) vs (state = last known reality) vs (actual cloud) to compute changes.
- **Never edit it by hand.** Use `terraform state` subcommands instead.

```bash
terraform state list          # list managed resources
terraform state show <addr>   # inspect one resource
```
- **Why these commands:** They let you inspect and surgically manipulate state safely.

---

# Stage 3 — Inputs, Outputs, Locals, Data Sources

### 3.1 Input variables

`variables.tf`:
```hcl
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "tags" {
  type = map(string)
  default = {
    Project = "learning"
    Owner   = "me"
  }
}
```
- **Why variables:** They parameterize configs so the same code runs across environments/regions without edits.
- **Why `type` and `description`:** Type constraints catch errors early; descriptions document intent and appear in tooling.

**Ways to set variables (precedence, lowest to highest):**
1. `default` in the declaration
2. `terraform.tfvars` / `*.auto.tfvars` files
3. `-var-file=` flag
4. `-var="name=value"` flag
5. `TF_VAR_name` environment variables
- **Why know precedence:** In real projects values come from many places; knowing who wins prevents confusion.

### 3.2 Validation and sensitive variables

```hcl
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env must be dev, staging, or prod."
  }
}

variable "db_password" {
  type      = string
  sensitive = true
}
```
- **Why `validation`:** Fails fast with a clear message instead of a confusing cloud error later.
- **Why `sensitive`:** Keeps the value out of CLI output and logs.

### 3.3 Locals

```hcl
locals {
  name_prefix = "${var.env}-app"
  common_tags = merge(var.tags, { Environment = var.env })
}
```
- **Why locals:** DRY — compute a value once and reuse it. Unlike variables, they can't be overridden from outside, so they're for internal computed values.

### 3.4 Outputs

```hcl
output "bucket_name" {
  description = "The created bucket name"
  value       = aws_s3_bucket.demo.bucket
}
```
- **Why outputs:** Expose useful values after apply (IDs, endpoints), feed them to other tools, and pass them between modules.

### 3.5 Data sources (read existing infra)

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```
- **Why data sources:** They *read* existing resources you don't manage (e.g., look up the latest AMI ID) so your config stays dynamic instead of hardcoding IDs.

**Lab 3:** Parameterize your S3 bucket name with a variable, tag it with `local.common_tags`, output the bucket ARN, and use a data source to fetch the latest Amazon Linux AMI ID.

---

# Stage 4 — Meta-Arguments

Meta-arguments change how Terraform manages a resource regardless of provider.

### 4.1 `count`

```hcl
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}
```
- **Why `count`:** Create N identical resources. Reference them as `aws_instance.web[0]`.
- **Gotcha:** Removing an item from the middle of a count list re-indexes and can destroy/recreate resources.

### 4.2 `for_each` (usually preferred)

```hcl
resource "aws_instance" "web" {
  for_each      = toset(["blue", "green"])
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  tags = { Name = "web-${each.key}" }
}
```
- **Why `for_each` over `count`:** Keys are stable, so adding/removing one item doesn't disturb the others. Use it whenever instances have identity.

### 4.3 `depends_on`

```hcl
resource "aws_iam_role_policy" "p" { /* ... */ }

resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.p]
}
```
- **Why:** Terraform infers most dependencies automatically from references. Use `depends_on` only for *hidden* dependencies it can't see.

### 4.4 `lifecycle`

```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags["LastModified"]]
  }
}
```
- **Why `create_before_destroy`:** Zero-downtime replacement — new resource comes up before the old goes away.
- **Why `prevent_destroy`:** Guards critical resources (databases) from accidental deletion.
- **Why `ignore_changes`:** Stop Terraform from fighting external changes to specific attributes.

### 4.5 `provider` meta-argument (multi-region/account)

```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_s3_bucket" "west_bucket" {
  provider = aws.west
  bucket   = "tf-demo-west-..."
}
```
- **Why aliases:** Deploy to multiple regions or accounts from one config.

**Lab 4:** Convert your instances from `count` to `for_each`, add a `lifecycle` block with `create_before_destroy`, and deploy an S3 bucket to a second region using a provider alias.

---

# Stage 5 — Expressions, Functions, Dynamic Blocks

### 5.1 Conditionals and operators

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
```
- **Why:** Ternaries let one config adapt per environment.

### 5.2 `for` expressions (loops in values)

```hcl
locals {
  upper_names = [for n in var.names : upper(n)]
  name_map    = { for n in var.names : n => length(n) }
}
```
- **Why:** Transform lists/maps inline without extra resources.

### 5.3 Built-in functions (know these families)

- **String:** `format`, `join`, `split`, `replace`, `trimspace`, `lower/upper`.
- **Collection:** `length`, `merge`, `concat`, `contains`, `keys`, `values`, `lookup`, `flatten`, `distinct`.
- **Numeric:** `min`, `max`, `abs`, `ceil`, `floor`.
- **Encoding:** `jsonencode`, `jsondecode`, `base64encode`, `yamlencode`.
- **Filesystem:** `file`, `templatefile`, `fileset`, `pathexpand`.
- **IP:** `cidrsubnet`, `cidrhost`.
- **Date/crypto:** `timestamp`, `formatdate`, `uuid`, `sha256`.
- **Why learn them:** Functions eliminate hardcoding. Try each in `terraform console` (an interactive REPL) — the fastest way to learn them.

```bash
terraform console
> cidrsubnet("10.0.0.0/16", 8, 2)
"10.0.2.0/24"
```
- **Why `terraform console`:** Instant experimentation without applying anything.

### 5.4 `templatefile` for rendered configs

```hcl
user_data = templatefile("${path.module}/init.sh.tpl", {
  app_port = var.app_port
})
```
- **Why:** Inject Terraform values into scripts/config files cleanly.

### 5.5 Dynamic blocks

```hcl
resource "aws_security_group" "web" {
  name = "web-sg"
  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```
- **Why dynamic blocks:** Generate repeated nested blocks (like SG rules) from a variable instead of copy-pasting.

**Lab 5:** Use `cidrsubnet` to compute subnet CIDRs from a base VPC CIDR, and use a `dynamic` block to create security group rules from a list of ports.

---

# Stage 6 — Modules (Reusability)

Modules are the unit of reuse and the key to scalable Terraform.

### 6.1 Anatomy

Every Terraform folder is technically a module. A reusable module has:
- `variables.tf` (inputs), `main.tf` (resources), `outputs.tf` (outputs).
- **Why:** Clean input/output contract lets others use it without reading internals.

### 6.2 Create a local module

`modules/s3-bucket/main.tf`:
```hcl
variable "bucket_name" { type = string }
variable "versioning"  { type = bool, default = true }

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.versioning ? "Enabled" : "Suspended"
  }
}

output "arn" { value = aws_s3_bucket.this.arn }
```

### 6.3 Call the module

```hcl
module "logs_bucket" {
  source      = "./modules/s3-bucket"
  bucket_name = "my-logs-bucket-123"
  versioning  = true
}

output "logs_arn" {
  value = module.logs_bucket.arn
}
```
- **Why modules:** Encapsulate best practices once, reuse everywhere, and change behavior in one place.

### 6.4 Module sources & versioning

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"
  # ...
}
```
- **Why the Terraform Registry:** Battle-tested community modules (like the official AWS VPC module) save enormous effort.
- **Why pin `version`:** Reproducible builds; an upstream change can't silently break you.
- **Other sources:** Git (`git::https://...`), local paths, S3, GCS.

### 6.5 Module composition best practices

- Keep modules **small and focused** (one concern each).
- Don't hardcode provider config inside modules — pass it in.
- Expose only the inputs users need; provide sensible defaults.
  - **Why:** Composable, testable, and reusable modules beat one giant module.

**Lab 6:** Refactor your S3 + instances into modules. Then use the official `terraform-aws-modules/vpc/aws` module to create a real VPC and pass its outputs into your own module.

---

# Stage 7 — State Management (Team-Scale)

This is where hobby Terraform becomes production Terraform.

### 7.1 Why local state doesn't scale

- `terraform.tfstate` on your laptop can't be shared, isn't locked, and can be lost.
  - **Why remote state:** Teams need a shared, locked, durable state store.

### 7.2 Remote backend (S3 + DynamoDB lock)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-locks"
    encrypt        = true
  }
}
```
- **Why S3 backend:** Durable, versioned, encrypted central state.
- **Why DynamoDB table:** Provides **state locking** so two people can't `apply` simultaneously and corrupt state.
- **Why `encrypt`:** State can contain sensitive values; encrypt at rest.

> Note: In newer Terraform, S3 backend can use native lockfile (`use_lockfile`) — but DynamoDB locking is the widely-used pattern to know.

### 7.3 State commands you must know

```bash
terraform state list
terraform state show <addr>
terraform state mv <src> <dst>     # rename/move without destroy
terraform state rm <addr>          # stop managing (doesn't delete cloud resource)
terraform import <addr> <id>       # bring existing infra under management
terraform refresh                  # reconcile state with real world (careful)
```
- **Why `state mv`:** Refactor code (rename resources / move into modules) without recreating resources.
- **Why `state rm`:** Hand a resource off to another config or stop managing it.
- **Why `import`:** Adopt manually-created resources into Terraform.

### 7.4 `import` blocks (declarative import, TF 1.5+)

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "existing-bucket-name"
}
```
- **Why:** Plan-reviewable, repeatable imports (better than the imperative command for many resources). Can even generate config with `-generate-config-out`.

### 7.5 Workspaces

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select dev
```
- **Why workspaces:** Multiple state files from one config (e.g., `terraform.workspace` to vary names). 
- **Caution:** Good for small env separation; for serious multi-env, prefer **separate state files + directories** or a tool like Terragrunt. Know both approaches and their trade-offs.

**Lab 7:** Migrate your local state to an S3 backend with DynamoDB locking. Then use `terraform state mv` to move a resource into a module without destroying it, and `import` a manually-created bucket.

---

# Stage 8 — Provisioners, null_resource, and Timing

Use these sparingly — they're escape hatches, not the primary tool.

### 8.1 Provisioners

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "remote-exec" {
    inline = ["sudo yum install -y nginx", "sudo systemctl start nginx"]
    connection {
      type        = "ssh"
      host        = self.public_ip
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
    }
  }
}
```
- **Why provisioners are a last resort:** They break the declarative model and aren't tracked in state. Prefer `user_data`, cloud-init, or config-management tools (Ansible) instead. Know them, but reach for them rarely.

### 8.2 `null_resource` + `triggers`

```hcl
resource "null_resource" "run_when_changed" {
  triggers = { version = var.app_version }
  provisioner "local-exec" {
    command = "echo deploying ${var.app_version}"
  }
}
```
- **Why:** Run an action when an arbitrary value changes. (Modern alternative: `terraform_data` resource, which needs no provider.)

### 8.3 `time` provider

- `time_sleep`, `time_rotating` for delays and rotation.
  - **Why:** Some cloud resources need a settle time before dependents are created.

**Lab 8:** Use `terraform_data` with a `triggers_replace` to run a `local-exec` only when a variable changes.

---

# Stage 9 — Testing & Code Quality

### 9.1 Built-in checks

```bash
terraform fmt -recursive     # canonical formatting
terraform validate           # syntax + internal consistency
terraform plan               # the ultimate dry-run check
```
- **Why:** `fmt` keeps diffs clean; `validate` catches errors before hitting the cloud.

### 9.2 Linting and security scanning

- **TFLint** — catches provider-specific mistakes and enforces conventions.
- **tfsec / Trivy / Checkov** — scan for insecure configurations (open security groups, unencrypted volumes).
  - **Why:** Automated guardrails catch mistakes humans miss and enforce security baselines.

### 9.3 Native testing (`terraform test`, TF 1.6+)

`tests/bucket.tftest.hcl`:
```hcl
run "creates_bucket" {
  command = plan
  assert {
    condition     = aws_s3_bucket.this.bucket == "expected-name"
    error_message = "Bucket name did not match."
  }
}
```
- **Why native tests:** Validate modules with plan/apply assertions, no third-party framework needed.

### 9.4 Terratest (Go)

- Write Go tests that `apply`, make real assertions (HTTP calls, API checks), then `destroy`.
  - **Why:** Real end-to-end integration testing for critical modules.

**Lab 9:** Add `fmt`, `validate`, TFLint, and a Checkov scan to your project. Write one `terraform test` file asserting a module output.

---

# Stage 10 — Collaboration & CI/CD

### 10.1 Git hygiene

`.gitignore`:
```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars        # if they contain secrets
.terraform.lock.hcl   # DO commit this one
```
- **Why ignore state/`.terraform`:** They're large, machine-specific, or sensitive.
- **Why commit `.terraform.lock.hcl`:** It pins exact provider versions/hashes for reproducible, secure builds across the team.

### 10.2 The team workflow

1. Branch → change code → open PR.
2. CI runs `fmt -check`, `validate`, `plan` → posts the plan on the PR.
3. Reviewer reads the **plan** (the real diff) and approves.
4. Merge → CI runs `apply`.
- **Why plan-on-PR:** The plan is the review artifact. You review *infrastructure changes* the way you review code.

### 10.3 Example GitHub Actions pipeline

```yaml
name: terraform
on: [pull_request]
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -no-color
```
- **Why CI/CD:** Removes "works on my laptop," enforces checks, and creates an auditable history of every change.

### 10.4 Terraform Cloud / HCP Terraform

- Remote runs, remote state, private module registry, VCS-driven workflows, and policy enforcement.
  - **Why:** Managed collaboration — no need to build your own state backend and runner infrastructure.

**Lab 10:** Put your project in a Git repo, add the GitHub Actions plan workflow, and open a PR to see the plan posted automatically.

---

# Stage 11 — Security & Governance

### 11.1 Handling secrets

- **Never** hardcode secrets in `.tf` or commit `.tfvars` with secrets.
- Pull from **AWS Secrets Manager / SSM Parameter Store / Vault** via data sources.
- Mark outputs/variables `sensitive = true`.
- Remember: **secrets can still land in state** — so encrypt and restrict access to state.
  - **Why:** State is the most under-appreciated secret-leak vector. Lock it down.

### 11.2 Policy as Code

- **Sentinel** (HashiCorp, in HCP Terraform) or **OPA/Conftest** (open source).
- Example rule: "deny any security group open to `0.0.0.0/0` on port 22."
  - **Why:** Enforce org-wide guardrails automatically in the pipeline, before apply.

### 11.3 Drift detection

- Run `terraform plan` on a schedule; a non-empty plan means someone changed infra outside Terraform (**drift**).
  - **Why:** Detecting drift keeps reality and code in sync and prevents nasty surprises.

### 11.4 Least-privilege for Terraform itself

- Give the Terraform runner an IAM role scoped to only what it manages.
  - **Why:** Limits blast radius if the pipeline credentials leak.

**Lab 11:** Add a Checkov/OPA policy that fails the plan if an S3 bucket lacks encryption, and set up a scheduled drift-detection plan.

---

# Stage 12 — Advanced Topics & Ecosystem

### 12.1 Project structure at scale

- **Split state by blast radius:** separate stacks for network, data, app — each with its own state.
  - **Why:** A change to the app shouldn't require touching (or risking) the network state. Smaller state = faster, safer plans.
- **Terragrunt:** DRY wrapper for backends, provider config, and multi-env orchestration.
  - **Why:** Reduces repetition across many environments/accounts.

### 12.2 Advanced language features

- `moved` blocks — refactor resource addresses without destroy/recreate.
- `check` blocks — post-apply assertions and health checks.
- `precondition` / `postcondition` in resources — validate assumptions.
- `optional()` object type attributes with defaults.
  - **Why:** These make large codebases refactorable and self-validating.

### 12.3 CDK for Terraform (CDKTF)

- Write infrastructure in TypeScript/Python/Go, synthesized to Terraform.
  - **Why:** Use real programming languages and loops/abstractions when HCL feels limiting.

### 12.4 Writing a custom provider

- Providers are Go programs using the Terraform Plugin Framework.
  - **Why:** When you need to manage an internal API or a service without an existing provider.

### 12.5 Performance & operations

- `-target` (surgical apply, use sparingly), `-parallelism`, `-refresh=false`.
- Provider plugin cache to speed up `init`.
  - **Why:** Large states and many resources need tuning to stay fast.

**Capstone Lab:** Build the full multi-environment project described below.

---

## Capstone Project — Production-Grade Multi-Environment Stack

**Goal:** Prove you've mastered everything.

**Requirements:**
1. **Reusable modules** for network (VPC), compute, and data.
2. **Three environments** (dev/staging/prod) with separate state files.
3. **Remote backend** (S3 + DynamoDB locking, encrypted).
4. **Registry module** usage (official AWS VPC module) alongside your own.
5. **Variables + validation** driving all environment differences.
6. **CI/CD**: plan on PR (posted to the PR), apply on merge to main.
7. **Policy checks** (Checkov/OPA) that block insecure configs.
8. **Testing**: `terraform test` on modules + `fmt`/`validate`/`tflint` in CI.
9. **Secrets** pulled from SSM/Secrets Manager, never committed.
10. **Drift detection** on a schedule.
11. **Documentation** generated with `terraform-docs`.

- **Why this capstone:** It mirrors exactly how mature teams run Terraform — modular, tested, policy-gated, multi-env, and automated. Finishing it means you've left nothing out.

---

## Command Cheat Sheet

```bash
terraform init [-upgrade]        # init dir, download providers/modules
terraform fmt -recursive         # format
terraform validate               # validate config
terraform plan [-out=tfplan]     # preview / save a plan
terraform apply [tfplan]         # apply (a saved plan is safest)
terraform destroy                # tear down
terraform show                   # human-readable state/plan
terraform output [name]          # read outputs
terraform state list|show|mv|rm  # manage state
terraform import <addr> <id>     # adopt existing resource
terraform taint/untaint          # (legacy) force recreate → prefer -replace
terraform apply -replace=<addr>  # force recreate a resource
terraform workspace list|new|select
terraform console                # interactive expression REPL
terraform graph                  # dependency graph (DOT)
terraform providers              # provider requirements tree
terraform force-unlock <id>      # release a stuck state lock (careful)
```

---

## Common Pitfalls (and Why They Bite)

- **Editing state by hand** → corruption. Use `state` subcommands.
- **Not pinning versions** → surprise breakage. Pin Terraform + providers + modules.
- **Storing state locally on a team** → conflicts and loss. Use remote state + locking.
- **Overusing provisioners** → brittle, untracked changes. Prefer cloud-native init.
- **Giant monolithic state** → slow, risky plans. Split by blast radius.
- **Committing secrets / state** → leaks. `.gitignore` them; encrypt state.
- **Ignoring the plan** → unintended destroys. Always read the plan before apply.
- **`count` where identity matters** → re-index churn. Prefer `for_each`.

---

## Certification: HashiCorp Terraform Associate (003)

Focus areas: IaC concepts, Terraform workflow, state, modules, variables/outputs, provisioners, HCP/Terraform Cloud, debugging, and the CLI. Study this guide, do every lab, and read the official docs' "Associate tutorials." The exam is practical — hands-on practice is the best prep.

---

## Best Free Resources

- **Official docs:** developer.hashicorp.com/terraform (docs + guided tutorials)
- **Terraform Registry:** registry.terraform.io (providers + modules)
- **`terraform-docs`, TFLint, Checkov, tfsec** — install and use them early.

---

*You become a Terraform pro by shipping the capstone, refactoring it, breaking state on purpose, and recovering — not by reading. Build, plan, apply, destroy, repeat.*
