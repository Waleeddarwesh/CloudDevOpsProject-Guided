# 🏗️ Module 02 — Terraform

**Time:** ~8 hours · **Cost:** 💰 **~$0.30/hour from `apply` onward**

---

## ⚠️ Before you start

- [ ] Budget alarm set ([Module 00 § 3](00-prerequisites.md))
- [ ] `terraform version` reports **≥ 1.10**
- [ ] You know your public IP
- [ ] You plan to `terraform destroy` at the end of the session

**Destroy when you stop for the day.** Rebuilding is 20 minutes and reinforces the material. Leaving it up is ~$7/day.

---

## 🎯 What you'll build

```text
02-Terraform/
├── bootstrap/          ← run ONCE: creates the S3 state bucket
├── versions.tf         ← version constraints
├── providers.tf        ← AWS provider + default_tags
├── backend.tf          ← remote state
├── variables.tf        ← inputs + validation
├── locals.tf           ← computed names
├── main.tf             ← wires the 4 modules
├── outputs.tf
└── modules/
    ├── network/        VPC · subnets · IGW · NAT · route tables · NACLs · flow logs
    ├── server/         SG · IAM role · instance profile · EC2 · EIP
    ├── eks/            cluster · node group · OIDC · IRSA ×3 · add-ons · access
    └── ecr/            3 repositories · lifecycle policy
```

~80 AWS resources.

---

## 📋 Contents

