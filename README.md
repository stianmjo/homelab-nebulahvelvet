# homelab-nebulahvelvet

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Flux](https://img.shields.io/badge/Flux-5468FF?style=flat-square&logo=flux&logoColor=white)](https://fluxcd.io)
[![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=flat-square&logo=authentik&logoColor=white)](https://goauthentik.io)
[![Gitea](https://img.shields.io/badge/Gitea-609926?style=flat-square&logo=gitea&logoColor=white)](https://gitea.io)

> Self-hosted Kubernetes infrastructure. GitOps-managed.

A bare-metal Kubernetes cluster running on commodity hardware, fully managed through Git. Every change flows through version control — infrastructure as code.

---

## Stack

| | Layer | Component | Role |
|:-:|:--|:--|:--|
| ![](https://img.shields.io/badge/-5468FF?style=flat-square) | Core | Flux v2 | Continuous reconciliation from Git to cluster state |
| ![](https://img.shields.io/badge/-24A1C1?style=flat-square) | Core | Traefik | Dual ingress — external with TLS + internal for LAN services |
| ![](https://img.shields.io/badge/-326CE5?style=flat-square) | Core | MetalLB | Layer 2 load balancing for bare-metal environments |
| ![](https://img.shields.io/badge/-0DB7ED?style=flat-square) | Core | cert-manager | Automated certificate lifecycle via Let's Encrypt DNS-01 |
| | | | |
| ![](https://img.shields.io/badge/-FD4B2D?style=flat-square) | Identity | Authentik | Single sign-on with OIDC/SAML and Traefik ForwardAuth |
| ![](https://img.shields.io/badge/-609926?style=flat-square) | Identity | Gitea | Git hosting, Actions CI/CD, and container registry |
| | | | |
| ![](https://img.shields.io/badge/-5F259F?style=flat-square) | Data | Longhorn | Distributed block storage with replication and snapshots |
| ![](https://img.shields.io/badge/-00ADD8?style=flat-square) | Data | SeaweedFS | S3-compatible object storage for backups and artifacts |
| ![](https://img.shields.io/badge/-F5A623?style=flat-square) | Data | NFS | Shared storage for media libraries and bulk data |
| ![](https://img.shields.io/badge/-336791?style=flat-square) | Data | CloudNativePG | PostgreSQL operator for stateful database workloads |

---

## How It Works

The cluster follows a GitOps workflow — all configuration lives in this repository under `cluster/`. Flux watches the `main` branch and continuously reconciles the desired state. No manual `kubectl apply`, no configuration drift, no surprises.

**Ingress** splits into two Traefik instances: one faces the internet with TLS termination, rate limiting, and bot protection; the other serves internal services over plain HTTP on the local network.

**Identity** is centralized through Authentik, providing SSO across services. Gitea mirrors repositories from GitHub and runs CI/CD pipelines locally, pushing built images to its own container registry.

**Storage** is layered: Longhorn handles stateful workloads needing block storage, SeaweedFS provides S3 for object data, and NFS shares bulk storage across pods.

---

## Repository Structure

```
cluster/
├── flux/             Flux configuration and sync settings
├── traefik/          External and internal ingress controllers
├── cert-manager/     TLS certificate automation
├── metalllb/         Load balancer IP pool configuration
│
├── authentik/        SSO provider with CNPG PostgreSQL backend
├── gitea/            Git server, Actions runner, container registry
│
├── longhorn/         Block storage and retention policies
├── seaweedfs/        S3 object storage operator and cluster
├── nfs-provisioner/  Dynamic NFS volume provisioner
├── cnpg/             CloudNativePG operator
│
├── renovate/         Automated dependency updates
├── github-runner/    Self-hosted GitHub Actions runners
└── ...
```

---

## Secrets

Credentials are never committed to the repository. Each component that requires secrets includes a `*-secret-template.yaml` file documenting the expected structure. Apply secrets manually before deploying workloads.

---

## Bootstrap

New nodes are provisioned using [midgardssmie](https://github.com/stianmjo/midgardssmie) — an Ansible playbook that takes bare Ubuntu machines to a production-ready Kubernetes cluster with Cilium networking and Flux pre-configured.