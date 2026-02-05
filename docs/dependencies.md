# Service Dependencies

## Overview

This document maps the relationships between services. This is critical for recovery, because bringing services back up in the wrong order will result in broken functionality even if everything is technically running.

---

## Dependency Chain

```
OPNsense (firewall/DHCP)
    |
    |-- Pi-hole (standalone, no deps)
    |-- Twingate (standalone, no deps)
    |-- RustDesk (standalone, no deps)
    |-- UpSnap (standalone, no deps)
    |
    |-- Grafana Stack
    |       |
    |       |-- pve_exporter (needs Proxmox API token configured)
    |       |-- Prometheus (needs pve_exporter running to scrape)
    |       |-- Loki (standalone log collector)
    |       └-- Grafana (needs Prometheus and Loki as data sources)
    |
    |-- *arr Stack
    |       |
    |       |-- Gluetun VPN (must be up before qBittorrent)
    |       |-- qBittorrent (needs Gluetun running for VPN routing)
    |       |-- Prowlarr (indexer hub, needs qBittorrent as download client)
    |       |-- FlareSolverr (Cloudflare bypass, used by Prowlarr)
    |       |-- Sonarr (needs Prowlarr + qBittorrent)
    |       |-- Radarr (needs Prowlarr + qBittorrent)
    |       |-- Lidarr (needs Prowlarr + qBittorrent)
    |       └-- Jellyseer (frontend — needs Jellyfin + at least one *arr service)
    |
    └-- Jellyfin (standalone media server, but Jellyseer depends on it)

External Access (Cloudflare -> OPNsense -> Caddy)
    |
    |-- Jellyfin (proxied via Caddy)
    └-- Jellyseer (proxied via Caddy)
```

---

## Standalone Services (No Dependencies)

These can be brought up in any order, at any time. They don't rely on other internal services to function.

| Service | Host | Why It's Standalone |
|---------|------|---------------------|
| Pi-hole | lxc-pihole (prox2) | DNS filtering only, no internal service dependencies |
| Twingate | lxc-twingate (prox2) | Connects outbound to Twingate cloud, no local deps |
| RustDesk | lxc-rustdesk (prox1) | Relay/rendezvous server, no local deps |
| UpSnap | lxc-upsnap (prox1) | Wake-on-LAN manager, no local deps |
| Gluetun | ubuntu-arr (prox1) | VPN container, connects outbound only |

---

## Grafana Observability Stack

**Host:** docker-grafana (prox2, 172.0.0.51)

All components run as Docker containers on the same VM.

| Order | Service | Depends On | Reason |
|-------|---------|------------|--------|
| 1 | pve_exporter | Proxmox API token | Scrapes metrics from each Proxmox node. Needs the `prometheus@pve` user and API token configured on the cluster first. |
| 2 | Prometheus | pve_exporter | Scrapes metrics from pve_exporter (and Caddy). No data to collect if exporters aren't running. |
| 3 | Loki | — | Log aggregator. Can come up alongside Prometheus, no hard dependency between them. |
| 4 | Grafana | Prometheus, Loki | Dashboards pull from Prometheus and Loki as data sources. Must be configured after both are running. |

---

## *arr Media Stack

**Host:** ubuntu-arr (prox1, 172.0.0.53)

| Order | Service | Depends On | Reason |
|-------|---------|------------|--------|
| 1 | Gluetun | — | VPN tunnel. Must be established before qBittorrent starts routing traffic. |
| 2 | qBittorrent | Gluetun | All download traffic is routed through the Gluetun VPN. If Gluetun is down, downloads won't work. |
| 3 | FlareSolverr | — | Cloudflare bypass solver. Can come up alongside qBittorrent, but must be running before Prowlarr can use it. |
| 4 | Prowlarr | qBittorrent, FlareSolverr | Indexer hub that aggregates torrent sources. Needs qBittorrent configured as its download client and FlareSolverr for sites that use Cloudflare protection. |
| 5 | Sonarr | Prowlarr, qBittorrent | TV show automation. Searches via Prowlarr, downloads via qBittorrent. |
| 5 | Radarr | Prowlarr, qBittorrent | Movie automation. Same dependency chain as Sonarr. |
| 5 | Lidarr | Prowlarr, qBittorrent | Music automation. Same dependency chain as Sonarr. |
| 6 | Jellyseer | Jellyfin, Sonarr/Radarr/Lidarr | Request frontend. Needs Jellyfin to know what's already in the library, and at least one *arr service to submit requests to. |

Note: Sonarr, Radarr, and Lidarr are all order 5. They can come up in any order or simultaneously.

---

## Jellyfin

**Host:** vm-jellyfin (prox3, 172.0.0.56)

Jellyfin itself has no internal service dependencies — it's a standalone media server. However, two other services depend on it:

- **Jellyseer** needs Jellyfin running to display the current media library
- **Caddy** proxies external access to Jellyfin, so Jellyfin should be up before you expect external access to work

---

## External Access Chain

For any service to be reachable from outside the network, the full chain must be healthy:

```
Cloudflare (DNS + proxy) -> OPNsense (port forward) -> Caddy (reverse proxy) -> Target Service
```

Currently proxied services: **Jellyfin** and **Jellyseer**. Both require Caddy to be running. Other services (Grafana, *arr UIs, etc.) have Caddyfile entries but are commented out.