- [Part 1 — Remote state (the chicken-and-egg)](#part-1)
- [Part 2 — Root scaffolding](#part-2)
- [Part 3 — Network module](#part-3)
- [Part 4 — ECR module](#part-4)
- [Part 5 — Server module](#part-5)
- [Part 6 — EKS module](#part-6)
- [Part 7 — Wire it up and apply](#part-7)

---

<a id="part-1"></a>

# Part 1 · Remote state

## 🔨 Break it first: why local state fails

```bash
mkdir -p /tmp/tfdemo && cd /tmp/tfdemo
cat > main.tf <<'EOF'
resource "random_pet" "demo" {
  length = 2
}
EOF
terraform init && terraform apply -auto-approve
cat terraform.tfstate | head -20
```

You'll see a JSON file mapping your code to a real resource ID. Now:

```bash
rm terraform.tfstate
terraform apply -auto-approve
```

Terraform creates a **second** resource. It has no memory of the first — that resource is now orphaned, invisible, and (if it were an EKS cluster) still billing you.

**State is the only record linking your code to real infrastructure.** Three reasons it must not live on your laptop:

| Problem | Consequence |
|---|---|
| Lose the file | Terraform re-creates everything; the originals are orphaned |
| Two people, two copies | Each overwrites the other's resources |
| Contains **plaintext secrets** | DB passwords, private keys, generated tokens |

```bash
cd /tmp/tfdemo && terraform destroy -auto-approve && cd - && rm -rf /tmp/tfdemo
```

## The bootstrap module

The backend bucket must exist *before* Terraform can use it. Solve it with a separate root module that uses local state on purpose.

Create `workspace/02-Terraform/bootstrap/main.tf`:

```hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.80" }
  }
}

provider "aws" {
  region = var.aws_region
}

data "aws_caller_identity" "current" {}

locals {
  # S3 bucket names are globally unique across ALL AWS accounts, so
  # "ivolve-terraform-state" is certainly taken. Embedding the account ID
  # makes collision impossible while keeping the name predictable.
  bucket_name = "${var.project_name}-tfstate-${data.aws_caller_identity.current.account_id}-${var.aws_region}"
}

resource "aws_s3_bucket" "state" {
  bucket = local.bucket_name

  # Deliberately NOT force_destroy. This bucket holds the only record of
  # what infrastructure exists.
  force_destroy = false
}

# The single most important setting on this bucket.
# Terraform OVERWRITES the state object on every apply. Without versioning,
# a corrupted state is unrecoverable.
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# State files contain every value Terraform touched, in PLAINTEXT.
# Treat this bucket as a secrets store, because that is what it is.
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
    bucket_key_enabled = true
  }
}

# Four independent switches, all on. Publicly readable state buckets are a
# recurring source of real-world cloud breaches.
resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

`bootstrap/variables.tf`:

```hcl
variable "project_name" {
  type    = string
  default = "ivolve"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}
```

`bootstrap/outputs.tf`:

```hcl
output "state_bucket_name" {
  value = aws_s3_bucket.state.id
}
```

Run it — **once, ever**:

```bash
cd workspace/02-Terraform/bootstrap
terraform init
terraform apply
```

**Copy the bucket name from the output.**

---

<a id="part-2"></a>

# Part 2 · Root scaffolding

## `versions.tf`

```hcl
terraform {
  # >= 1.10 required for S3 native state locking (use_lockfile) below.
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
      # ~> 5.80 means ">= 5.80.0, < 6.0.0" — patches and minors allowed,
      # a breaking 6.x release cannot be pulled in silently.
      version = "~> 5.80"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

## `backend.tf`

```hcl
terraform {
  backend "s3" {
    # bucket/key/region come from backend.hcl via:
    #   terraform init -backend-config=backend.hcl
    #
    # Backend blocks CANNOT use variables — a hard Terraform limitation,
    # because the backend initialises before variables are evaluated.
    # Partial configuration is the supported workaround.

    encrypt = true

    # State locking via a native S3 conditional-write lock file.
    # Replaces the old `dynamodb_table` argument (deprecated in 1.11) —
    # one less resource to provision, pay for, and forget to clean up.
    use_lockfile = true
  }
}
```

`backend.hcl` (gitignored — bucket names are account-specific):

```hcl
bucket = "ivolve-tfstate-111122223333-us-east-1"   # ← YOUR bucket
key    = "capstone/dev/terraform.tfstate"
region = "us-east-1"
```

## `providers.tf`

```hcl
provider "aws" {
  region = var.aws_region

  # Applies these tags to EVERY taggable resource this provider creates —
  # across all four modules — without repeating a tags block each time.
  #
  # This matters beyond tidiness:
  #   Cost Explorer can only break spend down by tags that exist.
  #   `aws resourcegroupstaggingapi get-resources --tag-filters
  #      Key=Project,Values=ivolve` finds every orphan at teardown.
  #   ManagedBy=Terraform tells an on-call engineer not to fix it by hand.
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = var.owner
    }
  }
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
```

## `variables.tf` — with the guard that protects you

```hcl
variable "project_name" {
  type    = string
  default = "ivolve"

  validation {
    # These names end up in DNS names, IAM role names and S3 keys,
    # all of which reject uppercase and underscores.
    condition     = can(regex("^[a-z][a-z0-9-]{1,20}$", var.project_name))
    error_message = "project_name must be 2-21 lowercase alphanumeric/hyphen characters."
  }
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "owner" {
  type    = string
  default = "your-name"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least two AZs are required — EKS refuses to create a single-AZ cluster."
  }
}

variable "public_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "key_name" {
  type    = string
  default = "ivolve-key"
}

# ------------------------------------------------------------------------------
# THE GUARD
# ------------------------------------------------------------------------------
variable "allowed_ssh_cidrs" {
  description = "CIDRs allowed to reach port 22. Set to YOUR_IP/32."
  type        = list(string)
  default     = []

  validation {
    condition     = !contains(var.allowed_ssh_cidrs, "0.0.0.0/0")
    error_message = "Refusing to open SSH to the entire internet. Use [\"YOUR_IP/32\"]."
  }
}

variable "allowed_jenkins_ui_cidrs" {
  type    = list(string)
  default = []

  validation {
    condition     = !contains(var.allowed_jenkins_ui_cidrs, "0.0.0.0/0")
    error_message = "Refusing to expose the Jenkins UI to the entire internet."
  }
}

variable "kubernetes_version" {
  type    = string
  default = "1.31"
}

variable "ecr_repository_names" {
  type    = list(string)
  default = ["ivolve-frontend", "ivolve-auth-service", "ivolve-roadmap-service"]
}
```

> 💡 **Test the guard.** Set `allowed_ssh_cidrs = ["0.0.0.0/0"]` and run `terraform plan`. It fails before touching AWS. Validation blocks catch mistakes at plan time instead of at 3am.

## `locals.tf`

```hcl
locals {
  # Every resource is named "<project>-<env>-<thing>": ivolve-dev-vpc, etc.
  name_prefix  = "${var.project_name}-${var.environment}"
  cluster_name = "${local.name_prefix}-eks"

  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name

  ecr_registry = "${local.account_id}.dkr.ecr.${local.region}.amazonaws.com"

  # default_tags cannot reference computed values, so cluster-discovery
  # tags live here instead.
  common_tags = { Cluster = local.cluster_name }
}
```

---

<a id="part-3"></a>

# Part 3 · Network module

## The mental model

```text
Internet
   │
   ▼
  IGW ─────────────────────────────────────────┐
   │                                            │
 PUBLIC 10.0.1.0/24 (AZ-a)  10.0.2.0/24 (AZ-b) │
   · ALB          · NAT Gateway     · Jenkins   │
        │                                       │
        │ NAT — outbound only                   │
        ▼                                       │
 PRIVATE 10.0.10.0/24        10.0.11.0/24       │
   · EKS worker 1            · EKS worker 2     │
   · no public IP — unreachable from internet   │
──────────────────────────────────────────────── ┘
```

**"Public" vs "private" is decided entirely by the route table.** A public subnet has a `0.0.0.0/0` route to the IGW; a private one routes to a NAT Gateway. There is no `is_public` flag.

Create `modules/network/variables.tf`:

```hcl
variable "name_prefix"          { type = string }
variable "vpc_cidr"             { type = string }
variable "availability_zones"   { type = list(string) }
variable "public_subnet_cidrs"  { type = list(string) }
variable "private_subnet_cidrs" { type = list(string) }
variable "cluster_name"         { type = string }
variable "single_nat_gateway"   { type = bool, default = true }
variable "tags"                 { type = map(string), default = {} }
```

`modules/network/main.tf`:

```hcl
resource "aws_vpc" "this" {
  cidr_block = var.vpc_cidr

  # BOTH are mandatory for EKS. Without DNS support, in-cluster service
  # discovery and the CSI drivers cannot resolve AWS API endpoints.
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, { Name = "${var.name_prefix}-vpc" })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name_prefix}-igw" })
}

# ------------------------------------------------------------------------------
# Public subnets
# ------------------------------------------------------------------------------
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-public-${var.availability_zones[count.index]}"

    # ⚠️ EKS / Load Balancer Controller DISCOVERY TAGS.
    # The controller lists subnets and filters on these to decide where to
    # place a load balancer. Omit them and Ingress creation fails with
    # "couldn't auto-discover subnets" — a very common EKS bug.
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  # Explicitly false — worker nodes must not be directly addressable.
  map_public_ip_on_launch = false

  tags = merge(var.tags, {
    Name                                        = "${var.name_prefix}-private-${var.availability_zones[count.index]}"
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  })
}

# ------------------------------------------------------------------------------
# NAT Gateway
# ------------------------------------------------------------------------------
resource "aws_eip" "nat" {
  count  = var.single_nat_gateway ? 1 : length(var.public_subnet_cidrs)
  domain = "vpc"

  tags = merge(var.tags, { Name = "${var.name_prefix}-nat-eip-${count.index + 1}" })

  # The EIP cannot attach to a NAT Gateway until the IGW is attached,
  # otherwise apply fails with InvalidGateway.NotAttached.
  depends_on = [aws_internet_gateway.this]
}

resource "aws_nat_gateway" "this" {
  count = var.single_nat_gateway ? 1 : length(var.public_subnet_cidrs)

  allocation_id = aws_eip.nat[count.index].id

  # ⚠️ A NAT Gateway lives in a PUBLIC subnet — that is what gives it a
  # route to the IGW. Putting it in a private subnet is a common mistake
  # that produces a gateway with no internet path.
  subnet_id = aws_subnet.public[count.index].id

  tags       = merge(var.tags, { Name = "${var.name_prefix}-nat-${count.index + 1}" })
  depends_on = [aws_internet_gateway.this]
}

# ------------------------------------------------------------------------------
# Route tables — this is what makes a subnet public or private
# ------------------------------------------------------------------------------
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name_prefix}-public-rt" })
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id     # ← IGW = public
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# One table PER private subnet, even sharing one NAT. Costs nothing, and
# switching single_nat_gateway to false later changes only the route target.
resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name_prefix}-private-rt-${count.index}" })
}

