# W10 Lab - File tạo thêm và luồng hoạt động

Tài liệu này mô tả các file đã bổ sung trong repo và cách chúng phối hợp với nhau trong lab W10. Nội dung được tổng hợp từ `README.md`, `docs/plan.md`, các manifest GitOps và runbook hiện có.

## 1. Mục tiêu lab

Lab này mở rộng hệ thống W10 theo 4 lớp chính:

- Progressive delivery: deploy API bằng Argo Rollouts theo canary `10% -> 50% -> 100%`.
- Observability: Prometheus scrape `/metrics`, AnalysisTemplate kiểm tra success rate và Alertmanager gửi cảnh báo khi vi phạm SLO.
- Governance: RBAC, Gatekeeper admission policy và custom policy giới hạn số replica.
- Secrets và supply chain: External Secrets Operator sync secret từ AWS Secrets Manager; GitHub Actions scan image bằng Trivy, ký bằng Cosign và Sigstore Policy Controller kiểm tra chữ ký khi admission.

## 2. Nhóm file GitOps điều phối

### `argocd/root.yaml`

Đây là root Application theo mô hình App of Apps. Khi chạy:

```powershell
kubectl apply -f argocd/root.yaml
```

Argo CD đọc thư mục `argocd/apps` từ repo `https://github.com/MinhKhoa2209/temp.git`, rồi tự tạo toàn bộ child Application bên dưới. Root app bật:

- `prune: true`: xóa tài nguyên không còn trong Git.
- `selfHeal: true`: tự kéo cluster về đúng trạng thái Git khi có drift.

### `argocd/apps/*.yaml`

Các file trong thư mục này là child Application. Chúng chia lab thành từng phần nhỏ để Argo CD sync theo đúng thứ tự:

| File | Application | Vai trò | Sync wave |
| --- | --- | --- | --- |
| `app-common.yaml` | `common` | Tạo namespace `demo` | `-1` |
| `payments.yaml` | `payments` | Tạo namespace, RBAC, quota, limit và NetworkPolicy cho team `payments` | `-1` |
| `k8s-prometheus.yaml` | `kube-prometheus-stack` | Cài Prometheus, Alertmanager, Grafana | `0` |
| `k8s-rollout.yaml` | `argo-rollouts` | Cài Argo Rollouts controller và CRD | `0` |
| `gatekeeper.yaml` | `gatekeeper` | Cài Gatekeeper controller | `0` |
| `eso.yaml` | `eso` | Cài External Secrets Operator và CRD | `0` |
| `app-analysis.yaml` | `analysis` | Deploy AnalysisTemplate | `1` |
| `app-alert.yaml` | `alert` | Deploy PrometheusRule cảnh báo SLO | `1` |
| `rbac.yaml` | `rbac` | Deploy Role, ClusterRole, Binding | `1` |
| `gatekeeper-policies.yaml` | `gatekeeper-policies` | Deploy ConstraintTemplate và Constraint | `1` |
| `eso-secrets.yaml` | `eso-secrets` | Deploy SecretStore và ExternalSecret | `1` |
| `policies.yaml` | `policies` | Deploy Sigstore ClusterImagePolicy | `1` |
| `app-api.yaml` | `api` | Deploy Rollout, Service, ServiceMonitor | `2` |
| `payments-app.yaml` | `payments-app` | Deploy workload riêng của team `payments` | `2` |

Luồng sync này giúp controller và CRD có trước, policy/config có sau, cuối cùng mới deploy workload API.

## 3. Nhóm file ứng dụng API

### `src/api/app.py`

Đây là Flask API dùng cho demo. App có 3 điểm quan trọng:

- Route `/` trả JSON thành công hoặc lỗi 500 tùy biến môi trường `ERROR_RATE`.
- Route `/healthz` dùng cho liveness/readiness probe.
- `PrometheusMetrics(app)` tự expose `/metrics` để Prometheus scrape.

Biến `ERROR_RATE` là công tắc demo:

- `ERROR_RATE="0"`: API gần như luôn thành công, canary pass.
- `ERROR_RATE="0.10"`: success rate khoảng 90%, canary pass nếu ngưỡng là 90% nhưng SLO 95% có thể alert.
- `ERROR_RATE="0.15"`: success rate khoảng 85%, AnalysisRun fail và Rollout rollback.

### `src/api/Dockerfile`

