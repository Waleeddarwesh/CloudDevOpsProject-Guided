# 🧰 Module 00 — Prerequisites

**Time:** ~45 minutes · **Cost:** free · **AWS resources created:** none (except a key pair)

---

## 🎯 What you'll have at the end

Seven working tools, credentials that authenticate, a budget alarm protecting you, and the application source on disk.

Nothing here is interesting. Do it carefully anyway — **every hour lost later to a "weird bug" traces back to a skipped step here.**

---

## 1 · Install the tools

| Tool | Minimum | Why this project needs it |
|---|---|---|
| AWS CLI | **v2** | ECR login, `eks update-kubeconfig`. v1 lacks `get-login-password`. |
| Terraform | **≥ 1.10** ⚠️ | We use S3 native state locking (`use_lockfile`), added in 1.10 |
| Docker Desktop | recent | Building images, running the local stack |
| kubectl | ≥ 1.29 | Talking to the cluster. Bundles `kustomize`. |
| Helm | 3.x | Installing the LB Controller and monitoring stack |
| Ansible | ≥ 2.15 | Configuring the Jenkins server |
| Git | any | Everything |

### Windows

Ansible **does not run natively on Windows.** Install WSL2:

```powershell
wsl --install
```

Then run all Ansible commands from inside WSL. Everything else works in PowerShell or Git Bash.

```powershell
winget install Amazon.AWSCLI
winget install HashiCorp.Terraform
winget install Docker.DockerDesktop
winget install Kubernetes.kubectl
winget install Helm.Helm
winget install Git.Git
```

### macOS

```bash
brew install awscli terraform kubectl helm ansible git
brew install --cask docker
```

### Ubuntu / WSL

```bash
sudo apt-get update && sudo apt-get install -y ansible git curl unzip

# AWS CLI v2 — the apt package is v1, which will not work
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install

# Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
```

### ✅ Verify all seven

```bash
for t in aws terraform docker kubectl helm ansible git; do
  printf "%-12s " "$t"
  command -v $t >/dev/null && $t --version 2>&1 | head -1 || echo "MISSING"
done
```

> ⚠️ **Check the Terraform line reads 1.10 or higher.** On 1.9 you'll hit `Unsupported argument: use_lockfile` in Module 02 — a confusing error that looks like a typo in your code.

---

## 2 · AWS credentials

```bash
aws configure
```

Verify — **write down the Account ID, you need it repeatedly**:

```bash
aws sts get-caller-identity
```

```json
{
    "UserId": "AIDA...",
    "Account": "111122223333",      ← THIS
    "Arn": "arn:aws:iam::111122223333:user/your-user"
}
```

> ⚠️ **Never use the root account.** Create an IAM user. For a learning project `AdministratorAccess` is simplest; in a real job you'd scope it down.

<details>
<summary><b>Why not root?</b></summary>

Root can close the account, change billing, and cannot be restricted by any policy. A leaked root key is unrecoverable — you cannot revoke its permissions, only delete the key after the damage. An IAM user's blast radius is whatever policy you attached.
</details>

---

## 3 · 💰 Set a budget alarm — do this now

**Two minutes. Do not skip.** From Module 02 you're spending ~$0.30/hour.

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

aws budgets create-budget --account-id "$ACCOUNT" --budget '{
  "BudgetName": "ivolve-learning",
  "BudgetLimit": {"Amount": "25", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}'
```

Add an email alert at 80%:

```bash
aws budgets create-notification --account-id "$ACCOUNT" \
  --budget-name ivolve-learning \
  --notification '{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"}' \
  --subscribers '[{"SubscriptionType":"EMAIL","Address":"YOUR_EMAIL@example.com"}]'
```

> 💡 Budgets are evaluated a few times a day, not in real time. This is a safety net, not a hard stop. **Destroying resources when you finish a session is the real protection.**

---

## 4 · Create the EC2 key pair

Terraform will reference this by name; it must exist first.

```bash
aws ec2 create-key-pair --key-name ivolve-key \
  --query KeyMaterial --output text > ~/.ssh/ivolve-key.pem

chmod 400 ~/.ssh/ivolve-key.pem
```

<details>
<summary><b>Windows PowerShell — <code>&gt;</code> corrupts the key</b></summary>

PowerShell's `>` writes UTF-16 with a BOM. SSH rejects the file with `invalid format`.

```powershell
aws ec2 create-key-pair --key-name ivolve-key `
  --query KeyMaterial --output text | Out-File -Encoding ascii $HOME\.ssh\ivolve-key.pem
```

Verify it starts correctly:
```powershell
Get-Content $HOME\.ssh\ivolve-key.pem -TotalCount 1
# -----BEGIN RSA PRIVATE KEY-----
```
</details>

**Why `chmod 400`?** SSH refuses to use a private key that others can read:

```text
Permissions 0644 for 'ivolve-key.pem' are too open.
It is required that your private key files are NOT accessible by others.
```

---

## 5 · Find your public IP

```bash
curl -s https://checkip.amazonaws.com
```

Write it down. In Module 02 you'll use it as `YOUR_IP/32` to lock SSH and the Jenkins UI to just your machine.

> ⚠️ **This is the single most important security setting in the project.** An SSH port open to `0.0.0.0/0` is found by automated scanners within *minutes*. The usual result is a crypto-mining bill.
>
> The Terraform you write in Module 02 will **refuse to apply** if you leave it open. That guard is deliberate.

