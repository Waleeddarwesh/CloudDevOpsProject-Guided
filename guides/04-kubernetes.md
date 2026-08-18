# ☸️ Module 04 — Kubernetes

**Time:** ~6 hours · **Cost:** the ALB adds ~$17/mo while it exists

---

## 🎯 What you'll build

37+ objects, layered from the ground up:

```text
00-namespace.yaml         Namespace · ResourceQuota · LimitRange
01-rbac.yaml              4 ServiceAccounts · Roles · RoleBindings
02-config.yaml            ConfigMap · Secret
03-storage.yaml           StorageClass  (ebs.csi.aws.com)
04-database.yaml          StatefulSet · headless Service
05-auth-service.yaml      Deployment (+ initContainer) · Service
06-roadmap-service.yaml   Deployment · Service
07-frontend.yaml          Deployment · Service · NodePort
08-ingress.yaml           Ingress  (alb class)
12-network-policies.yaml  8 policies, default-deny baseline
kustomization.yaml
```

> 💡 **The reference implementation has 3 additional manifests** (`09-jenkins.yaml`, `10-sonarqube.yaml`, `11-argocd-ingress.yaml`) that deploy Jenkins and SonarQube *inside the cluster* and expose ArgoCD via an ALB Ingress. These are optional — the core learning path uses Jenkins on EC2 (Module 03) and SonarQube via Docker on the same EC2. Explore them after completing the core modules if you want to see in-cluster CI/CD.

**Build them in order and apply as you go.** Each layer depends on the one before, and applying incrementally means you see exactly which piece breaks.

---

# Part 0 · Install the Load Balancer Controller first

An `Ingress` object **does nothing on its own.** It's an inert record until a controller watches it and provisions real infrastructure.

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

> ⚠️ **The ServiceAccount name must be exactly `aws-load-balancer-controller`.** The IRSA trust policy you wrote in Module 02 only accepts tokens from that exact `namespace:serviceaccount` pair. A different name → `AccessDenied` → no ALB, with the error buried in controller logs.

---

# Part 1 · Namespace and governance

`workspace/04-Kubernetes/manifests/00-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ivolve
  labels:
    # Pod Security Admission. `restricted` is the strictest built-in
    # profile: no privileged containers, no host namespaces, must run
    # as non-root, must drop ALL capabilities.
    #
    # Your Dockerfiles from Module 01 already comply — which is exactly
    # why hardening them there pays off here.
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ivolve-quota
  namespace: ivolve
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "4"
    pods: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ivolve-limits
  namespace: ivolve
spec:
  limits:
    - type: Container
      # Applied to any container that doesn't specify its own.
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
```

> ⚠️ **A ResourceQuota has a side effect people trip over:** once a namespace has a CPU/memory quota, **every** container must declare requests and limits. A pod without them is rejected with `must specify limits.cpu`. The LimitRange supplies defaults so that doesn't happen.

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl describe quota -n ivolve
```

---

# Part 2 · Config and secrets

`02-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ivolve-config
  namespace: ivolve
data:
  # ⚠️ Read from the SOURCE, not from a tutorial:
  #   grep -n "os.environ" src/auth-service/app.py
  # It is DB_USER — not DB_USERNAME.

  # Stable DNS name of the StatefulSet pod.
  # Format: <pod>.<headless-service>.<namespace>.svc.cluster.local
  DB_HOST: "mysql-0.mysql.ivolve.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "ivolve"
  DB_USER: "ivolve_user"

  # Kubernetes DNS lets you use the short name inside the same namespace.
  AUTH_SERVICE_URL: "http://auth-service:5000"
  ROADMAP_SERVICE_URL: "http://roadmap-service:8080"
---
apiVersion: v1
kind: Secret
metadata:
  name: ivolve-secret
  namespace: ivolve
