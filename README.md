## context-alarm-gitops

GitOps repo for Context Alarm on k3s using Argo CD + SealedSecrets.

### What’s deployed

- **Namespace**: `context-alarm` (`cluster/namespace.yaml`)
- **API**
  - **Deployment**: `cluster/api/deployment.yaml`
  - **Service**: `cluster/api/service.yaml` (Service port 80 → container port 9090)
  - **Env/Config**: loaded from Kubernetes Secret `api-secret`
  - **Image**: `ghcr.io/context-alarm/context-alarm-backend:<tag>`
- **Checker Worker**
  - **Deployment**: `cluster/checkerApi/deployment.yaml`
  - **Type**: internal background worker (no Service/Ingress)
  - **Env/Config**: loaded from Kubernetes Secret `api-secret`
  - **Image**: `ghcr.io/context-alarm/context-alarm-checker:<tag>`
- **UI**
  - **Deployment**: `cluster/ui/deployment.yaml`
  - **Service**: `cluster/ui/service.yaml` (Service port 80 -> container port 3000)
  - **Image**: `ghcr.io/context-alarm/context-alarm-frontend:<tag>`
- **Secrets**
  - **App Secret**: `cluster/api/sealed-api-secret.yaml` (Bitnami SealedSecret → creates/updates `Secret/api-secret`)
- **Cloudflare Tunnel (in-cluster)**
  - **Deployment**: `cluster/ingress/cloudflared-api.yaml`
  - **ConfigMap**: `cluster/ingress/cloudflared-config.yaml`
  - **Public URL**: `https://api.contextalarm.com`
  - **Public URL**: `https://contextalarm.com`

### Argo CD

Argo CD `Application` is defined in `bootstrap/root-app.yaml`:

- **source**: this repo
- **path**: `cluster/` (directory recurse enabled)
- **syncPolicy**: automated (prune + self-heal)

Any Kubernetes manifests added/changed under `cluster/` are automatically applied to the k3s cluster.

### Updating API environment variables (.env → SealedSecret)

This repo stores encrypted app config as a SealedSecret (`cluster/api/sealed-api-secret.yaml`).

Workflow used:

1. Update values in your `.env` locally (do **not** commit `.env`)
2. Generate a plain Secret manifest from `.env` and seal it:

```bash
kubectl create secret generic api-secret \
  --from-env-file=".env" \
  --namespace=context-alarm \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-cert.pem --format yaml > cluster/api/sealed-api-secret.yaml
```

3. Commit + push the updated `cluster/api/sealed-api-secret.yaml`
4. Argo CD syncs it and the SealedSecrets controller updates `Secret/api-secret`
5. Reloader automatically rolls `api` and `checker-api` Deployments when `api-secret` changes

If your `.env` path contains spaces, quote it:
`--from-env-file="/path/with spaces/.env"`

### Auto-rollout on secret changes (Reloader)

`cluster/api/deployment.yaml` and `cluster/checkerApi/deployment.yaml` include Reloader annotations so pods restart automatically when `Secret/api-secret` changes.

Install Reloader once in the cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
```

Verify Reloader is running:

```bash
kubectl -n reloader get deploy,pods
```

### Private GHCR images (imagePullSecret)

The API image is private in GHCR, so k3s needs an image pull secret.

Secret created on the cluster:

```bash
kubectl create secret docker-registry ghcr-cred \
  --namespace context-alarm \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<PAT-with-read:packages> \
  --docker-email=you@example.com
```

The API Deployment references it via `imagePullSecrets`.

### Cloudflare Tunnel (in-cluster)

Goal: expose both API and UI through one tunnel.

Cloudflare side:

- **Tunnel name**: `context-alarm-api`
- **Tunnel ID**: `25a04e47-53c7-4411-812a-0712f562e85d`
- **DNS route**: `api.contextalarm.com` → tunnel (CNAME)

k3s side:

- Tunnel credentials are stored as a Kubernetes Secret:
  - **Secret**: `cloudflared-tunnel-cred`
  - **Namespace**: `context-alarm`
  - **Key**: `credentials.json`

Created on the cluster (example):

```bash
kubectl -n context-alarm create secret generic cloudflared-tunnel-cred \
  --from-file=credentials.json=clfare.json
```

`cloudflared` runs inside the cluster and targets the API Service using Kubernetes service DNS:

- **Internal service URL**: `http://api.context-alarm.svc.cluster.local:80`
- **Internal service URL**: `http://ui.context-alarm.svc.cluster.local:80`

This URL is deterministic:
`http://<service-name>.<namespace>.svc.cluster.local:<service-port>`

### Useful commands

- **Pods**

```bash
kubectl -n context-alarm get pods
kubectl -n context-alarm describe pod <pod>
```

- **API logs**

```bash
kubectl -n context-alarm logs -f -l app=api
```

- **Cloudflared logs**

```bash
kubectl -n context-alarm logs -f -l app=cloudflared-api
```

- **External test**

```bash
curl -i https://api.contextalarm.com/v1/stripe/pricing
```

### Follow-ups (do later)

1. **Automate image updates (best-practice GitOps)**
  - API CI should push an immutable image tag (e.g. git SHA) and update `cluster/api/deployment.yaml` `image:` tag in this repo.
  - Checker CI should set `GITOPS_MANIFEST_PATH=cluster/checkerApi/deployment.yaml` and update that manifest `image:` tag.
  - UI CI should set `GITOPS_MANIFEST_PATH=cluster/ui/deployment.yaml` and update that manifest `image:` tag.
2. **Secret portability**
   - `ghcr-cred` and `cloudflared-tunnel-cred` are created manually on the cluster right now.
   - Later: move them to SealedSecrets or an external secret manager for easier cluster rebuilds.
3. **DB TLS cleanup**
   - Long-term clean fix: use a DB hostname + proper TLS SANs (instead of connecting by IP), or issue a cert with IP SANs.