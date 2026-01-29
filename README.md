# proxStack-config
Steps for and files for the proxStack cluster.

1.) Create an lxc container for each: caddy, upsnap, rustdesk, twingate, and pihole
2.) Caddy instructions:
    a. Use this script to install caddy onto the container:
        sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
        curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
        curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
        chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
        chmod o+r /etc/apt/sources.list.d/caddy-stable.list
        sudo apt update
        sudo apt install caddy
    b. Copy the caddyfile and caddy_exporter directory into the lxc
    c. As root, execute the command ‘caddy run’ as a test
    d. Test connectivity by going to the domain jellyfin.big-mac.net
    e. Done!

3.) upsnap instructions:
    a. install docker, create directory 'upsnap'.
    b. copy the upsnap-compose.yml into the directory
    c. run 'docker compose up -d'
    d. configure upsnap in the Web GUI to point to Main-PC

4.) rustdesk instructions:
    a. install docker, create directory 'rustdesk'
    b. copy the rustdesk-compose.yml file into the directory
    c. run 'docker compose up -d'
    d. point Main-PC to rustdesk in the client
    e. keys are in Important Info

5.) twingate instructions:
    a. follow instructions in this webpage: https://www.twingate.com/docs/proxmox-container-deployment
    b. go to twingate dashboard and create a connector pointed at the lxc
    c. deploy and complete

6.) pihole instructions: 
    a. create lxc with static IP
    b. use this command to interactively install pihole:  curl -sSL https://install.pi-hole.net | bash
    c. go to web ui and configure there

7.) Create a VM for each of the other docker hosts: 