**If your ISP gives you a dynamic IP** it will change. When SSH suddenly fails, re-run this command and update `terraform.tfvars`.

---

## 6 · GitHub Personal Access Token

The Jenkins pipeline commits updated manifests back to your repo.

1. GitHub → **Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. **Generate new token**
3. Repository access → **Only select repositories** → your project repo
4. Permissions → Repository permissions → **Contents: Read and write**
5. Copy it — **shown once**

> ⚠️ Use a **fine-grained** token. A classic token with `repo` scope grants access to *every* repository you own. If Jenkins is compromised, so is all of it.

---

## 7 · Get the application source

You're building the DevOps platform, not the app. Clone the app into your workspace:

```bash
cd workspace
git clone --depth 1 https://github.com/Ibrahim-Adel15/iVolveFinalProject.git /tmp/app-src
cp -r /tmp/app-src/frontend /tmp/app-src/auth-service /tmp/app-src/roadmap-service src/
rm -rf /tmp/app-src
```

Verify:

```bash
find src -maxdepth 2 -type f | sort
```

```text
src/auth-service/Dockerfile
src/auth-service/app.py
src/auth-service/requirements.txt
src/frontend/Dockerfile
src/frontend/package.json
src/frontend/server.js
src/roadmap-service/Dockerfile
src/roadmap-service/pom.xml
```

---

## 8 · Read the application before you containerise it

**Spend ten minutes here.** You cannot write a correct Dockerfile or Kubernetes manifest for software you haven't looked at.

### The three services

| Service | Language | Port | Talks to |
|---|---|:---:|---|
| `frontend` | Node.js 22 · Express · EJS | 3000 | auth + roadmap |
| `auth-service` | **Python 3.12 · Flask** | 5000 | MySQL |
| `roadmap-service` | **Java 21 · Spring Boot** | 8080 | nothing |

> ⚠️ **The upstream README is wrong.** It claims auth-service is Java and roadmap-service is Python. **The code says the opposite.** Check for yourself:
>
> ```bash
> head -5 src/auth-service/app.py        # → import flask   (Python)
> grep artifactId src/roadmap-service/pom.xml | head -2   # → spring-boot  (Java)
> ```
>
> **Always trust the code over the docs.** This is a genuinely useful habit, and this project hands you a free example of why.

### Environment variables — read these from the source

```bash
grep -n "os.environ" src/auth-service/app.py
grep -n "process.env" src/frontend/server.js
```

```python
# auth-service/app.py
"host":     os.environ.get("DB_HOST"),
"port":     os.environ.get("DB_PORT", "3306"),
"user":     os.environ.get("DB_USER"),        # ← DB_USER, not DB_USERNAME
"password": os.environ.get("DB_PASSWORD"),
"database": os.environ.get("DB_NAME"),
```

```javascript
// frontend/server.js
const AUTH_SERVICE_URL    = process.env.AUTH_SERVICE_URL    || "http://localhost:5000";
const ROADMAP_SERVICE_URL = process.env.ROADMAP_SERVICE_URL || "http://localhost:8080";
      secret:               process.env.SESSION_SECRET      || "change-me-in-k8s";
```

**Three things to notice:**

1. **`DB_USER`, not `DB_USERNAME`.** Guess wrong and MySQL rejects the connection with `Access denied`, which looks like a password problem.
2. **The service URLs default to `localhost`** — fine on your laptop, useless in a container. You must supply them.
3. **`SESSION_SECRET` defaults to the literal string `change-me-in-k8s`.** Anyone who knows that value can forge a session cookie for any user and skip login entirely. Always override it.

### The endpoints

```bash
grep -n "@app\." src/auth-service/app.py
```

```text
@app.get("/health")              ← used by the container healthcheck & K8s probes
@app.post("/api/auth/signup")
@app.post("/api/auth/login")
```

```bash
grep -n "GetMapping" src/roadmap-service/src/main/java/com/ivolve/roadmap/RoadmapController.java
```

```text
@GetMapping("/api/roadmap")      ← the ONLY route
```

> 💡 **roadmap-service has no `/health` endpoint.** Only `/api/roadmap`. When you write its healthcheck and its Kubernetes probes in the next modules, you must point them at `/api/roadmap`. Pointing at `/health` gives you a permanently unhealthy container and a very confusing hour.

---

## ✅ Checkpoint

```bash
echo "── tools ──"
for t in aws terraform docker kubectl helm ansible git; do
  printf "%-12s " "$t"; command -v $t >/dev/null && echo OK || echo MISSING
done

echo "── aws ──"
aws sts get-caller-identity --query Account --output text

echo "── key ──"
ls -l ~/.ssh/ivolve-key.pem

echo "── ip ──"
curl -s https://checkip.amazonaws.com

echo "── source ──"
ls workspace/src/
```

Everything present? Tick Module 00 in [PROGRESS.md](../PROGRESS.md).

---

## 🧠 Before you continue

Answer these from memory. If you can't, re-read the section.

1. Why must Terraform be ≥ 1.10?
2. What breaks if you set `allowed_ssh_cidrs = ["0.0.0.0/0"]`?
3. Which service is Python, and how do you *know* — not from the README?
4. What is the env var for the database user, exactly?
5. Which endpoint must roadmap-service's healthcheck use, and why not `/health`?

---

<div align="center">

**Next → [Module 01 — Docker](01-docker.md)**

*Free, no AWS. You'll have the full app running on your laptop.*

</div>
