# 📋 Progress Tracker

Tick these off as you go. Each checkpoint has a **verification command** — do not tick until yours matches.

> 💡 Commit after each ✅. Your history becomes a record of your own learning.

---

## Module 00 — Prerequisites

- [ ] AWS CLI v2 installed · `aws --version`
- [ ] Terraform **≥ 1.10** installed · `terraform version`
- [ ] Docker Desktop running · `docker ps`
- [ ] kubectl installed · `kubectl version --client`
- [ ] Helm installed · `helm version`
- [ ] Ansible installed (WSL2 on Windows) · `ansible --version`
- [ ] AWS credentials working · `aws sts get-caller-identity`
- [ ] **Budget alarm set at $25** ⚠️
- [ ] EC2 key pair created · `ls ~/.ssh/ivolve-key.pem`
- [ ] Public IP noted · `curl -s https://checkip.amazonaws.com`
- [ ] GitHub fine-grained PAT created (Contents: read+write)
- [ ] Application source cloned into `workspace/src/`

**Checkpoint:** all seven tools report a version.

---

## Module 01 — Docker  *(free, no AWS)*

- [ ] `src/frontend/Dockerfile` — multi-stage, non-root, healthcheck
- [ ] `src/auth-service/Dockerfile` — wheel build stage, gunicorn
- [ ] `src/roadmap-service/Dockerfile` — Maven build → JRE runtime
- [ ] Three `.dockerignore` files
- [ ] `01-Docker/docker-compose.yml`
- [ ] `01-Docker/.env.example` + your own `.env`
- [ ] All four images build · `docker compose build`
- [ ] Stack starts healthy · `docker compose ps`
- [ ] **Signed up and logged in at <http://localhost:3000>**
- [ ] Roadmap page renders 8 topics

**Checkpoint:**
```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}"
# all four: Up (healthy)
```

---

## Module 02 — Terraform  💰 *billing starts*

- [ ] `bootstrap/` — S3 state bucket, versioned + encrypted
- [ ] `versions.tf` · `providers.tf` · `backend.tf` · `variables.tf` · `locals.tf`
- [ ] `modules/network/` — VPC, subnets, IGW, NAT, route tables, NACLs, flow logs
- [ ] `modules/server/` — SG, IAM role, instance profile, EC2, EIP
- [ ] `modules/eks/` — cluster, node group, OIDC, IRSA ×3, add-ons, access entries
- [ ] `modules/ecr/` — 3 repositories, lifecycle policy
- [ ] `main.tf` wires all four modules
- [ ] `terraform fmt -recursive -check` clean
- [ ] `terraform validate` passes
- [ ] `terraform plan` shows ~80 resources, 0 errors
- [ ] `terraform apply` completes
- [ ] `kubectl get nodes` → **2 nodes Ready in different AZs**

