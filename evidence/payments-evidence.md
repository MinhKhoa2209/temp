# Payments Tenant Evidence

Generated on 2026-06-19 from Kubernetes context `w10`.

## 1. Docker Desktop and minikube state

```text
kubectl config current-context
w10

minikube status -p w10
w10
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

docker ps
NAMES     STATUS
w10       Up 24 minutes
```

## 2. ArgoCD and workload state

```text
kubectl get applications payments payments-app root gatekeeper gatekeeper-policies -n argocd
NAME                  SYNC STATUS   HEALTH STATUS
payments              Synced        Healthy
payments-app          Synced        Healthy
root                  Synced        Healthy
gatekeeper            Synced        Healthy
gatekeeper-policies   Synced        Healthy

kubectl get deploy,pods,svc -n payments
NAME                           READY   UP-TO-DATE   AVAILABLE
deployment.apps/payments-api   2/2     2            2

NAME                                READY   STATUS    RESTARTS
pod/payments-api-5d444c775d-9lhtc   1/1     Running   0
pod/payments-api-5d444c775d-r7sm8   1/1     Running   0

NAME                   TYPE        CLUSTER-IP     PORT(S)
service/payments-api   ClusterIP   10.110.72.44   80/TCP
```

## 3. RBAC isolation

```text
kubectl auth can-i create deploy -n payments --as payments-dev
yes
kubectl auth can-i create deploy -n demo --as payments-dev
no
kubectl auth can-i get secret -n payments --as payments-dev
no
kubectl auth can-i create rolebinding -n payments --as payments-dev
no
```

## 4. Quota and LimitRange

```text
kubectl get quota,limitrange,networkpolicy,role,rolebinding -n payments
resourcequota/payments-quota   pods: 2/10, requests.cpu: 100m/1, requests.memory: 128Mi/1Gi, services: 1/5
limitrange/payments-default-limits
networkpolicy/default-deny-ingress
networkpolicy/allow-same-namespace-and-dns-egress
role/payments-developer
rolebinding/payments-dev

kubectl apply --dry-run=server -f runbooks/payments-default-limits-pod.yaml
pod/payments-default-limits created (server dry run)
Error from server (Forbidden): pods "payments-over-quota" is forbidden: exceeded quota: payments-quota
```

## 5. NetworkPolicy isolation

The `w10` profile was started with Calico CNI, so Kubernetes NetworkPolicy is enforced.

```text
kubectl get svc api -n demo
NAME   TYPE        CLUSTER-IP       PORT(S)
api    ClusterIP   10.100.129.180   80/TCP

kubectl run payments-curl -n payments --image=curlimages/curl:8.8.0 --rm -i --restart=Never -- ...
curl: (28) Connection timed out after 5045 milliseconds
pod payments/payments-curl terminated (Error)
```

## 6. Existing constraints block violations

```text
kubectl apply --dry-run=server -f runbooks/payments-violating-pod.yaml
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[disallow-root-user] container <bad> must not run as root
[disallow-latest-tag] container <bad> uses disallowed image tag <latest>
```
