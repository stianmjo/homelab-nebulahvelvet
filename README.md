# homelab-nebulahvelvet

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Flux](https://img.shields.io/badge/Flux-5468FF?style=flat-square&logo=flux&logoColor=white)](https://fluxcd.io)
[![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=flat-square&logo=authentik&logoColor=white)](https://goauthentik.io)

> Bare-metal Kubernetes, reconciled from Git.

Everything under `cluster/` is the desired state. Flux watches `main` and applies it, so the repo is the only way in.

## Core Platform

```mermaid
flowchart LR
    git[("GitHub<br/>main")]:::src --> flux{{"Flux"}}:::gitops

    subgraph net["Network"]
        direction TB
        mlb["MetalLB · L2<br/>10.0.2.80–90"]
        ext["Traefik external<br/>:443 · *.nebulahvelvet.no"]
        int["Traefik internal<br/>:80 · *.local.net"]
        mlb -- ".82" --> ext
        mlb -- ".83" --> int
    end

    subgraph sec["Identity & Trust"]
        direction TB
        ak["Authentik<br/>SSO · OIDC · ForwardAuth"]
        cm["cert-manager<br/>Let's Encrypt DNS-01"]
        eso["External Secrets"]
        relay["proton-relay"]
        relay --> eso
    end

    subgraph data["Storage"]
        direction TB
        lh[("Longhorn<br/>replicated block")]
        sw[("SeaweedFS<br/>S3 object")]
        nfs[("NFS<br/>bulk / media")]
        pg[("CloudNativePG<br/>Postgres")]
    end

    internet((Internet)):::edge --> mlb
    lan((LAN)):::edge --> mlb
    pass[["Proton Pass"]]:::src --> relay
    dns[["Domeneshop DNS"]]:::src --> cm

    ext -- "middleware chain" --> ak
    cm -- "TLS secrets" --> ext
    ak --> pg
    pg -- "nightly dump" --> sw
    int --> sw

    flux -.-> net & sec & data

    classDef src fill:#1f2937,stroke:#6b7280,color:#f9fafb
    classDef edge fill:#0f766e,stroke:#14b8a6,color:#f0fdfa
    classDef gitops fill:#5468ff,stroke:#3b4fe0,color:#fff
    style net fill:#24a1c114,stroke:#24a1c1
    style sec fill:#fd4b2d14,stroke:#fd4b2d
    style data fill:#f59e0b14,stroke:#f59e0b
```

| Layer | Component | Role |
|:--|:--|:--|
| GitOps | Flux v2 | Reconciles `cluster/` from `main`, alerts to Discord |
| Network | MetalLB | L2 LoadBalancer IPs from `10.0.2.80–90` |
| Network | Traefik ×2 | External (TLS, bot wrangler, fail2ban, rate limits) and internal LAN, routed by `IngressRoute` only |
| Trust | cert-manager | Let's Encrypt via DNS-01 on Domeneshop |
| Trust | External Secrets + proton-relay | Every secret pulled from Proton Pass |
| Auth | Authentik | SSO with OIDC, SAML and Traefik ForwardAuth |
| Storage | Longhorn | Replicated block storage; `longhorn-retain` for state that must survive |
| Storage | SeaweedFS | S3 object storage for backups |
| Storage | NFS | `nfs-odin` class for media and bulk data |
| Storage | CloudNativePG | Postgres operator, backing Authentik |

## Routing

Two Traefik instances split the edge. **traefik-external** takes `:443` for public hosts. It terminates TLS with cert-manager certificates, and every route passes a middleware chain (bot wrangler, fail2ban, rate and in-flight limits). **traefik-internal** serves `*.local.net` over plain HTTP on the LAN.

Routes are Traefik `IngressRoute` CRDs; the Kubernetes Ingress provider is disabled. Each route carries `kubernetes.io/ingress.class: traefik-external` or `traefik-internal` to pick its instance. Public routes ship their own `Certificate`. Start from [cluster/template/ingressRoute.yaml](cluster/template/ingressRoute.yaml).

## Secrets

Nothing sensitive lives in Git. Secrets come from a Proton Pass vault through [proton-relay](https://github.com/stianmjo/proton-relay) and into the cluster via External Secrets.

```
Proton Pass → proton-relay → ClusterSecretStore → ExternalSecret → Secret
```

Force a resync:

```sh
kubectl annotate externalsecret <name> -n <namespace> force-sync=$(date +%s) --overwrite
```

## Layout

```
cluster/
├── flux/              Flux instance and Discord alerts
├── metalllb/          IP pool and L2 advertisement
├── traefik/           External and internal ingress controllers
├── cert-manager/      ClusterIssuer and Domeneshop DNS-01 webhook
├── external-secrets/  ESO and the proton-relay bridge
├── authentik/         SSO with CNPG Postgres and S3 backups
├── longhorn/          Block storage
├── seaweedfs/         S3 object storage
├── nfs-provisioner/   NFS storage class
├── cnpg/              CloudNativePG operator
└── …                  Workloads on top of the platform
```
