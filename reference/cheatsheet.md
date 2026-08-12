# ⚡ Cheatsheet

Commands you'll reach for constantly. Keep this open in a second tab.

---

## 🐳 Docker

```bash
docker build -t name:tag .                  # build
docker build --no-cache -t name:tag .       # ignore layer cache
docker build --target runtime -t n:t .      # stop at a specific stage

docker run --rm -it image sh                # poke around inside
docker run --rm image whoami                # WHO does it run as?
docker exec -it container sh                # shell into a RUNNING container

docker images | sort -k7 -h                 # by size
docker history image                        # which layer is fat?
docker inspect image | jq '.[0].Config'     # user, env, entrypoint

docker system df                            # what's using disk
docker system prune -af --volumes           # reclaim everything ⚠️
```

### Compose

```bash
docker compose config --quiet               # VALIDATE before running
docker compose config                       # show fully-resolved file
docker compose up -d --build
docker compose ps
docker compose logs -f auth-service
docker compose exec mysql mysql -u root -p
docker compose down                         # keeps volumes
docker compose down -v                      # DELETES volumes ⚠️
```

---

## 🏗️ Terraform

```bash
terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan
terraform plan -out=tfplan && terraform apply tfplan   # apply exactly what you reviewed
terraform apply -target=module.network                 # one module (debugging only)
terraform destroy

terraform output
terraform output -raw eks_cluster_name      # no quotes — for use in scripts
terraform state list
terraform state show module.eks.aws_eks_cluster.this
terraform console                           # REPL for testing expressions
```

### When state and reality disagree

```bash
terraform refresh                           # re-read real resources
terraform import aws_s3_bucket.x bucket-name
terraform state rm aws_instance.x           # forget it (does NOT delete)
terraform taint module.server.aws_instance.jenkins   # force recreate next apply
```

> ⚠️ `terraform state rm` makes Terraform *forget* a resource — it keeps running and keeps billing. Use `destroy` to actually remove it.

---

## 🤖 Ansible

```bash
ansible-inventory --graph                   # what hosts were discovered
ansible-inventory --graph --flush-cache     # ignore the cache
ansible-inventory --list -vvv               # plugin errors

ansible role_jenkins -m ping
ansible role_jenkins -m setup               # dump all facts
ansible role_jenkins -a "df -h"             # ad-hoc command

ansible-playbook playbook.yml
ansible-playbook playbook.yml --check --diff        # DRY RUN
ansible-playbook playbook.yml --tags docker
ansible-playbook playbook.yml --skip-tags sonarqube
ansible-playbook playbook.yml --start-at-task "Install Jenkins"
ansible-playbook playbook.yml -vvv                  # debug SSH
```

### Vault

```bash
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault view    group_vars/all/vault.yml      # read without decrypting to disk
ansible-vault edit    group_vars/all/vault.yml
ansible-vault rekey   group_vars/all/vault.yml
```

---

## ☸️ kubectl

### Looking around

```bash
kubectl get pods -n ivolve
kubectl get pods -n ivolve -o wide           # + node, IP
kubectl get all -n ivolve
kubectl get events -n ivolve --sort-by=.lastTimestamp   # ⭐ start here when stuck

kubectl describe pod <pod> -n ivolve         # ⭐ events at the bottom
kubectl logs <pod> -n ivolve
kubectl logs <pod> -n ivolve -f              # follow
kubectl logs <pod> -n ivolve --previous      # ⭐ logs from the CRASHED container
kubectl logs <pod> -n ivolve -c wait-for-mysql   # a specific initContainer
```

### Debugging

```bash
kubectl exec -it <pod> -n ivolve -- sh
kubectl port-forward svc/frontend -n ivolve 3000:3000

# ⭐ Is the Service actually pointing at anything?
kubectl get endpoints -n ivolve
# Empty endpoints = no pod is passing its readiness probe

# DNS test from inside the cluster
kubectl run -it --rm dnstest --image=busybox:1.36 --restart=Never -- \
  nslookup auth-service.ivolve.svc.cluster.local

# Connectivity test
kubectl run -it --rm nettest --image=busybox:1.36 --restart=Never -- \
  wget -qO- http://auth-service.ivolve:5000/health
```

### Kustomize

```bash
kubectl kustomize .                          # render WITHOUT applying
kubectl kustomize . | grep image:            # ⭐ verify image substitution
kubectl apply -k .
kubectl diff -k .                            # what WOULD change
kustomize edit set image name=new:tag
```

### Workloads