type: Opaque
stringData:
  # stringData takes plaintext and base64-encodes it for you.
  # `data` would require you to encode by hand.
  MYSQL_ROOT_PASSWORD: "CHANGE_ME_root"

  # ⚠️ THESE TWO MUST MATCH. They are the same credential seen from two
  # sides: MySQL creates the account with MYSQL_PASSWORD, auth-service
  # authenticates with DB_PASSWORD. Mismatch → MySQL starts fine and
  # auth-service dies with "Access denied", which looks like a bug in the app.
  MYSQL_PASSWORD: "CHANGE_ME_app"
  DB_PASSWORD: "CHANGE_ME_app"

  SESSION_SECRET: "CHANGE_ME_session"
```

> ⚠️ **A plaintext Secret in Git is a known compromise**, made here so the project is reviewable. Kubernetes Secrets are only **base64-encoded**, not encrypted — `echo <value> | base64 -d` reads them.
>
> Production answer: **Sealed Secrets** or **External Secrets Operator**. Say this out loud if you present the project — knowing the limitation is worth more than pretending it isn't there.

---

# Part 3 · Storage

`03-storage.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ivolve-storage
provisioner: ebs.csi.aws.com
# ⚠️ NOT kubernetes.io/aws-ebs.
#
# That is the legacy IN-TREE provisioner, removed from Kubernetes 1.27+.
# Many tutorials still show it. Using it gives you a PVC stuck Pending
# forever with no useful event — one of the most frustrating EKS failures
# because everything *looks* correct.

parameters:
  type: gp3
  encrypted: "true"

reclaimPolicy: Retain
# Retain keeps the EBS volume when the PVC is deleted. `Delete` would
# destroy your database the moment someone removes the namespace.

allowVolumeExpansion: true

volumeBindingMode: WaitForFirstConsumer
# ⚠️ CRITICAL on multi-AZ clusters.
#
# `Immediate` creates the volume as soon as the PVC exists — possibly in
# us-east-1a — and then the scheduler may place the pod in us-east-1b.
# EBS volumes CANNOT cross AZs, so the pod is unschedulable forever:
#   "volume node affinity conflict"
#
# WaitForFirstConsumer delays creation until the pod is scheduled, then
# creates the volume in the right AZ.
```

```bash
kubectl apply -f manifests/03-storage.yaml
kubectl get storageclass
```

---

# Part 4 · The database (StatefulSet)

## Why not a Deployment?

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | random (`mysql-7d9f-x2k`) | **stable** (`mysql-0`) |
| Storage | all replicas share one PVC | **each gets its own** |
| DNS | none per-pod | `mysql-0.mysql.ns.svc...` |

A database needs a **stable identity** — writes must reach a specific instance.

`04-database.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: ivolve
spec:
  # HEADLESS SERVICE. clusterIP: None means no load balancer and no virtual
  # IP — DNS returns the POD's IP directly.
  #
  # This is what gives mysql-0 a stable DNS name. A normal ClusterIP would
  # round-robin across replicas, which is wrong for a database.
  clusterIP: None
  ports:
    - port: 3306
      name: mysql
  selector:
    app.kubernetes.io/name: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: ivolve
