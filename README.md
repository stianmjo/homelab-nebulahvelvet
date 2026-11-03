<div align="center">

# homelab-nebulahvelvet

A bare-metal Kubernetes cluster at home, run entirely from this repo.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Flux](https://img.shields.io/badge/Flux-5468FF?style=flat-square&logo=flux&logoColor=white)](https://fluxcd.io)
[![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=flat-square&logo=authentik&logoColor=white)](https://goauthentik.io)
[![Renovate](https://img.shields.io/badge/Renovate-enabled-1A1F6C?style=flat-square&logo=renovate&logoColor=white)](https://docs.renovatebot.com)

</div>

---

Four small machines, one Git repo. Everything the cluster runs lives under `cluster/`, and Flux keeps the cluster in sync with `main`.

## Hardware

| Node | Role | CPU | RAM | OS |
|:--|:--|:--|:--|:--|
| `mpc01` | control plane | 12 cores | 32 GB | Ubuntu 24.04 |
| `mpc02` | worker | 12 cores | 32 GB | Ubuntu 24.04 |
| `mpc03` | worker | 12 cores | 32 GB | Ubuntu 24.04 |
| `mpc04` | worker | 16 cores | 32 GB | Ubuntu 24.04 |

## Overview

Requests come in at the top and work their way down. Each layer relies on the one below it.

```mermaid
flowchart TB
    users(["Internet · LAN"]):::entry
    git(["GitHub · main"]):::entry

    subgraph edge["Front door"]
        direction LR
        mlb["MetalLB"]:::front --> traefik["Traefik ×2"]:::front
    end

    apps{{"Self-hosted apps"}}:::app

    subgraph platform["Platform"]
        direction LR
        ak["Authentik"]:::plat ~~~ kuma["Uptime Kuma"]:::plat ~~~ certs["cert-manager"]:::plat ~~~ secrets["External Secrets"]:::plat
    end

    subgraph storage["Storage"]
        direction LR
        lh["Longhorn"]:::store ~~~ sw["SeaweedFS"]:::store ~~~ nfs["NFS"]:::store ~~~ pg["CloudNativePG"]:::store
    end

    users ==> edge ==> apps ==> platform ==> storage
    git -. Flux .-> edge

    classDef entry fill:#1F2937,stroke:#9CA3AF,color:#F9FAFB
    classDef front fill:#0EA5E9,stroke:#0369A1,color:#FFFFFF
    classDef app   fill:#8B5CF6,stroke:#5B21B6,color:#FFFFFF
    classDef plat  fill:#F97316,stroke:#C2410C,color:#FFFFFF
    classDef store fill:#10B981,stroke:#047857,color:#FFFFFF

    style edge     fill:#0EA5E914,stroke:#0EA5E9,stroke-width:2px
    style platform fill:#F9731614,stroke:#F97316,stroke-width:2px
    style storage  fill:#10B98114,stroke:#10B981,stroke-width:2px
    linkStyle default stroke:#9CA3AF,stroke-width:2px
```

| Layer | Component | What it does |
|:--|:--|:--|
| Sync | Flux | Watches `main` and applies it. Pings Discord when something changes |
| Front door | MetalLB | LoadBalancer IPs on the LAN |
| Front door | Traefik ×2 | One for the internet, one for the LAN |
| Platform | Authentik | Single sign-on for everything |
| Platform | Uptime Kuma | Status page and monitors for every service |
| Platform | cert-manager | Free Let's Encrypt certificates, renewed automatically |
| Platform | External Secrets + proton-relay | Pulls every secret out of Proton Pass |
| Platform | kube-network-policies | Enforces NetworkPolicies so namespaces can't talk freely |
| Storage | Longhorn | Replicated disks for anything that needs to keep data |
| Storage | SeaweedFS | Internal S3 storage, used for backups |
| Storage | NFS | Big shared storage for bulk data |
| Storage | CloudNativePG | Postgres databases (Authentik's lives here) |

## Networking

Two Traefik instances, each with its own IP:

- **traefik-external** serves public sites over HTTPS, behind bot blocking, fail2ban and rate limits.
- **traefik-internal** serves sites on the home network.

All routes are Traefik `IngressRoute`s that select their Traefik with `spec.ingressClassName`; regular Kubernetes Ingress is off. Public routes carry their own `certificate.yaml`.

### Network policies

App namespaces are default-deny, with ingress and egress allowed per app. Policies live next to each app as `networkpolicy.yaml`.

## Secrets

Nothing secret is stored in Git. Secrets live in a Proton Pass vault, and [proton-relay](https://github.com/stianmjo/proton-relay) hands them to External Secrets, which turns them into Kubernetes Secrets.

```
Proton Pass → proton-relay → External Secrets → Kubernetes Secret
```

## Updates

Renovate runs nightly inside the cluster and opens a PR for every new version, majors included. It reads chart versions in `HelmRelease`s, `image:` lines, and `image.repository` / `image.tag` in `HelmRelease` values. Versions it can't see carry a comment on the line above:

```yaml
# renovate: datasource=github-tags depName=tomMoulard/fail2ban
version: v0.9.0
```

## Directories

```
cluster/
├── flux/              Flux itself and Discord alerts
├── network-policies/  kube-network-policies agent
├── metalllb/          IP pool for LoadBalancers
├── traefik/           Internet and LAN routers
├── cert-manager/      Let's Encrypt via Domeneshop DNS
├── external-secrets/  External Secrets and proton-relay
├── authentik/         Single sign-on and its database
├── longhorn/          Replicated disks
├── seaweedfs/         S3 storage
├── nfs-provisioner/   NFS storage
├── cnpg/              Postgres operator
├── renovate/          Nightly dependency updates
├── template/          Starting point for a new app
└── …                  The apps
```
