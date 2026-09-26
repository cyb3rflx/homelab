# homelab

GitOps repository for my bare-metal Kubernetes homelab. Everything in the cluster is declared here and deployed by **Argo CD** using the *App of Apps* pattern.

## Stack

| Component | Purpose |
|---|---|
| Cilium | CNI, kube-proxy replacement, Gateway API, LB-IPAM, L2 announcements |
| Argo CD | GitOps, App of Apps |
| cert-manager + step-issuer | TLS certificates from a private step-ca |
| external-dns | Creates DNS records in Pi-hole from routes |
| Dashboard | FastAPI backend, React frontend, PostgreSQL |

## Repository layout

```
.
├── bootstrap/
│   └── root-app.yaml          # The only manifest applied by hand
├── apps/                      # One Argo CD Application per component (watched by root-app)
│   ├── cilium.yaml
│   ├── cert-manager.yaml
│   ├── step-issuer.yaml
│   ├── external-dns.yaml
│   ├── infrastructure.yaml
│   └── dashboard.yaml
└── manifests/                 # Helm values and plain manifests referenced by the apps
    ├── cilium/values.yaml
    ├── cert-manager/values.yaml
    ├── external-dns/          # Deployment, RBAC
    ├── infrastructure/        # Gateway, LB IP pool, L2 announcement policy
    └── dashboard/             # Frontend, backend, PostgreSQL, HTTPRoute, storage
```

- **`bootstrap/`**: `app-root` points at `apps/` and syncs it automatically.
- **`apps/`**: Every file is an Argo CD `Application`. Adding a file here adds it to the cluster.
- **`manifests/`**: Two kinds of content:
  - **Helm values** (`cilium/`, `cert-manager/`): the charts are pulled from their upstream repositories; only the values live here, wired in via Argo CD multi-source (`ref: values` → `$values/manifests/...`).
  - **Plain Kubernetes manifests** (`dashboard/`, `external-dns/`, `infrastructure/`): deployed directly by Argo CD from the folder.

## Applications

| Application | Namespace | Sync |
|---|---|---|
| `cilium` | `kube-system` | Manual |
| `cert-manager` | `cert-manager` | Auto |
| `step-issuer` | `step-issuer` | Auto |
| `external-dns-app` | `infrastructure` | Auto |
| `infrastructure-components` | `infrastructure` | Auto |
| `dashboard-app` | `dashboard` | Auto |

Cilium is synced manually on purpose: it provides the cluster network, so a bad sync would also take down Argo CD.

## Networking

```
Client → Pi-hole DNS → 192.168.178.230 → Cilium Gateway → HTTPRoute → Service
```

- Cilium assigns `192.168.178.230` to the Gateway and announces it in the LAN.
- The Gateway serves `*.home.arpa` on HTTP and HTTPS, with TLS from cert-manager.
- external-dns creates the matching DNS records in Pi-hole.

## Secrets

No secrets are stored in this repository. They are created in the cluster separately.
