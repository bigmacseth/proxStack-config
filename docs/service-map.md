# Service Map

## Overview

All homelab services are hosted across a 3-node Proxmox cluster. Each service runs as either an LXC container or a VM on one of the nodes. This document maps every service to its host node, IP, and container/VM type.

---

## Prox1 — 172.0.0.3

| IP | Hostname | Type | Services |
|----|----------|------|----------|
| 172.0.0.57 | lxc-caddy | LXC | Caddy reverse proxy (routes external traffic to internal services via Cloudflare) |
| 172.0.0.60 | lxc-upsnap | LXC | UpSnap — wake-on-LAN manager |
| 172.0.0.58 | lxc-rustdesk | LXC | RustDesk relay/rendezvous server (remote desktop) |
| 172.0.0.53 | ubuntu-arr | VM | *arr stack: Sonarr, Radarr, Lidarr, Prowlarr, qBittorrent/Gluetun, FlareSolverr, Jellyseer |

---

## Prox2 — 172.0.0.52

| IP | Hostname | Type | Services |
|----|----------|------|----------|
| 172.0.0.54 | lxc-twingate | LXC | Twingate connector (secure remote access) |
| 172.0.0.59 | lxc-pihole | LXC | Pi-hole DNS ad blocker |
| 172.0.0.51 | docker-grafana | VM | Grafana, Prometheus, Loki, pve_exporter |

---

## Prox3 — 172.0.0.55

| IP | Hostname | Type | Services |
|----|----------|------|----------|
| 172.0.0.56 | vm-jellyfin | VM | Jellyfin media server (GPU passthrough for hardware transcoding) |

---

## Other Hosts

| IP | Hostname | Type | Role |
|----|----------|------|------|
| 172.0.0.50 | seth | Gaming PC | Primary workstation. Managed via UpSnap (WoL) and RustDesk for remote access. Manual Pi-hole DNS assignment for ad blocking. |

---

## Docker Compose File Reference

All compose files are stored in the root of the proxStack-config repo. This table maps each file to its host and service.

| Compose File | Host | Service(s) |
|--------------|------|------------|
| cadvisor-compose.yml | docker-grafana (prox2) | cAdvisor container metrics |
| grafana-loki-compose.yml | docker-grafana (prox2) | Grafana Enterprise, Loki |
| prometheus-compose.yml | docker-grafana (prox2) | Prometheus |
| pve_exporter-compose.yml | docker-grafana (prox2) | Proxmox exporter for Prometheus |
| sonarr-compose.yml | ubuntu-arr (prox1) | Sonarr |
| radarr-compose.yml | ubuntu-arr (prox1) | Radarr |
| lidarr-compose.yml | ubuntu-arr (prox1) | Lidarr |
| prowlarr-compose.yml | ubuntu-arr (prox1) | Prowlarr |
| qbittorrent-nox-gluetun-compose.yml | ubuntu-arr (prox1) | qBittorrent + Gluetun VPN |
| flaresolverr-compose.yml | ubuntu-arr (prox1) | FlareSolverr |
| jellyseer-compose.yml | ubuntu-arr (prox1) | Jellyseer |
| upsnap-compose.yml | lxc-upsnap (prox1) | UpSnap |
| rustdesk-compose.yml | lxc-rustdesk (prox1) | RustDesk |
