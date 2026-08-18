<div align="center">

# 🎓 Build It Yourself — Cloud DevOps Guided Capstone

**A guided course where *you* write every line. Nothing is copied.**

![Modules](https://img.shields.io/badge/Modules-8-blue?style=flat-square)
![Time](https://img.shields.io/badge/Est._Time-25--35_hours-orange?style=flat-square)
![Level](https://img.shields.io/badge/Level-Intermediate-green?style=flat-square)

</div>

---

## 📌 What this is

[`CloudDevOpsProject`](https://github.com/Waleeddarwesh/CloudDevOpsProject) is a **finished reference implementation** — complete, validated, deployable.

**This repository is the opposite.** It is empty on purpose. You will build the same platform from nothing, one file at a time, and every line you type will be explained *before* you type it.

### Clone both, side by side

The guides refer to the reference implementation by relative path, so put the
two repositories next to each other:

```bash
mkdir cloud-devops && cd cloud-devops
git clone https://github.com/Waleeddarwesh/CloudDevOpsProject.git
git clone https://github.com/Waleeddarwesh/CloudDevOpsProject-Guided.git
cd CloudDevOpsProject-Guided
```

```text
cloud-devops/
├── CloudDevOpsProject/          the answer key      ← look here when stuck
└── CloudDevOpsProject-Guided/   your workbench      ← you are here
```

> 💡 **The reference implementation is not cheating.** Use it when you're stuck for more than 15 minutes. The goal is understanding, not suffering. But *try first* — the struggle is where the learning happens.

---

## 🧭 How these guides teach

Most tutorials hand you a finished 200-line file and say "this is the config." You copy it, it works, and you learn nothing.

These guides do the opposite. Every module follows the same rhythm:

```text
  ┌─────────────────────────────────────────────────────────────┐
  │  1. THE PROBLEM     What are we trying to solve, and why    │
  │                     does the naive approach fail?           │
  ├─────────────────────────────────────────────────────────────┤
  │  2. THE SMALLEST    Type ~10 lines. Run it. See it work.    │
  │     THING THAT      No abstractions yet.                    │
  │     WORKS                                                   │
  ├─────────────────────────────────────────────────────────────┤
  │  3. BREAK IT        Deliberately introduce the mistake      │
  │                     everyone makes. See the real error      │
  │                     message. Now you'll recognise it.       │
  ├─────────────────────────────────────────────────────────────┤
  │  4. FIX IT          Add the line that solves it — and now   │
  │     PROPERLY        you know *why* that line exists.        │
  ├─────────────────────────────────────────────────────────────┤
  │  5. CHECKPOINT      A command that proves it works.         │
  │                     Expected output shown.                  │
  └─────────────────────────────────────────────────────────────┘
```

**Step 3 is the important one.** You will spend more of your career reading error messages than writing config. Meeting each error deliberately, in a safe context, is worth ten tutorials.

---

## 📚 Module Map

| # | Module | Build | Time | Cost |
|:-:|---|---|:---:|:---:|
| **00** | [Prerequisites](guides/00-prerequisites.md) | Tools, AWS account, SSH key | 45 m | free |
| **01** | [Docker](guides/01-docker.md) | 3 Dockerfiles + Compose stack | 4 h | free |
| **02** | [Terraform](guides/02-terraform.md) | VPC · EC2 · EKS · ECR, 4 modules | 8 h | 💰 |
| **03** | [Ansible](guides/03-ansible.md) | Dynamic inventory + 9 roles + Vault | 5 h | — |
| **04** | [Kubernetes](guides/04-kubernetes.md) | 37 objects, namespace → ingress | 6 h | — |
| **05** | [Jenkins](guides/05-jenkins.md) | Shared library + 9-stage pipeline | 5 h | — |
| **06** | [ArgoCD](guides/06-argocd.md) | GitOps reconciliation | 2 h | — |
| **07** | [Monitoring](guides/07-monitoring.md) | Prometheus · Grafana · Alertmanager | 3 h | — |

> ⚠️ **Modules 00 and 01 are free and use no AWS.** Do them first. You'll have a working application on your laptop before spending a cent — and if something breaks later, you'll know it's infrastructure, not code.

---

## 💰 Cost — read before Module 02

From Module 02 onward this provisions **real, billable AWS infrastructure**.

| Resource | Rate | If left running |
|---|---|---|
| EKS control plane | $0.10/hr | ~$73/mo |
| 2 × t3.medium nodes | $0.083/hr | ~$60/mo |
| Jenkins t3.medium | $0.042/hr | ~$30/mo |
| NAT Gateway | $0.045/hr | ~$33/mo |
| ALB | $0.023/hr | ~$17/mo |
| **Total** | **~$0.30/hr** | **~$215/mo** |

**≈ $7 per day if you forget.** None of it is free-tier eligible — EKS has no free tier at all.

### Two habits that will save you money

**1. Set a budget alarm before you start.** Two minutes, once:

```bash
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{"BudgetName":"ivolve-learning","BudgetLimit":{"Amount":"25","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}'
```

**2. Destroy at the end of every session.** You are learning, not running production. Rebuilding takes 20 minutes and reinforces the material:

```bash
cd workspace/02-Terraform && terraform destroy
```

> 💡 Being *comfortable* destroying and rebuilding infrastructure is itself a core DevOps skill. If tearing down makes you nervous, your automation isn't good enough yet.

---

## 📂 Your workspace

You'll create files under `workspace/`. It starts empty — that's the point.

```text
CloudDevOpsProject-Guided/
│
├── README.md               ← you are here
├── PROGRESS.md             ← tick things off as you go
│
├── guides/                 ← the course
│   ├── 00-prerequisites.md
│   ├── 01-docker.md
│   ├── 02-terraform.md
│   ├── 03-ansible.md
│   ├── 04-kubernetes.md
│   ├── 05-jenkins.md
│   ├── 06-argocd.md
│   └── 07-monitoring.md
│
├── reference/
│   ├── cheatsheet.md       ← commands you'll forget
│   └── errors.md           ← every error message, decoded
│
└── workspace/              ← YOUR files go here
    ├── src/                (empty — Module 01 fills it)
    ├── 01-Docker/          (empty)
    ├── 02-Terraform/       (empty)
    ├── 03-Ansible/         (empty)
    ├── 04-Kubernetes/      (empty)
    ├── 05-Jenkins/         (empty)
    ├── 06-ArgoCD/          (empty)
    └── 07-Monitoring/      (empty)
```

---

## 🛠 Working method

### Type the code. Don't paste it.

This sounds like busywork. It isn't.

Typing forces you to read every character. You will notice that `image_tag_mutability` is spelled with an underscore, that YAML cares about two spaces, that `ingressClassName` is camelCase while `image-tag` is kebab. Pasting skips all of that, and then those details bite you at 2am during a real incident.

**A reasonable compromise:** type everything in Modules 00–04. By Module 05 you'll have the muscle memory; paste the boilerplate and type the parts that are new.

### Commit after every checkpoint

```bash
cd workspace
git add .
git commit -m "module 01: compose stack running locally"
```

Your commit history becomes a record of your own learning — and it lets you `git diff` against the reference implementation to see exactly where you diverged.

### Keep two terminals open

| Terminal | For |
|---|---|
| **A** | typing commands from the guide |
| **B** | `watch kubectl get pods -n ivolve`, `docker compose logs -f`, `terraform plan` |

Watching state change in real time as you apply config is the single fastest way to build intuition.

---

## ✅ Checkpoints

Every module ends with a checkpoint — a command with expected output. **Do not move on until yours matches.**

```bash
# Module 01 checkpoint, for example
docker compose ps
```

```text
NAME                     STATUS
ivolve-mysql             Up (healthy)
ivolve-auth-service      Up (healthy)
ivolve-roadmap-service   Up (healthy)
ivolve-frontend          Up (healthy)
```

A broken foundation compounds. If Module 02's network is subtly wrong, Module 04's pods won't schedule and you'll spend three hours debugging Kubernetes when the bug is in a route table.

---

## 🆘 When you're stuck

In order:

1. **Read the error message.** Actually read it. Most say exactly what's wrong.
2. **Check [reference/errors.md](reference/errors.md)** — every error these guides can produce, decoded.
3. **Diff against the reference:**
   ```bash
   diff workspace/01-Docker/docker-compose.yml \
        ../CloudDevOpsProject/01-Docker/docker-compose.yml
   ```
4. **Read the reference file's comments.** Every file in `CloudDevOpsProject/` is heavily commented with *why*, not just *what*.

> 💡 **15-minute rule.** Stuck longer than 15 minutes with no new information? Look at the answer. Then close it and retype the fix from memory. You keep the learning without losing the afternoon.

---

## ⚖️ Differences from Reference

This Guided project is designed for learning, while the reference implementation (`CloudDevOpsProject`) includes a few advanced additions that you can explore later:

1. **Docker Compose Complexity:** The reference implementation's `docker-compose.yml` uses advanced YAML anchors (`x-logging`, `x-service-defaults`) to reduce duplication. The version taught here is simpler to help you understand the core mechanics first.
2. **Additional Kubernetes Manifests:** The reference implementation deploys Jenkins and SonarQube *inside* the cluster as optional extras (manifests `09` and `10`), plus an ArgoCD Ingress (`11`). This Guided course focuses on the core path: running Jenkins directly on the EC2 instance provisioned in Module 03.
3. **Network Policies:** Because the reference has extra manifests, it also contains 8 NetworkPolicies instead of the 6 taught here.

---

## 🎯 What you'll be able to do afterwards

Not "I followed a tutorial" but genuinely:

- Design a VPC with public/private subnets and explain **why** the NAT Gateway lives in the public one
- Write a Terraform module with a clean input/output interface, and explain why `for_each` beats `count`
- Explain what IRSA is, why it exists, and what breaks without it
- Debug a `Pending` pod down to the missing CSI driver
- Explain why `sed` on a Kubernetes manifest is dangerous, and what to use instead
- Explain why Jenkins holds no cluster credentials in a GitOps setup

Each of those is a real interview question.

---

## 🚦 Start here

```bash
cd guides
```

Open **[00-prerequisites.md](guides/00-prerequisites.md)**.

---

<div align="center">

**Reference implementation:** [https://github.com/Waleeddarwesh/CloudDevOpsProject](https://github.com/Waleeddarwesh/CloudDevOpsProject) · **Progress tracker:** [PROGRESS.md](PROGRESS.md)

</div>
