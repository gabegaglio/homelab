# homelab

A Proxmox-based homelab (Nov. 2025 – present) running 10 LXC containers on a single node (`dev1`), behind a baremetal OPNsense router. Single-purpose containers: app stacks run as Docker-in-LXC, infrastructure runs on the host.

## Network architecture

- **OPNsense (baremetal, FreeBSD)** — edge router at `192.168.1.1`. DHCP, DNS routing, firewall rules. Enforces segmentation between internal-only services and externally exposed ones.
- **Managed switch with dual-bridge VLAN topology** — Proxmox carries two bridges, one per VLAN:
  - **`vmbr0`** (`192.168.1.0/24`) — internal services: `obsidian` (200), `pihole` (300), `pbs` (500). Host on `192.168.1.2`.
  - **`vmbr2`** (`10.10.20.0/24`) — externally-fronted / isolated services: `nousena` (100), `mc-server` (101), `mc-modded` (103), `immich` (400).
  - **IP convention:** the LAN host octet is `floor(CTID/10)` — e.g., CT 200 → `.20`, CT 300 → `.30`, CT 100 → `.10`. `pct list` lines up with `ip a` by inspection.
- **Tailscale VPN mesh** — dev1 advertises as exit node; IP forwarding persisted in `/etc/sysctl.d/99-tailscale.conf`. DERP relay handles restrictive networks. Pi-hole DNS pushed mesh-wide.
- **Cloudflare Tunnel** — only inbound path from the public internet; fronts [nousena.com](https://nousena.com) from CT 100. No WAN ports opened.
- **Pi-hole + dnscrypt-proxy** (CT 300) — network-wide DNS filtering; upstream resolution encrypted to Cloudflare DoH. Local DNS records resolve `immich`, `obsidian`, `dev1` mesh-wide.
- **Nginx reverse proxy** on dev1 — single Tailscale hop to LAN services:

  | Hostname | Port | → | Backend |
  |---|---|---|---|
  | `pihole`   | 8080 | → | `192.168.1.30:80` (CT 300) |
  | `obsidian` | 8081 | → | `192.168.1.31:3001` HTTPS (CT 201, Selkies in Docker) |
  | `dev1`     | 2283 | → | `10.10.20.40:2283` (CT 400, Immich on vmbr2) |

## Container map

| CTID | Name | Bridge | Role |
|------|------|--------|------|
| 100  | [nousena](#100--nousena)         | vmbr2 | Docker host — [nousena.com](https://nousena.com) dashboard stack |
| 101  | [mc-server](#101--mc-server)     | vmbr2 | Paper Minecraft server (custom plugin: [lumberjack](https://github.com/gabegaglio/lumberjack)) |
| 103  | [mc-modded](#103--mc-modded)     | vmbr2 | Modded Fabric Minecraft server |
| 200  | [obsidian](#200--obsidian)       | vmbr0 | Obsidian vault — source of truth, Syncthing send-receive |
| 201  | [obsidian-ui](#201--obsidian-ui) | vmbr0 | Obsidian web UI (Docker-in-LXC, Selkies/WebRTC), receive-only |
| 300  | [pihole](#300--pihole)           | vmbr0 | Pi-hole v6 DNS + dnscrypt-proxy → Cloudflare DoH |
| 400  | [immich](#400--immich)           | vmbr2 | Immich photo server (Docker compose) |
| 500  | [pbs](#500--pbs)                 | vmbr0 | Proxmox Backup Server 4.2 |
| 600  | [home-assistant](#600--home-assistant) | vmbr0 | Home Assistant |
| 700  | [splunk](#700--splunk)           | vmbr0 | Splunk Enterprise 10.4 — log indexer for the fleet |

## Host (`dev1`)

Proxmox node. Acts as the control plane:

- **Nginx reverse proxy** — see network table above
- **Tailscale exit node** — advertises exit-node, pushes Pi-hole DNS
- **Telegraf** — runs directly on the host (needs raw cgroup/disk/net visibility). 30s interval, InfluxDB line protocol, output to `/var/log/proxmox_metrics/telegraf_metrics.log`. Captures per-LXC `veth` interfaces so per-container network throughput is recoverable from the host's view alone.
- **Per-LXC metric scripts:**
  - `/usr/local/bin/proxmox-lxc-telegraf.sh` — pulls CPU/mem via the Proxmox API (`pvesh get /nodes/.../lxc/<ctid>/status/current`) and emits InfluxDB line protocol. No `pct exec` overhead, honors container cgroup limits.
  - `/usr/local/bin/lxc-metrics.sh` — plain-text dump for grep/awk.
- **Vault bind-mount** — `/data/vault` mounted from CT 200, symlinked at `/vault`.
- **`ct` wrapper** — thin shim over `pct exec`.
- **[vigosk](https://github.com/gabegaglio/vigosk)** — homelab monitor running directly on dev1 (not in any container). Independent of Splunk; watches container state, host health, and service liveness across the node.

```bash
ct list                          # status of all containers
ct <name> <command>              # exec inside a container
pct exec <ctid> -- <command>     # raw form, works for any CTID
```

## Observability stack

Splunk (logs) + Telegraf (metrics). Architecture:

```
containers ── tcp:9997 ──▶  Splunk indexer (CT 700)  ──▶ dashboards & SPL
Proxmox host ──────────▶  Telegraf  ──▶ /var/log/proxmox_metrics/*.log
```

- **Indexer** (CT 700) — Splunk Enterprise 10.4, free tier (500 MB/day). Receiver on `:9997`, web on `:8000`, mgmt on `:8089`.
- **Per-source indexes** in `/opt/splunk/etc/apps/search/local/indexes.conf`: `dev1`, `nousena`, `mc-vanilla`, `mc-modded`, `pihole`.
- **Universal Forwarders** on CT 101, 103, 200, 300 — tail `/var/log` and ship over TCP to `:9997`. Deployed in one pass via `/root/splunk.sh`, which also re-binds the systemd unit to the `splunk` user (the installer leaves it as root).
- **Example SPL:**

  ```spl
  index=pihole status=2 | top 10 domain                 # most-blocked domains
  index=mc-vanilla "joined the game" | timechart count  # player joins over time
  index=dev1 sourcetype=linux_secure "Failed password"  # bruteforce attempts
  ```

---

### 100 — nousena

Privileged LXC running Docker. Hosts the [nousena.com](https://nousena.com) dashboard stack: FastAPI backend, React/Vite frontend, MariaDB, separate prod and dev environments, fronted by a Cloudflare tunnel.

| Container | Role |
|-----------|------|
| `nousena-frontend` / `nousena-api` / `nousena-db` | Prod stack |
| `income-frontend-dev` / `income-api-dev` / `income-db-dev` | Dev stack |
| `cloudflared` | Cloudflare tunnel → public DNS |

Application lives at `/opt/dashboard/`. The `dashrun` script wraps common docker compose flows for both environments.

### 101 — mc-server

Vanilla Paper Minecraft server, runs as `minecraft.service` (systemd). Regularly hosts 10+ concurrent players. A Splunk universal forwarder ships game logs to CT 700 under `index=mc-vanilla`. Backups and monitoring use dedicated shell scripts on the host, not the Splunk pipeline.

Runs a custom Java plugin — **[lumberjack](https://github.com/gabegaglio/lumberjack)** — that chains tree-chopping in-game (fell the trunk, the whole tree comes down).

### 103 — mc-modded

Modded Fabric Minecraft server. Same operational pattern as `mc-server`; logs to `index=mc-modded`. Mod loader is Fabric rather than Paper, so the lumberjack Paper plugin doesn't apply here.

### 200 — obsidian

Unprivileged LXC. Holds the canonical Obsidian vault at `/vault` and runs Syncthing (send-receive) as the `syncthing` user — never as root. UID-mapped to `101000:101000` to match the unprivileged container.

```
/vault/
├── notes/       # human-only, never written by automation
├── claude/      # skills/, runbooks/, hosts/
├── memories/    # per-host auto + long-term memory (Syncthing-replicated)
└── shared/      # read-only reference
```

**Knowledge pipeline:** multi-node Claude Code setup with Gemini embeddings (`gemini-embedding-2-preview`) over the shared vault. A vault watcher on dev1 (`~/.local/bin/vault-watcher`) re-embeds on any `.md` change; a PostToolUse hook re-embeds after Edit/Write; the `claude-memory` MCP server (`~/.claude/memory/server.py`) serves semantic search across the vault and per-host memories.

**Syncthing mesh** — multiple endpoints with role-based access:
- `obsidian` (CT 200) — send-receive, source of truth
- `obsidian-ui` (CT 201) — receive-only mirror
- workstation + mobile clients — send-receive

### 201 — obsidian-ui

Receive-only Syncthing consumer. Runs the Obsidian web app inside Docker-in-LXC via Selkies (WebRTC). Reverse-proxied through nginx on dev1 at `obsidian:8081`.

### 300 — pihole

Pi-hole v6 for network-wide DNS filtering. Upstream resolution goes through `dnscrypt-proxy` → Cloudflare DoH so queries leave the LAN encrypted. Local DNS records live in `/etc/pihole/pihole.toml` under the `[dns]` section; FTL log shipped to Splunk under `index=pihole`.

### 400 — immich

Privileged LXC running the Immich photo server via Docker compose:

```
immich_server  immich_postgres  immich_machine_learning  immich_redis
```

LXC rootfs on NVMe (24 GB), media + Postgres on a 500 GB HDD-backed mount at `/mnt/immich`. Reachable at `dev1:2283` (LAN/Tailscale) and a separate HTTP port for the mobile app (which rejects self-signed certs).

### 500 — pbs

Proxmox Backup Server 4.2. Rootfs and 200 GB datastore (`main`) both live on the safe HDD — backup target is independent of NVMe failure. PVE authenticates with an API token (`root@pam!pve-backup`) granted `DatastoreBackup` on `/datastore/main`.

```bash
vzdump <CTID> --storage pbs-main --mode snapshot
pct restore <newCTID> pbs-main:backup/ct/<srcCTID>/<timestamp>
```

### 600 — home-assistant

Home Assistant for smart-home automation and device control.

### 700 — splunk

Splunk Enterprise 10.4 (`/opt/splunk`, `Splunkd.service`, web on `:8000`, mgmt on `:8089`, receiver on `:9997`). Runs as the `splunk` user under a systemd-managed unit. Dashboards built with `splunk-dashboard-studio` and `splunk-ai-canvas`. See [Observability stack](#observability-stack) above for the indexer/forwarder/index layout and the deploy script.

## Conventions

- **Container ops** — always `pct exec`, never `pct enter` (loses host shell context)
- **Syncthing** — runs as the `syncthing` user inside CT 200 / 201, never as root
- **Vault writes** — Claude-authored content goes under `/vault/claude/`; `/vault/notes/` is human-only
- **Skills & runbooks** — single source of truth at `/vault/claude/skills/{hosts,runbooks}/`, reindexed by the vault-watcher + PostToolUse hook into the `claude-memory` MCP server
- **Topology drift** — `CLAUDE.md` container maps are cross-checked against `pct list` and the vault memory index at session start
