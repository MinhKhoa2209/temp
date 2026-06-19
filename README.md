# W10 - Progressive Delivery with Analysis

GitOps setup for API deployment với Argo Rollouts + AnalysisTemplate.

## Concept

Deploy API với **canary strategy** và **automated analysis**:
- Rollout: 10% → 50% → 100%
- AnalysisTemplate query Prometheus để check success rate ≥ 95%
- Auto rollback nếu analysis fail
- AlertManager gửi email khi có SLO violation

## Requirements

- Docker Desktop
- kubectl
- minikube
- git

## Structure

```
w10/
├── app-api/              # API Rollout manifests
│   ├── rollout.yaml      # Argo Rollout với canary strategy
│   ├── service.yaml      # Service expose API
│   └── servicemonitor.yaml # Prometheus metrics scraper
├── app-analysis/         # Analysis manifests
│   └── analysis-template.yaml # Template phân tích success rate
├── app-alert/            # Alert manifests
│   ├── prometheus-rules.yaml # PrometheusRule cho SLO alerts
│   ├── email-secret.yaml # Gmail password (NOT COMMITTED)
│   └── README.md         # Alert setup guide
├── app-common/           # Common resources
│   └── demo-namespace.yaml # Namespace demo
├── tenants/
│   └── payments/         # Namespace, RBAC, quota, limit, NetworkPolicy cho team payments
├── apps/
│   └── payments/         # Workload GitOps của team payments
├── src/                  # Source code
│   └── api/              # Flask API application
├── argocd/
│   ├── apps/             # ArgoCD Application manifests
│   │   ├── app-api.yaml  # Deploy API Rollout
│   │   ├── app-analysis.yaml # Deploy AnalysisTemplate
│   │   ├── app-alert.yaml # Deploy PrometheusRule
│   │   ├── app-common.yaml # Deploy common resources
│   │   ├── payments.yaml # Deploy tenant resources cho payments
│   │   ├── payments-app.yaml # Deploy workload team payments
│   │   ├── k8s-prometheus.yaml # Prometheus + AlertManager
│   │   └── k8s-rollout.yaml # Argo Rollouts controller
│   └── root.yaml         # App of Apps pattern
└── README.md
```

## Quick Start

### 1. Setup Cluster
```bash
minikube start -p w10 --driver=docker
kubectl config use-context w10
```

### 2. Install ArgoCD
```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### 3. Access ArgoCD UI
```bash
# Port forward
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### 4. Deploy App of Apps
```bash
kubectl apply -f argocd/root.yaml
```

### 5. Setup Email Alert (Optional)
```bash
# Follow instructions in app-alert/README.md
cp app-alert/email-secret.yaml.example app-alert/email-secret.yaml
kubectl apply -f app-alert/email-secret.yaml
```

## Components

### Core
- **Argo Rollouts**: Progressive delivery controller
- **Prometheus Stack**: Metrics collection + AlertManager
- **API**: Flask application với metrics endpoint

### GitOps Applications
- `app-api`: API Rollout với canary strategy
- `payments`: Namespace, RBAC, ResourceQuota, LimitRange và NetworkPolicy cho team payments
- `payments-app`: Workload riêng của team payments
- `app-analysis`: AnalysisTemplate cho automated validation
- `app-alert`: PrometheusRule cho runtime alerting
- `app-common`: Shared resources (namespace)
- `k8s-prometheus`: Monitoring stack
- `k8s-rollout`: Argo Rollouts controller

## Verify Deployment

### Check Rollout Status
```bash
# Watch rollout progress
kubectl get rollout api -n demo -w

# Check current state
kubectl get rollout api -n demo

# Check pods
kubectl get pods -n demo -l app=api
```

### Check AnalysisRun
```bash
# List analysis runs
kubectl get analysisrun -n demo

# Watch latest analysis
kubectl get analysisrun -n demo --sort-by=.metadata.creationTimestamp | tail -1

# Describe for detailed metrics
kubectl describe analysisrun -n demo <name>
```

### Query Prometheus Metrics
```bash
# Success rate metric
kubectl run test-query --image=curlimages/curl:latest --rm -i --restart=Never -n monitoring -- \
  curl -s 'http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query=api:success_rate:5m'
```

## Test Scenarios (GitOps)

### Test 1: Successful Deployment (Success Rate ≥ 90%)
```bash
# Edit rollout to deploy with no errors
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0"

git add app-api/rollout.yaml
git commit -m "test: deploy with 0% error rate"
git push origin main

# Watch AnalysisRun succeed
kubectl get analysisrun -n demo -w
```

