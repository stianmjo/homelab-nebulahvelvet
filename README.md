# homelab-nebulahvelvet

GitOps repository for a self-hosted Kubernetes homelab cluster, managed with Flux CD v2.

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
| Monitoring | VictoriaMetrics · Grafana · Alloy · node-exporter |
| Identity / SSO | [Authentik](https://goauthentik.io) + CloudNativePG (Traefik ForwardAuth) |
| Notifications | Flux → Discord |
| Updates | [Renovate](https://docs.renovatebot.com) (self-hosted) |

## Applications

| Category | App |
|---|---|
| Media | Jellyfin, Sonarr, Radarr, Bazarr, Prowlarr, qBittorrent (VPN), Jellyseerr |
| Gaming | Crafty (Minecraft server manager) |
| Monitoring | Uptime Kuma |
| CI/CD | GitHub Actions self-hosted runners |
| Identity | Authentik |

## Repository structure

```
cluster/
├── flux/               # FluxInstance and notification provider
├── traefik/            # Ingress (traefik-internal, traefik-external)
├── metalllb/           # MetalLB LoadBalancer
├── cert-manager/       # TLS certificate management
├── longhorn/           # Block storage + S3 backup
├── nfs-provisioner/    # NFS-backed dynamic PVC provisioner
├── seaweedfs/          # S3-compatible object store
├── monitoring/         # VictoriaMetrics, Grafana, node-exporter
├── media/              # Media stack (Jellyfin, *arr, qBittorrent)
├── crafty/             # Minecraft server manager
├── uptime-kuma/        # Uptime monitoring
├── cnpg/               # CloudNativePG operator
├── authentik/          # SSO / identity provider (ForwardAuth for Traefik)
├── renovate/           # Dependency update automation
└── github-runner/      # Self-hosted GitHub Actions runners
```

## Secrets

Secrets are not committed to the repository. Each component that requires credentials has a `*-secret-template.yaml` file describing the expected secret shape. Secrets must be applied to the cluster manually before the relevant workload is deployed.
