# homelab-nebulahvelvet

<div align="center">

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Flux](https://img.shields.io/badge/Flux_v2-5468FF?style=for-the-badge&logo=flux&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=for-the-badge&logo=traefikproxy&logoColor=white)
![Gitea](https://img.shields.io/badge/Gitea-609926?style=for-the-badge&logo=gitea&logoColor=white)
![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=for-the-badge&logo=authentik&logoColor=white)

**Self-hosted Kubernetes infrastructure managed with GitOps**

[Architecture](#architecture) • [Stack](#stack) • [Structure](#repository-structure) • [Secrets](#secrets)

</div>

---

## Architecture

```
                            ┌─────────────────────────────────────────────────────────┐
                            │                    INTERNET                             │
                            └─────────────────────────┬───────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  EDGE                                                                                               │
│  ┌────────────────────────────────────────┐    ┌──────────────────────────────────────────────────┐ │
│  │  traefik-external (10.0.2.82)          │    │  cert-manager                                    │ │
│  │  ├─ TLS termination                    │    │  └─ Let's Encrypt DNS-01 via Domeneshop          │ │
│  │  ├─ fail2ban + bot-wrangler            │    └──────────────────────────────────────────────────┘ │
│  │  └─ ForwardAuth → Authentik            │                                                         │
│  └────────────────────────────────────────┘                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  SERVICES                                                                                           │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────────────────────────────┐ │
│  │  Authentik           │  │  Gitea               │  │  traefik-internal (10.0.2.83)              │ │
│  │  ├─ SSO / OIDC       │  │  ├─ Git hosting      │  │  └─ *.local.net (no TLS)                   │ │
│  │  └─ CNPG PostgreSQL  │  │  ├─ Actions CI/CD    │  └────────────────────────────────────────────┘ │
│  └──────────────────────┘  │  ├─ Container registry│                                                │
│                            │  └─ CNPG PostgreSQL  │                                                 │
│                            └──────────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  DATA                                                                                               │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────────────────────────┐   │
│  │  Longhorn            │  │  SeaweedFS           │  │  NFS (nfs-odin)                          │   │
│  │  └─ Block storage    │  │  └─ S3 object store  │  │  └─ Shared media / backups               │   │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  PLATFORM                                                                                           │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────────────────────────┐   │
│  │  Flux v2             │  │  MetalLB             │  │  CloudNativePG                           │   │
│  │  └─ GitOps sync      │  │  └─ L2 LoadBalancer  │  │  └─ PostgreSQL operator                  │   │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Stack

### Platform

| Component | Description |
|:--|:--|
| **Flux v2** | GitOps continuous delivery — syncs this repo to the cluster |
| **MetalLB** | Bare-metal LoadBalancer (L2 mode, pool `10.0.2.80-90`) |
| **CloudNativePG** | PostgreSQL operator for Authentik and Gitea databases |

### Networking & Security

| Component | Description |
|:--|:--|
| **Traefik** | Dual-instance ingress — `external` (HTTPS + middlewares) and `internal` (HTTP, `*.local.net`) |
| **cert-manager** | Automatic TLS via Let's Encrypt DNS-01 (Domeneshop webhook) |
| **Authentik** | Identity provider with OIDC/SAML, Traefik ForwardAuth integration |

### Storage

| Component | Description |
|:--|:--|
| **Longhorn** | Distributed block storage with S3 backup support |
| **SeaweedFS** | S3-compatible object storage |
| **NFS** | Shared storage via `nfs-subdir-external-provisioner` |

### CI/CD & Automation

| Component | Description |
|:--|:--|
| **Gitea** | Self-hosted Git with Actions runners and container registry |
| **GitHub Runners** | Self-hosted Actions runners (ARC scale sets) |
| **Renovate** | Automated dependency updates |

---

## Repository Structure

```
cluster/
├── flux/                # FluxInstance, GitRepository, Discord notifications
├── traefik/             # Ingress controllers (internal + external)
├── metalllb/            # L2 LoadBalancer IP pool
├── cert-manager/        # TLS certificates (Let's Encrypt + Domeneshop DNS-01)
├── longhorn/            # Block storage + retain StorageClass
├── nfs-provisioner/     # NFS-backed dynamic provisioner
├── seaweedfs/           # S3 object storage (operator + cluster)
├── cnpg/                # CloudNativePG operator
├── authentik/           # SSO / identity provider
├── gitea/               # Git server, Actions runner, container registry
├── github-runner/       # Self-hosted GitHub Actions (ARC)
├── renovate/            # Dependency automation
├── media/               # Media applications
├── crafty/              # Minecraft server manager
├── uptime-kuma/         # Uptime monitoring
└── wol/                 # Wake-on-LAN service
```

---

## Secrets

Secrets are **never committed**. Each component requiring credentials includes a `*-secret-template.yaml` describing the expected shape. Apply secrets manually before deploying workloads.

---

## Network

| IP | Service |
|:--|:--|
| `10.0.2.82` | traefik-external |
| `10.0.2.83` | traefik-internal |
| `10.0.2.89` | Crafty Minecraft |

| Domain | Target |
|:--|:--|
| `auth.nebulahvelvet.no` | Authentik |
| `git.nebulahvelvet.no` | Gitea |
| `*.local.net` | Internal services |

---

## Bootstrap

This cluster is bootstrapped via [midgardssmie](https://github.com/stianmjo/midgardssmie) — an Ansible playbook that provisions bare Ubuntu nodes to Kubernetes + Cilium + Flux.

---

<div align="center">

*Infrastructure as code. GitOps as lifestyle.*

</div>