resource "aws_route" "private_nat" {
  count                  = length(var.private_subnet_cidrs)
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = var.single_nat_gateway ? aws_nat_gateway.this[0].id : aws_nat_gateway.this[count.index].id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

## Network ACLs — and why they're not redundant with Security Groups

| | Security Group | Network ACL |
|---|---|---|
| Attaches to | an ENI (instance) | a **subnet** |
| State | **stateful** — replies auto-allowed | **stateless** — replies need their own rule |
| Rules | allow only | allow **and deny** |

Because NACLs are stateless, a client in a subnet opening a connection gets its reply on an **ephemeral port (1024–65535)** — so you need an ingress rule for those even though nothing "listens" there.

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.this.id
  subnet_ids = aws_subnet.public[*].id
  tags       = merge(var.tags, { Name = "${var.name_prefix}-public-nacl" })
}

# Rules are evaluated in ascending rule_number; first match wins.
resource "aws_network_acl_rule" "public_in_http" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

resource "aws_network_acl_rule" "public_in_https" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# ⚠️ Return traffic for connections STARTED from this subnet.
# Without this, the NAT Gateway's outbound flows get no replies and every
# package download in the private subnets hangs.
resource "aws_network_acl_rule" "public_in_ephemeral" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 120
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

resource "aws_network_acl_rule" "public_in_admin" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 130
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 22
  to_port        = 9000
}

