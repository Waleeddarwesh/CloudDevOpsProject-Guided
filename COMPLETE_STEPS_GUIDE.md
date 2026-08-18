# 📘 CloudDevOpsProject — Complete Step-by-Step Guide

**All steps from cloning the source repository to production deployment.**

> This document consolidates every actionable step across all 8 project modules into a single, ordered checklist. Follow it top-to-bottom.

---

## Table of Contents

1. [GitHub Repository Setup](#1-github-repository-setup)
2. [Containerization with Docker](#2-containerization-with-docker)
3. [Infrastructure Provisioning with Terraform](#3-infrastructure-provisioning-with-terraform)
4. [Configuration Management with Ansible](#4-configuration-management-with-ansible)
5. [Container Orchestration with Kubernetes](#5-container-orchestration-with-kubernetes)
6. [Continuous Integration with Jenkins](#6-continuous-integration-with-jenkins)
7. [Continuous Deployment with ArgoCD](#7-continuous-deployment-with-argocd)
8. [Documentation](#8-documentation)

---

## Prerequisites — Module 00 (~45 min, Free)

### Step 0.1 — Install Required Tools

| Tool | Minimum Version | Purpose |
|---|---|---|
| AWS CLI | v2 | ECR login, `eks update-kubeconfig` |
| Terraform | ≥ 1.10 | S3 native state locking (`use_lockfile`) |
| Docker Desktop | recent | Building images, local stack |
| kubectl | ≥ 1.29 | Talking to the cluster (bundles `kustomize`) |
| Helm | 3.x | LB Controller and monitoring stack |
| Ansible | ≥ 2.15 | Configuring the Jenkins server |
| Git | any | Version control |

**Windows install:**
```powershell
wsl --install                       # Ansible runs only in WSL
winget install Amazon.AWSCLI
winget install HashiCorp.Terraform
winget install Docker.DockerDesktop
winget install Kubernetes.kubectl
winget install Helm.Helm
winget install Git.Git
```

**macOS install:**
```bash
brew install awscli terraform kubectl helm ansible git
brew install --cask docker
```

**Ubuntu / WSL install:**
```bash
sudo apt-get update && sudo apt-get install -y ansible git curl unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
```

**Verify all tools:**
```bash
for t in aws terraform docker kubectl helm ansible git; do
  printf "%-12s " "$t"
  command -v $t >/dev/null && $t --version 2>&1 | head -1 || echo "MISSING"
done
```

### Step 0.2 — Configure AWS Credentials

```bash
aws configure
aws sts get-caller-identity          # Note down the Account ID
```

> ⚠️ Never use the root account. Create an IAM user with `AdministratorAccess`.

### Step 0.3 — Set a Budget Alarm ($25)

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

aws budgets create-budget --account-id "$ACCOUNT" --budget '{
  "BudgetName": "ivolve-learning",
  "BudgetLimit": {"Amount": "25", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}'

aws budgets create-notification --account-id "$ACCOUNT" \
  --budget-name ivolve-learning \
  --notification '{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"}' \
  --subscribers '[{"SubscriptionType":"EMAIL","Address":"YOUR_EMAIL@example.com"}]'
```

### Step 0.4 — Create EC2 Key Pair

```bash
aws ec2 create-key-pair --key-name ivolve-key \
  --query KeyMaterial --output text > ~/.ssh/ivolve-key.pem
chmod 400 ~/.ssh/ivolve-key.pem
```

**Windows PowerShell alternative:**
```powershell
aws ec2 create-key-pair --key-name ivolve-key `
  --query KeyMaterial --output text | Out-File -Encoding ascii $HOME\.ssh\ivolve-key.pem
```

### Step 0.5 — Find Your Public IP

```bash
curl -s https://checkip.amazonaws.com
```

> Write this down. You'll use it as `YOUR_IP/32` to restrict SSH and Jenkins UI access.

### Step 0.6 — Create GitHub Personal Access Token

1. GitHub → **Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. **Generate new token**
3. Repository access → **Only select repositories** → your project repo
4. Permissions → Repository permissions → **Contents: Read and write**
5. Copy the token (shown once)

### Step 0.7 — Clone the Application Source

```bash
cd workspace
git clone --depth 1 https://github.com/Ibrahim-Adel15/iVolveFinalProject.git /tmp/app-src
cp -r /tmp/app-src/frontend /tmp/app-src/auth-service /tmp/app-src/roadmap-service src/
rm -rf /tmp/app-src
```

**Verify:**
```bash
find src -maxdepth 2 -type f | sort
```

### Step 0.8 — Read the Application Source

Understand the three microservices before writing any configuration:

| Service | Language | Port | Talks to |
|---|---|:---:|---|
| `frontend` | Node.js 22 · Express · EJS | 3000 | auth + roadmap |
| `auth-service` | Python 3.12 · Flask | 5000 | MySQL |
| `roadmap-service` | Java 21 · Spring Boot | 8080 | nothing |

**Key environment variables:**
```bash
grep -n "os.environ" src/auth-service/app.py
grep -n "process.env" src/frontend/server.js
```

> ⚠️ The env var is `DB_USER`, NOT `DB_USERNAME`. The roadmap-service has no `/health` endpoint — use `/api/roadmap` for healthchecks.

---

## 1. GitHub Repository Setup

### Step 1.1 — Create the GitHub Repository

1. Go to [github.com/new](https://github.com/new)
2. Repository name: `CloudDevOpsProject`
3. Initialize with a README ✅
4. Create repository

### Step 1.2 — Clone and Set Up Workspace Structure

```bash
git clone https://github.com/YourUser/CloudDevOpsProject.git
cd CloudDevOpsProject

mkdir -p src/{frontend,auth-service,roadmap-service}
mkdir -p 01-Docker
mkdir -p 02-Terraform/{bootstrap,modules/{network,server,eks,ecr}}
mkdir -p 03-Ansible/{inventory,group_vars/all,roles/{common,java,docker,jenkins,trivy,aws_cli,kubectl,helm,sonarqube}/{tasks,defaults,handlers,templates,meta}}
mkdir -p 04-Kubernetes/manifests
mkdir -p 05-Jenkins/{vars,Jenkinsfiles}
mkdir -p 06-ArgoCD/applications
```

### Step 1.3 — Copy the Source Code

```bash
git clone --depth 1 https://github.com/Ibrahim-Adel15/iVolveFinalProject.git /tmp/app-src
cp -r /tmp/app-src/frontend/* src/frontend/
cp -r /tmp/app-src/auth-service/* src/auth-service/
cp -r /tmp/app-src/roadmap-service/* src/roadmap-service/
rm -rf /tmp/app-src
```

### Step 1.4 — Create `.gitignore`

```bash
cat > .gitignore <<'EOF'
.env
*.pem
*.key
.vault_pass
*.retry
ansible.log
.ansible_*
terraform.tfstate
terraform.tfstate.backup
.terraform/
backend.hcl
terraform.tfvars
EOF
```

### Step 1.5 — Initial Commit

```bash
git add .
git commit -m "chore: initial project structure with source code"
git push origin main
```

**✅ Deliverable:** URL to the GitHub repository

---

## 2. Containerization with Docker (~4 hours, Free)

### Step 2.1 — Create Frontend Dockerfile

Create `src/frontend/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev --no-audit --no-fund

FROM node:22-alpine AS runtime
LABEL org.opencontainers.image.title="ivolve-frontend" \
      org.opencontainers.image.description="iVolve DevOps Roadmap — web frontend"

RUN apk add --no-cache tini

ENV NODE_ENV=production \
    PORT=3000

WORKDIR /app
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node server.js ./
COPY --chown=node:node views ./views
COPY --chown=node:node public ./public

USER node
EXPOSE 3000

HEALTHCHECK --interval=15s --timeout=5s --start-period=20s --retries=3 \
    CMD node -e "require('http').get('http://127.0.0.1:3000/',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

### Step 2.2 — Create Auth-Service Dockerfile

Add gunicorn to requirements:
```bash
echo "gunicorn==23.0.0" >> src/auth-service/requirements.txt
```

Create `src/auth-service/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS builder
WORKDIR /build
RUN apt-get update \
    && apt-get install --no-install-recommends -y build-essential libffi-dev \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
RUN apt-get update \
    && apt-get install --no-install-recommends -y curl tini \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --create-home --shell /usr/sbin/nologin --uid 10001 appuser
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY --chown=appuser:appuser app.py .

USER appuser
EXPOSE 5000

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=5 \
    CMD curl --fail --silent http://127.0.0.1:5000/health || exit 1

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "4", "--timeout", "60", "--access-logfile", "-", "app:app"]
```

### Step 2.3 — Create Roadmap-Service Dockerfile

Create `src/roadmap-service/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM maven:3.9.11-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime
RUN apk add --no-cache curl tini \
    && addgroup --system --gid 10001 appgroup \
    && adduser  --system --uid 10001 --ingroup appgroup --shell /sbin/nologin appuser
WORKDIR /app
COPY --from=build --chown=appuser:appgroup /build/target/roadmap-service-1.0.0.jar app.jar

USER appuser
EXPOSE 8080

ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseSerialGC -XX:+ExitOnOutOfMemoryError"

HEALTHCHECK --interval=15s --timeout=5s --start-period=45s --retries=5 \
    CMD curl --fail --silent http://127.0.0.1:8080/api/roadmap || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
```

### Step 2.4 — Create Three `.dockerignore` Files

**`src/frontend/.dockerignore`:**
```
node_modules/
npm-debug.log*
.env
.env.*
*.pem
*.key
.git/
.gitignore
.dockerignore
Dockerfile
*.md
.vscode/
.idea/
.DS_Store
```

**`src/auth-service/.dockerignore`:**
```
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/
.env
.env.*
*.pem
*.key
.git/
.dockerignore
Dockerfile
*.md
```

**`src/roadmap-service/.dockerignore`:**
```
target/
.mvn/
mvnw
mvnw.cmd
.env
*.pem
*.key
.git/
.dockerignore
Dockerfile
*.md
*.iml
```

### Step 2.5 — Create `docker-compose.yml`

Create `01-Docker/docker-compose.yml`:

```yaml
name: ivolve

services:

  mysql:
    image: mysql:8.0
    container_name: ivolve-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD in .env}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-ivolve}
      MYSQL_USER: ${MYSQL_USER:-ivolve_user}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD in .env}
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root -p\"$$MYSQL_ROOT_PASSWORD\" --silent"]
      interval: 10s
      timeout: 5s
      start_period: 40s
      retries: 10

  auth-service:
    build:
      context: ../src/auth-service
      target: runtime
    image: ivolve-auth-service:local
    container_name: ivolve-auth-service
    restart: unless-stopped
    environment:
      DB_HOST: mysql
      DB_PORT: "3306"
      DB_NAME: ${MYSQL_DATABASE:-ivolve}
      DB_USER: ${MYSQL_USER:-ivolve_user}
      DB_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD in .env}
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      mysql:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:5000/health"]
      interval: 15s
      timeout: 5s
      start_period: 30s
      retries: 5

  roadmap-service:
    build:
      context: ../src/roadmap-service
      target: runtime
    image: ivolve-roadmap-service:local
    container_name: ivolve-roadmap-service
    restart: unless-stopped
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:8080/api/roadmap"]
      interval: 15s
      timeout: 5s
      start_period: 45s
      retries: 5

  frontend:
    build:
      context: ../src/frontend
      target: runtime
    image: ivolve-frontend:local
    container_name: ivolve-frontend
    restart: unless-stopped
    environment:
      AUTH_SERVICE_URL: http://auth-service:5000
      ROADMAP_SERVICE_URL: http://roadmap-service:8080
      SESSION_SECRET: ${SESSION_SECRET:?set SESSION_SECRET in .env}
    networks: [ivolve-net]
    ports:
      - "3000:3000"
    depends_on:
      auth-service:
        condition: service_healthy
      roadmap-service:
        condition: service_healthy

networks:
  ivolve-net:
    name: ivolve-net
    driver: bridge

volumes:
  mysql-data:
    name: ivolve-mysql-data
```

### Step 2.6 — Create `.env.example` and `.env`

Create `01-Docker/.env.example`:
```bash
MYSQL_ROOT_PASSWORD=CHANGE_ME_root_password
MYSQL_DATABASE=ivolve
MYSQL_USER=ivolve_user
MYSQL_PASSWORD=CHANGE_ME_app_password
SESSION_SECRET=CHANGE_ME_long_random_secret
```

Create your `.env`:
```bash
cd 01-Docker
cp .env.example .env
# Generate passwords: openssl rand -base64 24
# Edit .env with your generated values
```

### Step 2.7 — Build and Test Locally

```bash
cd 01-Docker
docker compose config --quiet && echo "compose valid"
docker compose up -d --build
docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

### Step 2.8 — Verify the Application

1. Open http://localhost:3000
2. Sign up (password ≥ 8 characters)
3. Log in
4. Verify the DevOps Roadmap page shows 8 topics

```bash
curl -s http://127.0.0.1:5000/health | jq
curl -s http://127.0.0.1:8080/api/roadmap | jq '.[0]'
```

### Step 2.9 — Commit Docker Files

```bash
docker compose down -v
git add .
git commit -m "feat: docker-compose and Dockerfiles for all microservices"
git push origin main
```

**✅ Deliverable:** `docker-compose.yml` committed to the repository

---

## 3. Infrastructure Provisioning with Terraform (~8 hours, 💰 ~$0.30/hr)

### Step 3.1 — Create Bootstrap Module (S3 State Bucket)

Create `02-Terraform/bootstrap/main.tf`:

```hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.80" }
  }
}

provider "aws" { region = var.aws_region }

data "aws_caller_identity" "current" {}

locals {
  bucket_name = "${var.project_name}-tfstate-${data.aws_caller_identity.current.account_id}-${var.aws_region}"
}

resource "aws_s3_bucket" "state" {
  bucket        = local.bucket_name
  force_destroy = false
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Create `02-Terraform/bootstrap/variables.tf` and `outputs.tf`.

**Run bootstrap (once, ever):**
```bash
cd 02-Terraform/bootstrap
terraform init
terraform apply
# Copy the bucket name from output
```

### Step 3.2 — Create Root Scaffolding Files

Create the following files in `02-Terraform/`:

- **`versions.tf`** — Terraform ≥ 1.10, AWS provider ~> 5.80, TLS provider ~> 4.0
- **`backend.tf`** — S3 backend with `use_lockfile = true` (native S3 locking)
- **`backend.hcl`** — Partial backend config with bucket name, key, region (gitignored)
- **`providers.tf`** — AWS provider with `default_tags` (Project, Environment, ManagedBy, Owner)
- **`variables.tf`** — All inputs with validations:
  - `project_name`, `environment`, `owner`, `aws_region`
  - `vpc_cidr` (default `10.0.0.0/16`)
  - `availability_zones` (min 2 AZs required)
  - `public_subnet_cidrs`, `private_subnet_cidrs`
  - `key_name`, `kubernetes_version`, `ecr_repository_names`
  - **`allowed_ssh_cidrs`** — with validation that REFUSES `0.0.0.0/0`
  - **`allowed_jenkins_ui_cidrs`** — with validation that REFUSES `0.0.0.0/0`
- **`locals.tf`** — `name_prefix`, `cluster_name`, `account_id`, `ecr_registry`
- **`terraform.tfvars`** — Your specific values (gitignored)

### Step 3.3 — Create Network Module

Create `02-Terraform/modules/network/` with `variables.tf`, `main.tf`, `outputs.tf`:

**Resources to create:**
- VPC with `enable_dns_support` and `enable_dns_hostnames`
- Internet Gateway
- 2 Public subnets (with `kubernetes.io/role/elb` tag for ALB discovery)
- 2 Private subnets (with `kubernetes.io/role/internal-elb` tag)
- Elastic IPs for NAT
- NAT Gateway (in public subnet)
- Public route table → IGW
- Private route tables → NAT
- Route table associations
- Network ACLs (public: HTTP/HTTPS/ephemeral/admin inbound, all outbound)

**Outputs:** `vpc_id`, `vpc_cidr`, `public_subnet_ids`, `private_subnet_ids`

### Step 3.4 — Create ECR Module

Create `02-Terraform/modules/ecr/` with `main.tf`, `variables.tf`, `outputs.tf`:

**Resources to create:**
- ECR repositories using `for_each` (not `count`) with `IMMUTABLE` tags, `scan_on_push`
- Lifecycle policies: expire untagged after 1 day, keep 10 most recent tagged

**Outputs:** `repository_urls`, `repository_arns`

### Step 3.5 — Create Server Module (Jenkins EC2)

Create `02-Terraform/modules/server/` with `main.tf`, `variables.tf`, `outputs.tf`:

**Resources to create:**
- Ubuntu 22.04 AMI data source
- Security group with separate ingress rules (SSH, Jenkins UI 8080)
- Egress rule (all outbound)
- IAM role with ECR push/pull permissions and EKS describe
- Instance profile
- EC2 instance with:
  - IMDSv2 enforced (`http_tokens = "required"`)
  - `gp3` encrypted root volume
  - `ignore_changes = [ami]`
  - Tag `Role = "jenkins"` (for Ansible dynamic inventory)
- Elastic IP

### Step 3.6 — Create EKS Module

Create `02-Terraform/modules/eks/` with `main.tf`, `variables.tf`, `outputs.tf`:

**Resources to create:**
- EKS cluster IAM role + policy attachments
- EKS cluster with `bootstrap_cluster_creator_admin_permissions = true`
- Node group IAM role + 3 policy attachments (Worker, CNI, ECR)
- Node group: 2 workers in private subnets, `t3.medium`, `AL2023`
- OIDC provider for IRSA
- IRSA roles:
  - EBS CSI Driver (`system:serviceaccount:kube-system:ebs-csi-controller-sa`)
  - AWS Load Balancer Controller (`system:serviceaccount:kube-system:aws-load-balancer-controller`)
- EKS add-ons: `vpc-cni`, `kube-proxy`, `coredns`, `aws-ebs-csi-driver`
- Jenkins access entry

**Download ALB controller policy:**
```bash
curl -sSL -o modules/eks/policies/aws-load-balancer-controller.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

### Step 3.7 — Wire Modules in `main.tf`

Create `02-Terraform/main.tf` connecting all four modules:

```hcl
module "network" {
  source               = "./modules/network"
  name_prefix          = local.name_prefix
  vpc_cidr             = var.vpc_cidr
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  cluster_name         = local.cluster_name
  tags                 = local.common_tags
}

module "ecr" {
  source           = "./modules/ecr"
  repository_names = var.ecr_repository_names
  tags             = local.common_tags
}

module "server" {
  source                   = "./modules/server"
  name_prefix              = local.name_prefix
  vpc_id                   = module.network.vpc_id
  subnet_id                = module.network.public_subnet_ids[0]
  key_name                 = var.key_name
  allowed_ssh_cidrs        = var.allowed_ssh_cidrs
  allowed_jenkins_ui_cidrs = var.allowed_jenkins_ui_cidrs
  ecr_repository_arns      = module.ecr.repository_arns
  eks_cluster_name         = local.cluster_name
  tags                     = local.common_tags
}

module "eks" {
  source             = "./modules/eks"
  cluster_name       = local.cluster_name
  kubernetes_version = var.kubernetes_version
  private_subnet_ids = module.network.private_subnet_ids
  public_subnet_ids  = module.network.public_subnet_ids
  jenkins_role_arn   = module.server.iam_role_arn
  tags               = local.common_tags
}
```

### Step 3.8 — Initialize, Validate, and Apply

```bash
cd 02-Terraform

cat > terraform.tfvars <<'EOF'
allowed_ssh_cidrs        = ["YOUR_IP/32"]
allowed_jenkins_ui_cidrs = ["YOUR_IP/32"]
EOF

terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan          # ~80 resources, 0 errors
terraform apply         # 15–20 minutes
```

### Step 3.9 — Verify Infrastructure

```bash
aws eks update-kubeconfig --region us-east-1 --name $(terraform output -raw eks_cluster_name)

kubectl get nodes -o custom-columns=\
NAME:.metadata.name,STATUS:.status.conditions[-1].type,ZONE:.metadata.labels.'topology\.kubernetes\.io/zone'
# → 2 nodes, Ready, different AZs

kubectl get pods -n kube-system | grep -E "coredns|aws-node|kube-proxy|ebs-csi"
# → All Running
```

### Step 3.10 — Commit Terraform Files

```bash
git add 02-Terraform/
git commit -m "feat: terraform modules for VPC, EC2, EKS, ECR with S3 backend"
git push origin main
```

**✅ Deliverable:** Terraform modules committed to the repository

---

## 4. Configuration Management with Ansible (~5 hours)

### Step 4.1 — Create Dynamic Inventory

Create `03-Ansible/inventory/aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Role: jenkins
  instance-state-name:
    - running
hostnames:
  - tag:Name
  - dns-name
keyed_groups:
  - key: tags.Role
    prefix: role
    separator: "_"
compose:
  ansible_host: public_ip_address
  ansible_user: "'ubuntu'"
  ansible_ssh_private_key_file: "'~/.ssh/ivolve-key.pem'"
cache: true
cache_plugin: jsonfile
cache_connection: ./.ansible_inventory_cache
cache_timeout: 300
```

### Step 4.2 — Create `ansible.cfg`

```ini
[defaults]
inventory        = ./inventory
remote_user      = ubuntu
private_key_file = ~/.ssh/ivolve-key.pem
host_key_checking = False
become = True
stdout_callback   = yaml
callbacks_enabled = timer, profile_tasks
vault_password_file = ./.vault_pass

[inventory]
enable_plugins = amazon.aws.aws_ec2, ini, yaml

[ssh_connection]
ssh_args  = -o ControlMaster=auto -o ControlPersist=300s -o PreferredAuthentications=publickey
pipelining = True
```

### Step 4.3 — Install Ansible Requirements

Create `03-Ansible/requirements.yml`:
```yaml
collections:
  - name: amazon.aws
    version: ">=9.0.0"
  - name: community.docker
    version: ">=4.0.0"
  - name: community.general
    version: ">=10.0.0"
  - name: ansible.posix
    version: ">=1.6.0"
```

```bash
cd 03-Ansible
ansible-galaxy collection install -r requirements.yml
pip install boto3 botocore
```

### Step 4.4 — Test Dynamic Inventory

```bash
ansible-inventory --graph
# Should show: @role_jenkins: ivolve-dev-jenkins

ansible role_jenkins -m ping
# Should return: SUCCESS => { "ping": "pong" }
```

### Step 4.5 — Set Up Ansible Vault

```bash
openssl rand -base64 32 > .vault_pass
chmod 600 .vault_pass

mkdir -p group_vars/all
cat > group_vars/all/vault.yml <<'EOF'
---
vault_sonarqube_admin_password: "CHANGE_ME"
vault_github_username: "YourGitHubUser"
vault_github_token: "CHANGE_ME_pat"
EOF

# Edit vault.yml with real values, then encrypt:
ansible-vault encrypt group_vars/all/vault.yml
```

### Step 4.6 — Create 9 Ansible Roles

Create roles under `03-Ansible/roles/`:

| # | Role | Key Actions |
|---|---|---|
| 1 | **common** | APT cache update, baseline packages, APT keyring dir, kernel params (`inotify`, `vm.max_map_count`) |
| 2 | **java** | Install OpenJDK 21 (Jenkins dependency) |
| 3 | **docker** | Remove distro packages, add Docker repo, install Docker CE + Compose, configure `daemon.json` (log rotation) |
| 4 | **jenkins** | Add Jenkins repo, install Jenkins, systemd override (heap, NOFILE, timeout), wait for HTTP 200/403, add to docker group |
| 5 | **trivy** | Add Aqua Security repo, install Trivy |
| 6 | **aws_cli** | Install AWS CLI v2 |
| 7 | **kubectl** | Install kubectl |
| 8 | **helm** | Install Helm 3 |
| 9 | **sonarqube** | Run SonarQube via Docker container |

Role structure:
```text
roles/<role_name>/
├── tasks/main.yml       # What to do
├── defaults/main.yml    # Overridable variables
├── handlers/main.yml    # Restart triggers
├── templates/           # Jinja2 templates
└── meta/main.yml        # Dependencies
```

### Step 4.7 — Create the Playbook

Create `03-Ansible/playbook.yml`:

```yaml
---
- name: Configure Jenkins CI server
  hosts: role_jenkins
  become: true
  any_errors_fatal: true

  pre_tasks:
    - name: Wait for SSH
      ansible.builtin.wait_for_connection:
        delay: 5
        timeout: 300

    - name: Verify the target is Ubuntu 22.04+
      ansible.builtin.assert:
        that:
          - ansible_facts['distribution'] == 'Ubuntu'
          - ansible_facts['distribution_major_version'] is version('22', '>=')

    - name: Wait for Terraform bootstrap to finish
      ansible.builtin.wait_for:
        path: /etc/ivolve-bootstrap-complete
        timeout: 600

  roles:
    - { role: common,    tags: [common] }
    - { role: java,      tags: [java] }
    - { role: docker,    tags: [docker] }
    - { role: jenkins,   tags: [jenkins] }
    - { role: trivy,     tags: [trivy] }
    - { role: aws_cli,   tags: [aws] }
    - { role: kubectl,   tags: [kubectl] }
    - { role: helm,      tags: [helm] }
    - { role: sonarqube, tags: [sonarqube] }

  post_tasks:
    - name: Show the summary
      ansible.builtin.debug:
        msg: |
          Jenkins   http://{{ ansible_host }}:8080
          SonarQube http://{{ ansible_host }}:9000
```

### Step 4.8 — Run the Playbook

```bash
ansible-playbook playbook.yml --check --diff     # Dry run first
ansible-playbook playbook.yml                    # ~10 minutes
```

### Step 4.9 — Verify Idempotency

```bash
ansible-playbook playbook.yml | tail -3
# ok=48  changed=0  unreachable=0  failed=0
```

### Step 4.10 — Commit Ansible Files

```bash
git add 03-Ansible/
git commit -m "feat: ansible playbook with 9 roles and dynamic inventory"
git push origin main
```

**✅ Deliverable:** Ansible modules, inventory, and playbook committed to the repository

---

## 5. Container Orchestration with Kubernetes (~6 hours)

### Step 5.1 — Install AWS Load Balancer Controller

```bash
CLUSTER=$(terraform -chdir=../02-Terraform output -raw eks_cluster_name)
ROLE_ARN=$(terraform -chdir=../02-Terraform output -raw aws_load_balancer_controller_role_arn)

helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=$ROLE_ARN"

kubectl -n kube-system rollout status deploy/aws-load-balancer-controller
```

### Step 5.2 — Create Namespace (`00-namespace.yaml`)

- Namespace `ivolve` with Pod Security Admission labels (`restricted`)
- ResourceQuota (CPU: 4 req / 8 limit, Memory: 8Gi req / 16Gi limit, 4 PVCs, 20 pods)
- LimitRange (default CPU: 500m/100m, Memory: 512Mi/128Mi)

```bash
kubectl apply -f manifests/00-namespace.yaml
```

### Step 5.3 — Create RBAC (`01-rbac.yaml`)

- 4 ServiceAccounts: `frontend`, `auth-service`, `roadmap-service`, `mysql`
- Roles and RoleBindings for each

### Step 5.4 — Create ConfigMap and Secret (`02-config.yaml`)

**ConfigMap (`ivolve-config`):**
| Key | Value |
|---|---|
| `DB_HOST` | `mysql-0.mysql.ivolve.svc.cluster.local` |
| `DB_PORT` | `3306` |
| `DB_NAME` | `ivolve` |
| `DB_USER` | `ivolve_user` |
| `AUTH_SERVICE_URL` | `http://auth-service:5000` |
| `ROADMAP_SERVICE_URL` | `http://roadmap-service:8080` |

**Secret (`ivolve-secret`):**
`MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `DB_PASSWORD`, `SESSION_SECRET`

### Step 5.5 — Create StorageClass (`03-storage.yaml`)

- Name: `ivolve-storage`
- Provisioner: `ebs.csi.aws.com` (NOT legacy `kubernetes.io/aws-ebs`)
- Parameters: `gp3`, `encrypted: "true"`
- `reclaimPolicy: Retain`, `allowVolumeExpansion: true`
- `volumeBindingMode: WaitForFirstConsumer` (critical for multi-AZ)

### Step 5.6 — Create Database StatefulSet (`04-database.yaml`)

- **Headless Service** (`clusterIP: None`)
- **StatefulSet** (1 replica, MySQL 8.0):
  - `securityContext`: `runAsNonRoot`, uid/gid 999, `fsGroup: 999`
  - `startupProbe`, `readinessProbe`, `livenessProbe`
  - `volumeClaimTemplates`: 10Gi using `ivolve-storage`

```bash
kubectl apply -f manifests/02-config.yaml -f manifests/04-database.yaml
kubectl get pvc -n ivolve    # data-mysql-0 should be Bound
```

### Step 5.7 — Create Auth-Service Deployment (`05-auth-service.yaml`)

- 2 replicas, `initContainer` (`wait-for-mysql` using `busybox:1.36`)
- `envFrom` referencing ConfigMap and Secret
- Probes on `/health`, `readOnlyRootFilesystem: true`
- ClusterIP Service on port 5000

### Step 5.8 — Create Roadmap-Service Deployment (`06-roadmap-service.yaml`)

- 2 replicas
- Probes on `/api/roadmap` (NOT `/health`)
- ClusterIP Service on port 8080

### Step 5.9 — Create Frontend Deployment (`07-frontend.yaml`)

- 2 replicas
- NodePort Service on port 3000

### Step 5.10 — Create Ingress (`08-ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: ivolve
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  ingressClassName: alb      # NOT the deprecated annotation
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  name: http
```

### Step 5.11 — Create Network Policies (`12-network-policies.yaml`)

| Policy | Purpose |
|---|---|
| `default-deny-all` | Deny all ingress/egress by default |
| `allow-dns-egress` | Allow DNS to kube-system (critical!) |
| `mysql-policy` | Only auth-service → MySQL |
| `auth-service-policy` | Only frontend → auth-service |
| `roadmap-service-policy` | Only frontend → roadmap-service |
| `frontend-policy` | Allow ingress from ALB |
| `jenkins-policy` | (Reference only) Jenkins access |
| `sonarqube-policy` | (Reference only) SonarQube access |

> 💡 **Note:** The reference implementation includes manifests `09-jenkins.yaml`, `10-sonarqube.yaml`, and `11-argocd-ingress.yaml` which are optional in-cluster deployments not required for the core path.

### Step 5.12 — Create `kustomization.yaml`

- List all 10 (or 13 if using the optional ones) manifests as resources
- `images:` mapping service names to ECR registry URLs
- `labels:` with `app.kubernetes.io/part-of: ivolve`, `includeSelectors: false`

### Step 5.13 — Apply and Verify

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/123456789012/$ACCOUNT/g" kustomization.yaml

kubectl apply -k .
kubectl get pods,pvc,ingress -n ivolve
# All pods Running, PVC Bound, Ingress has an ADDRESS
```

### Step 5.14 — Commit Kubernetes Files

```bash
git add 04-Kubernetes/
git commit -m "feat: kubernetes manifests with namespace, statefulset, deployments, ingress"
git push origin main
```

**✅ Deliverable:** Required YAML files committed to the repository

---

## 6. Continuous Integration with Jenkins (~5 hours)

### Step 6.1 — Unlock Jenkins

```bash
ssh -i ~/.ssh/ivolve-key.pem ubuntu@<JENKINS_IP> \
  'sudo cat /var/lib/jenkins/secrets/initialAdminPassword'
```

Open `http://<JENKINS_IP>:8080`, paste the password.
**Select plugins → "Select none"** → Create admin user.

### Step 6.2 — Register Shared Library

**Manage Jenkins → System → Global Pipeline Libraries → Add:**

| Field | Value |
|---|---|
| Name | `shared-library` |
| Default version | `main` |
| Retrieval | Modern SCM → Git → your repo |

### Step 6.3 — Add Credentials

**Manage Jenkins → Credentials → System → Global → Add:**

| ID (exact match!) | Kind | Value |
|---|---|---|
| `github-token` | Username with password | GitHub user + PAT |
| `sonar-token` | Secret text | SonarQube token |

### Step 6.4 — Configure SonarQube

1. Open `http://<JENKINS_IP>:9000` — login `admin`/`admin`, change password
2. **My Account → Security → Generate Token** → add to Jenkins as `sonar-token`
3. **Manage Jenkins → System → SonarQube servers → Add:**
   - Name: `SonarQube`, URL: `http://localhost:9000`, Token: `sonar-token`
4. **⚠️ THE WEBHOOK (most commonly missed step!):**
   - SonarQube → Administration → Configuration → Webhooks → Create
   - Name: `Jenkins`, URL: `http://localhost:8080/sonarqube-webhook/`

### Step 6.5 — Create Shared Library (8 Groovy Steps)

Create files in `05-Jenkins/vars/`:

| File | Purpose |
|---|---|
| `dockerBuildImage.groovy` | Build Docker image with OCI labels |
| `trivyScan.groovy` | 2-pass Trivy scan: report + gate on CRITICAL |
| `ecrPush.groovy` | Push to ECR via IAM instance profile (no creds!) |
| `runUnitTests.groovy` | Containerized unit tests per language |
| `sonarQubeScan.groovy` | SonarQube analysis + blocking quality gate |
| `updateManifests.groovy` | `kustomize edit set image` (NOT sed!) |
| `pushManifests.groovy` | Git commit + push with `[skip ci]` to prevent loops |
| `microservicePipeline.groovy` | 9-stage orchestrator |

**9 Pipeline Stages:**
```
1. Checkout           → tag = <build>-<git-sha>, never :latest
2. Unit Tests         → containerized, per language
3. SonarQube Analysis → analysis + blocking quality gate
4. Build Image        → multi-stage Docker build
5. Scan Image         → Trivy: fixable CRITICAL ⇒ BUILD FAILS
6. Push Image         → ECR (IAM instance profile)
7. Delete Image       → reclaim local disk space
8. Update Manifests   → kustomize edit set image
9. Push Manifests     → git commit [skip ci] → ArgoCD deploys
```

### Step 6.6 — Create 3 Jenkinsfiles

**`05-Jenkins/Jenkinsfiles/frontend.Jenkinsfile`:**
```groovy
@Library('shared-library') _

microservicePipeline(
    serviceName: 'ivolve-frontend',
    sourceDir:   'src/frontend',
    language:    'node',
    ecrRegistry: '<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com',
    gitRepo:     'github.com/YourUser/CloudDevOpsProject.git'
)
```

Repeat for:
- `auth-service.Jenkinsfile` (`language: 'python'`)
- `roadmap-service.Jenkinsfile` (`language: 'java'`)

### Step 6.7 — Set Real Account ID

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/123456789012/$ACCOUNT/g" 05-Jenkins/Jenkinsfiles/*.Jenkinsfile
git commit -am "chore: real AWS account ID" && git push
```

### Step 6.8 — Create Jenkins Pipeline Jobs

For each service → **New Item → Pipeline:**
- Definition: **Pipeline script from SCM**
- SCM: Git → your repo → credentials `github-token`
- Branch: `*/main`
- Script Path: `05-Jenkins/Jenkinsfiles/<service>.Jenkinsfile`

### Step 6.9 — Build and Verify

```bash
# After all 3 pipelines succeed:
aws ecr list-images --repository-name ivolve-frontend \
  --query 'imageIds[*].imageTag' --output table

git pull && git log --oneline -3
# Should show Jenkins' ci() commit with [skip ci]
```

### Step 6.10 — Commit Jenkins Files

```bash
git add 05-Jenkins/
git commit -m "feat: jenkins shared library and pipeline files"
git push origin main
```

**✅ Deliverables:**
- Jenkins files committed to the repository
- Shared library directory (`vars/`) committed to the repository

---

## 7. Continuous Deployment with ArgoCD (~2 hours)

### Step 7.1 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### Step 7.2 — Access the ArgoCD UI

```bash
# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080 · user: admin
```

### Step 7.3 — Create AppProject

Create `06-ArgoCD/project.yaml`:
- Project name: `ivolve`, namespace: `argocd`
- Restricted `sourceRepos` to your GitHub repo only
- Restricted `destinations` to `ivolve` namespace only
- `clusterResourceWhitelist` for `StorageClass`

### Step 7.4 — Create ArgoCD Application

Create `06-ArgoCD/applications/ivolve-app.yaml`:
- Source: your repo, path `04-Kubernetes/manifests` (auto-detects Kustomize)
- Destination: in-cluster, namespace `ivolve`
- **Automated sync policy:**
  - `prune: true` — delete resources removed from Git
  - `selfHeal: true` — revert manual `kubectl` changes
- Retry with exponential backoff
- `ignoreDifferences` for `/spec/replicas` (prevents HPA fight)
- `resources-finalizer.argocd.argoproj.io` finalizer

### Step 7.5 — Add Sync Wave Annotations

Add `argocd.argoproj.io/sync-wave` to Module 04 manifests:

| Wave | Resources |
|:---:|---|
| `-1` | Namespace, ResourceQuota, LimitRange, RBAC |
| `0` | ConfigMap, Secret, StorageClass |
| `1` | MySQL StatefulSet + headless Service |
| `2` | auth-service, roadmap-service |
| `3` | frontend |
| `4` | Ingress |

### Step 7.6 — Apply and Verify

```bash
kubectl apply -f 06-ArgoCD/project.yaml
kubectl apply -f 06-ArgoCD/applications/ivolve-app.yaml

kubectl get application -n argocd -w
# NAME: ivolve   SYNC STATUS: Synced   HEALTH STATUS: Healthy
```

### Step 7.7 — Prove Self-Heal Works

```bash
# Terminal B: watch
kubectl get deploy frontend -n ivolve -w

# Terminal A: manual change
kubectl scale deployment frontend -n ivolve --replicas=7
# → Watch ArgoCD revert it back to 2 within ~3 minutes

kubectl delete deployment frontend -n ivolve
# → Watch ArgoCD recreate it
```

### Step 7.8 — Commit ArgoCD Files

```bash
git add 06-ArgoCD/
git commit -m "feat: argocd project and application with auto-sync and self-heal"
git push origin main
```

**✅ Deliverable:** ArgoCD Application committed to the repository

---

## 8. Documentation

### Step 8.1 — Architecture Overview

Include in `README.md`:

**CI/CD Pipeline Flow:**
```
Developer → Git Push → Jenkins (Build → Test → SonarQube → Scan → Push ECR)
                                    ↓
                          Git Commit (kustomization.yaml update)
                                    ↓
                    ArgoCD (detect commit → sync) → EKS Cluster
```

**Infrastructure Diagram:**
```
Internet
   │
   ▼
  IGW ──────────────────────────────────────────────┐
   │                                                 │
 PUBLIC 10.0.1.0/24 (AZ-a)  10.0.2.0/24 (AZ-b)    │
   · ALB          · NAT Gateway     · Jenkins EC2   │
        │                                            │
        │ NAT — outbound only                        │
        ▼                                            │
 PRIVATE 10.0.10.0/24        10.0.11.0/24            │
   · EKS worker 1            · EKS worker 2         │
   · No public IP                                    │
─────────────────────────────────────────────────────┘
```

### Step 8.2 — Setup Instructions

Document in `README.md` or `SETUP.md`:
1. Prerequisites — AWS account, tools, credentials
2. Bootstrap — Create S3 state bucket (`terraform apply` in bootstrap/)
3. Infrastructure — `terraform apply` in 02-Terraform/
4. Configuration — `ansible-playbook playbook.yml` in 03-Ansible/
5. Application — `kubectl apply -k` in 04-Kubernetes/manifests/
6. CI Setup — Jenkins web setup, credentials, SonarQube webhook
7. CD Setup — ArgoCD install, project + application
8. Teardown — Proper destroy order (Ingress → namespaces → terraform destroy)

### Step 8.3 — Commit Documentation

```bash
git add README.md
git commit -m "docs: comprehensive setup instructions and architecture overview"
git push origin main
```

**✅ Deliverable:** Comprehensive documentation committed to the repository

---

## 🎓 Final Verification — The Complete Loop

1. Change a line in `src/frontend/views/roadmap.ejs`
2. `git add . && git commit -m "feat: update heading" && git push`
3. Jenkins builds → tests → Sonar → scans → pushes to ECR
4. Jenkins commits the new image tag to `kustomization.yaml` with `[skip ci]`
5. ArgoCD detects the commit and syncs
6. Rolling update deploys the new version
7. **Your change is live in the browser — with no `kubectl apply` anywhere**

**That is the entire point of the project.**

---

## 🧹 Teardown — ⚠️ DO NOT SKIP

```bash
# 1 — Delete Kubernetes resources FIRST
kubectl delete ingress --all -n ivolve
kubectl delete namespace ivolve argocd --ignore-not-found

# 2 — Confirm the ALB is gone
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'

# 3 — Destroy infrastructure
cd 02-Terraform && terraform destroy

# 4 — Clean up
aws ec2 delete-key-pair --key-name ivolve-key

# 5 — Check billing dashboard
```

> ⚠️ **Why Ingress first?** The ALB is owned by the Ingress controller, not Terraform. Destroying the VPC while it exists leaves an orphaned ALB and Terraform hangs ~20 minutes on `DependencyViolation`.
