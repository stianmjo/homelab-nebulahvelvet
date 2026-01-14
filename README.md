# proton-bridge

Experimental Proton Pass provider for **External Secrets Operator** using the **webhook** provider.  
It runs a small HTTP API that External Secrets can call to fetch secrets from a Proton Pass vault, and exposes health/readiness endpoints suitable for Kubernetes probes and Uptime Kuma.

> **Ports**
> - `8080` = API (used by External Secrets)
> - `8081` = Monitor endpoints (`/healthz`, `/readyz`)

---

## Image

Container image:

- `ghcr.io/stianmjo/proton-bridge:<tag>`
- Example pinned tag: `ghcr.io/stianmjo/proton-bridge:v0.1.0`

---

## Configuration

### Required persistence
Proton Pass CLI stores session/state on disk. This image expects persistent storage mounted at:

- `/data`  (also treated as HOME)

**Important:**
- Use `PROTON_PASS_KEY_PROVIDER=fs` (containers typically don’t have an OS keyring).
- Ensure `/data` is writable by UID/GID **10001**.

### Basic Auth (recommended)
The API is protected with **Basic Auth**:

- `BASIC_USER`
- `BASIC_PASS`

You must configure the **same credentials** in:
1) **proton-bridge** (server-side auth)
2) **External Secrets** `SecretStore/ClusterSecretStore` webhook client (so requests include the Basic Auth header)

---

## Running with Docker

### 1) Create a local data directory
```bash
mkdir -p ./proton-bridge-data