resource "aws_network_acl_rule" "public_out_all" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  egress         = true
  protocol       = "-1"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 0
  to_port        = 0
}
```

> 💡 The NACL is deliberately broad at the subnet level; the **Security Group** does the precise per-instance filtering (your IP only). Defence in depth means each layer does what it's good at.

`modules/network/outputs.tf`:

```hcl
output "vpc_id"             { value = aws_vpc.this.id }
output "vpc_cidr"           { value = aws_vpc.this.cidr_block }
output "public_subnet_ids"  { value = aws_subnet.public[*].id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
```

---

<a id="part-4"></a>

# Part 4 · ECR module

`modules/ecr/main.tf`:

```hcl
resource "aws_ecr_repository" "this" {
  # ⚠️ for_each over a SET, not count over a list.
  #
  # With `count`, removing the first repository shifts every index — and
  # Terraform plans to DESTROY AND RECREATE the rest, deleting live images.
  # for_each keys state by NAME, so unrelated entries are untouched.
  for_each = toset(var.repository_names)

  name = each.value

  # IMMUTABLE is the most valuable setting here.
  #
  # With MUTABLE tags, a rebuild silently replaces the image behind a tag
  # already running in the cluster — so `:42` no longer means what it meant
  # when it was scanned and approved. Immutability makes every tag a
  # permanent, auditable reference, which is exactly what GitOps records.
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    # A second opinion alongside the Trivy gate: ECR re-evaluates STORED
    # images as new CVEs are published, so an image clean at build time
    # gets re-flagged when a new advisory lands.
    scan_on_push = true
  }

  encryption_configuration { encryption_type = "AES256" }

  force_delete = var.force_delete

  tags = merge(var.tags, { Name = each.value })
}

