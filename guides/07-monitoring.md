# 📊 Module 07 — Monitoring

**Time:** ~3 hours

---

## 🎯 What you'll build

```text
Prometheus ──scrapes──► pods · nodes · kube-state-metrics · blackbox
Prometheus ──pushes───► Alertmanager        (when a rule fires)
Grafana    ──queries──► Prometheus          (Grafana is a CLIENT)
```

> ⚠️ **Get these directions right.** Almost every hand-drawn diagram shows `Prometheus → Grafana → Alertmanager` as a chain. That's wrong: Grafana is never in the alerting path.

---

## 📖 The pull model

Most older monitoring tools **push** metrics to a central collector. Prometheus **pulls** — it scrapes an HTTP endpoint on each target.

| | Push | Pull |
|---|---|---|
| Target discovery | each target must be configured with the server address | server discovers targets via the K8s API |
| A dead target | silence, indistinguishable from "no data" | **scrape fails — you know immediately** |
| Ephemeral pods | must register/deregister | discovered and dropped automatically |

**"A dead target is detectable" is the key property.** In a push system, a crashed service and a healthy-but-idle service look identical.

---

# Part 1 · Install the stack

`kube-prometheus-stack` bundles Prometheus, Alertmanager, Grafana, node-exporter and kube-state-metrics with the Operator.

`workspace/07-Monitoring/values.yaml`:

```yaml
prometheus:
  prometheusSpec:
    retention: 7d
    retentionSize: 8GB

    # ⚠️ Persist metrics. Without a volumeClaimTemplate, Prometheus stores
    # data in an emptyDir — and every restart loses ALL history. You find
    # this out during your first incident.
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ivolve-storage    # from Module 04
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

    # ⚠️ THE SETTING THAT CATCHES EVERYONE.
    #
    # By default the Operator only discovers ServiceMonitors carrying the
    # Helm release label. Your own ServiceMonitors won't have it, so they
    # are silently ignored — no error, the target simply never appears.
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

    resources:
      requests: { cpu: 200m, memory: 1Gi }
      limits:   { cpu: "1",  memory: 2Gi }

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: ivolve-storage
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 2Gi } }

  config:
    route:
      # Group related alerts into ONE notification. A node failure fires
      # 30 alerts; without grouping that's 30 messages at 3am.
      group_by: ['alertname', 'namespace']
      group_wait: 30s          # wait for related alerts before sending
      group_interval: 5m       # then batch updates
      repeat_interval: 4h      # re-notify unresolved alerts
      receiver: 'null'

    receivers:
      - name: 'null'
      # Add Slack/email here in a real deployment.

grafana:
  adminPassword: "CHANGE_ME"

  persistence:
    enabled: true
    storageClassName: ivolve-storage
    size: 5Gi

  # Prometheus is pre-wired as a datasource by the chart.
  defaultDashboardsEnabled: true

nodeExporter:
  enabled: true      # a DaemonSet — one pod per node (internship Lab 19)

kubeStateMetrics:
  enabled: true      # exposes Deployment/Pod/PVC object state as metrics
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f workspace/07-Monitoring/values.yaml

kubectl -n monitoring rollout status deploy/kube-prometheus-stack-grafana
```

---

# Part 2 · Monitoring the app without touching it

The three microservices expose **no `/metrics` endpoint**. You have two options:

| Option | Cost |
|---|---|
| Add Prometheus client libraries to each service | modifies application code in 3 languages |
| **Blackbox exporter** — probe them over HTTP from outside | zero app changes |

We use blackbox. It answers *"is it up, and how fast does it respond?"* — which covers most of what you need, with no code change.

```bash
helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  -n monitoring
```

`workspace/07-Monitoring/manifests/01-probes.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: ivolve-services
  namespace: monitoring
spec:
  interval: 30s
  module: http_2xx        # succeed only on a 2xx response
  prober:
    url: blackbox-exporter-prometheus-blackbox-exporter:9115
  targets:
    staticConfig:
      static:
        # Cross-namespace: must use the FQDN, not the short name.
        - http://frontend.ivolve.svc.cluster.local:3000/
        - http://auth-service.ivolve.svc.cluster.local:5000/health
        # ⚠️ /api/roadmap — this service has no /health route.
        - http://roadmap-service.ivolve.svc.cluster.local:8080/api/roadmap
```