```bash
kubectl rollout status deploy/frontend -n ivolve
kubectl rollout history deploy/frontend -n ivolve
kubectl rollout undo deploy/frontend -n ivolve       # roll back
kubectl rollout restart deploy/frontend -n ivolve    # restart without changing spec

kubectl scale deploy/frontend -n ivolve --replicas=3
kubectl top pods -n ivolve                           # needs metrics-server
```

### Useful output tricks

```bash
kubectl get pods -n ivolve -o jsonpath='{..image}' | tr ' ' '\n' | sort -u
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,ZONE:.metadata.labels.'topology\.kubernetes\.io/zone'
kubectl get secret ivolve-secret -n ivolve -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

---

## ☁️ AWS CLI

```bash
aws sts get-caller-identity
aws sts get-caller-identity --query Account --output text     # just the ID

# EKS
aws eks list-clusters
aws eks update-kubeconfig --region us-east-1 --name <cluster>
aws eks describe-cluster --name <cluster> --query 'cluster.status'
aws eks list-addons --cluster-name <cluster>

# ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin <acct>.dkr.ecr.us-east-1.amazonaws.com
aws ecr describe-repositories --query 'repositories[].repositoryName'
aws ecr list-images --repository-name ivolve-frontend --query 'imageIds[*].imageTag'

# EC2
aws ec2 describe-instances \
  --filters "Name=tag:Role,Values=jenkins" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress]' --output table

# Find EVERYTHING tagged with this project — use before teardown
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Project,Values=ivolve \
  --query 'ResourceTagMappingList[].ResourceARN' --output table
```

---

## 🔄 ArgoCD

```bash
kubectl get application -n argocd
kubectl describe application ivolve -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

kubectl port-forward svc/argocd-server -n argocd 8080:443

# CLI
argocd app get ivolve
argocd app sync ivolve
argocd app history ivolve
argocd app rollback ivolve <id>
argocd app diff ivolve
```

---

## 📊 Prometheus / Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

### PromQL starters

```promql
probe_success                                    # blackbox: is it up?
up{namespace="ivolve"}                           # are scrapes working?

# Restarts in the last 15 minutes
increase(kube_pod_container_status_restarts_total{namespace="ivolve"}[15m])

# Memory usage vs LIMIT (this is what gets you OOM-killed)
container_memory_working_set_bytes{namespace="ivolve"}
  / on(container,pod) kube_pod_container_resource_limits{resource="memory"}

# CPU cores used
rate(container_cpu_usage_seconds_total{namespace="ivolve"}[5m])

# PVC free space
kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes
```

---

## 🔐 Secret generation

```bash
openssl rand -base64 24                     # passwords
openssl rand -base64 32                     # vault password / session secret
openssl rand -hex 16

curl -s https://checkip.amazonaws.com       # your public IP
```

---

## 🧹 Teardown — in this order

```bash
# 1 — Kubernetes FIRST. The ALB is owned by the controller, not Terraform.
kubectl delete ingress --all -n ivolve
kubectl delete svc --all-namespaces --field-selector spec.type=LoadBalancer
kubectl delete namespace ivolve argocd monitoring --ignore-not-found

# 2 — confirm it's gone
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'

# 3 — infrastructure
cd workspace/02-Terraform && terraform destroy

# 4 — what Terraform doesn't own
aws ec2 delete-key-pair --key-name ivolve-key
rm -f ~/.ssh/ivolve-key.pem

# 5 — verify nothing survives
aws eks list-clusters
aws ec2 describe-nat-gateways --query 'NatGateways[?State==`available`].NatGatewayId'
aws ec2 describe-instances --filters "Name=tag:Project,Values=ivolve" \
  --query 'Reservations[].Instances[?State.Name!=`terminated`].InstanceId'
```

---

## 🎯 Debug order — when something is broken

```text
1.  kubectl get events -n <ns> --sort-by=.lastTimestamp
        ⭐ 80% of answers are here

2.  kubectl describe <kind> <name> -n <ns>
        events at the bottom

3.  kubectl logs <pod> -n <ns> --previous
        the CRASHED container, not the new one

4.  kubectl get endpoints -n <ns>
        empty = readiness probe failing

5.  DNS test with a busybox pod

6.  controller logs
        kubectl logs -n kube-system deploy/aws-load-balancer-controller
```

> 💡 **`kubectl get events` first, always.** Scheduling failures, image pull errors, probe failures, volume problems — all show up there with a plain-English reason.

---

<div align="center">

**[Error decoder](errors.md)** · **[Back to the course](../README.md)**

</div>