spec:
  # Links the StatefulSet to the headless Service above — this is what
  # creates the per-pod DNS records.
  serviceName: mysql

  # ⚠️ ONE replica, deliberately.
  #
  # MySQL does not replicate by raising this number. A second pod would
  # start as an INDEPENDENT, EMPTY database — not a replica. Real HA needs
  # an operator or a bootstrap sidecar configuring replication.
  # Setting this to 2 silently produces two divergent databases.
  replicas: 1

  selector:
    matchLabels:
      app.kubernetes.io/name: mysql

  template:
    metadata:
      labels:
        app.kubernetes.io/name: mysql
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999          # the mysql user in the official image
        runAsGroup: 999
        fsGroup: 999
        # fsGroup makes the kubelet chown the mounted volume to this group.
        # Without it the non-root process cannot write to its own PVC.
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql

          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef: { name: ivolve-secret, key: MYSQL_ROOT_PASSWORD }
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef: { name: ivolve-config, key: DB_NAME }
            - name: MYSQL_USER
              valueFrom:
                configMapKeyRef: { name: ivolve-config, key: DB_USER }
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef: { name: ivolve-secret, key: MYSQL_PASSWORD }

          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql

          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits:   { cpu: "1",  memory: 1Gi }

          # Three probes, three different jobs:
          #
          # startupProbe   — "has it finished booting?"  Disables the other
          #                  two until it passes. MySQL's first boot
          #                  initialises the data dir and takes minutes;
          #                  without this the liveness probe kills it
          #                  mid-initialisation, forever.
          # readinessProbe — "should it receive traffic?"  Failing removes
          #                  the pod from Service endpoints.
          # livenessProbe  — "is it wedged?"  Failing RESTARTS the container.
          startupProbe:
            exec:
              command: ["sh", "-c", "mysqladmin ping -h 127.0.0.1 -u root -p\"$MYSQL_ROOT_PASSWORD\""]
            failureThreshold: 30
            periodSeconds: 10        # allows up to 5 minutes

          readinessProbe:
            exec:
              command: ["sh", "-c", "mysqladmin ping -h 127.0.0.1 -u root -p\"$MYSQL_ROOT_PASSWORD\""]
            periodSeconds: 10

          livenessProbe:
            tcpSocket: { port: 3306 }
            periodSeconds: 20

          securityContext:
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
            # MySQL writes to /var/run and /tmp, so a read-only root FS
            # needs extra emptyDir mounts. Left false for simplicity.
            readOnlyRootFilesystem: false

  # volumeClaimTemplates — NOT a plain `volumes:` entry.
  # Each replica gets its OWN PVC, named <template>-<pod>: data-mysql-0.
  # Deleting the StatefulSet does NOT delete these — deliberate.
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ivolve-storage
        resources:
          requests:
            storage: 10Gi
```

```bash
kubectl apply -f manifests/02-config.yaml -f manifests/04-database.yaml
kubectl get pvc,pod -n ivolve -w
```

## ✅ The moment of truth

```bash
kubectl get pvc -n ivolve
```

```text
NAME           STATUS   VOLUME        CAPACITY   STORAGECLASS
data-mysql-0   Bound    pvc-a1b2...   10Gi       ivolve-storage
```

**`Bound` proves the entire Module 02 IRSA chain works** — OIDC provider → trust policy → EBS CSI driver → EC2 CreateVolume.

> ⚠️ **Stuck `Pending`?**
> ```bash
> kubectl describe pvc data-mysql-0 -n ivolve
> kubectl get pods -n kube-system | grep ebs-csi
> ```
> Almost always: CSI driver add-on missing, or the IRSA annotation is wrong.

---

# Part 5 · The application services

## auth-service — with an initContainer

`05-auth-service.yaml` (key parts):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: ivolve
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: auth-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: auth-service
    spec:
      serviceAccountName: auth-service

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001          # matches the Dockerfile's appuser
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile: { type: RuntimeDefault }

      # initContainers run to COMPLETION, in order, before any app
      # container starts. This is the Kubernetes equivalent of Compose's
      # `depends_on: condition: service_healthy`.
      initContainers:
        - name: wait-for-mysql
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z mysql-0.mysql.ivolve.svc.cluster.local 3306; do
                echo "waiting for mysql..."
                sleep 3
              done
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }

      containers:
        - name: auth-service
          image: ivolve-auth-service      # kustomize rewrites this
          ports:
            - containerPort: 5000
              name: http

          # envFrom pulls in EVERY key, saving 5 explicit blocks.
          # The app reads DB_HOST/DB_PORT/DB_NAME/DB_USER from the
          # ConfigMap and DB_PASSWORD from the Secret.
          envFrom:
            - configMapRef: { name: ivolve-config }
            - secretRef:    { name: ivolve-secret }

          readinessProbe:
            httpGet: { path: /health, port: http }
            initialDelaySeconds: 10
            periodSeconds: 10

          livenessProbe:
            httpGet: { path: /health, port: http }
            initialDelaySeconds: 30
            periodSeconds: 20

          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }

          securityContext:
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
            readOnlyRootFilesystem: true
```

