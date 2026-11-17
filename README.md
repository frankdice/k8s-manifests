# k8s-manifests (`bluey`)

GitOps repository for home server applications and cluster infrastructure running on **k3s** (`bluey`), managed by **Flux v2** and **Kustomize**.

---

## Repository Structure

```text
.
├── apps/
│   ├── _base/                  # Shared Kustomize base (Namespace, Deployment, Service, Ingress)
│   ├── _components/
│   │   ├── ingress-tls/        # Kustomize component adding Cloudflare cert-manager TLS
│   │   └── pvc-longhorn/       # Kustomize component adding a persistent volume claim & mount
│   └── <app>/                  # Per-application Kustomize manifests
├── cluster-services/           # Core infrastructure (Flux, cert-manager, haproxy-ingress, etc.)
├── clusters/
│   └── bluey/                  # Flux Kustomization entrypoints auto-discovered by Flux
│       ├── <app>.yaml          # App Flux Kustomizations (defines postBuild variables)
│       └── cluster-services/   # Infrastructure Flux Kustomizations
└── renovate.json               # Renovate config (PR-only updates, automerge disabled)
```

---

## Architecture & Conventions

### 1. Shared Cluster Variables (`postBuild`)
Global non-secret settings live in [cluster-services/fluxcd/cluster-settings.yaml](cluster-services/fluxcd/cluster-settings.yaml) (`ConfigMap/cluster-settings` in `flux-system`) and are injected into manifests by Flux `postBuild.substituteFrom`:
- `${SECRET_DOMAIN}`: `babyvelociraptor.com`
- `${NAS_IP}`: `192.168.1.35`
- `${CLUSTER_NODE}`: `bluey`

### 2. Three-Tier Storage Strategy
Storage is chosen per application based on durability needs:
1. **Local Node (`storageClassName: local-path`):** Single-node local storage on `bluey` (either dynamic `local-path` PVCs or static PVs under `/storage/<app>-...`) for rebuildable configs and caches.
2. **Backed-Up Local State (`storageClassName: longhorn`):** Block storage managed by Longhorn with snapshots/backups pushed to NFS (`${NAS_IP}:/Media/Backups/Bluey`).
3. **Shared Media (`nfs`):** Direct NFS mounts from `${NAS_IP}:/Media/...` for media libraries and downloads.

### 3. Ingress, DNS & Certificates
- **Ingress Controller:** `haproxy-ingress` (`ingressClassName: haproxy-ingress`) bound via LoadBalancer to `192.168.1.5` on ports `80` (redirects to HTTPS) and `443`.
- **TLS:** `cert-manager` issues Let's Encrypt certificates via Cloudflare DNS-01 (`ClusterIssuer/cloudflare`).
- **DNS:** `external-dns` automatically manages DNS records for ingresses/services.

### 4. Secrets Management
- Secrets are stored in 1Password (`Bluey` vault) and synced into Kubernetes `Secret`s via `1password-connect` and `external-secrets` (`ClusterSecretStore/bluey`).

### 5. Automated Updates (Renovate)
- [renovate.json](renovate.json) scans all Flux `HelmRelease`s, Kustomize `images:` blocks, and Kubernetes manifests.
- `automerge` is set to `false`—Renovate only opens pull requests and maintains a Dependency Dashboard issue so all upgrades remain manual and explicit.

---

## Adding a New Application

Most applications use [apps/_base](apps/_base) + optional [apps/_components](apps/_components) and only require **two small files**:

### 1. Create `apps/<app>/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: my-app
resources:
  - ../_base
components:
  - ../_components/ingress-tls
  # Optional: include if the app needs a PVC
  - ../_components/pvc-longhorn
images:
  - name: app-image
    newName: org/my-app
    newTag: 1.2.3
```

### 2. Create `clusters/bluey/<app>.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  path: ./apps/my-app
  prune: true
  postBuild:
    substitute:
      APP_NAME: my-app
      APP_PORT: "8080"
      # Optional: if using ../_components/pvc-longhorn
      PVC_NAME: data
      PVC_MOUNT_PATH: /data
      PVC_STORAGE_CLASS: longhorn # or local-path
      PVC_SIZE: 10Gi
    substituteFrom:
      - kind: ConfigMap
        name: cluster-settings
```

*(If an app also needs NFS mounts or custom environment variables, add a `patches:` block in `apps/<app>/kustomization.yaml`—see [apps/readarr/kustomization.yaml](apps/readarr/kustomization.yaml) for an example.)*

---

## Bootstrapping & Day-2 Operations

### Configure `KUBECONFIG` on `bluey`
Standalone CLIs (`flux`, `helm`) require `KUBECONFIG` to be pointed at k3s's config:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube && ln -sf /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

### Bootstrap 1Password Connect & External Secrets (One-Time Manual Step)
```bash
kubectl create ns 1password
kubectl -n 1password create secret generic op-credentials \
  --from-file=1password-credentials.json=./1password-credentials.json

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bluey-token
  namespace: 1password
type: Opaque
stringData:
  token: <BLUEY_VAULT_TOKEN>
EOF
```

### Bootstrap Flux CD
```bash
cd cluster-services/fluxcd/
kubectl kustomize . | kubectl apply --server-side --force-conflicts -f -

cd ../../clusters/bluey/cluster-services/
kubectl apply -f fluxcd.yaml
```

### Common Flux Commands
```bash
# Pull latest Git commits and reconcile all Kustomizations
flux reconcile source git k8s-manifests -n flux-system
flux reconcile kustomization k8s-manifests -n flux-system

# Check status of all Flux resources
flux get all -A

# Validate all Kustomizations locally before committing
for d in apps/*/ cluster-services/*/; do
  [ "$d" != "apps/_components/" ] && [ "$d" != "cluster-services/ingress-nginx/" ] && kubectl kustomize "$d" > /dev/null
done
```

### Upgrade Flux CD
```bash
# 1. On Mac/workstation: export new Flux manifests into cluster-services/fluxcd/fluxcd.yaml
flux install --components=source-controller,kustomize-controller,helm-controller,notification-controller --export > cluster-services/fluxcd/fluxcd.yaml
flux migrate -f . --yes

# 2. On bluey: upgrade CLI, migrate storedVersions in etcd, and apply
curl -s https://fluxcd.io/install.sh | sudo bash
flux migrate
flux install --components=source-controller,kustomize-controller,helm-controller,notification-controller --export | kubectl apply --server-side --force-conflicts -f -
flux check
```

---

## Host OS & Troubleshooting Notes

### AdGuard Home (Port 53) vs. `systemd-resolved`
Disable `systemd-resolved` on `bluey` so AdGuard's LoadBalancer can bind port 53:
- Reference: https://github.com/AdguardTeam/AdGuardHome/wiki/Docker#resolved

### Force-Remove Stuck Namespace Finalizers
```bash
kubectl get ns <NS> -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/<NS>/finalize" -f -
```

---

## Look Into Later
- openbooks
- PhotoPrism / Immich
- Invidious
- wikijs
- kimai (time tracking)
- n8n (workflows)
- draw.io (`https://github.com/jgraph/docker-drawio`)
- ErsatzTV (`https://github.com/jasongdove/ErsatzTV`)
- Nightscout (`https://github.com/nightscout/cgm-remote-monitor`)
- archiveBox (use n8n to push RSS feed from linkding to back up bookmarked pages)