**Checkpoint:**
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.'topology\.kubernetes\.io/zone'
# 2 rows, 2 different zones
```

---

## Module 03 — Ansible

- [ ] `ansible.cfg`
- [ ] `inventory/aws_ec2.yml` — dynamic inventory
- [ ] `requirements.yml` — collections
- [ ] `group_vars/all/main.yml`
- [ ] `group_vars/all/vault.yml` — **encrypted**
- [ ] `.vault_pass` — created, gitignored, saved to password manager
- [ ] Roles: `common` `java` `docker` `jenkins` `trivy` `aws_cli` `kubectl` `helm` `sonarqube`
- [ ] `playbook.yml` with pre_tasks + roles + post_tasks
- [ ] Instance discovered · `ansible-inventory --graph`
- [ ] `ansible role_jenkins -m ping` → pong
- [ ] Playbook completes
- [ ] **Second run reports `changed=0`** (idempotency proof)

**Checkpoint:**
```bash
ansible-playbook playbook.yml | tail -1
# ok=NN  changed=0  unreachable=0  failed=0
```

---

## Module 04 — Kubernetes

- [ ] `00-namespace.yaml` — namespace, ResourceQuota, LimitRange
- [ ] `01-rbac.yaml` — 4 ServiceAccounts, Roles, RoleBindings
- [ ] `02-config.yaml` — ConfigMap + Secret
- [ ] `03-storage.yaml` — StorageClass (`ebs.csi.aws.com`)
- [ ] `04-database.yaml` — StatefulSet + headless Service
- [ ] `05-auth-service.yaml` — Deployment (+ initContainer) + Service
- [ ] `06-roadmap-service.yaml` — Deployment + Service
- [ ] `07-frontend.yaml` — Deployment + Service + NodePort
- [ ] `08-ingress.yaml` — Ingress (`alb` class)
- [ ] `12-network-policies.yaml` — 8 policies, default-deny baseline
- [ ] `kustomization.yaml`
- [ ] AWS Load Balancer Controller installed via Helm
- [ ] `kubectl kustomize` renders 37+ resources
- [ ] All pods Running
- [ ] PVC **Bound** (proves the EBS CSI driver + IRSA work)
- [ ] **App reachable at the ALB address**

> 💡 The reference also has `09-jenkins.yaml`, `10-sonarqube.yaml`, and `11-argocd-ingress.yaml` — optional in-cluster deployments. See the reference's K8s README.

**Checkpoint:**
```bash
kubectl get pods,pvc -n ivolve
# 7 pods Running, 1 PVC Bound
```

---

## Module 05 — Jenkins

- [ ] Jenkins unlocked, admin user created
- [ ] Shared library registered as `shared-library`
- [ ] Credentials `github-token` and `sonar-token` added
- [ ] SonarQube admin password changed
- [ ] **SonarQube webhook → `http://localhost:8080/sonarqube-webhook/`** ⚠️
- [ ] `vars/dockerBuildImage.groovy`
- [ ] `vars/trivyScan.groovy`
- [ ] `vars/ecrPush.groovy`
- [ ] `vars/runUnitTests.groovy`
- [ ] `vars/sonarQubeScan.groovy`
- [ ] `vars/updateManifests.groovy`
- [ ] `vars/pushManifests.groovy`
- [ ] `vars/microservicePipeline.groovy` — 9 stages
- [ ] 3 Jenkinsfiles
- [ ] 3 pipeline jobs created
- [ ] **All three builds green**
- [ ] Images visible in ECR

**Checkpoint:**
```bash
aws ecr list-images --repository-name ivolve-frontend --query 'imageIds[*].imageTag'
# at least one <build>-<sha> tag
```

---

## Module 06 — ArgoCD

- [ ] ArgoCD installed in-cluster
- [ ] `project.yaml` — AppProject
- [ ] `applications/ivolve-app.yaml` — Application with sync waves
- [ ] Application shows **Synced / Healthy**
- [ ] **Self-heal proven:** delete a pod by hand, watch ArgoCD restore it

**Checkpoint:**
```bash
kubectl get application -n argocd
# SYNC STATUS: Synced   HEALTH STATUS: Healthy
```

---

## Module 07 — Monitoring

- [ ] `kube-prometheus-stack-values.yaml`
- [ ] Stack installed via Helm
- [ ] `manifests/01-blackbox-probes.yaml`
- [ ] Alert rules defined
- [ ] Grafana dashboard imported
- [ ] Prometheus targets all **UP**
- [ ] Grafana renders live data
- [ ] **Alert fired deliberately** (scale a deployment to 0 and watch)

**Checkpoint:**
```bash
kubectl get pods -n monitoring
# prometheus, grafana, alertmanager, node-exporter all Running
```

---

## 🎓 Final — the whole loop

- [ ] Change a line in `src/frontend/views/roadmap.ejs`
- [ ] `git push`
- [ ] Jenkins builds, scans, pushes to ECR
- [ ] Jenkins commits the manifest change
- [ ] ArgoCD detects and syncs
- [ ] **Your change is live in the browser** — with no `kubectl apply` anywhere

That is the entire point of the project. When it works end to end without you touching the cluster, you're done.

---

## 🧹 Teardown  ⚠️ do not skip

```bash
# 1 — Kubernetes FIRST. The ALB is owned by the Ingress controller, not
#     Terraform. Destroying the VPC while it exists leaves an orphaned ALB
#     and Terraform hangs ~20 min on DependencyViolation.
kubectl delete ingress --all -n ivolve
kubectl delete namespace ivolve argocd monitoring --ignore-not-found

# 2 — confirm the load balancer is actually gone
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'

# 3 — now destroy
cd workspace/02-Terraform && terraform destroy

# 4 — clean up what Terraform doesn't own
aws ec2 delete-key-pair --key-name ivolve-key
```

- [ ] Ingress deleted
- [ ] Namespaces deleted
- [ ] `terraform destroy` completed
- [ ] `aws eks list-clusters` → empty
- [ ] Key pair deleted
- [ ] **Billing dashboard checked**