# Without this, storage grows forever — a pipeline pushing on every commit
# accumulates hundreds of ~200 MB images.
resource "aws_ecr_lifecycle_policy" "this" {
  for_each   = aws_ecr_repository.this
  repository = each.value.name

  policy = jsonencode({
    rules = [
      {
        # Untagged images are previous versions of a moving tag — they
        # consume storage and can never be pulled by tag again.
        # This rule MUST come first: each image is actioned by the FIRST
        # rule it matches.
        rulePriority = 1
        description  = "Expire untagged after 1 day"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Keep the 10 most recent builds"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = { type = "expire" }
      },
    ]
  })
}
```

`modules/ecr/variables.tf` and `outputs.tf`:

```hcl
variable "repository_names" { type = list(string) }
variable "force_delete"     { type = bool, default = true }
variable "tags"             { type = map(string), default = {} }
```

```hcl
output "repository_urls" { value = [for r in aws_ecr_repository.this : r.repository_url] }
output "repository_arns" { value = [for r in aws_ecr_repository.this : r.arn] }
```

---

<a id="part-5"></a>

# Part 5 · Server module

## The security decisions

Three choices worth understanding before you type:

**1. IAM instance profile, not access keys.** The instance gets temporary, auto-rotating credentials from the metadata service. No `AKIA...` key is ever written to disk — that leak is the most common way a compromised CI box becomes a compromised AWS account.

**2. IMDSv2 enforced.** IMDSv1 answers any HTTP GET from inside the instance, so an SSRF bug in *any* app on the host can read the role's credentials. IMDSv2 requires a PUT to get a token first, which SSRF cannot do.

**3. Elastic IP.** A stopped/started instance gets a new public IP, breaking your Ansible inventory and every bookmark.

`modules/server/main.tf` — the key parts:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical's official account

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_security_group" "jenkins" {
  name        = "${var.name_prefix}-jenkins-sg"
  description = "Jenkins CI server"
  vpc_id      = var.vpc_id

  lifecycle { create_before_destroy = true }

  tags = merge(var.tags, { Name = "${var.name_prefix}-jenkins-sg" })
}

# Rules as SEPARATE resources, not inline `ingress {}` blocks.
#
# Inline blocks are authoritative: Terraform deletes any rule it doesn't
# know about — so a rule added in the console during an incident vanishes
# on the next apply with no warning.
resource "aws_vpc_security_group_ingress_rule" "ssh" {
  count = length(var.allowed_ssh_cidrs)

  security_group_id = aws_security_group.jenkins.id
  cidr_ipv4         = var.allowed_ssh_cidrs[count.index]
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
}

resource "aws_vpc_security_group_ingress_rule" "jenkins_ui" {
  count = length(var.allowed_jenkins_ui_cidrs)

  security_group_id = aws_security_group.jenkins.id
  cidr_ipv4         = var.allowed_jenkins_ui_cidrs[count.index]
  ip_protocol       = "tcp"
  from_port         = 8080
  to_port           = 8080
}

resource "aws_vpc_security_group_egress_rule" "all" {
  security_group_id = aws_security_group.jenkins.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

The IAM role, with a detail worth noticing:

```hcl
data "aws_iam_policy_document" "ecr" {
  # ecr:GetAuthorizationToken is ACCOUNT-scoped and cannot be restricted to
  # a repository — AWS only accepts "*". Splitting it into its own statement
  # keeps the wildcard confined to this one harmless read action.
  statement {
    sid       = "EcrAuthToken"
    actions   = ["ecr:GetAuthorizationToken"]
    resources = ["*"]
  }

  # The actions that actually MOVE image data ARE scoped.
  statement {
    sid = "EcrPushPull"
    actions = [
      "ecr:BatchCheckLayerAvailability", "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage", "ecr:InitiateLayerUpload", "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload", "ecr:PutImage",
    ]
    resources = var.ecr_repository_arns
  }
}
```

The instance:

```hcl
resource "aws_instance" "jenkins" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.jenkins.id]
  iam_instance_profile   = aws_iam_instance_profile.jenkins.name

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"        # ← IMDSv2 only
    # Hop limit 1 stops a container on this host from reaching the metadata
    # endpoint — a container's packet has already taken one hop.
    http_put_response_hop_limit = 1
  }

  root_block_device {
    volume_size           = var.root_volume_size
    volume_type           = "gp3"     # 3000 IOPS baseline regardless of size
    encrypted             = true      # workspaces hold source + build secrets
    delete_on_termination = true
  }

  lifecycle {
    # Canonical publishes new AMIs constantly. Without this, every plan
    # shows a pending REPLACEMENT of the instance — destroying Jenkins and
    # all its build history.
    ignore_changes = [ami]
  }

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-jenkins"
    # ⚠️ Ansible's dynamic inventory filters on tag:Role = jenkins.
    # Change this string and Module 03 finds nothing.
    Role = "jenkins"
  })
}