> 💡 **`readOnlyRootFilesystem: true` works here only because Module 01 set `PYTHONDONTWRITEBYTECODE=1`.** Without it Python writes `.pyc` files and the container crashes on a read-only filesystem. Decisions compound.

## roadmap-service — the probe trap

```yaml
          # ⚠️ /api/roadmap, NOT /health.
          #
          # This service has exactly one route. Pointing probes at /health
          # gives 404 → readiness never passes → the pod never enters
          # Service endpoints → the frontend gets "service unavailable",
          # while the pod itself looks Running.
          readinessProbe:
            httpGet: { path: /api/roadmap, port: http }
            initialDelaySeconds: 20

          livenessProbe:
            httpGet: { path: /api/roadmap, port: http }
            initialDelaySeconds: 60      # JVM + Spring startup is slow
```

Verify for yourself before writing it:

```bash
grep -rn "GetMapping" ../src/roadmap-service/src/main/java/
```

---

# Part 6 · Ingress

`08-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: ivolve
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing

    # target-type: ip sends traffic straight to POD IPs.
    # `instance` mode would route via NodePort — an extra hop, and it
    # requires the nodes to be in the ALB's subnets.
    alb.ingress.kubernetes.io/target-type: ip

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  # ⚠️ spec.ingressClassName — NOT the old
  # `kubernetes.io/ingress.class` ANNOTATION.
  #
  # The annotation is deprecated and silently IGNORED by newer controllers.
  # The symptom: an Ingress that is never reconciled, with no error and no
  # ADDRESS, forever.
  ingressClassName: alb

  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend      # ← the ONLY externally reachable service
                port:
                  name: http
```

```bash
kubectl apply -f manifests/
kubectl get ingress -n ivolve -w
```

Wait for `ADDRESS` to appear (~2 min), then **wait another 2–3 minutes** before browsing — ALB target-group health checks take that long. A `503` before then is expected.

---

# Part 7 · Network policies

`09-network-policies.yaml` — start with deny-all:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ivolve
spec:
  # An EMPTY podSelector selects EVERY pod in the namespace.
  podSelector: {}
  policyTypes: [Ingress, Egress]
  # No ingress/egress rules = deny everything.
---
# ⚠️ ADD THIS OR NOTHING WORKS.
# default-deny blocks DNS too, so every service name fails to resolve and
# you get errors that look like anything except a network policy.
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
---
# Only auth-service may reach the database.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-policy
  namespace: ivolve
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: mysql
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: auth-service
      ports:
        - { protocol: TCP, port: 3306 }
```

Add matching policies for `frontend` (from the ALB), `auth-service` and `roadmap-service` (from frontend only).

> 💡 **NetworkPolicies are additive allow-lists.** With no policy, everything is allowed. The moment *one* policy selects a pod, everything not explicitly allowed for that pod is denied.

## Prove it works

```bash
# roadmap-service must NOT be able to reach the database
kubectl exec -n ivolve deploy/roadmap-service -- \
  timeout 5 sh -c "</dev/tcp/mysql-0.mysql.ivolve.svc.cluster.local/3306" \
  && echo "REACHABLE — policy broken" || echo "blocked (correct)"
