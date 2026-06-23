# homelab

A Proxmox-based homelab running 10 LXC containers on a single node (`dev1`). Each container is a single-purpose unit: app stacks run in Docker-in-LXC, infrastructure runs on the host.

## Topology

| CTID | Name | Role |
|------|------|------|
| 100  | [nousena](#100--nousena) | Docker host for the income/dashboard stack — [nousena.com](https://nousena.com) |
| 101  | [mc-server](#101--mc-server) | Paper Minecraft server, managed by [lumberjack](https://github.com/gabegaglio/lumberjack) |
| 103  | [mc-modded](#103--mc-modded) | Modded Fabric Minecraft server, managed by [lumberjack](https://github.com/gabegaglio/lumberjack) |
| 200  | [obsidian](#200--obsidian) | Obsidian vault — source of truth, Syncthing send-receive |
| 201  | [obsidian-ui](#201--obsidian-ui) | Obsidian web UI (Docker-in-LXC), Syncthing receive-only |
| 300  | [pihole](#300--pihole) | Pi-hole v6 DNS + dnscrypt-proxy → Cloudflare DoH |
| 400  | [immich](#400--immich) | Immich photo server (Docker compose) |
| 500  | [pbs](#500--pbs) | Proxmox Backup Server 4.2 |
| 600  | [home-assistant](#600--home-assistant) | Home Assistant |
| 700  | [splunk](#700--splunk) | Splunk Enterprise 10.4 — central log analytics |

## Host (`dev1`)

Proxmox node. Everything else is an LXC underneath. Acts as the control plane:

- **Nginx reverse proxy** — fronts pihole (`:8080`), obsidian-ui (`:8081`), immich (`:2283`)
- **Tailscale exit node** — pushes Pi-hole DNS to every device; DERP relay configured
- **Vault bind-mount** — `/data/vault` is mounted from CT 200 and symlinked at `/vault`
- **`ct` wrapper** — thin shim around `pct exec` for friendlier container ops
- **[vigosk](https://github.com/gabegaglio/vigosk)** — homelab monitor running directly on dev1 (not in any container). Independent of Splunk; watches container state, host health, and service liveness across the node.

```bash
ct list                          # status of all containers
ct <name> <command>              # exec inside a container
pct exec <ctid> -- <command>     # raw form, works for any CTID
```

---

### 100 — nousena

Privileged LXC running Docker. Hosts the [nousena.com](https://nousena.com) dashboard stack — a FastAPI backend, React/Vite frontend, MariaDB, with separate prod and dev environments, fronted by a Cloudflare tunnel.

| Container | Role |
|-----------|------|
| `nousena-frontend` / `nousena-api` / `nousena-db` | Prod stack |
| `income-frontend-dev` / `income-api-dev` / `income-db-dev` | Dev stack |
| `cloudflared` | Cloudflare tunnel → public DNS |

Application lives at `/opt/dashboard/`. The `dashrun` script wraps common docker compose flows for both environments.

### 101 — mc-server

Vanilla Paper Minecraft server. Runs as a `systemd` unit (`minecraft.service`). A `splunkforwarder` package ships game logs to CT 700 for analysis. Backups and monitoring are handled by **[lumberjack](https://github.com/gabegaglio/lumberjack)**.

### 103 — mc-modded

Modded Fabric Minecraft server. Same operational pattern as `mc-server` — also wired into **[lumberjack](https://github.com/gabegaglio/lumberjack)**.

### 200 — obsidian

Unprivileged LXC. Holds the canonical Obsidian vault at `/vault` and runs Syncthing (send-receive) as the `syncthing` user — never as root. The vault is the **source of truth** for notes, Claude skills, runbooks, and per-host memory.

```
/vault/
├── notes/       # human-only, never written by automation
├── claude/      # skills, hosts/, runbooks/
├── memories/    # per-host auto + long-term memory
└── shared/      # read-only reference
```

### 201 — obsidian-ui

Receive-only Syncthing node serving the Obsidian web UI inside Docker-in-LXC (WebRTC). Read-only mirror of CT 200.

### 300 — pihole

Pi-hole v6 for network-wide DNS filtering. Upstream resolution goes through `dnscrypt-proxy` → Cloudflare DoH so queries leave the LAN encrypted. Local A records resolve `immich`, `obsidian`, `dev1` to the dev1 Tailscale IP.

### 400 — immich

Privileged LXC running the Immich photo server via Docker compose:

```
immich_server  immich_postgres  immich_machine_learning  immich_redis
```

App data on NVMe; media library on a dedicated HDD. Reverse-proxied at `:2283`.

### 500 — pbs

Proxmox Backup Server 4.2. Datastore lives on a separate "safe" HDD — snapshot/restore target for every other CT on dev1.

### 600 — home-assistant

Home Assistant for smart-home automation and device control.

### 700 — splunk

Splunk Enterprise 10.4 (`/opt/splunk`, `Splunkd.service`, web on `:8000`, mgmt on `:8089`). Central log analytics for the homelab — Minecraft servers ship logs via `splunkforwarder`; system logs are collected through the `journald_input` app. Dashboards built with `splunk-dashboard-studio` and `splunk-ai-canvas`.

## Conventions

- **Container ops** — always `pct exec`, never `pct enter` (loses host shell context)
- **Syncthing** — runs as the `syncthing` user inside CT 200 / 201, never as root
- **Vault writes** — Claude-authored content goes under `/vault/claude/`; `/vault/notes/` is human-only
- **Skills & runbooks** — single source of truth at `/vault/claude/skills/{hosts,runbooks}/`, reindexed automatically by a vault-watcher + PostToolUse hook feeding the `claude-memory` MCP server
