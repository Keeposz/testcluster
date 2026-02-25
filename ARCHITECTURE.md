# Architecture

Technical documentation covering the design and implementation decisions of this cluster.

## Infrastructure

| Component | Details |
|-----------|---------|
| Server | Hetzner dedicated, Debian Trixie |
| Kubernetes | RKE2 v1.34.3+rke2r3 |
| CNI | Flannel (VXLAN overlay) |
| Container runtime | containerd (via RKE2) |
| Firewall | UFW — TCP: 22, 80, 443, 6443, 9345 / UDP: 8472 |
| Security | fail2ban, SSH key-only auth, root login disabled |

## Ansible provisioning

The cluster is provisioned via `ansible-playbook site.yml`. The playbook is fully idempotent.

**Roles:**

| Role | Source | Responsibilities |
|------|--------|-----------------|
| `debian-base` | External | Base config, security hardening |
| `rke2` | Local (`roles/rke2/`) | RKE2 installation, kubectl symlink, Helm |
| `tailscale` | Local (`roles/tailscale/`) | Tailscale installation + auth |
| `longhorn` | Local (`roles/longhorn/`) | open-iscsi prerequisite |

**Idempotency pattern:**

```yaml
- name: check if rke2-server service exists
  stat:
    path: /usr/local/lib/systemd/system/rke2-server.service
  register: rke2_service

- name: install rke2
  shell: curl -sfL https://get.rke2.io | sh -
  when: not rke2_service.stat.exists
```

## GitOps — ArgoCD

All deployments are managed by ArgoCD using the **wrapper chart pattern**:

```
deployments/<component>/
├── Chart.yaml          # Declares the upstream Helm chart as a dependency
├── Chart.lock
├── charts/             # Downloaded upstream chart (run: helm dependency update)
└── application.yaml    # ArgoCD Application CRD with valuesObject
```

ArgoCD manages itself via `application.yaml` in `deployments/argocd/argocd/`. This solves the chicken-and-egg problem: bootstrap once manually with `helm install`, then ArgoCD syncs itself from Git.

## Networking — Tailscale zero-trust

All private services are **not publicly reachable**. Access goes through Tailscale:

```
Browser → Tailscale VPN → Subnet router pod → NGINX ClusterIP → service
```

**Setup:**
- Tailscale subnet router pod advertises Kubernetes CIDRs: `10.42.0.0/16` (pods) and `10.43.0.0/16` (services)
- A dedicated NGINX Ingress `ClusterIP` Service (`nginx-ingress-tailscale`) acts as the entry point for Tailscale traffic
- Cloudflare DNS A-records point to this ClusterIP, not to the public server IP
- Without Tailscale `--accept-routes` → timeout (zero-trust enforcement)

**NGINX dual access:**

Two Services for NGINX Ingress serving two distinct use cases:

| Service type | Access | Use case |
|--------------|--------|----------|
| `hostPort` | Public | CI/CD webhooks from Git provider |
| `ClusterIP` | Tailscale only | All other services |

The `/hook` webhook path is publicly reachable via hostPort; the Drone UI is only accessible via Tailscale.

## Secret management

Secrets follow a single pipeline:

```
Vault (KV v2) ──► External Secrets Operator ──► Kubernetes Secret ──► App
```

**Components:**

- **PostgreSQL** — Vault storage backend (Bitnami Helm chart, Longhorn PVC)
- **Vault** — KV v2 secrets engine, Kubernetes auth method, PostgreSQL storage
- **External Secrets Operator** — `ClusterSecretStore` as the bridge between Vault and Kubernetes
- **ExternalSecret** — declares which Vault keys map to which Kubernetes Secret keys

```yaml
# ExternalSecret example
spec:
  secretStoreRef:
    name: vault-secret-store
    kind: ClusterSecretStore
  target:
    name: cloudflare-api-token   # Name the app expects
  data:
    - secretKey: api-token       # Key in the Kubernetes Secret
      remoteRef:
        key: cert-manager/cloudflare
        property: api-token      # Key in Vault
```