Dockerfile build Flask API thành container image. GitHub Actions build image này và push lên GHCR theo tên:

```text
ghcr.io/minhkhoa2209/w10-api:<version>
```

### `app-api/rollout.yaml`

File này thay Deployment thường bằng Argo Rollouts `Rollout`.

Các điểm chính:

- Chạy 4 replicas trong namespace `demo`.
- Image hiện tại: `ghcr.io/minhkhoa2209/w10-api:0.0.1`.
- Container chạy non-root với `runAsUser: 1000`, `runAsGroup: 1000`.
- Có CPU/memory requests và limits để pass Gatekeeper policy.
- Canary strategy đi qua các bước `10%`, pause 2 phút, `50%`, pause 2 phút, rồi `100%`.
- Từ `startingStep: 1`, Rollout gọi AnalysisTemplate `success-rate` để kiểm tra chất lượng trước khi promote tiếp.

### `app-api/service.yaml`

Tạo Service `api` trong namespace `demo`, expose port `80` và forward vào container port `8080`. Service này là điểm để Prometheus và client nội bộ gọi API.

### `app-api/servicemonitor.yaml`

Tạo `ServiceMonitor` để kube-prometheus-stack scrape endpoint:

```text
http://api.demo.svc/metrics
```

Prometheus dùng metrics này cho cả AnalysisTemplate và PrometheusRule.

## 4. Nhóm file phân tích và cảnh báo

### `app-analysis/analysis-template.yaml`

Tạo AnalysisTemplate `success-rate` trong namespace `demo`. Template query Prometheus:

```promql
sum(rate(flask_http_request_duration_seconds_count{namespace="demo",app="api",status!~"5.."}[2m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo",app="api"}[2m]))
```

Điều kiện pass:

```text
result >= 0.90
```

Nếu success rate thấp hơn ngưỡng nhiều lần, AnalysisRun fail và Argo Rollouts rollback bản canary.

### `app-alert/prometheus-rules.yaml`

Tạo `PrometheusRule` trong namespace `monitoring`:

- Recording rule `api:success_rate:5m` tính success rate trong 5 phút.
- Alert `SLOViolation` bắn khi `api:success_rate:5m < 0.95` trong 2 phút.

Điểm khác nhau giữa canary analysis và SLO alert:

- Canary analysis dùng ngưỡng 90% để quyết định promote hoặc rollback.
- SLO alert dùng ngưỡng 95% để cảnh báo chất lượng service đang xấu.

### `app-alert/email-secret.yaml.example`

Đây là file mẫu để tạo secret email cho Alertmanager. File secret thật `app-alert/email-secret.yaml` không được commit vì chứa Gmail App Password.

## 5. Nhóm file RBAC

### `rbac/roles.yaml`

Tạo 3 nhóm quyền:

- `Role developer` trong namespace `demo`: được thao tác workload chính như pods, services, deployments, replicasets, rollouts trong `demo`.
- `ClusterRole sre`: có quyền thao tác workload tương tự ở cấp cluster.
- `ClusterRole viewer`: chỉ được `get`, `list`, `watch` toàn bộ tài nguyên.

### `rbac/rolebindings.yaml`

Gán quyền cho user demo:

- `alice` nhận Role `developer` trong namespace `demo`.
- `bob` nhận ClusterRole `sre`.
- `carol` nhận ClusterRole `viewer`.

Lệnh kiểm tra mẫu:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Kỳ vọng: `alice` chỉ mạnh trong `demo`, `bob` có quyền SRE với workload, `carol` chỉ xem được.

## 6. Nhóm file Gatekeeper admission policy

### `gatekeeper/templates/*.yaml`

Các ConstraintTemplate định nghĩa loại policy mà Gatekeeper hiểu được:

- `k8s-disallowed-tags.yaml`: tạo kind `K8sDisallowedTags`.
- `k8s-required-resources.yaml`: tạo kind `K8sRequiredResources`.
- `k8s-psp-allowed-users.yaml`: tạo kind `K8sPSPAllowedUsers`.
- `k8s-psp-host-networking-ports.yaml`: tạo kind `K8sPSPHostNetworkingPorts`.
- `k8s-max-replicas.yaml`: custom template tạo kind `K8sMaxReplicas`.

### `gatekeeper/constraints/*.yaml`

Các Constraint áp policy vào namespace `demo` và `payments`:

