# Secret Rotation Runbook

## Goal

Prove that AWS Secrets Manager rotation is synced to Kubernetes by External Secrets Operator in less than 60 seconds, without restarting the API pod.

## Preconditions

- `eso` and `eso-secrets` Argo CD Applications are Synced/Healthy.
- Local Kubernetes Secret `aws-credentials` exists in namespace `demo`.
- AWS Secrets Manager secret `w10/demo/api-db-password` exists with property `password`.
- API reads the password from the mounted Kubernetes Secret or from the synced Secret value.

## Verify Current State

```powershell
kubectl get application eso eso-secrets -n argocd
kubectl get secretstore aws-store -n demo
kubectl get externalsecret api-db-password -n demo
kubectl get secret api-db-password -n demo
kubectl get pods -n demo -l app=api
```

Record the current pod name and restart count:

```powershell
kubectl get pods -n demo -l app=api -o wide
```

## Rotate Secret

Update the AWS Secrets Manager value:

```powershell
aws secretsmanager put-secret-value `
  --region ap-southeast-1 `
  --secret-id w10/demo/api-db-password `
  --secret-string '{"password":"replace-with-new-password"}'
```

Watch the Kubernetes Secret:

```powershell
kubectl get secret api-db-password -n demo -w
```

## Acceptance

- Kubernetes Secret updates in less than 60 seconds.
- API pod name remains the same.
- API pod restart count does not increase.
- App reads the rotated value successfully.
