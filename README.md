# RKE2 Kubernetes Homelab

Production-grade Kubernetes cluster on Hetzner bare-metal, fully automated with Ansible and managed via ArgoCD GitOps.

## Stack

| Layer | Technology |
|-------|------------|
| Provisioning | Ansible |
| Kubernetes | RKE2 v1.34 |
| GitOps | ArgoCD v3.3 |
| CI/CD | Drone CI (Docker runner) |
| Secret management | HashiCorp Vault v1.21 + External Secrets Operator |
| Networking | Tailscale (zero-trust) + NGINX Ingress |
| Storage | Longhorn |
| Monitoring | Prometheus + Grafana |
| SSL/TLS | cert-manager + Let's Encrypt DNS-01 via Cloudflare |
| Database | PostgreSQL (Bitnami, Vault storage backend) |
| OS | Debian Trixie on Hetzner dedicated server |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Internet                                │
└──────────────┬──────────────────────────────┬───────────────────┘
               │ HTTPS (webhook)               │ Tailscale VPN
               ▼                               ▼
     ┌─────────────────┐             ┌──────────────────┐
     │  NGINX hostPort │             │  NGINX ClusterIP │
     │  (public)       │             │  (private)       │
     └────────┬────────┘             └────────┬─────────┘
              │                               │
              ▼                               ▼
     ┌─────────────────────────────────────────────────┐
     │                 RKE2 Kubernetes                  │
     │                                                  │
     │  ┌──────────┐  ┌────────┐  ┌─────────────────┐  │
     │  │  ArgoCD  │  │ Drone  │  │  Observability  │  │
     │  │ (GitOps) │  │ CI/CD  │  │ Prometheus      │  │
     │  └──────────┘  └────────┘  │ Grafana         │  │
     │                             │ Gatus           │  │
     │  ┌──────────┐  ┌────────┐  └─────────────────┘  │
     │  │  Vault   │◄─│  ESO   │                        │
     │  │ (secrets)│  │        │  ┌─────────────────┐  │
     │  └──────────┘  └────────┘  │    Longhorn     │  │
     │                             │   (storage)     │  │
     │  ┌──────────────────────┐  └─────────────────┘  │
     │  │  Tailscale subnet    │                        │
     │  │  router              │                        │
     │  └──────────────────────┘                        │
     └─────────────────────────────────────────────────┘
```

**Traffic flows:**
- **Private:** Browser → Tailscale VPN → NGINX ClusterIP → service
- **Public (webhook):** Git provider → DNS → NGINX hostPort → `/hook` → Drone

## Repository structure

```
├── ansible.cfg                  # Ansible configuration
├── inventory.yml                # Server definition
├── site.yml                     # Main playbook
├── .drone.yml                   # CI/CD pipeline (lint + build + push)
├── host_vars/testcluster/       # Host-specific Ansible variables
├── roles/
│   ├── rke2/                    # RKE2 installation + kubectl setup (idempotent)
│   ├── tailscale/               # Tailscale installation (idempotent)
│   └── longhorn/                # Longhorn prerequisite: open-iscsi
└── deployments/                 # ArgoCD Applications (Helm wrapper charts)
    ├── argocd/                  # ArgoCD self-management
    ├── cert-manager/            # Let's Encrypt SSL via DNS-01
    ├── drone/                   # Drone CI server + Docker runner
    ├── longhorn/                # Distributed storage
    ├── nginx-ingress-config.yaml
    ├── observability/
    │   ├── gatus/               # Uptime monitoring
    │   ├── grafana/             # Dashboards (3x Kubernetes)
    │   └── prometheus/          # Metrics collection
    ├── tailscale-subnet-router/ # Zero-trust network access
    └── vault/
        ├── external-secrets/    # External Secrets Operator + ClusterSecretStore
        ├── postgresql/          # Vault storage backend
        └── vault/               # HashiCorp Vault (KV v2, Kubernetes auth)
```

## Prerequisites

- Hetzner dedicated server running Debian Trixie
- Cloudflare account with a domain (for DNS-01 SSL and A-records)
- Tailscale account (for zero-trust network access)
- Ansible installed on your local machine
- SSH key access to the server

## Getting started

**1. Clone and replace placeholders:**

| Placeholder | Description |
|-------------|-------------|
| `<YOUR_GITHUB_USERNAME>` | Your GitHub username — used in all `application.yaml` `repoURL` fields |
| `<SERVER_IP>` | Public IP of your server (`inventory.yml`) |
| `<INGRESS_CLUSTER_IP>` | ClusterIP of the NGINX Ingress Service used for Tailscale routing |
| `<YOUR_EMAIL>` | Email for Let's Encrypt notifications (`cert-manager/application.yaml`) |
| `<YOUR_DOMAIN>` | Your domain for SSL certificates (`cert-manager/application.yaml`) |
| `<YOUR_REGISTRY>` | Container registry path (`.drone.yml`) |
| `<YOUR_GIT_SERVER>` | Git server hostname if using SSH (`argocd/application.yaml`) |
| `<YOUR_GIT_USERNAME>` | Your Git username for Drone admin access (`drone/application.yaml`) |

**2. Deploy the cluster with Ansible:**

```bash
ansible-playbook --syntax-check site.yml  # Syntax check
ansible-playbook --check site.yml         # Dry run
ansible-playbook site.yml                 # Deploy
```

**3. Bootstrap ArgoCD:**

```bash
# Download Helm chart dependencies first
helm dependency update deployments/argocd/argocd/

kubectl apply -f deployments/argocd/namespace.yaml
helm install argocd deployments/argocd/argocd/ -n argocd

# ArgoCD then manages itself via the Application CRD
kubectl apply -f deployments/argocd/argocd/application.yaml
```

**4. Bootstrap secrets (manual, one-time):**

```bash
# Cloudflare API token for cert-manager
kubectl create secret generic cloudflare-api-token \
  -n cert-manager --from-literal=api-token=<TOKEN>

# Tailscale auth key (gitignored)
echo "<AUTH_KEY>" > secrets/tailscale-auth-key
```

> After bootstrapping, all secrets are managed by Vault + External Secrets Operator.

## Key design decisions

**Zero-trust networking via Tailscale**
All private services are only reachable via Tailscale. A subnet router pod advertises the pod/service CIDRs. Cloudflare DNS points to the ClusterIP of the NGINX Ingress Service — without Tailscale, everything times out.

**Secret management pipeline**
`Vault (KV v2)` → `External Secrets Operator` → `Kubernetes Secret`. No plaintext secrets in Git. Vault uses PostgreSQL as a storage backend for persistence.

**GitOps via ArgoCD wrapper charts**
All deployments use the wrapper chart pattern: a local `Chart.yaml` declares the upstream dependency, `application.yaml` contains the Helm values. ArgoCD syncs from this repository.

**NGINX Ingress dual access**
Two services for NGINX: `hostPort` (public, for Drone webhooks) and `ClusterIP` (private, for Tailscale). This keeps webhooks reachable without exposing all services publicly.

## Links

- [RKE2 docs](https://docs.rke2.io)
- [ArgoCD docs](https://argo-cd.readthedocs.io)
- [HashiCorp Vault](https://developer.hashicorp.com/vault/docs)
- [External Secrets Operator](https://external-secrets.io)
- [Tailscale subnet routing](https://tailscale.com/kb/1019/subnets)
- [cert-manager DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/)