- `disallow-latest-tag.yaml`: reject Pod dùng image tag `latest`.
- `require-resource-limits.yaml`: reject Pod thiếu `resources.limits.cpu` hoặc `resources.limits.memory`.
- `disallow-root-user.yaml`: reject Pod/container chạy root.
- `disallow-host-network.yaml`: reject Pod bật `hostNetwork: true`.
- `max-replicas.yaml`: reject Deployment trong `demo` hoặc `payments` nếu `spec.replicas > 5`.

Policy chỉ match namespace workload (`demo`, `payments`) để tránh chặn nhầm Argo CD, Prometheus, Gatekeeper, Rollouts controller hoặc namespace hệ thống.

## 7. Nhóm file tenant payments

### `tenants/payments/namespace.yaml`

Tạo namespace `payments` cho team mới. Namespace này không commit secret và dùng các guardrail Gatekeeper hiện hữu để kiểm soát workload team B.

### `tenants/payments/rbac.yaml`

Tạo Role `payments-developer` và RoleBinding `payments-dev` trong namespace `payments`.

Quyền được cấp chỉ nằm trong namespace `payments` và chỉ cho workload thường như pods, services, configmaps, deployments, replicasets. Role này không cấp quyền đọc `secrets`, tạo `roles` hoặc tạo `rolebindings`, nên user `payments-dev` không tự nâng quyền và không chạm được namespace `demo`.

Lệnh kiểm tra:

```powershell
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secret -n payments --as payments-dev
kubectl auth can-i create rolebinding -n payments --as payments-dev
```

Kỳ vọng: `yes`, `no`, `no`, `no`.

### `tenants/payments/quota.yaml`

Tạo `ResourceQuota` để giới hạn tổng ngân sách namespace và `LimitRange` để đặt default request/limit cho container. Phần này giúp team mới không dùng quá tài nguyên platform.

### `tenants/payments/networkpolicy.yaml`

Tạo 2 NetworkPolicy:

- `default-deny-ingress`: chặn toàn bộ traffic đi vào pod trong namespace `payments`.
- `allow-same-namespace-and-dns-egress`: chỉ cho pod trong `payments` gọi pod cùng namespace và DNS ở `kube-system`.

Vì egress bị giới hạn, pod trong `payments` không gọi được service `api.demo.svc.cluster.local`. Điều này cần CNI có enforce NetworkPolicy, ví dụ minikube bật Calico.

### `apps/payments/deployment.yaml`

Deploy app team B qua GitOps. Manifest dùng lại image đã ký `ghcr.io/minhkhoa2209/w10-api@sha256:616ae00b4ad60d69be24ba476e1ab87d6f2c7b9a3195caa90dccaa7ef1c46705`, đặt `VERSION=payments-v0.0.1`, chạy non-root và có đủ CPU/memory requests + limits để pass Gatekeeper/Sigstore.

### `runbooks/payments-violating-pod.yaml`

Đây là manifest cố ý sai để chứng minh constraint cũ tự áp cho namespace mới. File dùng image tag `latest`, thiếu resource limits và không khai báo non-root.

Kiểm tra:

```powershell
kubectl apply --dry-run=server -f runbooks/payments-violating-pod.yaml
```

Kỳ vọng: admission reject bởi các constraint Gatekeeper hiện hữu, không cần tạo luật mới.

## 8. Nhóm file External Secrets Operator

### `eso/secret-store.yaml`

Tạo `SecretStore` tên `aws-store` trong namespace `demo`. SecretStore khai báo cách ESO kết nối AWS Secrets Manager:

- Service: `SecretsManager`.
- Region: `ap-southeast-1`.
- Credential lấy từ Kubernetes Secret local `aws-credentials`.

Lưu ý: `aws-credentials` là secret thủ công/local, không commit vào repo.

### `eso/external-secret.yaml`

Tạo `ExternalSecret` tên `api-db-password`. File này yêu cầu ESO:

- Đọc remote secret `w10/demo/api-db-password`.
- Lấy property `password`.
- Tạo hoặc cập nhật Kubernetes Secret `api-db-password` trong namespace `demo`.
- Refresh mỗi `60s`.
- Giữ lại Kubernetes Secret khi ExternalSecret bị xóa nhờ `deletionPolicy: Retain`.

Luồng rotation:

1. Người vận hành đổi value trong AWS Secrets Manager.
2. ESO reconcile theo `refreshInterval: 60s`.
3. Kubernetes Secret `api-db-password` được cập nhật.
4. Pod API không cần restart nếu app đọc secret theo cơ chế phù hợp.