resource "aws_eip" "jenkins" {
  instance = aws_instance.jenkins.id
  domain   = "vpc"
}
```

---

<a id="part-6"></a>

# Part 6 · EKS module

This is the hardest module. Four things must all be right or the cluster comes up broken in confusing ways.

## 6.1 · The cluster

```hcl
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    # BOTH tiers. Workers live in private; the public subnets are where the
    # Load Balancer Controller places the internet-facing ALB.
    subnet_ids = concat(var.private_subnet_ids, var.public_subnet_ids)

    endpoint_private_access = true
    endpoint_public_access  = true
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  access_config {
    authentication_mode = "API_AND_CONFIG_MAP"

    # ⚠️ WITHOUT THIS the cluster is created and immediately unreachable —
    # every kubectl call returns "You must be logged in to the server".
    # A classic EKS footgun.
    bootstrap_cluster_creator_admin_permissions = true
  }

  depends_on = [aws_iam_role_policy_attachment.cluster_eks_policy]
}
```

## 6.2 · Node group

```hcl
resource "aws_eks_node_group" "this" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "${var.cluster_name}-nodes"
  node_role_arn   = aws_iam_role.node.arn

  # Passing BOTH private subnets is what satisfies "2 workers in different
  # subnets and AZs" — the ASG balances instances across every subnet given.
  subnet_ids = var.private_subnet_ids

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 4
  }

  instance_types = ["t3.medium"]
  ami_type       = "AL2023_x86_64_STANDARD"

  lifecycle {
    # The autoscaler changes desired_size at runtime. Without this, the next
    # apply scales the cluster back down and evicts running pods.
    ignore_changes = [scaling_config[0].desired_size]
  }

  # Nodes register the instant they boot. If IAM isn't attached yet,
  # registration fails and creation times out after ~20 min.
  depends_on = [
    aws_iam_role_policy_attachment.node_worker,
    aws_iam_role_policy_attachment.node_cni,
    aws_iam_role_policy_attachment.node_ecr,
  ]
}
```

**The three node policies — omit any one and you get a confusing failure:**

| Policy | Without it |
|---|---|
| `AmazonEKSWorkerNodePolicy` | nodes never reach `Ready` |
| `AmazonEKS_CNI_Policy` | pods stick in `ContainerCreating`: "failed to assign an IP address" |
| `AmazonEC2ContainerRegistryReadOnly` | `ImagePullBackOff` / "no basic auth credentials" |

## 6.3 · IRSA — the part everyone skips

**The problem:** a pod needing AWS permissions must either ship static keys in a Secret, or borrow the *node's* role — in which case **every pod on that node** inherits those permissions.

**IRSA:** EKS runs an OIDC provider that issues a signed JWT naming the pod's namespace and ServiceAccount. IAM trusts that provider, but only for a specific `sub`. Result: per-ServiceAccount permissions, no static keys, automatic rotation.

```hcl
# IAM pins the provider by the SHA-1 thumbprint of its CA. Reading it
# dynamically keeps working when AWS rotates the cert — a hardcoded
# thumbprint eventually breaks every IRSA pod at once.
data "tls_certificate" "oidc" {
  url = aws_eks_cluster.this.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "this" {
  url             = aws_eks_cluster.this.identity[0].oidc[0].issuer
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.oidc.certificates[0].sha1_fingerprint]
}

locals {
  oidc_provider_url = replace(aws_eks_cluster.this.identity[0].oidc[0].issuer, "https://", "")
}