**Bootstrap order** (chicken-and-egg):
1. Deploy PostgreSQL
2. Deploy Vault → initialize → unseal
3. Enable Vault KV v2 + configure Kubernetes auth
4. Deploy ESO + create ClusterSecretStore
5. Add secrets to Vault via UI or CLI
6. Apply ExternalSecrets → sync to Kubernetes Secrets

> Vault must be manually unsealed after every pod restart (3 of 5 Shamir keys required). Vault auto-unseal is a planned improvement.

## SSL/TLS — cert-manager

Certificates are automatically requested and renewed via Let's Encrypt.

- **Challenge type:** DNS-01 via Cloudflare API (works behind Tailscale, no inbound HTTP required)
- **ClusterIssuer:** `letsencrypt-prod`
- **Scope:** all subdomains of `<YOUR_DOMAIN>`

```
cert-manager → Cloudflare API → DNS TXT record → Let's Encrypt validation → certificate
```

## CI/CD — Drone

```
Git push → webhook → NGINX /hook (hostPort) → Drone server → Docker runner
```

**Pipeline steps** (`.drone.yml`):
1. `yaml-lint` + `ansible-lint` (via Docker containers)
2. `build` — build Docker image tagged with commit SHA
3. `test` — run container as smoke test
4. `push` — push image to registry (main branch only)

Credentials (OAuth, registry, RPC secret) are injected via Vault + ESO as Kubernetes Secrets.

## Monitoring

| Tool | Role |
|------|------|
| Prometheus | Metrics collection (6 scrapeConfigs, direct kubelet scraping) |
| Grafana | Dashboards (Global, Nodes, Pods via sidecar ConfigMaps) |
| Gatus | Uptime monitoring (ArgoCD health check endpoint) |

Grafana dashboards are managed as Kubernetes ConfigMaps in Git (label `grafana_dashboard: "1"`). The Grafana sidecar picks them up automatically.

## Troubleshooting

**RKE2 installs on every run**
```bash
ssh admin@<SERVER_IP> "ls /usr/local/lib/systemd/system/rke2-server.service"
# File missing → service check fails → install triggers again
```

**Vault sealed after restart**
```bash
kubectl exec -n vault vault-0 -- vault operator unseal <KEY1>
kubectl exec -n vault vault-0 -- vault operator unseal <KEY2>
kubectl exec -n vault vault-0 -- vault operator unseal <KEY3>
```

**ClusterSecretStore not ready**
```bash
kubectl exec -n vault vault-0 -- vault status  # Check if Vault is unsealed
kubectl get clustersecretstore                  # Check ESO store status
```

**cert-manager certificate stuck**
```bash
kubectl describe challenge -n <namespace>
# Check: Cloudflare API token empty? Newline in secret?
kubectl get secret cloudflare-api-token -n cert-manager -o yaml
```

**ArgoCD redirect loop (307)**
```bash
kubectl get configmap argocd-cmd-params-cm -n argocd \
  -o jsonpath='{.data.server\.insecure}'  # Must return "true"
```

**Drone runner Unauthorized**
```bash
# Runner and server must share the exact same RPC secret
kubectl rollout restart deployment drone-runner-docker -n drone
```

## Quick reference

```bash
# Deploy cluster
ansible-playbook site.yml

# Cluster status
kubectl get nodes
kubectl get pods -A

# ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Grafana admin password
kubectl get secret grafana -n observability \
  -o jsonpath='{.data.admin-password}' | base64 -d

# Vault status + unseal
kubectl exec -n vault vault-0 -- vault status
kubectl exec -n vault vault-0 -- vault operator unseal <KEY>

# ESO sync status
kubectl get externalsecret -A

# Tailscale subnet router
kubectl get pods -n tailscale
kubectl logs -n tailscale -l app=tailscale-subnet-router

# Certificates
kubectl get certificate -A
```