### Test 2: Failed Deployment (Success Rate < 90%)
```bash
# Edit rollout to deploy with 15% error rate
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.15"

git add app-api/rollout.yaml
git commit -m "test: deploy with 15% error rate (should fail)"
git push origin main

# Watch AnalysisRun fail and auto rollback
kubectl get analysisrun -n demo -w
kubectl get rollout api -n demo
```

### Test 3: Trigger SLO Alert Email
```bash
# Edit rollout to set 10% error rate (triggers alert, but passes canary)
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.10"

git add app-api/rollout.yaml
git commit -m "test: deploy with 10% error rate (90% success)"
git push origin main

# Canary passes (≥90%) but SLO alert fires (below 95%)
# Wait 2-3 minutes, then check email inbox
```


## Configuration Reference

### Sync Waves
ArgoCD applications deploy in order:
- Wave -1: `app-common` (namespace)
- Wave -1: `payments` (namespace + tenant guardrails)
- Wave 0: `k8s-prometheus`, `k8s-rollout` (infrastructure)
- Wave 1: `app-analysis`, `app-alert` (configuration)
- Wave 2: `app-api`, `payments-app` (applications)

## Onboard Team Payments

Team `payments` có namespace riêng, Role/RoleBinding riêng, quota/limit riêng và NetworkPolicy riêng:

- `tenants/payments/namespace.yaml`: tạo namespace `payments` và bật Sigstore admission label.
- `tenants/payments/rbac.yaml`: user `payments-dev` chỉ quản workload trong namespace `payments`; không có quyền với `secrets`, `roles` hoặc `rolebindings`.
- `tenants/payments/quota.yaml`: đặt `ResourceQuota` và `LimitRange` để giới hạn ngân sách tài nguyên.
- `tenants/payments/networkpolicy.yaml`: default-deny ingress và chỉ cho egress cùng namespace + DNS, nên pod `payments` không gọi sang service `demo/api`.
- `apps/payments/*`: deploy app team B bằng image đã ký, pin digest và manifest có đủ non-root + resource limits.

Guardrail Gatekeeper cũ vẫn được dùng lại: các constraint trong `gatekeeper/constraints` chỉ được mở rộng match thêm namespace `payments`, không tạo policy/template mới.

Vì sao guardrail cũ tự áp cho team B? Gatekeeper constraint là policy cấp cluster, nên khi thêm `payments` vào `spec.match.namespaces`, cùng luật `disallow-latest`, `require-resource-limits`, `disallow-root-user`, `disallow-host-network` và `max-replicas` tự kiểm tra tenant mới mà không cần viết lại template.

Vì sao Role/RoleBinding giữ cô lập tốt hơn ClusterRoleBinding? RoleBinding `payments-dev` chỉ trỏ tới Role trong namespace `payments`, còn ClusterRoleBinding sẽ cấp quyền theo phạm vi cluster và dễ làm user team B chạm sang namespace/team khác.

Kiểm tra RBAC:

```bash
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secret -n payments --as payments-dev
kubectl auth can-i create rolebinding -n payments --as payments-dev
```

Kỳ vọng lần lượt: `yes`, `no`, `no`, `no`.

Kiểm tra workload và quota:

```bash
kubectl get application payments payments-app -n argocd
kubectl get deploy,svc,quota,limitrange -n payments
kubectl apply --dry-run=server -f runbooks/payments-default-limits-pod.yaml
```

Kỳ vọng pod `payments-default-limits` được admit và tự có default request/limit, còn pod `payments-over-quota` bị quota từ chối.

Kiểm tra NetworkPolicy cần CNI có enforce NetworkPolicy, ví dụ minikube chạy với Calico:

```bash
kubectl run payments-curl -n payments --image=curlimages/curl:latest --rm -i --restart=Never -- \
  curl -m 5 http://api.demo.svc.cluster.local
```

Kỳ vọng request timeout hoặc bị chặn.

Kiểm tra constraint cũ chặn manifest vi phạm trong `payments`:

```bash
kubectl apply --dry-run=server -f runbooks/payments-violating-pod.yaml
```

Kỳ vọng bị reject vì image dùng tag `latest`, thiếu resource limits và không khai báo non-root.

## Cleanup

```bash
# Delete ArgoCD applications
kubectl delete -f argocd/root.yaml

# Wait for resources to be cleaned up
kubectl get all -n demo
kubectl get all -n monitoring

# Delete ArgoCD
kubectl delete ns argocd

# Stop minikube
minikube stop -p w10
minikube delete -p w10
```
