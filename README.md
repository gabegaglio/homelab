# homelab

Self-hosted infrastructure on Proxmox VE — 10 LXC containers across a segmented network behind a baremetal OPNsense router. Built Nov. 2025 – present.

Production-style operations: VLAN segmentation, encrypted DNS, zero-WAN-ingress public services via Cloudflare Tunnel, full-fleet log aggregation with Splunk + Universal Forwarders, host-level metrics with Telegraf, automated snapshot backups to a physically separate disk, and a multi-node knowledge pipeline backed by semantic search.

## Stack

| Layer | Tooling |
|---|---|
| Virtualization | Proxmox VE · LXC · Docker-in-LXC |
| Networking | OPNsense (FreeBSD) edge router · managed switch · dual-VLAN topology · Tailscale mesh + exit node · Cloudflare Tunnel |
| DNS & security | Pi-hole v6 · dnscrypt-proxy → Cloudflare DoH · nginx reverse proxy · self-signed TLS |
| Observability | Splunk Enterprise 10.4 + Universal Forwarders · Telegraf (InfluxDB line protocol) |
| Backups | Proxmox Backup Server 4.2 on a separate disk · API-token auth · RBAC |
| Applications | FastAPI · React + Vite · MariaDB · Immich · Home Assistant · Paper / Fabric Minecraft |
| AI tooling | Claude Code · Gemini embeddings · custom MCP server · Syncthing-replicated knowledge vault |
| Languages | Python · TypeScript · Bash · Java |

## Network architecture

Edge router is OPNsense on baremetal (FreeBSD), handling DHCP, DNS, firewall, and L3 segmentation. A managed switch enforces two VLANs at L2; Proxmox carries both as separate bridges:

- **`vmbr0` — internal LAN (`192.168.1.0/24`)** · Obsidian vault, Pi-hole, Proxmox Backup Server, Splunk, Home Assistant
- **`vmbr2` — external / isolated (`10.10.20.0/24`)** · nousena dashboard, Minecraft servers, Immich