# ------------------------------------------------------------------------------
# IRSA role — EBS CSI Driver.  REQUIRED for this project to work at all.
# ------------------------------------------------------------------------------
# Module 04's MySQL StatefulSet declares a PVC. Something must call
# CreateVolume/AttachVolume against EC2. Without this role the PVC stays
# Pending FOREVER and the MySQL pod never starts.
data "aws_iam_policy_document" "ebs_csi_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.this.arn]
    }

    # ⚠️ Restricts the role to ONE ServiceAccount. Without this condition,
    # ANY pod in the cluster could assume it and detach other workloads' volumes.
    condition {
      test     = "StringEquals"
      variable = "${local.oidc_provider_url}:sub"
      values   = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_provider_url}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ebs_csi" {
  name               = "${var.cluster_name}-ebs-csi-irsa"
  assume_role_policy = data.aws_iam_policy_document.ebs_csi_assume_role.json
}

resource "aws_iam_role_policy_attachment" "ebs_csi" {
  role       = aws_iam_role.ebs_csi.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}
```

Repeat the same pattern for the **AWS Load Balancer Controller** (ServiceAccount `kube-system:aws-load-balancer-controller`). Its policy has no AWS-managed equivalent — download it:

```bash
mkdir -p workspace/02-Terraform/modules/eks/policies
curl -sSL -o workspace/02-Terraform/modules/eks/policies/aws-load-balancer-controller.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

## 6.4 · Add-ons — a bare EKS cluster is not a working cluster

| Add-on | Without it |
|---|---|
| `vpc-cni` | no pod networking at all |
| `kube-proxy` | ClusterIP DNS resolves but connections hang |
| `coredns` | `mysql:3306` and every service name fails to resolve |
| **`aws-ebs-csi-driver`** | **PVCs stay `Pending` forever** |

> ⚠️ The EBS CSI driver stopped being bundled with EKS in 1.23. That's why a StatefulSet that worked on an older cluster silently hangs on a new one — and it's the most commonly missed piece of EKS setup.

```hcl
data "aws_eks_addon_version" "this" {
  for_each = toset(["vpc-cni", "kube-proxy", "coredns", "aws-ebs-csi-driver"])

  addon_name         = each.value
  kubernetes_version = aws_eks_cluster.this.version
  most_recent        = false     # use AWS's default — the tested choice
}

resource "aws_eks_addon" "ebs_csi" {
  cluster_name  = aws_eks_cluster.this.name
  addon_name    = "aws-ebs-csi-driver"
  addon_version = data.aws_eks_addon_version.this["aws-ebs-csi-driver"].version

  # THIS is the link that makes IRSA work: EKS annotates the driver's
  # ServiceAccount with this role ARN, and the trust policy above only
  # accepts tokens from that exact namespace/ServiceAccount pair.
  service_account_role_arn = aws_iam_role.ebs_csi.arn

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [aws_eks_node_group.this, aws_iam_role_policy_attachment.ebs_csi]
}
```

Add `vpc-cni`, `kube-proxy` and `coredns` the same way (no `service_account_role_arn`). CoreDNS is a Deployment, so it needs `depends_on = [aws_eks_node_group.this]` — installing it before nodes exist leaves it `DEGRADED`.

---

<a id="part-7"></a>

# Part 7 · Wire it up

## ⚠️ The dependency cycle you will hit

Naively:

```hcl
module "server" { eks_cluster_arn = module.eks.cluster_arn }     # server → eks
module "eks"    { jenkins_role_arn = module.server.iam_role_arn } # eks → server
```

```text
Error: Cycle: module.server, module.eks
```

**Fix:** pass the cluster *name* (a local, known before either module runs) and reconstruct the ARN inside the server module:

```hcl
resources = ["arn:aws:eks:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:cluster/${var.eks_cluster_name}"]
```

## A second, subtler trap

```hcl
count = var.jenkins_role_arn != null ? 1 : 0
```

```text
Error: Invalid count argument
The "count" value depends on resource attributes that cannot be determined until apply.
```