```

---

# Part 8 · Kustomize

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ivolve

resources:
  - 00-namespace.yaml
  - 01-rbac.yaml
  - 02-config.yaml
  - 03-storage.yaml
  - 04-database.yaml
  - 05-auth-service.yaml
  - 06-roadmap-service.yaml
  - 07-frontend.yaml
  - 08-ingress.yaml
  - 09-network-policies.yaml

images:
  - name: ivolve-frontend
    newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ivolve-frontend
    newTag: dev
  - name: ivolve-auth-service
    newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ivolve-auth-service
    newTag: dev
  - name: ivolve-roadmap-service
    newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ivolve-roadmap-service
    newTag: dev

labels:
  - pairs:
      app.kubernetes.io/part-of: ivolve
    # ⚠️ includeSelectors: false — and use `labels`, NOT the deprecated
    # `commonLabels`.
    #
    # commonLabels injects into spec.selector too, and a Deployment's
    # selector is IMMUTABLE. Adding one to an existing Deployment fails
    # with "field is immutable" and the only fix is delete + recreate —
    # an outage caused purely by adding a label.
    includeSelectors: false
```

## 🔨 Why this beats `sed`

```bash
kubectl kustomize . | grep "image:"
```

```text
image: 1234....dkr.ecr.../ivolve-auth-service:dev
image: busybox:1.36                                  ← untouched ✅
image: 1234....dkr.ecr.../ivolve-frontend:dev
image: 1234....dkr.ecr.../ivolve-roadmap-service:dev
image: mysql:8.0                                     ← untouched ✅
```

**5 image lines. Kustomize rewrote exactly the 3 it was told to.**

`sed -i 's|image: .*|image: NEW|g'` would have rewritten **all five** — replacing your database and your init container with an application image. The failure shows up at deploy time as an inexplicable `CrashLoopBackOff`.

Remember this in Module 05.

---

# ✅ Checkpoint

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/123456789012/$ACCOUNT/g" kustomization.yaml

kubectl apply -k .
kubectl get pods,pvc,ingress -n ivolve
```

```text
NAME                                   READY   STATUS
pod/auth-service-xxxx                  1/1     Running
pod/auth-service-xxxx                  1/1     Running
pod/frontend-xxxx                      1/1     Running
pod/frontend-xxxx                      1/1     Running
pod/mysql-0                            1/1     Running
pod/roadmap-service-xxxx               1/1     Running
pod/roadmap-service-xxxx               1/1     Running

NAME                                 STATUS   CAPACITY
persistentvolumeclaim/data-mysql-0   Bound    10Gi

NAME                        ADDRESS
ingress.../frontend         k8s-ivolve-xxxx.us-east-1.elb.amazonaws.com
```

Open the ADDRESS in a browser. Sign up. Log in. **Same app as Module 01, now on EKS.**

> 💡 Until Module 05 pushes real images, the pods will be `ImagePullBackOff` — that's expected. To test now, push manually:
> ```bash
> aws ecr get-login-password | docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
> docker tag ivolve-frontend:local $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/ivolve-frontend:dev
> docker push $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/ivolve-frontend:dev
> ```

---

## 🧹 End of session

```bash
kubectl delete ingress --all -n ivolve     # ⚠️ ALWAYS delete the Ingress first
kubectl delete namespace ivolve
cd ../02-Terraform && terraform destroy
```

**Why the Ingress first?** The ALB is owned by the controller, not Terraform. Destroying the VPC while it exists leaves an orphaned ALB and Terraform hangs ~20 minutes on `DependencyViolation`.

---

## 🧠 Before you continue

1. Why a StatefulSet for MySQL, not a Deployment?
2. What does `clusterIP: None` do, and why does the StatefulSet need it?
3. Why `WaitForFirstConsumer`? What error does `Immediate` cause?
4. What does a `Bound` PVC prove about Module 02?
5. Why does roadmap-service probe `/api/roadmap`?
6. What do the three probe types each do?
7. What breaks if you forget `allow-dns-egress`?
8. Why does `spec.ingressClassName` matter vs the annotation?
9. Why is `sed` on manifests dangerous — with numbers?

---

<div align="center">

**Next → [Module 05 — Jenkins](05-jenkins.md)**

</div>
