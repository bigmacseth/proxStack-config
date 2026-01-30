# proxStack-config

Configuration steps and files for the proxStack cluster.

## Container Setup

Create an LXC container for each service:
- Caddy
- UpSnap
- RustDesk
- Twingate
- Pi-hole

---

## Service Deployment Instructions

### 1. Caddy

**Install Caddy:**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

**Configure:**
1. Copy the `Caddyfile` and `caddy_exporter` directory into the LXC
2. As root, test with: `caddy run`
3. Verify connectivity at: `jellyfin.big-mac.net`

---

### 2. UpSnap

1. Install Docker and create directory: `upsnap`
2. Copy `upsnap-compose.yml` into the directory
3. Run: `docker compose up -d`
4. Configure UpSnap in Web GUI to point to Main-PC

---

### 3. RustDesk

1. Install Docker and create directory: `rustdesk`
2. Copy `rustdesk-compose.yml` into the directory
3. Run: `docker compose up -d`
4. Point Main-PC to RustDesk in the client
5. Keys are stored in Important Info

---

### 4. Twingate

1. Follow deployment guide: https://www.twingate.com/docs/proxmox-container-deployment
2. Create a connector in Twingate dashboard pointed at the LXC
3. Deploy and verify connection

---

### 5. Pi-hole

1. Create LXC with static IP
2. Run interactive installer: `curl -sSL https://install.pi-hole.net | bash`
3. Configure via Web UI

---

### 6. Docker Host VMs

Create a VM for each additional Docker host service...
