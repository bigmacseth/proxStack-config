# Network Topology

## Physical Layout

```
Internet
    |
    v
AT&T Modem/Router (also serves as wireless AP for home network)
    |
    | (Ethernet - WAN)
    v
Sophos SG-135 Rev. 2 (running OPNsense)
    |
    | (Ethernet - LAN)
    v
8-Port NetGear Switch
    |
    |--- prox1 (172.0.0.3)
    |--- prox2 (172.0.0.52)
    |--- prox3 (172.0.0.55)
    └--- Seth / Gaming PC (172.0.0.50)
```

## Subnets

| Subnet          | Gateway       | Purpose                                          |
|-----------------|---------------|--------------------------------------------------|
| 192.168.1.0/24  | 192.168.1.1   | Home network — wireless devices via AT&T router  |
| 172.0.0.0/24    | 172.0.0.1     | Homelab — all wired infrastructure via OPNsense |

No VLANs are currently configured. All homelab traffic is on a single flat subnet.

## Proxmox Nodes

Static IPs are assigned directly in Proxmox (not via OPNsense DHCP leases).

| Hostname | IP Address   | Notes                        |
|----------|--------------|------------------------------|
| prox1    | 172.0.0.3    | Proxmox cluster node         |
| prox2    | 172.0.0.52   | Proxmox cluster node         |
| prox3    | 172.0.0.55   | Proxmox cluster node         |

## Service Assignments

Static leases managed via OPNsense. All services hosted as LXC containers or VMs on the Proxmox cluster.

| IP            | Hostname        | Type | Services                                  |
|---------------|-----------------|------|-------------------------------------------|
| 172.0.0.10    | *(stale lease)* | —    | No longer in use — can be removed         |
| 172.0.0.50    | seth            | PC   | Gaming PC                                 |
| 172.0.0.51    | docker-grafana  | VM   | Grafana, Prometheus, Loki, pve_exporter   |
| 172.0.0.53    | ubuntu-arr      | VM   | *arr stack (Sonarr, Radarr, Lidarr, Prowlarr, qBittorrent/Gluetun, FlareSolverr) |
| 172.0.0.54    | lxc-twingate    | LXC  | Twingate connector (remote access)        |
| 172.0.0.56    | vm-jellyfin     | VM   | Jellyfin media server                     |
| 172.0.0.57    | lxc-caddy       | LXC  | Caddy reverse proxy                       |
| 172.0.0.58    | lxc-rustdesk    | LXC  | RustDesk relay/rendezvous server          |
| 172.0.0.59    | lxc-pihole      | LXC  | Pi-hole DNS ad blocker                    |
| 172.0.0.60    | lxc-upsnap      | LXC  | UpSnap wake-on-LAN manager               |

## Stale Leases to Clean Up

The following OPNsense leases are no longer accurate and should be removed:

- **172.0.0.10** — old Pi-hole entry, Pi-hole is now at 172.0.0.59
- **172.0.0.53** — was mislabeled as `jellyfin`, is actually the *arr stack (`ubuntu-arr`)
