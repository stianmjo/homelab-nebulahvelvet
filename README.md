# homelab-nebulahvelvet

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Flux](https://img.shields.io/badge/Flux-5468FF?style=flat-square&logo=flux&logoColor=white)](https://fluxcd.io)
[![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=flat-square&logo=authentik&logoColor=white)](https://goauthentik.io)
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white)](https://prometheus.io)

> Self-hosted Kubernetes infrastructure. GitOps-managed.

A bare-metal Kubernetes cluster running on commodity hardware, fully managed through Git. Every change flows through version control as infrastructure as code.

## Stack

| Layer | Component | Role |
|:--|:--|:--|
| Core | Flux v2 | Continuous reconciliation from Git to cluster state |
| Core | Traefik | Dual ingress, external with TLS and internal for LAN services |
| Core | MetalLB | Layer 2 load balancing for bare metal |
| Core | cert-manager | Certificate lifecycle via Let's Encrypt DNS-01 |
| Core | External Secrets Operator | Secret sync from Proton Pass through the proton-relay bridge |
| Identity | Authentik | Single sign-on with OIDC/SAML and Traefik ForwardAuth |
| Data | Longhorn | Distributed block storage with replication and snapshots |
| Data | SeaweedFS | S3-compatible object storage for backups and artifacts |
| Data | NFS | Shared storage for media libraries and bulk data |
| Data | CloudNativePG | PostgreSQL operator for stateful database workloads |
| Observability | Prometheus and Grafana | Cluster metrics collection and dashboards |
| Observability | Splunk | Log analytics and security event search |
| Observability | Uptime Kuma | Service and certificate uptime monitoring |
| Ops | Renovate | Automated dependency updates |
| Ops | GitHub Runners | Self-hosted Actions runners for CI/CD |
| Ops | apt-scanner | Scheduled CVE and package audit across nodes, reported to Discord |

## How It Works

All configuration lives in this repository under `cluster/`. Flux watches the `main` branch and continuously reconciles the desired state, so there is no manual `kubectl apply` and no configuration drift.

**Ingress** splits into two Traefik instances. One faces the internet with TLS termination, rate limiting, and bot protection. The other serves internal services over plain HTTP on the local network.

**Identity** is centralized through Authentik, providing SSO across services with OIDC, SAML, and Traefik ForwardAuth.

**Storage** is layered. Longhorn handles stateful workloads needing block storage, SeaweedFS provides S3 for object data, NFS shares bulk storage across pods, and CloudNativePG runs the Postgres databases.

**Observability** covers metrics, logs, and availability. Prometheus and Grafana collect and visualize cluster metrics, Splunk indexes logs for search and security analysis, and Uptime Kuma tracks service and certificate health.

**Workloads** on top of the platform include a media automation stack and a Minecraft server, all served through internal ingress.

**Ops** is automated where possible. apt-scanner audits upgradable packages against the Ubuntu CVE API and posts to Discord, Renovate keeps dependencies current, and self-hosted runners execute CI/CD.

## Repository Structure

```
cluster/
├── flux/             Flux configuration and sync settings
├── traefik/          External and internal ingress controllers
├── metalllb/         Load balancer IP pool configuration
├── cert-manager/     TLS certificate automation
├── external-secrets/ External Secrets Operator and the proton-relay bridge
│
├── authentik/        SSO provider with CNPG PostgreSQL backend
│
├── longhorn/         Block storage and retention policies
├── seaweedfs/        S3 object storage operator and cluster
├── nfs-provisioner/  Dynamic NFS volume provisioner
├── cnpg/             CloudNativePG operator
│
├── monitoring/       Prometheus, Grafana, and Alertmanager
├── splunk/           Log analytics and security event search
├── uptime-kuma/      Uptime and certificate monitoring
│
├── media/            Media automation stack
├── crafty/           Minecraft server controller
│
├── apt-scanner/      Node CVE and package audit CronJob
├── renovate/         Automated dependency updates
└── github-runner/    Self-hosted GitHub Actions runners
```

## Secrets

Credentials are never committed to the repository. Secrets are managed through two complementary patterns.

**Bootstrap secrets** are credentials required before workloads can start, such as PATs, database passwords, and API keys. They are applied manually with `kubectl create secret`. Each component that needs one includes a `*-secret-template.yaml` documenting the expected structure and key names.

**Runtime secrets** are managed by [External Secrets Operator](https://external-secrets.io) via [proton-relay](https://github.com/stianmjo/proton-relay), a lightweight bridge that pulls secrets from Proton Pass vaults into Kubernetes Secrets on a configurable refresh interval.

```
Proton Pass vault → proton-relay → ESO ClusterSecretStore → ExternalSecret → Kubernetes Secret
```

To force an immediate sync of a secret:

```sh
kubectl annotate externalsecret <name> -n <namespace> force-sync=$(date +%s) --overwrite
```