# Supply Chain Verify Runbook

## Goal

Prove that the API image is scanned by Trivy, signed by Cosign, and verified by admission before running in Kubernetes.

## Preconditions

- GitHub repository secrets exist:
  - `COSIGN_PRIVATE_KEY`
  - `COSIGN_PASSWORD`
- `signing/cosign.pub` contains the matching public key.
- `policies/cluster-image-policy.yaml` contains the same public key.
- `policy-controller` and `policies` Argo CD Applications are Synced/Healthy.

## Verify CI Scan And Sign

Trigger the workflow:

```powershell
git push origin main
```

Confirm in GitHub Actions:

- Trivy fails the workflow for `HIGH` or `CRITICAL` CVEs.
- If Trivy passes, the image is pushed to GHCR.
- Cosign signs `ghcr.io/minhkhoa2209/w10-api:0.0.1`.

Verify the signature locally:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/minhkhoa2209/w10-api:0.0.1
```

## Enable Admission Enforcement

Only label the namespace after the current API image is signed:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

## Admission Tests

Unsigned image should be rejected:

```powershell
kubectl run unsigned-test `
  -n demo `
  --image=ghcr.io/minhkhoa2209/w10-api:unsigned-test `
  --restart=Never
```

Signed image should pass:

```powershell
kubectl run signed-test `
  -n demo `
  --image=ghcr.io/minhkhoa2209/w10-api:0.0.1 `
  --restart=Never `
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true},"containers":[{"name":"signed-test","image":"ghcr.io/minhkhoa2209/w10-api:0.0.1","securityContext":{"runAsUser":1000,"runAsGroup":1000,"allowPrivilegeEscalation":false},"resources":{"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}}]}}'
```

## Acceptance

- CI fails on blocked CVEs.
- Unsigned image is rejected by admission.
- Signed image is admitted and starts normally.
