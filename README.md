# homelab-nebulahvelvet

<div align="center">

![Flux](https://img.shields.io/badge/GitOps-Flux_v2-5468FF?style=for-the-badge&logo=flux&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Traefik](https://img.shields.io/badge/Ingress-Traefik-24A1C1?style=for-the-badge&logo=traefikproxy&logoColor=white)
![Renovate](https://img.shields.io/badge/Updates-Renovate-1A1F6C?style=for-the-badge&logo=renovate&logoColor=white)

*A self-hosted Kubernetes homelab managed with GitOps principles.*

</div>

---

## Stack

| Layer | Tool |
|---|---|
| GitOps | [Flux v2](https://fluxcd.io) |
| Ingress | [Traefik](https://traefik.io) (internal + external instances) |
| Load Balancer | [MetalLB](https://metallb.universe.tf) (L2 mode) |
| TLS | [cert-manager](https://cert-manager.io) with DNS-01 webhook |
| Block Storage | [Longhorn](https://longhorn.io) |
| Object Storage | [SeaweedFS](https://github.com/seaweedfs/seaweedfs) |
| NFS Storage | nfs-subdir-external-provisioner |
| Identity / SSO | [Authentik](https://goauthentik.io) + CloudNativePG (Traefik ForwardAuth) |
| Notifications | Flux → Discord |
| Updates | [Renovate](https://docs.renovatebot.com) (self-hosted) |

---

## Applications

| Category | App |
|---|---|
| Personal Site | [Huginn](https://nebulahvelvet.no) — personal Next.js webpage |
| Media | Jellyfin, Sonarr, Radarr, Bazarr, Prowlarr, qBittorrent (VPN), Jellyseerr |
| Gaming | Crafty (Minecraft server manager) |
| Monitoring | Uptime Kuma |
| CI/CD | GitHub Actions self-hosted runners |
| Identity | Authentik |

---

## Repository Structure

```
cluster/
├── flux/               # FluxInstance and notification provider
├── traefik/            # Ingress (traefik-internal, traefik-external)
├── metalllb/           # MetalLB LoadBalancer
├── cert-manager/       # TLS certificate management
├── longhorn/           # Block storage + S3 backup
├── nfs-provisioner/    # NFS-backed dynamic PVC provisioner
├── seaweedfs/          # S3-compatible object store
├── huginn/             # Personal Next.js website (nebulahvelvet.no)
├── media/              # Media stack (Jellyfin, *arr, qBittorrent)
├── crafty/             # Minecraft server manager
├── uptime-kuma/        # Uptime monitoring
├── cnpg/               # CloudNativePG operator
├── authentik/          # SSO / identity provider (ForwardAuth for Traefik)
├── renovate/           # Dependency update automation
└── github-runner/      # Self-hosted GitHub Actions runners
```

---

## Secrets

Secrets are not committed to the repository. Each component that requires credentials has a `*-secret-template.yaml` file describing the expected secret shape. Secrets must be applied to the cluster manually before the relevant workload is deployed.