Public exposure is a single Cloudflare Tunnel out of CT 100 to [nousena.com](https://nousena.com) — no inbound WAN ports.

Remote access runs over a Tailscale mesh, with dev1 advertising as exit node and pushing Pi-hole DNS mesh-wide. Nginx on the host fronts internal services so remote clients reach them in a single Tailscale hop instead of relaying twice through DERP.

DNS is Pi-hole v6 for network-wide filtering, with upstream queries encrypted through dnscrypt-proxy to Cloudflare DoH.

## Containers

| CTID | Name | Bridge | Role |
|------|------|--------|------|
| 100  | [nousena](#nousena-ct-100--nousenacom)              | vmbr2 | Docker host — nousena.com dashboard stack |
| 101  | [mc-server](#minecraft-servers-ct-101--103)         | vmbr2 | Paper Minecraft server (custom plugin: lumberjack) |
| 103  | [mc-modded](#minecraft-servers-ct-101--103)         | vmbr2 | Modded Fabric Minecraft server |
| 200  | [obsidian](#obsidian-vault--knowledge-pipeline-ct-200--201) | vmbr0 | Obsidian vault — Syncthing source of truth |
| 201  | [obsidian-ui](#obsidian-vault--knowledge-pipeline-ct-200--201) | vmbr0 | Obsidian web UI (Docker-in-LXC, Selkies/WebRTC) |
| 300  | [pihole](#pi-hole-ct-300)                           | vmbr0 | Pi-hole v6 + dnscrypt-proxy → Cloudflare DoH |
| 400  | [immich](#immich-ct-400)                            | vmbr2 | Immich photo server (Docker compose) |
| 500  | [pbs](#proxmox-backup-server-ct-500)                | vmbr0 | Proxmox Backup Server 4.2 |
| 600  | [home-assistant](#home-assistant-ct-600)            | vmbr0 | Home Assistant |
| 700  | [splunk](#observability-stack)                      | vmbr0 | Splunk Enterprise 10.4 — log indexer |

### nousena (CT 100) → [nousena.com](https://nousena.com)

Privileged LXC running Docker. Hosts a personal finance / dashboard application with isolated prod and dev environments:

- **Backend** — FastAPI + MariaDB
- **Frontend** — React + Vite + Tailwind
- **Ingress** — Cloudflare Tunnel (no inbound ports opened on the WAN side)
- **Tooling** — `dashrun` CLI wraps the docker compose flows for both environments

### Minecraft servers (CT 101 / 103)

Vanilla Paper (CT 101) and modded Fabric (CT 103). Regularly host 10+ concurrent players. Game logs are forwarded to Splunk under per-server indexes (`mc-vanilla`, `mc-modded`), making player joins, deaths, and chat queryable from the SPL pane.

CT 101 also runs **[lumberjack](https://github.com/gabegaglio/lumberjack)** — a custom Paper Java plugin that chains tree-felling: break the trunk and the entire tree comes down.

### Obsidian vault + knowledge pipeline (CT 200 / 201)

Unprivileged LXC holding the canonical Obsidian vault, bind-mounted to the Proxmox host so dev1 tooling can read it directly.

- **Sync** — Syncthing mesh with role-based access: source of truth on CT 200 (send-receive), read-only mirror on CT 201, workstation and mobile clients send-receive. All instances run under a dedicated non-root user.
- **Web UI** — CT 201 serves the Obsidian web app inside Docker-in-LXC via Selkies / WebRTC, reverse-proxied through nginx with self-signed TLS.
- **Knowledge pipeline** — a custom MCP server backs Claude Code with semantic search across the vault and per-host memory. Gemini embeddings, a vault watcher that re-embeds on any `.md` change, and PostToolUse hooks that re-embed after Claude edits keep the index live.

### Pi-hole (CT 300)

Pi-hole v6 + dnscrypt-proxy. Local DNS records resolve internal hostnames (`immich`, `obsidian`, `dev1`) over Tailscale. FTL query logs ship to Splunk for blocked-domain analysis.

### Immich (CT 400)

Privileged LXC running Immich via Docker compose (`server`, `postgres`, `machine_learning`, `redis`). LXC rootfs on NVMe for app speed; media library and Postgres on a dedicated HDD mount. Reachable on LAN, Tailscale, and the Immich mobile app.

### Proxmox Backup Server (CT 500)

PBS 4.2 with both its rootfs and the backup datastore on a physically separate disk from the live containers — a primary-disk failure does not take backups with it. PVE authenticates via an API token with `DatastoreBackup` RBAC; snapshot backups for the at-risk containers; single-command restores from the datastore.

### Home Assistant (CT 600)

Smart-home control plane.

## Observability stack

```
containers ── tcp:9997 ──▶  Splunk indexer (CT 700)  ──▶ dashboards & SPL
Proxmox host ──────────▶  Telegraf  ──▶ InfluxDB-line metric files
```

- **Splunk Enterprise 10.4** (CT 700) — receiver on `:9997`, web on `:8000`. Per-source indexes: `dev1`, `nousena`, `mc-vanilla`, `mc-modded`, `pihole`.
- **Universal Forwarders** on CT 101 / 103 / 200 / 300, tailing `/var/log` and shipping over TCP. Deployed in a single pass via a shell script that also fixes the installer's default root-binding and re-homes the systemd unit to a dedicated `splunk` user.
- **Telegraf** runs directly on the Proxmox host — it needs raw cgroup, disk, and network visibility that an LXC can't surface. 30 s interval, InfluxDB line protocol, captures every LXC's `veth` interface so per-container network throughput is recoverable from host view alone.
- **Per-LXC CPU / memory** via a custom script using the Proxmox API (`pvesh`) instead of `pct exec` — no shell-spawn overhead, cgroup-accurate values, and it still works against containers with shells disabled.
- **[vigosk](https://github.com/gabegaglio/vigosk)** — independent host-level monitor running on dev1, complementary to Splunk; watches container state, host health, and service liveness.

Sample SPL:

```spl
index=pihole status=2 | top 10 domain                 # most-blocked domains
index=mc-vanilla "joined the game" | timechart count  # player joins over time
index=dev1 sourcetype=linux_secure "Failed password"  # bruteforce attempts
```

## Operational practices

- **Single-purpose containers** — application stacks run as Docker-in-LXC; infrastructure (nginx, Tailscale, Telegraf, MCP server) runs on the host
- **Privilege minimisation** — Syncthing, Splunk, and forwarders all run under dedicated non-root users; the installer defaults that bind services to root are fixed at deploy time
- **Scripted ops over interactive ones** — operational entry points use `pct exec` rather than `pct enter`, so host shell context is preserved and commands stay scriptable
- **Versioned topology** — container map, network layout, and service inventory live in a vault that's Syncthing-replicated and semantically indexed for Claude Code

---

_Related repos: [nousena.com](https://nousena.com) · [vigosk](https://github.com/gabegaglio/vigosk) · [lumberjack](https://github.com/gabegaglio/lumberjack)_