## 9. Nhóm file supply chain security

### `.github/workflows/build-push.yml`

Workflow này chạy khi push thay đổi trong `src/api/**` hoặc workflow file, hoặc khi chạy manual `workflow_dispatch`.

Luồng chính:

1. Checkout code.
2. Tính semantic version.
3. Login GHCR.
4. Build Docker image từ `src/api`.
5. Chạy Trivy scan image, fail nếu có CVE `HIGH` hoặc `CRITICAL`.
6. Push image tags lên GHCR.
7. Cài Cosign.
8. Ghi private key từ GitHub Secret `COSIGN_PRIVATE_KEY` vào file tạm `cosign.key`.
9. Ký image bằng Cosign với `COSIGN_PASSWORD`.
10. Update `app-api/rollout.yaml` sang image version mới.
11. Commit/push thay đổi version và tạo Git tag.

Private key và password chỉ nằm trong GitHub Secrets, không nằm trong repo.

### `signing/cosign.pub`

Chứa Cosign public key được phép commit. Public key này dùng để verify image đã được ký bởi private key tương ứng.

Verify thủ công:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/minhkhoa2209/w10-api:0.0.1
```

### `policies/cluster-image-policy.yaml`

Tạo Sigstore `ClusterImagePolicy` để require chữ ký Cosign cho image:

```text
ghcr.io/minhkhoa2209/w10-api:*
ghcr.io/minhkhoa2209/w10-api@sha256:*
```

Policy đang ở `mode: enforce`, nghĩa là image không có chữ ký hợp lệ sẽ bị admission reject khi policy-controller được bật cho namespace phù hợp.

### `argocd/apps/policies.yaml`

Đưa `policies/cluster-image-policy.yaml` vào GitOps để Argo CD quản lý. File này cần Sigstore Policy Controller/CRD có trước, nên có `SkipDryRunOnMissingResource=true`.

## 10. Nhóm file runbook

### `runbooks/secret-rotation.md`

Mô tả cách chứng minh secret rotation:

- Kiểm tra ESO app, SecretStore, ExternalSecret và Kubernetes Secret.
- Ghi pod name và restart count trước khi rotate.
- Update AWS Secrets Manager bằng `aws secretsmanager put-secret-value`.
- Watch Kubernetes Secret.
- Acceptance: secret cập nhật dưới 60 giây, pod không đổi tên, restart count không tăng.

### `runbooks/supply-chain-verify.md`

Mô tả cách chứng minh supply-chain guardrail:

- GitHub Actions scan image bằng Trivy.
- Image pass scan mới được push và ký.
- Verify chữ ký bằng `cosign verify`.
- Bật admission enforcement bằng label namespace `demo`.
- Unsigned image bị reject, signed image được admit.

### `runbooks/cve-exception-adr.md`

Mẫu ADR cho trường hợp cần exception tạm thời với CVE `HIGH/CRITICAL`. Exception chỉ hợp lệ khi có CVE ID, owner, expiry date, lý do, mitigation và lệnh review lại.

## 11. Luồng hoạt động end-to-end

### 11.1. Bootstrap cluster và GitOps

1. Tạo cluster minikube.
2. Cài Argo CD.
3. Apply `argocd/root.yaml`.
4. Root app tạo các child app trong `argocd/apps`.
5. Argo CD sync theo wave:
   - Wave `-1`: namespace `demo`.
   - Wave `0`: controller nền như Prometheus, Rollouts, Gatekeeper, ESO.
   - Wave `1`: analysis, alert, RBAC, policy, secret sync.
   - Wave `2`: API Rollout.

### 11.2. Deploy API bằng canary

1. Argo CD sync `app-api/rollout.yaml`.
2. Argo Rollouts tạo ReplicaSet canary.
3. Canary nhận 10% traffic logic theo rollout step.
4. Sau pause, Rollout tạo AnalysisRun từ `success-rate`.
5. AnalysisRun query Prometheus.
6. Nếu success rate `>= 0.90`, Rollout tiếp tục lên 50% rồi 100%.
7. Nếu success rate thấp, Rollout fail analysis và rollback về version ổn định trước đó.

### 11.3. Metrics và alert

1. Flask API expose `/metrics`.
2. `ServiceMonitor` yêu cầu Prometheus scrape service `api`.
3. Prometheus tính `api:success_rate:5m`.
4. Nếu success rate dưới 95% trong 2 phút, `SLOViolation` firing.
5. Alertmanager nhận alert và gửi email bằng cấu hình trong `k8s-prometheus.yaml` cộng secret email local.

### 11.4. RBAC và admission

1. Argo CD sync `rbac/`.
2. Kubernetes API server dùng Role/Binding để trả lời các lệnh `kubectl auth can-i`.
3. Argo CD sync Gatekeeper controller và policy.
4. Khi có Pod/Deployment mới trong namespace `demo`, Kubernetes gọi Gatekeeper admission webhook.
5. Gatekeeper kiểm tra image tag, resource limits, non-root, hostNetwork và max replicas.
6. Manifest vi phạm bị reject trước khi chạy trong cluster.
7. Với namespace `payments`, cùng constraint cũ cũng reject manifest sai vì constraint đã match thêm namespace này.

### 11.5. Onboard team payments

1. Argo CD sync `payments.yaml` ở wave `-1`.
2. Namespace `payments`, Role/RoleBinding, quota/limit và NetworkPolicy được tạo trước workload.
3. Argo CD sync `payments-app.yaml` ở wave `2`.
4. Deployment `payments-api` chạy bằng image đã ký và đủ security/resource fields.
5. `payments-dev` chỉ thao tác được workload trong `payments`, không thao tác được `demo`, `secrets` hoặc `rolebindings`.
6. NetworkPolicy chặn traffic vào `payments` và chặn egress từ `payments` sang service của `demo`, chỉ giữ lại cùng namespace và DNS.

### 11.6. Secret rotation

1. ESO controller được cài từ Helm chart.
2. `SecretStore` biết cách đọc AWS Secrets Manager.
3. `ExternalSecret` map remote secret `w10/demo/api-db-password` sang Kubernetes Secret `api-db-password`.
4. Khi secret ở AWS đổi, ESO sync lại trong khoảng 60 giây.
5. Kubernetes Secret đổi mà không cần commit secret thật vào Git.

### 11.7. Build, scan, sign và verify image

1. Developer sửa code trong `src/api`.
2. Push lên GitHub.
3. GitHub Actions build image.
4. Trivy scan image:
   - Nếu có CVE `HIGH/CRITICAL` không được bỏ qua, workflow fail.
   - Nếu pass, image được push lên GHCR.
5. Cosign ký image bằng private key từ GitHub Secrets.
6. Workflow update `app-api/rollout.yaml` sang version mới và push lại.
7. Argo CD thấy Git thay đổi, sync Rollout.
8. Sigstore policy-controller verify chữ ký image khi admission.
9. Image không ký hoặc ký sai key bị reject.

## 12. Các lệnh kiểm tra nhanh

```powershell
kubectl get application -n argocd
kubectl get rollout api -n demo
kubectl get analysisrun -n demo
kubectl get pods -n demo -l app=api
kubectl get constrainttemplates
kubectl get constraints
kubectl get externalsecret -n demo
kubectl get secret api-db-password -n demo
kubectl get clusterimagepolicy
kubectl get application payments payments-app -n argocd
kubectl get deploy,svc,quota,limitrange,networkpolicy -n payments
```

Kiểm tra RBAC:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secret -n payments --as payments-dev
kubectl auth can-i create rolebinding -n payments --as payments-dev
```

Kiểm tra rollout:

```powershell
kubectl get rollout api -n demo -w
kubectl describe analysisrun -n demo <analysisrun-name>
```

Kiểm tra Cosign:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/minhkhoa2209/w10-api:0.0.1
kubectl apply --dry-run=server -f runbooks/payments-violating-pod.yaml
```

## 13. Điểm cần nhớ khi nộp lab

- Không commit secret thật: AWS key, Gmail App Password, Cosign private key, secret value.
- `docs/plan.md` và HTML lab trong `docs/` là tài liệu local/reference nếu `.gitignore` đang bỏ qua.
- Các policy Gatekeeper hiện enforce namespace `demo` và `payments`.
- Chỉ bật Sigstore admission label cho namespace `demo` sau khi image hiện tại đã được ký.
- Nếu image `ghcr.io/minhkhoa2209/w10-api:0.0.1` chưa tồn tại hoặc chưa ký, Rollout có thể lỗi `ImagePullBackOff` hoặc bị admission reject.
