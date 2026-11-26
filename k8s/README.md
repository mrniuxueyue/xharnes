# X-Harness Kubernetes Deployment Manifests

Production-ready raw Kubernetes manifests (`kubectl apply -k`) implementing the role-split architecture (`api`, `worker`, `feishu`, `scheduler`, `web`) plus isolated sandbox security boundaries.

## File Layout

| File | Description |
| --- | --- |
| `00-namespace.yaml` | `x-harness` control-plane namespace & `x-harness-sandbox` restricted sandbox namespace |
| `01-configmap.yaml` | `x-harness-config` (app/S3/native K8s sandbox env vars) & `x-harness-web-nginx` (SPA + SSE reverse proxy) |
| `02-secret.example.yaml` | Sensitive credentials template (DB, Redis, Master Key, S3, optional search keys) |
| `03-rbac.yaml` | `x-harness` (zero-privilege SA) & `x-harness-sandbox-ctrl` (least-privilege Role in `x-harness-sandbox` for native K8s Pod sandboxes) |
| `04-job-bootstrap.yaml` | Idempotent Job running `src/cli.ts bootstrap` (V1–V4 migrations + builtin tools/roles/admin) |
| `05-deployment-api.yaml` | Stateless API + SSE + Webhook server (`XH_ROLES=api`, 2 replicas) + `Service` |
| `06-deployment-worker.yaml` | Agent & multi-agent DAG execution worker (`XH_ROLES=worker`, `terminationGracePeriodSeconds: 150`) |
| `07-deployment-feishu-scheduler.yaml` | Feishu channel gateway (`XH_ROLES=feishu`) + leader-elected Scheduler (`XH_ROLES=scheduler`) |
| `08-deployment-web.yaml` | Unprivileged Nginx SPA (`nginxinc/nginx-unprivileged`, port 8080, `/healthz`) + `Service` |
| `09-ingress.yaml` | NGINX Ingress with SSE streaming (`proxy-buffering: off`, 3600s timeout), TLS, 50m upload limit |
| `10-autoscaling-pdb-netpol.yaml` | HPAs (`api`, `worker`), PDBs, and default-deny `NetworkPolicy` for `x-harness-sandbox` |
| `infra/` | Optional self-hosted backing services (PostgreSQL+pgvector, Redis Cache/Streams, MinIO S3, OpenSandbox) |

## Quick Start

### 1. Build & Push Container Images

```bash
REGISTRY=your-registry.corp.internal/xh
TAG=1.4.0

docker build -t $REGISTRY/server:$TAG ./server
docker build -t $REGISTRY/web:$TAG ./web
docker build -f ./deploy/sandbox/Dockerfile.data -t $REGISTRY/sbx-data:1.0.0 ./deploy/sandbox

docker push $REGISTRY/server:$TAG
docker push $REGISTRY/web:$TAG
docker push $REGISTRY/sbx-data:1.0.0
```

### 2. Configure Domain, Images & Secrets

1. Update `XH_PUBLIC_BASE_URL`, `XH_CORS_ORIGINS`, `XH_S3_BUCKET`, `XH_DEFAULT_SANDBOX_IMAGE` in `01-configmap.yaml`, and `host` / `tls.secretName` in `09-ingress.yaml`.
2. Update image repositories/tags via the `images:` override block in `kustomization.yaml`.
3. If using a private container registry for `sbx-data:1.0.0`, create a pull secret in `x-harness-sandbox` and set `XH_K8S_SANDBOX_IMAGE_PULL_SECRETS` in `01-configmap.yaml`:
   ```bash
   kubectl -n x-harness-sandbox create secret docker-registry regcred \
     --docker-server=<REGISTRY> --docker-username=<USER> --docker-password=<PASS>
   ```
4. Generate a production master key (`openssl rand -hex 32`) and replace all `CHANGE_ME_*` values in `02-secret.example.yaml` (recommended: replace with External Secrets / Sealed Secrets in production).

### 3. Deploy Backing Infrastructure (Optional)

Skip this step if you already have managed PostgreSQL 16 (pgvector), Redis, and S3. Native Kubernetes sandboxes (`XH_SANDBOX_PROVIDER=kubernetes`) require **no OpenSandbox server or CRDs**.

```bash
# Replace CHANGE_ME_* passwords inside deploy/k8s/infra/00-infra-stateful.yaml first
kubectl apply -k deploy/k8s/infra
kubectl -n x-harness rollout status statefulset/x-harness-postgres statefulset/x-harness-redis-streams statefulset/x-harness-minio
```

### 4. Deploy X-Harness

```bash
kubectl apply -k deploy/k8s

# Wait for schema migration + bootstrap Job
kubectl -n x-harness wait --for=condition=complete job/x-harness-bootstrap --timeout=180s
kubectl -n x-harness logs job/x-harness-bootstrap

# Verify all workloads are ready
kubectl -n x-harness rollout status deploy/x-harness-api deploy/x-harness-worker deploy/x-harness-feishu deploy/x-harness-scheduler deploy/x-harness-web
```

### 5. Post-Deploy Verification

```bash
kubectl -n x-harness get pods,svc,ingress,hpa,pdb
# Local smoke test via port-forward (if Ingress DNS is not bound yet):
kubectl -n x-harness port-forward svc/x-harness-web 8080:80
# Open http://localhost:8080 and sign in with username `admin` and `xh_seed_admin_password`
```
