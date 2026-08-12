# 🔄 Module 06 — ArgoCD

**Time:** ~2 hours

---

## 🎯 The idea

Module 05 ended with Jenkins **committing to Git**. It never ran `kubectl apply`.

That's the whole point of GitOps:

```text
PUSH model (traditional CI/CD)          PULL model (GitOps)
────────────────────────────────        ─────────────────────────────
Jenkins holds cluster credentials       Jenkins holds NO cluster creds
Jenkins runs `kubectl apply`            Jenkins commits to Git
Cluster state = whatever ran last       Cluster state = whatever Git says
Rollback = re-run an old build          Rollback = `git revert`
Drift is invisible                      Drift is detected and corrected
Audit trail = Jenkins logs              Audit trail = git log
```

**A compromised Jenkins in the pull model can push an image and open a commit — both of which leave a trail. It cannot silently change the cluster.**

---

# Part 1 · Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server
```

Get the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Access the UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080  ·  user: admin
```

> 💡 Self-signed certificate — your browser will warn. Expected for a lab.

---

# Part 2 · The AppProject

A **Project** is a security boundary: which repos an app may deploy *from*, which clusters/namespaces it may deploy *to*, and which resource kinds it may create.

`workspace/06-ArgoCD/project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ivolve
  namespace: argocd
spec:
  description: iVolve DevOps Roadmap platform

  # Only these repos. A compromised ArgoCD cannot be pointed at an
  # attacker's repository.
  sourceRepos:
    - https://github.com/YourUser/CloudDevOpsProject.git

  # Only these destinations.
  destinations:
    - server: https://kubernetes.default.svc
      namespace: ivolve

  # ⚠️ Cluster-scoped resources this project may create.
  # The StorageClass in Module 04 is cluster-scoped, so it must be listed —
  # otherwise the sync fails with a permission error that looks like RBAC.
  clusterResourceWhitelist:
    - group: storage.k8s.io
      kind: StorageClass

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

---

# Part 3 · The Application

`workspace/06-ArgoCD/applications/ivolve-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ivolve
  namespace: argocd
  finalizers:
    # Ensures `kubectl delete application ivolve` also removes the
    # deployed resources. Without it, deleting the Application orphans
    # everything it created.
    - resources-finalizer.argocd.argoproj.io
spec:
  project: ivolve

  source:
    repoURL: https://github.com/YourUser/CloudDevOpsProject.git
    targetRevision: HEAD
    # ArgoCD auto-detects kustomization.yaml and renders with Kustomize.
    path: 04-Kubernetes/manifests

  destination:
    server: https://kubernetes.default.svc
    namespace: ivolve

  syncPolicy:
    automated:
      # prune: delete resources removed from Git.
      # Without it, deleting a manifest leaves the object running forever —
      # Git and reality silently diverge, which is the exact drift GitOps
      # is supposed to prevent.
      prune: true

      # selfHeal: revert manual changes made with kubectl.
      # Someone runs `kubectl scale deploy/frontend --replicas=10` at 2am
      # to survive an incident. Without selfHeal that stays forever,
      # undocumented. With it, Git wins and the fix must go through review.
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true

    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m

  # Ignore fields that other controllers legitimately own, so ArgoCD
  # doesn't fight them in an endless sync loop.
  ignoreDifferences:
    - group: apps
      kind: Deployment
      # The HPA owns replicas. Without this, ArgoCD resets replicas to the
      # Git value, the HPA scales it back, and they oscillate forever.
      jsonPointers:
        - /spec/replicas
```

---

# Part 4 · Sync waves — ordering

Kubernetes has no built-in ordering. Applied all at once, auth-service starts before MySQL exists and crash-loops.

Add annotations to your Module 04 manifests:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

| Wave | Resources | Why |
|:---:|---|---|
| `-1` | Namespace, ResourceQuota, LimitRange, RBAC | must exist before anything in them |
| `0` | ConfigMap, Secret, StorageClass | workloads reference these |
| `1` | MySQL StatefulSet + headless Service | the dependency everything waits on |
| `2` | auth-service, roadmap-service | need the database |
| `3` | frontend | needs both backends |
| `4` | Ingress | needs frontend's Service |

ArgoCD applies each wave and **waits for it to become Healthy** before the next.

> 💡 The initContainer from Module 04 and sync waves are belt-and-braces. Waves order the *apply*; the initContainer handles the case where MySQL is applied but still initialising.

---

# ✅ Checkpoint

```bash
kubectl apply -f workspace/06-ArgoCD/project.yaml
kubectl apply -f workspace/06-ArgoCD/applications/ivolve-app.yaml

kubectl get application -n argocd -w
```

```text
NAME     SYNC STATUS   HEALTH STATUS
ivolve   Synced        Healthy
```

## 🔨 Prove self-heal works

This is the demo that makes GitOps click.

```bash
# Terminal B
kubectl get deploy frontend -n ivolve -w
```

```bash
# Terminal A — simulate a 2am manual "fix"
kubectl scale deployment frontend -n ivolve --replicas=7
```

Watch Terminal B: replicas jump to 7... then **ArgoCD pulls it back to 2** within ~3 minutes.

**Git is the source of truth.** Manual changes are reverted automatically. There is no such thing as undocumented drift.

```bash
kubectl delete deployment frontend -n ivolve
# ...and watch ArgoCD recreate it
```

## Prove the full loop

```bash
# Change something visible
vim workspace/src/frontend/views/roadmap.ejs
git add . && git commit -m "feat: update roadmap heading" && git push
```

Then watch:

1. Jenkins builds → tests → Sonar → scans → pushes to ECR
2. Jenkins commits the new image tag to `kustomization.yaml`
3. ArgoCD detects the commit
4. ArgoCD syncs → rolling update
5. **Your change is live**

**You never ran `kubectl apply`.** That's the deliverable.

---

## 🧠 Before you continue

1. What does Jenkins have that ArgoCD doesn't, and vice versa?
2. What does `prune: true` prevent?
3. What does `selfHeal: true` prevent?
4. Why must `StorageClass` be in `clusterResourceWhitelist`?
5. Why ignore `/spec/replicas` on Deployments?
6. What problem do sync waves solve?
7. How do you roll back a bad deploy in this model?

---

<div align="center">

**Next → [Module 07 — Monitoring](07-monitoring.md)**

</div>
