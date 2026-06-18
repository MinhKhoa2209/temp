# W10 Morning RBAC + Admission Policy Plan

## Summary

- Repo GitOps dùng `https://github.com/MinhKhoa2209/temp.git`.
- Container image dùng `ghcr.io/minhkhoa2209/w10-api:0.0.1`.
- Alertmanager gửi và nhận mail qua `minhkhoaoik2209@gmail.com`.
- Lab cần bổ sung RBAC, Gatekeeper controller, Gatekeeper constraints, và 1 custom policy reject Deployment nếu `replicas > 5`.
- Policy chỉ enforce namespace `demo` để tránh chặn nhầm Argo CD, Prometheus, Gatekeeper, Rollouts controller, hoặc namespace hệ thống.

## Existing Repo Changes

- Đổi toàn bộ Argo CD internal `repoURL` sang `https://github.com/MinhKhoa2209/temp.git`:
  - `argocd/root.yaml`
  - `argocd/apps/app-api.yaml`
  - `argocd/apps/app-analysis.yaml`
  - `argocd/apps/app-alert.yaml`
  - `argocd/apps/app-common.yaml`
- Giữ Helm repo external:
  - `https://argoproj.github.io/argo-helm`
  - `https://prometheus-community.github.io/helm-charts`
- Đổi API image trong `app-api/rollout.yaml`:
  - `ghcr.io/minhkhoa2209/w10-api:0.0.1`
- Đổi Alertmanager email trong `argocd/apps/k8s-prometheus.yaml`:
  - `to: minhkhoaoik2209@gmail.com`
  - `from: minhkhoaoik2209@gmail.com`
  - `auth_username: minhkhoaoik2209@gmail.com`

## New Files To Create

### `argocd/apps/rbac.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rbac
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/MinhKhoa2209/temp.git
    path: rbac
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - ServerSideApply=true
```

### `rbac/roles.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: demo
rules:
- apiGroups: ["", "apps", "argoproj.io"]
  resources: ["pods", "services", "deployments", "replicasets", "rollouts"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre
rules:
- apiGroups: ["", "apps", "argoproj.io"]
  resources: ["pods", "services", "deployments", "replicasets", "rollouts"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### `rbac/rolebindings.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: demo
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-sre
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: sre
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: carol-viewer
subjects:
- kind: User
  name: carol
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: viewer
  apiGroup: rbac.authorization.k8s.io
```

### `argocd/apps/gatekeeper.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gatekeeper
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://open-policy-agent.github.io/gatekeeper/charts
    chart: gatekeeper
    targetRevision: 3.22.2
  destination:
    server: https://kubernetes.default.svc
    namespace: gatekeeper-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

### `argocd/apps/gatekeeper-policies.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gatekeeper-policies
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/MinhKhoa2209/temp.git
    path: gatekeeper
    targetRevision: main
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: gatekeeper-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - ServerSideApply=true
    - SkipDryRunOnMissingResource=true
```

### `gatekeeper/templates/*.yaml`

Vendor 4 ConstraintTemplate từ Gatekeeper Library:

- `K8sDisallowedTags`: chặn image tag `latest`.
- `K8sRequiredResources`: bắt container có `resources.limits.cpu` và `resources.limits.memory`.
- `K8sPSPAllowedUsers`: chặn container chạy root.
- `K8sPSPHostNetworkingPorts`: chặn `hostNetwork: true`.

### `gatekeeper/constraints/disallow-latest-tag.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: disallow-latest-tag
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["demo"]
  parameters:
    tags: ["latest"]
```

### `gatekeeper/constraints/require-resource-limits.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["demo"]
  parameters:
    limits:
    - cpu
    - memory
```

### `gatekeeper/constraints/disallow-root-user.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPAllowedUsers
metadata:
  name: disallow-root-user
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["demo"]
  parameters:
    runAsUser:
      rule: MustRunAsNonRoot
```

### `gatekeeper/constraints/disallow-host-network.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPHostNetworkingPorts
metadata:
  name: disallow-host-network
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["demo"]
  parameters:
    hostNetwork: false
```

### `gatekeeper/templates/k8s-max-replicas.yaml`

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8smaxreplicas
spec:
  crd:
    spec:
      names:
        kind: K8sMaxReplicas
      validation:
        openAPIV3Schema:
          type: object
          properties:
            maxReplicas:
              type: integer
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8smaxreplicas

      violation[{"msg": msg}] {
        input.review.kind.kind == "Deployment"
        replicas := input.review.object.spec.replicas
        max := input.parameters.maxReplicas
        replicas > max
        msg := sprintf("Deployment replicas must be <= %v, got %v", [max, replicas])
      }
```

### `gatekeeper/constraints/max-replicas.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMaxReplicas
metadata:
  name: max-replicas
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces: ["demo"]
  parameters:
    maxReplicas: 5
```

## Existing File To Harden

Update `app-api/rollout.yaml` before enforcing Gatekeeper:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
      - name: api
        image: ghcr.io/minhkhoa2209/w10-api:0.0.1
        securityContext:
          runAsUser: 1000
          runAsGroup: 1000
          allowPrivilegeEscalation: false
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
```

## Test Plan

- RBAC:
  - `kubectl auth can-i create deploy -n demo --as alice` => `yes`
  - `kubectl auth can-i create deploy -n kube-system --as alice` => `no`
  - `kubectl auth can-i get pods -A --as bob` => `yes`
  - `kubectl auth can-i delete nodes --as carol` => `no`

- Gatekeeper readiness:
  - `kubectl get ns gatekeeper-system`
  - `kubectl -n gatekeeper-system rollout status deploy/gatekeeper-controller-manager`
  - `kubectl get constrainttemplates`
  - `kubectl get constraints`

- Admission:
  - Pod image `nginx:latest` => reject.
  - Pod thiếu limits => reject.
  - Pod `runAsUser: 0` => reject.
  - Pod `hostNetwork: true` => reject.
  - Pod hợp lệ: pinned image, limits, non-root => pass.
  - Deployment `replicas: 6` => reject.
  - Deployment `replicas: 5` => pass.

- GitOps:
  - Push changes to `https://github.com/MinhKhoa2209/temp.git`.
  - `kubectl apply -f argocd/root.yaml`
  - Confirm Argo CD apps are `Synced/Healthy`.
  - Confirm API Rollout remains healthy after policies enforce.

## Notes

- Nếu `ghcr.io/minhkhoa2209/w10-api:0.0.1` chưa tồn tại, build và push image trước khi sync, hoặc tạm dùng image cũ để tránh `ImagePullBackOff`.
- Vì Alertmanager đang dùng `smtp.gmail.com:587`, email `minhkhoaoik2209@gmail.com` cần dùng Gmail App Password. File local `app-alert/email-secret.yaml` đã chứa App Password và đang được `.gitignore` bỏ qua.