> ⚠️ **The blackbox exporter lives in `monitoring`, your app in `ivolve`.** The `default-deny-all` NetworkPolicy from Module 04 blocks that traffic. Add an allow rule or the probes fail with a timeout that looks like an app problem:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-ingress
  namespace: ivolve
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
```

---

# Part 3 · Alert rules

`workspace/07-Monitoring/manifests/02-alerts.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ivolve-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: ivolve.availability
      rules:
        - alert: ServiceDown
          expr: probe_success == 0
          # `for:` is what separates a real alert from noise. The condition
          # must hold for 2 minutes — a single failed scrape during a
          # rolling update does NOT page anyone.
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "{{ $labels.instance }} has been unreachable for 2 minutes"

        - alert: PodCrashLooping
          # increase(...[15m]) > 3 — restarting more than 3 times in 15 min.
          expr: increase(kube_pod_container_status_restarts_total{namespace="ivolve"}[15m]) > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "{{ $labels.pod }} is crash-looping"

        - alert: PodNotReady
          expr: kube_pod_status_ready{namespace="ivolve", condition="false"} == 1
          for: 10m
          labels:
            severity: warning

    - name: ivolve.resources
      rules:
        - alert: HighMemoryUsage
          # Actual usage vs the container's LIMIT — not vs node capacity.
          # Exceeding the limit gets the container OOM-killed.
          expr: |
            container_memory_working_set_bytes{namespace="ivolve"}
              / on(container, pod) kube_pod_container_resource_limits{namespace="ivolve", resource="memory"}
              > 0.9
          for: 5m
          labels:
            severity: warning

        - alert: PersistentVolumeFillingUp
          expr: kubelet_volume_stats_available_bytes{namespace="ivolve"} / kubelet_volume_stats_capacity_bytes{namespace="ivolve"} < 0.15
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is over 85% full"
```

> 💡 **Every alert needs a `for:` clause.** Without it, one transient scrape failure pages someone. Alerts that fire spuriously get muted, and a muted alert is worse than no alert — it creates false confidence.

---

# ✅ Checkpoint

```bash
kubectl get pods -n monitoring
```

All `Running`. Then:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

**<http://localhost:9090/targets>** — everything should be **UP**.

Try a query in the Graph tab:

```promql
probe_success
```

Should return `1` for all three services.

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

**<http://localhost:3000>** — `admin` + your password. Open *Dashboards → Kubernetes / Compute Resources / Namespace (Pods)* → select `ivolve`.

## 🔨 Fire an alert deliberately

```bash
kubectl scale deployment roadmap-service -n ivolve --replicas=0
```

Wait 2 minutes, then check **<http://localhost:9090/alerts>**:

```text
ServiceDown  (1 active)
  instance = http://roadmap-service.ivolve.svc.cluster.local:8080/api/roadmap
```

Restore it:

```bash
kubectl scale deployment roadmap-service -n ivolve --replicas=2
```

> 💡 **If ArgoCD's `selfHeal` is on, it will restore the replica count for you** — which is itself a nice demonstration that Modules 06 and 07 are both working.

**You just watched the full observability loop: a real failure → scrape fails → rule evaluates → `for:` elapses → alert fires.**

---

## 🧠 Before you finish

1. Why does Prometheus pull instead of push?
2. What does `serviceMonitorSelectorNilUsesHelmValues: false` fix?
3. What happens without a `volumeClaimTemplate` on Prometheus?
4. Why blackbox probes instead of instrumenting the apps?
5. What NetworkPolicy change did the monitoring namespace require, and why?
6. What does `for: 2m` prevent?
7. Which two components exchange alerts, and in which direction?

---

# 🎓 You're done

Look back at what you built:

| Module | You now understand |
|---|---|
| 01 | Multi-stage builds, layer caching, PID 1 signals, healthchecks |
| 02 | VPC design, route tables, NAT placement, IAM, IRSA/OIDC, EKS add-ons |
| 03 | Provisioning vs configuration, dynamic inventory, idempotency, Vault |
| 04 | StatefulSets, headless Services, CSI storage, probes, NetworkPolicies, Kustomize |
| 05 | Shared libraries, immutable tags, security gates, secret handling in CI |
| 06 | Push vs pull deployment, drift correction, sync waves |
| 07 | Pull-based metrics, alert design, blackbox probing |

**Now write it up.** Not "I followed a tutorial" — you can explain *why* the NAT Gateway is in the public subnet, what breaks without IRSA, and why `sed` on a manifest is dangerous. Those are interview answers.

---

## 🧹 Final teardown ⚠️

```bash
kubectl delete ingress --all -n ivolve
kubectl delete namespace ivolve argocd monitoring --ignore-not-found

# confirm the ALB is actually gone before destroying the VPC
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'

cd workspace/02-Terraform && terraform destroy

aws ec2 delete-key-pair --key-name ivolve-key
```

Verify nothing survives:

```bash
aws eks list-clusters
aws ec2 describe-nat-gateways --query 'NatGateways[?State==`available`].NatGatewayId'
aws ec2 describe-instances --filters "Name=tag:Project,Values=ivolve" \
  --query 'Reservations[].Instances[?State.Name!=`terminated`].InstanceId'
```

All empty. **Check your billing dashboard tomorrow** to be certain.

---

<div align="center">

**[← Back to the course](../README.md)** · **[Progress tracker](../PROGRESS.md)**

</div>
