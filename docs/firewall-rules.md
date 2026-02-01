# Firewall Rules

## Overview

OPNsense is running on a Sophos SG-135. The firewall manages traffic between the WAN (internet) and the single LAN segment (172.0.0.0/24). There are no VLANs or inter-subnet rules — the home network (192.168.1.0/24) is handled entirely by the AT&T router and does not pass through OPNsense.

## DNS Configuration

DHCP on the LAN serves `1.1.1.1` and `9.9.9.9` as DNS servers to all homelab clients. Pi-hole (.59) is available on the subnet but is only used by devices that manually assign it as their DNS server (e.g., the gaming PC for ad blocking). The Proxmox nodes and other infrastructure do not use Pi-hole.

## LAN Rules (172.0.0.0/24)

| # | Action | Source | Destination | Protocol | Description |
|---|--------|--------|-------------|----------|-------------|
| 1 | Pass   | LAN network | Any | IPv4 (any) | Default allow — all LAN clients can reach the internet and any other LAN host |
| 2 | Pass   | LAN network | Any | IPv6 (any) | Default allow — same as above for IPv6 |

These are the default OPNsense rules. No custom LAN rules have been added. All homelab traffic is permitted outbound by default.

## WAN Rules (Internet-facing)

All inbound WAN traffic is **blocked by default** except for the following explicit port forwards to Caddy. Both rules are restricted to Cloudflare IP ranges only — no other external traffic is permitted in.

| # | Action | Source | Destination | Protocol | Port | Description |
|---|--------|--------|-------------|----------|------|-------------|
| 1 | Pass   | CLOUDFLARE_IPS | 172.0.0.57 (lxc-caddy) | TCP | 80 | Caddy HTTP — Cloudflare proxy only |
| 2 | Pass   | CLOUDFLARE_IPS | 172.0.0.57 (lxc-caddy) | TCP | 443 | Caddy HTTPS — Cloudflare proxy only |

Both rules have logging enabled.

## NAT / Port Forwarding

Outbound NAT is set to automatic. Inbound port forwards mirror the WAN rules above:

| External Port | Internal Target | Internal Port | Description |
|---------------|-----------------|---------------|-------------|
| 80            | 172.0.0.57      | 80            | Caddy HTTP  |
| 443           | 172.0.0.57      | 443           | Caddy HTTPS |

## Firewall Alias

A named alias `CLOUDFLARE_IPS` is defined containing all current Cloudflare IPv4 and IPv6 ranges. This is used as the source filter on both WAN rules to ensure only legitimate Cloudflare proxy traffic reaches Caddy. If Cloudflare updates their IP ranges, this alias will need to be updated accordingly.

## Intrusion Detection (IDS/IPS)

OPNsense IDS is enabled and running in IPS mode on both LAN and WAN interfaces. The following rule sets are active:

- abuse.ch (FeodoTracker, SSLBlacklist, SSLIPBlacklist, ThreatFox, URLhaus)
- Emerging Threats (exploit, exploit_kit, FTP, malware, phishing, shellcode, web_client, web_server, and many others)
- ThreatView C2
- Tor detection
- OPNsense built-in rules (file transfer, mail, media streaming, messaging, social media)

## Recovery Notes

- The full OPNsense configuration can be backed up via **System -> Backup** as an XML file. This backup contains all firewall rules, NAT, DHCP leases, and certificates and is sufficient to restore the firewall to its current state.
- The `CLOUDFLARE_IPS` alias should be verified periodically against [Cloudflare's official IP list](https://www.cloudflare.com/ips/) to ensure inbound rules remain accurate.
- If the firewall is rebuilt from scratch, the DHCP static leases in this backup are critical — they tie MAC addresses to hostnames and IPs for all homelab services.