`count` must resolve at **plan** time, and `module.server.iam_role_arn` is `(known after apply)` on a fresh deployment. Use a plain bool instead:

```hcl
variable "create_jenkins_access_entry" { type = bool, default = true }
count = var.create_jenkins_access_entry ? 1 : 0
```

The unknown ARN is fine as an *attribute value* — only `count` is constrained.

## `main.tf`

```hcl
module "network" {
  source = "./modules/network"

  name_prefix          = local.name_prefix
  vpc_cidr             = var.vpc_cidr
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  cluster_name         = local.cluster_name      # ← local, not module.eks
  tags                 = local.common_tags
}

module "ecr" {
  source           = "./modules/ecr"
  repository_names = var.ecr_repository_names
  tags             = local.common_tags
}

module "server" {
  source = "./modules/server"

  name_prefix = local.name_prefix
  vpc_id      = module.network.vpc_id
  subnet_id   = module.network.public_subnet_ids[0]
  key_name    = var.key_name

  allowed_ssh_cidrs        = var.allowed_ssh_cidrs
  allowed_jenkins_ui_cidrs = var.allowed_jenkins_ui_cidrs

  ecr_repository_arns = module.ecr.repository_arns
  eks_cluster_name    = local.cluster_name       # ← name, breaks the cycle
  tags                = local.common_tags
}

module "eks" {
  source = "./modules/eks"

  cluster_name       = local.cluster_name
  kubernetes_version = var.kubernetes_version
  private_subnet_ids = module.network.private_subnet_ids
  public_subnet_ids  = module.network.public_subnet_ids
  jenkins_role_arn   = module.server.iam_role_arn
  tags               = local.common_tags
}
```

> 💡 **Notice there is no `depends_on` anywhere.** Passing `module.network.vpc_id` into another module *is* the dependency declaration — and it's more precise than a blanket `depends_on`. Terraform builds the graph from these references and parallelises everything independent.

## Apply

```bash
cd workspace/02-Terraform

cp terraform.tfvars.example terraform.tfvars   # or create it
# set: allowed_ssh_cidrs = ["YOUR_IP/32"]
#      allowed_jenkins_ui_cidrs = ["YOUR_IP/32"]

terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan      # ← READ IT. ~80 resources.
terraform apply
```

**15–20 minutes.** The EKS control plane alone is ~10.

---

# ✅ Checkpoint

```bash
aws eks update-kubeconfig --region us-east-1 --name $(terraform output -raw eks_cluster_name)

kubectl get nodes -o custom-columns=\
NAME:.metadata.name,STATUS:.status.conditions[-1].type,ZONE:.metadata.labels.'topology\.kubernetes\.io/zone'
```

```text
NAME                         STATUS   ZONE
ip-10-0-10-42.ec2.internal   Ready    us-east-1a
ip-10-0-11-88.ec2.internal   Ready    us-east-1b
```

**Two nodes, `Ready`, different AZs.** That single line proves the VPC, subnets, route tables, IAM, node group and CNI are all correct.

Verify the add-ons — especially the CSI driver:

```bash
kubectl get pods -n kube-system | grep -E "coredns|aws-node|kube-proxy|ebs-csi"
```

All `Running`. If `ebs-csi-controller` is missing, Module 04's database will never start.

---

## 🧹 End of session

```bash
terraform destroy
```

Yes, really. You'll rebuild in 20 minutes next time, and you'll understand it better the second time.

---

## 🧠 Before you continue

1. Why can't `backend.tf` use variables?
2. What makes a subnet "public"?
3. Why must the NAT Gateway be in a public subnet?
4. Why `for_each` instead of `count` for ECR repositories?
5. What breaks without the `kubernetes.io/role/elb` subnet tag?
6. What does IRSA solve that a node role doesn't?
7. What happens to a PVC without the EBS CSI driver?
8. Why did passing `module.eks.cluster_arn` into the server module fail?

---

<div align="center">

**Next → [Module 03 — Ansible](03-ansible.md)**

</div>
