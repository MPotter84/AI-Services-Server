# AI Services Server — Configuration Reference

> **Role:** AI automation and vector-search node in a self-hosted homelab.
> **Hostname:** `aiservices` | **VLAN:** 20 (192.168.20.x) | **Uptime:** 21+ days at time of writing

---

## Host Specifications

| Property | Value |
|---|---|
| **OS** | Ubuntu 22.04.5 LTS (Jammy Jellyfish) |
| **Kernel** | 5.15.0-176-generic (x86\_64) |
| **Hypervisor** | Proxmox VE (QEMU/KVM) |
| **vCPUs** | 6 |
| **RAM** | 8 GB |
| **Swap** | 4 GB swapfile (`vm.swappiness=10`) |
| **Root Disk** | ~98 GB LVM volume, ext4 |
| **Network** | Single NIC (`ens18`), DHCP on VLAN 20 |
| **Shell / Init** | bash / systemd |

---

## Docker Stack

All services are managed with **Docker Compose v2**. Three compose files provide a layered configuration:

| File | Purpose |
|---|---|
| `docker-compose.yml` | Core stack (always active) |
| `docker-compose.langsmith.yml` | Optional LangSmith overlay |
| `docker-compose.ngrok.yml` | Optional dev tunnel overlay |

### Core Services (`docker-compose.yml`)

#### nginx — Reverse Proxy
| Setting | Value |
|---|---|
| **Image** | `nginx:1.25-alpine` |
| **Ports** | `80:80`, `443:443` |
| **Memory limit** | 128 MB |
| **Restart policy** | `unless-stopped` |
| **Healthcheck** | `GET /healthz` every 30 s |

nginx is the single ingress point for all external traffic. Internal service ports are never exposed to the host directly.

#### n8n — Workflow Automation Engine
| Setting | Value |
|---|---|
| **Image** | `n8nio/n8n:latest` |
| **Internal port** | 5678 |
| **Memory limit** | 2 GB |
| **Restart policy** | `unless-stopped` |
| **Database backend** | PostgreSQL (via `DB_TYPE=postgresdb`) |
| **Execution mode** | `main` (single process, limits concurrency on 8 GB host) |
| **Binary data** | Filesystem mode |
| **Auth** | Built-in user management (n8n v1.x) with encryption key |
| **Tracing** | Optional LangSmith / LangChain tracing via env vars |

#### PostgreSQL — n8n Database
| Setting | Value |
|---|---|
| **Image** | `postgres:15-alpine` |
| **Internal port** | 5432 |
| **Memory limit** | 512 MB |
| **Restart policy** | `unless-stopped` |
| **Healthcheck** | `pg_isready` every 10 s, 5 retries |
| **Data persistence** | Docker named volume (`postgres_data`) |

#### ChromaDB — Vector Store
| Setting | Value |
|---|---|
| **Image** | `chromadb/chroma:latest` |
| **Internal port** | 8000 |
| **Memory limit** | 3 GB |
| **CPU cap** | 1.5 vCPUs (prevents embedding spikes from starving n8n) |
| **Restart policy** | `unless-stopped` |
| **Auth** | Token-based (`Authorization` header) |
| **Telemetry** | Disabled (`ANONYMIZED_TELEMETRY=FALSE`) |
| **Data persistence** | Docker named volume (`chromadb_data`) |

### Optional Overlay — LangSmith (`docker-compose.langsmith.yml`)

Adds self-hosted LangSmith for LLM run tracing. Activated with:

```bash
docker compose -f docker-compose.yml -f docker-compose.langsmith.yml up -d
```

> **Note:** Adds ~2–3 GB RAM usage. Monitor headroom with `docker stats`.

| Service | Image | Memory Limit |
|---|---|---|
| `langsmith-backend` | `langchain/langsmith-backend:latest` | 1 GB |
| `langsmith-frontend` | `langchain/langsmith-frontend:latest` | 256 MB |
| `langsmith-postgres` | `postgres:15-alpine` | 512 MB |
| `langsmith-redis` | `redis:7-alpine` | 256 MB |

### Optional Overlay — ngrok (`docker-compose.ngrok.yml`)

Exposes n8n webhooks to the internet during local development.

```bash
docker compose -f docker-compose.yml -f docker-compose.ngrok.yml up -d
```

| Setting | Value |
|---|---|
| **Image** | `ngrok/ngrok:latest` |
| **Tunnels** | `n8n:5678` → public HTTPS URL |
| **Dashboard** | Port 4040 (LAN-only) |
| **Memory limit** | 128 MB |

