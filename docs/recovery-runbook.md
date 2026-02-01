# Recovery Runbook

## Before You Start

This runbook assumes you have access to:
- The Proxmox web UI on at least one healthy node
- The OPNsense web UI (172.0.0.1)
- This repo (proxStack-config) cloned locally or accessible via GitHub
- The OPNsense backup XML (or a recently restored firewall)

Reference docs in this repo:
- `docs/network-topology.md` — IPs, subnets, physical layout
- `docs/firewall-rules.md` — OPNsense config and recovery notes
- `docs/service-map.md` — what runs where
- `docs/dependencies.md` — what needs what, and in what order

---

## Known Risk: Current Backup Limitations

Backups are currently stored on local node storage only (automated daily at 3am, keep-last=5). This means:

- **If a node fails completely, its backups are lost with it.**
- **Prox3 is the highest-risk node** — it hosts Jellyfin, the NFS media share, and is the only node with media storage. A prox3 failure means losing the media library entirely until the NFS share is recovered or rebuilt.

**TODO:** Once a NAS is available, migrate backups and the media library off local storage. Update this section when that's done.

---

## Scenario 1: Single Service Down

**Symptoms:** One service is unreachable, everything else is fine.

1. Check the Proxmox web UI — confirm the VM or LXC is running on its expected node (see `docs/service-map.md`).
2. If it's stopped, start it from the Proxmox UI.
3. If it's running but the service inside is broken, SSH into the VM/LXC and check the container status:
   ```
   docker ps -a
   ```
4. If the container is stopped or crashing, check logs:
   ```
   docker logs <container_name> --tail 50
   ```
5. If the container won't start, redeploy from the compose file in `configs/` or `compose/` in this repo:
   ```
   docker compose up -d
   ```
6. Verify the service is healthy before moving on.

---

## Scenario 2: Single Node Down

**Symptoms:** All services on one node are unreachable. Other nodes are healthy.

### Identify the impact

Check `docs/service-map.md` to see what was on the failed node:

| Failed Node | Services Lost | Additional Impact |
|-------------|---------------|-------------------|
| prox1 | Caddy, UpSnap, RustDesk, *arr stack + Jellyseer | External access is down (Caddy). No remote desktop (RustDesk). No media downloads or requests. |
| prox2 | Twingate, Pi-hole, Grafana/Prometheus/Loki | Remote access via Twingate is down. No DNS ad blocking. No monitoring dashboards. |
| prox3 | Jellyfin, NFS media share | Media library is inaccessible. Jellyseer on prox1 will be broken too since it depends on Jellyfin. *arr stack can still run but downloads have nowhere to go. |

### Restore from backup

1. On a healthy node, go to **Datacenter -> Backup** in the Proxmox UI.
2. Locate the most recent backup for each VM/LXC that was on the failed node.
3. Restore each one to a healthy node. Assign the same IP addresses as before — OPNsense DHCP leases are tied to MAC addresses, which are preserved in the backup.
4. Once restored, bring services back up in the order specified in `docs/dependencies.md`.

### Special case: prox3 failure

If prox3 is the failed node, the NFS media share is down. Jellyfin and Jellyseer will not function until storage is recovered. The *arr stack on prox1 will still run, but downloads will fail or have no valid destination until the share is back.

**TODO:** Once a NAS is in place, the media library and NFS share should be documented here with its own recovery steps.

---

## Scenario 3: OPNsense / Firewall Down

**Symptoms:** No internet access on any device. All external services unreachable.

1. Check the Sophos SG-135 physically — confirm it's powered on and the WAN/LAN LEDs are active.
2. If the hardware is fine but OPNsense is unresponsive, restore from the OPNsense backup XML:
   - Go to **System -> Backup -> Restore** in OPNsense
   - Upload the most recent backup XML (store a copy of this outside the homelab — see note below)
3. Once restored, verify:
   - DHCP leases are active and match `docs/network-topology.md`
   - Firewall rules are in place per `docs/firewall-rules.md`
   - Port forwards to Caddy (ports 80 and 443) are working
4. Confirm internet access from a homelab device before considering the firewall recovered.

**Important:** The OPNsense backup XML contains everything needed to restore the firewall — rules, NAT, DHCP leases, and certs. Keep a copy of this backup somewhere outside the homelab (USB drive, cloud storage, etc.). If the firewall and all nodes go down simultaneously, you need this file to be accessible without any of them.

---

## Scenario 4: Full Cluster Failure

**Symptoms:** All three Proxmox nodes are down or unresponsive.

This is the worst case. Recovery depends on whether the hardware is recoverable or not.

### If hardware is recoverable (power issue, crash, etc.)

1. Power on nodes one at a time. Proxmox cluster should reform automatically.
2. Once the cluster is healthy, verify all VMs and LXCs are in their expected state.
3. Start any stopped services following the dependency order in `docs/dependencies.md`.

### If hardware is not recoverable (hardware failure, data loss, etc.)

1. **Restore OPNsense first** — you need the firewall up before anything else matters. Use the backup XML (see Scenario 3).
2. Rebuild Proxmox nodes as needed. Restore VMs/LXCs from the most recent backups available.
3. Bring services back up in this order:
   - **Tier 1 — Infrastructure:** Pi-hole, Twingate, RustDesk, UpSnap (no dependencies, any order)
   - **Tier 2 — Networking:** Caddy (needed for external access)
   - **Tier 3 — Observability:** pve_exporter -> Prometheus/Loki -> Grafana
   - **Tier 4 — Media downloads:** Gluetun -> qBittorrent -> FlareSolverr -> Prowlarr -> Sonarr/Radarr/Lidarr
   - **Tier 5 — Media serving:** Jellyfin (requires NFS share to be healthy)
   - **Tier 6 — Frontends:** Jellyseer (needs both Jellyfin and *arr services running)
4. Verify external access via Cloudflare -> Caddy -> Jellyfin/Jellyseer.

---

## Post-Recovery Checklist

After any recovery event, verify the following before considering things resolved:

- [ ] All three Proxmox nodes are online and cluster is healthy
- [ ] OPNsense DHCP leases match `docs/network-topology.md`
- [ ] Firewall rules and port forwards are active per `docs/firewall-rules.md`
- [ ] All services are running per `docs/service-map.md`
- [ ] Grafana dashboards are showing data (confirms Prometheus and pve_exporter are healthy)
- [ ] Jellyfin is accessible and library is intact
- [ ] Jellyseer can submit requests
- [ ] External access works (test via browser outside the network or via Twingate)
- [ ] Automated backups are running (verify next day at 3am)
