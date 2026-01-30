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

Create a VM for each additional Docker host service:
  - Grafana
  - *arr stack
  - jellyfin

### 7. Grafana Stack

Use one vm (installed with ubuntu LTS) as a docker host, installing docker from this:
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF


sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
1. Make a directory in /home/$USER/ for each service (grafana-loki/ , prometheus/ ,  and pve_exporter/)

    - For grafana it’s simple, just copy the compose file from this repo and spin it up with ‘docker compose up -d’

2. For prometheus it’s a little more complicated.  In the directory prometheus/ place the compose file and make a new directory called prometheus/

    - Inside that second prometheus directory, place the prometheus.yml file and DO NOT RENAME IT. This sets up the default scrape jobs.
    - Then run ‘docker compose up -d’ to spin up the container
  
3. For pve_exporter, it is much more complicated.

    - Watch this youtube video for instructions on setting up a pve_exporter agent on each node in the ProxStack: https://www.youtube.com/watch?v=PtsdThgnZqs&pp=2AZc0gcJCZEKAYcqIYzv\
    - With the pve-user created, obtain an api token for the cluster.
    - Place the pve_exporter compose file into the main pve_exporter directory
    - Create a new directory inside the pve_exporter directory called ‘conf/’ and create a file in there called ‘pve.yml’
    - The format of the file should be:

```yml
default:
  user: “prometheus@pve”
  token-name: “exporter”
  token-value: “(api key goes here)”
  verify-ssl: false
```
 
4. Everything from here should be simple, but setting up grafana can be annoying.
    - go to grafana's ip-address:port and add prometheus as a data source.
    - I use Dashboards 10347 for proxmox monitoring and 22870 for caddy monitoring
    - caddy's job setup should look like "datasource: proxmox (or whatever you called your datasource)" "job: caddy" "Instance: 172.0.0.0:2019" "Interval: 10m"
    - proxmox's job setup should look  like "Instance: 172.0.0.3 (or whatever node you want to monitor, my first node is 172.0.0.3)
    - Done! (Loki logging coming soon)