---

## Nginx Configuration

### Upstreams

| Upstream | Backend | Keepalive |
|---|---|---|
| `n8n` | `n8n:5678` | 32 connections |
| `chromadb` | `chromadb:8000` | 16 connections |
| `langsmith` | `langsmith-frontend:80` | 8 connections |

### Routing (HTTP :80)

| Path | Destination | Notes |
|---|---|---|
| `/` | n8n upstream | WebSocket (`Upgrade`) headers forwarded; 300 s proxy timeout |
| `/webhook/` | n8n upstream | 100 MB body limit; 120 s timeout |
| `/chroma/` | chromadb upstream | Path rewritten; 300 s timeout for slow embedding queries |
| `/langsmith/` | langsmith-frontend | Active only when LangSmith overlay is running |
| `/langsmith/api/` | langsmith-backend:1984 | 120 s timeout |
| `/healthz` | inline 200 OK | Docker healthcheck probe; access log suppressed |

**Global client\_max\_body\_size:** 50 MB (overridden to 100 MB on `/webhook/`)

### HTTPS

TLS server block is defined but commented out — ready to activate by dropping `fullchain.pem` / `privkey.pem` into `./nginx/ssl/` and enabling `TLSv1.2 + TLSv1.3`.

---

## Security

### Firewall (UFW)

| Rule | Action |
|---|---|
| Default incoming | **DENY** |
| Default outgoing | Allow |
| SSH (22/tcp) | Allow |
| HTTP (80/tcp) | Allow |
| HTTPS (443/tcp) | Allow |
| All other internal service ports | **Blocked at host** (nginx proxies internally) |

### fail2ban

Enabled and running via systemd — protects SSH from brute-force attempts.

### SSH (`sshd_config`)

| Setting | Value |
|---|---|
| `KbdInteractiveAuthentication` | `no` |
| `UsePAM` | `yes` |
| `X11Forwarding` | `yes` |
| `Subsystem` | `sftp` via `/usr/lib/openssh/sftp-server` |

### Service Authentication

- **n8n:** Built-in user management with a randomly generated encryption key; credentials injected via environment variables at runtime.
- **ChromaDB:** Token-based auth on all API requests (`Authorization` header).
- **PostgreSQL:** Password auth; port never exposed outside the Docker bridge network.

---

## Maintenance & Observability

### Automated Tasks

| Schedule | Task |
|---|---|
| Daily 03:10 | `e2scrub_all -A -r` (online filesystem check) |
| Weekly Sun 03:30 | `e2scrub_all` (full e2fsck pass) |
| systemd | `unattended-upgrades` for security patches |

### Key System Services

| Service | Role |
|---|---|
| `ssh.service` | Remote access |
| `rsyslog.service` | System logging |
| `systemd-networkd` | DHCP / interface management |
| `systemd-timesyncd` | NTP time sync |
| `fail2ban.service` | Brute-force protection |
| `cron.service` | Scheduled tasks |

### Useful Runtime Commands

```bash
# Watch container resource usage
docker stats

# View all container logs
docker compose logs -f

# Health status
docker compose ps

# ngrok public URL (when overlay active)
docker logs ngrok 2>&1 | grep url
```

---

## Bootstrap

The `setup.sh` script provisions a fresh Ubuntu 22.04 VM in a single run:

1. System package update + upgrade
2. Install Docker CE (official repo, not snap), UFW, fail2ban, htop, iotop, ncdu
3. Create 4 GB swapfile and set `vm.swappiness=10`
4. Configure UFW rules (deny-by-default)
5. Enable fail2ban
6. Create data directories (`data/{n8n,postgres,chromadb,langsmith}`, `nginx/ssl/`)
7. Copy `.env.example` → `.env` and prompt for secrets
8. Pull all Docker images

---

## Memory Budget (8 GB Host)

| Service | Limit |
|---|---|
| n8n | 2 GB |
| ChromaDB | 3 GB |
| PostgreSQL (n8n) | 512 MB |
| nginx | 128 MB |
| **Core total** | **~5.6 GB** |
| LangSmith backend | 1 GB |
| LangSmith postgres | 512 MB |
| LangSmith frontend | 256 MB |
| LangSmith redis | 256 MB |
| **With LangSmith overlay** | **~7.6 GB** |
| Host OS + headroom | ~400 MB remaining |
| Swap safety net | 4 GB |
