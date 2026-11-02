# homelab-nebulahvelvet

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Flux](https://img.shields.io/badge/Flux-5468FF?style=flat-square&logo=flux&logoColor=white)](https://fluxcd.io)
[![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=flat-square&logo=authentik&logoColor=white)](https://goauthentik.io)

> A bare-metal Kubernetes cluster at home, run entirely from this repo.

Four small machines, one Git repo. Everything the cluster runs lives under `cluster/`, and Flux keeps the cluster in sync with `main`. Want to change something? Change the YAML and push.

## The big picture

Requests come in at the top and work their way down. Each layer leans on the one below it.

```mermaid
flowchart TB
    users(["Internet and LAN"])

    subgraph cluster["Kubernetes cluster, synced from GitHub by Flux"]
        direction TB

        subgraph edge["Front door"]
            direction LR
            mlb["MetalLB<br/>hands out IPs"] --> traefik["Traefik<br/>routes requests"]
        end

        apps["Apps"]

        subgraph platform["Platform"]
            direction LR
            ak["Authentik<br/>login"] ~~~ kuma["Uptime Kuma<br/>status"] ~~~ certs["cert-manager<br/>TLS"] ~~~ secrets["External Secrets<br/>secrets"] ~~~ netpol["Network policies<br/>firewall"]
        end

        subgraph storage["Storage"]
            direction LR
            lh["Longhorn<br/>disks"] ~~~ sw["SeaweedFS<br/>S3"] ~~~ nfs["NFS<br/>media"] ~~~ pg["CloudNativePG<br/>Postgres"]
        end

        edge --> apps --> platform --> storage
    end

    users --> edge
```

## What's running

| Layer | Piece | What it does |
|:--|:--|:--|
| Sync | Flux | Watches `main` and applies it. Pings Discord when something changes |
| Front door | MetalLB | Gives Traefik real LAN IPs (`10.0.2.82` public, `10.0.2.83` internal) |
| Front door | Traefik ×2 | One for the internet, one for the LAN |
| Platform | Authentik | Single sign-on for everything |
| Platform | Uptime Kuma | Status page and monitors for every service |
| Platform | cert-manager | Free Let's Encrypt certificates, renewed automatically |
| Platform | External Secrets + proton-relay | Pulls every secret out of Proton Pass |
| Platform | kube-network-policies | Enforces NetworkPolicies so namespaces can't talk freely |
| Storage | Longhorn | Replicated disks for anything that needs to keep data |
| Storage | SeaweedFS | S3 storage, used for backups |
| Storage | NFS | Big shared storage for media |
| Storage | CloudNativePG | Postgres databases (Authentik's lives here) |

## How traffic gets in

There are two Traefik instances, each with its own IP:

- **traefik-external** handles public `*.nebulahvelvet.no` sites over HTTPS. Every route passes bot blocking, fail2ban and rate limits before it reaches an app.
- **traefik-internal** handles `*.local.net` sites on the home network over plain HTTP.

Routes are Traefik `IngressRoute`s; regular Kubernetes Ingress is switched off. Each route picks its Traefik with `kubernetes.io/ingress.class`, and public routes bring their own `certificate.yaml`. To add one, copy [cluster/template/ingressRoute.yaml](cluster/template/ingressRoute.yaml).

## Network policies

Every app namespace starts closed. Nothing can reach its pods except:

- pods in the same namespace
- the Traefik instance that serves it, on the app's own port only

The secrets path is locked down the hardest. Only External Secrets can talk to proton-relay, and the relay can only make DNS lookups and HTTPS calls out to Proton. Uptime Kuma gets a separate health-only port so it can watch the relay without reaching any secrets.

Policies live next to each app as `networkpolicy.yaml`. If something breaks, reverting that one file's commit fixes it.

## Secrets

Nothing secret is stored in Git. Secrets live in a Proton Pass vault, and [proton-relay](https://github.com/stianmjo/proton-relay) hands them to External Secrets, which turns them into Kubernetes Secrets.

```
Proton Pass → proton-relay → External Secrets → Kubernetes Secret
```

To pull a secret again right now:

```sh
kubectl annotate externalsecret <name> -n <namespace> force-sync=$(date +%s) --overwrite
```

## Updates

Renovate runs nightly inside the cluster and opens a PR for every new version. It finds most versions by itself:

- chart versions in `HelmRelease`s
- `image:` lines, and `image.repository` / `image.tag` inside `HelmRelease` values

So the rule is simple: **pin an exact version and don't annotate it.** Only when Renovate can't see a version does it get a comment on the line above:

```yaml
# renovate: datasource=github-tags depName=tomMoulard/fail2ban
version: v0.8.9
```

## Where things live

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
└── …                  The apps
```
