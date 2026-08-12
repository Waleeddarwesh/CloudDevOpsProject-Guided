# 🚨 Error Decoder

Every error these guides can produce, with the real cause. **Search this file before searching the web** — the top Stack Overflow answer is often for a different cause of the same message.

---

## 📋 Quick index

| Symptom | Module | Jump |
|---|:---:|---|
| `Unsupported argument: use_lockfile` | 02 | [→](#tf-lockfile) |
| `Refusing to open SSH to the entire internet` | 02 | [→](#tf-ssh-guard) |
| `Cycle: module.server, module.eks` | 02 | [→](#tf-cycle) |
| `Invalid count argument` | 02 | [→](#tf-count) |
| `You must be logged in to the server` | 02 | [→](#eks-unauthorized) |
| `skipping: no hosts matched` | 03 | [→](#ansible-nohosts) |
| `Could not get lock /var/lib/dpkg/lock-frontend` | 03 | [→](#ansible-aptlock) |
| Jenkins service won't start | 03 | [→](#jenkins-nostart) |
| PVC stuck `Pending` | 04 | [→](#pvc-pending) |
| `volume node affinity conflict` | 04 | [→](#volume-affinity) |
| Ingress has no `ADDRESS` | 04 | [→](#ingress-noaddress) |
| Ingress returns `503` | 04 | [→](#ingress-503) |
| Pod `CreateContainerConfigError` | 04 | [→](#runasnonroot) |
| Pod `ImagePullBackOff` | 04 | [→](#imagepull) |
| auth-service `Access denied` | 04 | [→](#mysql-denied) |
| Everything breaks after NetworkPolicy | 04 | [→](#netpol-dns) |
| Pod `Running` but Service returns nothing | 04 | [→](#probe-wrong-path) |
| Build hangs at SonarQube | 05 | [→](#sonar-hang) |
| `CredentialNotFoundException` | 05 | [→](#jenkins-creds) |
| `unable to resolve class` | 05 | [→](#jenkins-library) |
| Infinite build loop | 05 | [→](#skipci) |
| `non-fast-forward` push rejected | 05 | [→](#git-race) |
| ArgoCD `ComparisonError` on StorageClass | 06 | [→](#argo-clusterscope) |
| ArgoCD oscillates on replicas | 06 | [→](#argo-hpa) |
| Prometheus target never appears | 07 | [→](#prom-selector) |
| Prometheus loses data on restart | 07 | [→](#prom-storage) |
| `terraform destroy` hangs | — | [→](#destroy-hang) |

---

# Terraform

<a id="tf-lockfile"></a>

## `Unsupported argument: use_lockfile`

```text
Error: Unsupported argument
  on backend.tf line 8, in terraform:
   8:     use_lockfile = true
```

**Cause:** Terraform < 1.10. S3 native state locking was added in 1.10.

```bash
terraform version
```

**Fix:** upgrade. Or, as a stopgap, remove the line and add a DynamoDB lock table (the legacy approach, deprecated in 1.11).

---

<a id="tf-ssh-guard"></a>

## `Refusing to open SSH (port 22) to the entire internet`

**This is not a bug.** It's the validation block you wrote, doing its job.

**Fix:**
```bash
curl -s https://checkip.amazonaws.com
```
```hcl
allowed_ssh_cidrs = ["203.0.113.9/32"]
```

> ⚠️ Do not "fix" this by deleting the validation block. An SSH port open to `0.0.0.0/0` is found by scanners within minutes.

---

<a id="tf-cycle"></a>

## `Cycle: module.server, module.eks`

```text
Error: Cycle: module.server.var.eks_cluster_arn, module.eks.var.jenkins_role_arn
```

**Cause:** each module references an output of the other.

```hcl
module "server" { eks_cluster_arn  = module.eks.cluster_arn }      # server → eks
module "eks"    { jenkins_role_arn = module.server.iam_role_arn }  # eks → server
```

**Fix:** break it with a value known before either runs. Pass the cluster *name* from `locals` and rebuild the ARN inside the server module:

```hcl
module "server" { eks_cluster_name = local.cluster_name }
```
```hcl
resources = ["arn:aws:eks:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:cluster/${var.eks_cluster_name}"]
```

---

<a id="tf-count"></a>

## `Invalid count argument`

```text
Error: Invalid count argument
The "count" value depends on resource attributes that cannot be determined
until apply.
```

**Cause:** `count` must resolve at **plan** time, but you used a value that is `(known after apply)`:

```hcl
count = var.jenkins_role_arn != null ? 1 : 0    # ARN unknown until apply
```

**Fix:** gate on a plain bool instead.

```hcl
variable "create_jenkins_access_entry" { type = bool, default = true }
count = var.create_jenkins_access_entry ? 1 : 0
```

The unknown ARN is fine as an *attribute value* — only `count` is constrained.

---

<a id="eks-unauthorized"></a>

## `error: You must be logged in to the server (Unauthorized)`

**Cause:** authenticating to EKS is **two** steps and people conflate them:

1. **IAM** decides whether you may *call* the endpoint
2. **Kubernetes RBAC** decides what you may *do*

Having `AdministratorAccess` satisfies step 1 and gives you **nothing** in step 2.

**Fix:** ensure the cluster has

```hcl
access_config {
  bootstrap_cluster_creator_admin_permissions = true
}
```

For a cluster already created, add an access entry:

```bash
aws eks create-access-entry --cluster-name <c> --principal-arn <your-arn>
aws eks associate-access-policy --cluster-name <c> --principal-arn <your-arn> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

---

# Ansible

<a id="ansible-nohosts"></a>

## `skipping: no hosts matched`

Three causes, in order of likelihood:

**1. Plugin not enabled.** `ansible.cfg` needs:
```ini
[inventory]
enable_plugins = amazon.aws.aws_ec2, ini, yaml
```

**2. Wrong filename.** Must end in `aws_ec2.yml` / `aws_ec2.yaml`. `inventory.yml` is silently ignored.

**3. Tag mismatch.**
```bash
aws ec2 describe-instances --filters "Name=tag:Role,Values=jenkins" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]'
```

**Diagnose:**
```bash
ansible-inventory --graph --flush-cache
ansible-inventory --list -vvv        # shows the plugin's own errors
```

---

<a id="ansible-aptlock"></a>

## `Could not get lock /var/lib/dpkg/lock-frontend`

**Cause:** Terraform's `user_data` and Ansible are both running `apt-get`.

**Fix:** the playbook must wait for the bootstrap marker:

```yaml
- name: Wait for the Terraform bootstrap to finish
  ansible.builtin.wait_for:
    path: /etc/ivolve-bootstrap-complete
    timeout: 600
```

---

<a id="jenkins-nostart"></a>

## Jenkins installs but won't start

```text
jenkins.service: Failed with result 'exit-code'
```

`journalctl -u jenkins` shows nothing useful.

**Cause:** no JVM. The Jenkins `.deb` declares **no JRE dependency**.

**Fix:** `java` must run before `jenkins`:

```yaml
# roles/jenkins/meta/main.yml
dependencies:
  - role: java
  - role: docker
```

---

# Kubernetes

<a id="pvc-pending"></a>

## PVC stuck `Pending`

```bash
kubectl describe pvc data-mysql-0 -n ivolve
```

| Event says | Cause | Fix |
|---|---|---|
| `no persistent volumes available` + provisioner `kubernetes.io/aws-ebs` | **legacy in-tree provisioner**, removed in K8s 1.27+ | use `ebs.csi.aws.com` |
| `waiting for first consumer` | normal with `WaitForFirstConsumer` | not an error — check the *pod* |
| `failed to provision volume` / `AccessDenied` | EBS CSI driver missing or IRSA wrong | below |

```bash
kubectl get pods -n kube-system | grep ebs-csi
kubectl logs -n kube-system deploy/ebs-csi-controller -c csi-provisioner
kubectl get sa ebs-csi-controller-sa -n kube-system -o yaml | grep role-arn
```

The annotation must match the IRSA role, and its trust policy must name **exactly** `system:serviceaccount:kube-system:ebs-csi-controller-sa`.

---

<a id="volume-affinity"></a>

## `volume node affinity conflict`

**Cause:** the EBS volume was created in one AZ, the pod scheduled in another. **EBS cannot cross AZs.**

Happens with `volumeBindingMode: Immediate` on a multi-AZ cluster.

**Fix:**
```yaml
volumeBindingMode: WaitForFirstConsumer
```

Delete the PVC and let it be recreated. Existing volumes cannot be moved.

---

<a id="ingress-noaddress"></a>

## Ingress has no `ADDRESS`

```bash
kubectl get ingress -n ivolve      # ADDRESS column empty forever
```

**Checklist:**

**1. Is the controller running?**
```bash
kubectl get deploy -n kube-system aws-load-balancer-controller
```
Missing → nothing reconciles the Ingress. Install it.

**2. Are you using the deprecated annotation?**
```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: alb    # ❌ IGNORED by modern controllers
spec:
  ingressClassName: alb                 # ✅
```

**3. Subnet tags missing?**
```bash
kubectl logs -n kube-system deploy/aws-load-balancer-controller | grep -i subnet
```
```text
couldn't auto-discover subnets: unable to resolve at least one subnet
```
Public subnets need `kubernetes.io/role/elb = 1` **and** `kubernetes.io/cluster/<name> = shared`.

**4. IRSA wrong?**
```bash
kubectl logs -n kube-system deploy/aws-load-balancer-controller | grep -i accessdenied
```
The ServiceAccount must be named exactly `aws-load-balancer-controller`.

---

<a id="ingress-503"></a>

## Ingress returns `503`

**Usually just impatience.** Target-group health checks take 2–3 minutes after the ADDRESS appears.

If it persists:
```bash
kubectl get endpoints -n ivolve frontend
```
Empty endpoints → **no pod is passing its readiness probe**. Check [probe path](#probe-wrong-path).

---

<a id="runasnonroot"></a>

## `CreateContainerConfigError` / `container has runAsNonRoot and image will run as root`

**Cause:** your `securityContext` demands non-root, but the image's default user is root.

**Fix:** in the Dockerfile —
```dockerfile
RUN adduser --system --uid 10001 appuser
USER appuser
```

And make the manifest's `runAsUser` match the Dockerfile's UID.

---

<a id="imagepull"></a>

## `ImagePullBackOff`

```bash
kubectl describe pod <pod> -n ivolve | tail -20
```

| Message | Cause |
|---|---|
| `manifest unknown` / `not found` | tag doesn't exist in ECR — has the pipeline run? |
| `no basic auth credentials` | nodes lack `AmazonEC2ContainerRegistryReadOnly` |
| `pull access denied` | wrong account ID in `kustomization.yaml` |

```bash
aws ecr list-images --repository-name ivolve-frontend
kubectl get deploy frontend -n ivolve -o jsonpath='{..image}'
```

---

<a id="mysql-denied"></a>

## auth-service: `Access denied for user 'ivolve_user'`

MySQL is `Running` and healthy. auth-service crash-loops.

**Cause:** `MYSQL_PASSWORD` ≠ `DB_PASSWORD` in your Secret. They are the same credential from two sides — MySQL *creates* the account with one, auth-service *authenticates* with the other.

```bash
kubectl get secret ivolve-secret -n ivolve -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d; echo
kubectl get secret ivolve-secret -n ivolve -o jsonpath='{.data.DB_PASSWORD}'    | base64 -d; echo
```

> ⚠️ **Changing the Secret is not enough.** MySQL only reads `MYSQL_PASSWORD` on **first boot**, when it initialises the data directory. You must delete the PVC:
> ```bash
> kubectl delete statefulset mysql -n ivolve
> kubectl delete pvc data-mysql-0 -n ivolve
> ```

Also check the variable name is `DB_USER`, not `DB_USERNAME`.

---

<a id="netpol-dns"></a>

## Everything breaks right after applying NetworkPolicies

Symptoms are wildly varied: `getaddrinfo ENOTFOUND`, connection timeouts, "service unavailable".

**Cause:** `default-deny-all` blocks **DNS**, so no service name resolves anywhere.

**Fix — you must add this:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: ivolve
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

---

<a id="probe-wrong-path"></a>

## Pod is `Running` but the Service returns nothing

```bash
kubectl get endpoints -n ivolve roadmap-service     # <none>
```

**Cause:** the readiness probe is failing, so the pod is excluded from Service endpoints. `Running` only means the process exists.

```bash
kubectl describe pod <pod> -n ivolve | grep -A3 Readiness
```

```text
Readiness probe failed: HTTP probe failed with statuscode: 404
```

**The classic instance:** probing `/health` on **roadmap-service**, which only has `/api/roadmap`.

```yaml
readinessProbe:
  httpGet:
    path: /api/roadmap      # ← not /health
```

---

# Jenkins

<a id="sonar-hang"></a>

## Build hangs at "SonarQube Analysis" then fails after 10 minutes

**Cause:** the **webhook is missing**. This is the single most common failure in the whole project.

The scanner uploads asynchronously and exits. `waitForQualityGate()` waits for SonarQube to *call back*. No webhook → no callback → timeout.

**Fix:** SonarQube → *Administration → Configuration → Webhooks → Create*

| Field | Value |
|---|---|
| Name | `Jenkins` |
| URL | `http://localhost:8080/sonarqube-webhook/` |

> ⚠️ The trailing slash matters.

---

<a id="jenkins-creds"></a>

## `CredentialNotFoundException`

```text
ERROR: Could not find credentials entry with ID 'github-token'
```

**Cause:** the ID in Jenkins doesn't match the string in the library. Note it fails **at runtime**, not at config time.

```groovy
credentialsId: 'github-token'    // must match EXACTLY
```

Check *Manage Jenkins → Credentials* — the ID field, not the description.

---

<a id="jenkins-library"></a>

## `unable to resolve class` / `No such DSL method`

**Cause 1 — the missing underscore:**

```groovy
@Library('shared-library')       // ❌
@Library('shared-library') _     // ✅
```

The annotation must attach to a statement; `_` is the conventional no-op.

**Cause 2 — `vars/` not at the repo root.** Jenkins cannot load `05-Jenkins/vars/`. The library needs its own repo with `vars/` at the top level.

**Cause 3 — library name mismatch.** The string in `@Library(...)` must equal the Name field in *Global Pipeline Libraries*.

---

<a id="skipci"></a>

## Builds trigger endlessly

Every build pushes a commit, which triggers another build, forever — until someone notices the executor is permanently busy.

**Fix:** `[skip ci]` in the commit message:

```groovy
git commit -m "ci(${imageName}): deploy ${newImage}

[skip ci]"
```

---

<a id="git-race"></a>

## `Updates were rejected because the remote contains work that you do not have`

**Cause:** two pipelines finished at once and both pushed to `main`.

**Fix:** rebase and retry.

```groovy
retry(3) {
    sh '''
        git pull --rebase origin main || true
        git push https://${GIT_USERNAME}:${GIT_TOKEN}@...
    '''
}
```

Plus `disableConcurrentBuilds()` in pipeline options.

---

# ArgoCD

<a id="argo-clusterscope"></a>

## `ComparisonError: StorageClass is not permitted in project`

**Cause:** `StorageClass` is **cluster-scoped**, and AppProjects deny cluster-scoped resources by default.

**Fix:**
```yaml
spec:
  clusterResourceWhitelist:
    - group: storage.k8s.io
      kind: StorageClass
```

---

<a id="argo-hpa"></a>

## ArgoCD and the HPA fight over replicas

Sync status flips between `Synced` and `OutOfSync` every few minutes.

**Cause:** the HPA scales the Deployment; ArgoCD resets it to the Git value; the HPA scales it again.

**Fix:**
```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

---

# Monitoring

<a id="prom-selector"></a>

## ServiceMonitor/Probe created, but the target never appears

No error anywhere. The target simply doesn't exist.

**Cause:** the Operator only discovers monitors carrying the Helm release label.

**Fix — in your values file:**
```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
```

Or add `labels: { release: kube-prometheus-stack }` to each monitor.

---

<a id="prom-storage"></a>

## Prometheus loses all history on restart

**Cause:** no `volumeClaimTemplate` → data lives in an `emptyDir`, which is destroyed with the pod.

**Fix:**
```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ivolve-storage
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 10Gi } }
```

---

# Teardown

<a id="destroy-hang"></a>

## `terraform destroy` hangs on the VPC (~20 min, then `DependencyViolation`)

```text
Error: deleting EC2 Subnet: DependencyViolation:
The subnet has dependencies and cannot be deleted.
```

**Cause:** the ALB was created by the Load Balancer Controller, **not Terraform**. Terraform doesn't know it exists and can't delete it — but it holds ENIs in your subnets.

**Fix — always delete Kubernetes resources first:**

```bash
kubectl delete ingress --all -n ivolve
kubectl delete svc --all-namespaces --field-selector spec.type=LoadBalancer

# confirm it's gone before continuing
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'

terraform destroy
```

**Already stuck?** Delete the ALB by hand, then re-run destroy:

```bash
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerArn'
aws elbv2 delete-load-balancer --load-balancer-arn <arn>
```

---

<div align="center">

**Still stuck?** Diff your file against [the reference implementation](https://github.com/Waleeddarwesh/CloudDevOpsProject) — every file there is commented with *why*.

</div>
