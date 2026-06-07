# Complete ARR Stack Installation on Proxmox

This guide is an extension of the excellent [Automation Avenue ARR Stack guide](https://github.com/automation-avenue/arr-new) and its related video tutorial: [ARR stack setup video](https://www.youtube.com/watch?v=-PQtE6Nb0Cw).

The goal is not to replace the original guide, but to document my own complete Proxmox-based setup and make the process easier to follow from start to finish. I also clarify and correct a few details that were easy to miss in the original instructions, especially around storage paths, Docker configuration, service setup, and first-run configuration.

This version also adds an optional Tailscale remote-access setup for Jellyfin and Seerr, inspired by this video: [Tailscale remote access tutorial](https://www.youtube.com/watch?v=E34gf3GC7xw).

This guide documents a complete homelab setup for running an ARR media automation stack inside an Ubuntu Server VM hosted on Proxmox VE.

The goal is to keep your main PC clean, use a dedicated second machine as a small server, and deploy the stack in a predictable way with clear storage, networking, and service paths.

This is an unofficial tutorial based on my personal setup. It may evolve over time as I add new services or improve the configuration.

---

## Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Network Plan](#network-plan)
4. [Install Proxmox VE](#install-proxmox-ve)
5. [Access the Proxmox Web UI](#access-the-proxmox-web-ui)
6. [Post-Install Proxmox Setup](#post-install-proxmox-setup)
7. [Storage Layout](#storage-layout)
8. [Create the Ubuntu VM](#create-the-ubuntu-vm)
9. [Prepare Ubuntu](#prepare-ubuntu)
10. [Install Docker and Docker Compose](#install-docker-and-docker-compose)
11. [Folder Structure](#folder-structure)
12. [Tailscale Setup](#tailscale-setup)
13. [Deploy the ARR Stack](#deploy-the-arr-stack)
14. [Service URLs and Ports](#service-urls-and-ports)
15. [Initial App Configuration](#initial-app-configuration)
16. [References](#references)

---

## Overview

After reading this guide, you will be able to:

- Install Proxmox VE on a dedicated homelab machine.
- Manage Proxmox from your main PC through the web interface.
- Create an Ubuntu Server virtual machine.
- Install Docker and Docker Compose inside the VM.
- Deploy an ARR stack with Docker Compose.
- Access and manage the stack locally.
- Access selected services remotely with Tailscale.
- Watch your media from your preferred devices.

This setup uses:

- A main PC for administration.
- A second machine dedicated to the homelab.
- One Ubuntu Server VM running Docker and the ARR stack.
- A shared `/data` storage layout for downloads and media.
- Dedicated `/docker/appdata` folders for application configuration.
- Tailscale for remote access to selected services.

Final target architecture:

```text
Main PC
  |
  | Browser / SSH
  v
Proxmox host: homelab PC
  |
  v
Ubuntu VM: arr
  |
  +-- Docker
      +-- Tailscale sidecars
      |   +-- Jellyfin
      |   +-- Seerr
      |
      +-- Radarr
      +-- Sonarr
      +-- Lidarr
      +-- Bazarr
      +-- Prowlarr
      +-- Flaresolverr
      +-- qBittorrent
      +-- Homarr
```

---

## Requirements

### Main PC

You need a machine that can access the Proxmox web UI and connect to the server over SSH.

Current main PC example:

| Component | Value |
| --- | --- |
| OS | Windows 11 Pro |
| Motherboard | Gigabyte Z690 AORUS ELITE DDR4 |
| CPU | Intel Core i7-12700K |
| RAM | 32 GB |
| GPU | AMD Radeon RX 6900 XT |
| Network | Realtek 2.5GbE |

### Dedicated Homelab PC

Current dedicated machine example:

| Component | Value |
| --- | --- |
| Machine | Lenovo PC |
| Hostname | `pve` |
| Proxmox IP | `192.168.XXX.XXX` |
| CPU | Intel Core i7-4790S @ 3.20 GHz |
| Cores / Threads | 4 cores / 8 threads |
| RAM | 15 GiB |
| SSD | 238.5 GB |
| HDD | 931.5 GB |
| Proxmox version | Proxmox VE 9.2.3 |
| Kernel | 7.0.6-2-pve |
| Virtualization | Intel VT-x |

### USB Installer

You also need:

- A USB stick with at least 16 GB of storage.
- [Rufus](https://rufus.ie/en/) to create the bootable USB stick.
- The Proxmox VE ISO.

---

## Network Plan

Use a small static range for homelab devices. This makes the server and VM easier to find and avoids IP changes after a reboot.

Main LAN example:

```text
Router / gateway: 192.168.XXX.XXX
Homelab range:    192.168.XXX.200-254
```

Example network plan:

| IP Address | Service / Device | Notes |
| --- | --- | --- |
| `192.168.XXX.XXX` | Main Windows PC | Administration machine |
| `192.168.XXX.201` | Lenovo Proxmox host | `pve` |
| `192.168.XXX.222` | Ubuntu VM | `arr` |

Replace these values with the addresses used on your own network.

---

## Install Proxmox VE

### Step 1: Reserve a Static IP Range

Before installing Proxmox, prepare your router so the homelab devices can use predictable IP addresses.

This step is important. If the IP range is not configured correctly, you may need to redo part of the setup later.

1. Log in to your router.
2. Open the LAN or DHCP settings.
3. Set the DHCP range so that part of the network is free for static addresses.
4. For example, set:
   - Beginning IP address: `192.168.1.201`
   - Ending IP address: `192.168.1.254`
5. Save the settings.

You can now assign static IP addresses to Proxmox and to the Ubuntu VM.

### Step 2: Create the Proxmox USB Installer

Rufus is a utility that formats a USB stick and turns it into a bootable installer.

1. Download and install [Rufus](https://rufus.ie/en/) on your main PC.
2. Download the Proxmox VE ISO from the [official Proxmox download page](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso).
3. Insert the USB stick into your PC.
4. Open Rufus.
5. Select your USB stick in the `Device` field.
6. Select the Proxmox VE ISO.
7. Click `Start`.

> Warning: Rufus will erase the entire USB stick. Make sure there is nothing important on it before continuing.

### Step 3: Configure the Homelab BIOS

Restart the homelab machine and enter the BIOS.

The exact menu names may be different depending on your machine, but the goal is to enable virtualization, disable Secure Boot, and boot from the USB stick.

Example BIOS settings:

```text
Advanced -> CPU Configuration -> Intel (VMX) Virtualization Technology -> Enabled
Security -> Secure Boot -> Disabled
Boot -> Boot Order Priorities -> USB Device as Boot Option #1
```

### Step 4: Install Proxmox from the USB Stick

1. Insert the Proxmox USB installer into the homelab machine.
2. Save the BIOS changes and reboot.
3. Select `Install Proxmox VE (Graphical)`.
4. Choose the target disk.
   - In this setup, Proxmox is installed on the SSD for better performance.
5. Set the root password.
   - The username is always `root`.
   - Keep the password safe. You will need it to log in to Proxmox.

### Step 5: Set the Proxmox Static IP

During the Proxmox installation, set the static IP address.

If you followed the example network plan, you can use:

```text
192.168.1.201/24
```

Keep the `/24` suffix unless your network uses a different subnet.

The gateway and DNS values should usually match your router settings.

### Step 6: Finish the Installation

At the end of the installer:

1. Untick `Automatically reboot after successful installation`.
2. Click `Install`.
3. When the machine turns off or finishes installing, remove the USB stick.
4. Restart the homelab machine.

Proxmox VE is now installed on your homelab machine.

---

## Access the Proxmox Web UI

From your main PC, open a browser and go to:

```text
https://192.168.1.201:8006
```

If you used a different Proxmox IP address, replace `192.168.1.201` with your own address.

Log in with:

```text
User: root
Password: <PROXMOX_ROOT_PASSWORD>
```

You are now connected to the Proxmox web interface from your main PC.

---

## Post-Install Proxmox Setup

After the first login, update and clean up the Proxmox installation.

One common option is to use the Proxmox post-install script from [community-scripts.org](https://community-scripts.org/scripts/post-pve-install).

The command is:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

To run it:

1. In the Proxmox web UI, select `pve`.
2. Open `Shell`.
3. Paste the command.
4. Follow the prompts.

If you prefer, review the script before running it.

---

## Storage Layout

Current storage design:

| Storage | Purpose |
| --- | --- |
| SSD, 238.5 GB | Proxmox OS, `local`, `local-lvm`, and small future VMs |
| HDD, 931.5 GB | Main storage for the `arr` VM disk |

Current Proxmox storage:

| Storage ID | Type | Purpose |
| --- | --- | --- |
| `local` | Directory | ISO files, backups, templates |
| `local-lvm` | LVM-thin | VM disks on the SSD |
| `hdd1tb` | Directory | VM disk storage on the HDD |

In this setup, the HDD is mounted on Proxmox at:

```text
/mnt/hdd1tb
```

Example `/etc/fstab` entry:

```fstab
UUID=<HDD_UUID> /mnt/hdd1tb ext4 defaults,nofail 0 2
```

Add the HDD to Proxmox as storage:

```bash
pvesm add dir hdd1tb --path /mnt/hdd1tb --content images,iso,backup --is_mountpoint 1
```

Verify that the storage is available:

```bash
pvesm status
df -h /mnt/hdd1tb
```

---

## Create the Ubuntu VM

The Ubuntu VM will run Docker and the full ARR stack.

### Download the Ubuntu ISO

Download the Ubuntu Server ISO from the [Ubuntu releases page](https://ubuntu-releases.mirrorservice.org/releases/24.04.3/ubuntu-24.04.4-live-server-amd64.iso).

Example ISO name:

```text
ubuntu-24.04.4-live-server-amd64.iso
```

### Upload the ISO to Proxmox

In the Proxmox web UI:

1. Go to `pve`.
2. Select `local`.
3. Open `ISO Images`.
4. Upload the Ubuntu Server ISO.

### Create the VM

Click `Create VM` in the top-right corner of the Proxmox web UI.

Use these settings as a baseline:

| Section | Setting |
| --- | --- |
| General | Name: `arr` |
| General | VM ID: `222` |
| OS | Ubuntu Server ISO |
| Disks | Use the large storage, for example `hdd1tb` |
| Disks | Enable `Discard` |
| CPU | Type: `host` |
| CPU | Choose the number of cores you want |
| Memory | 4096 MiB or more |

If your main storage is an SSD, open `Advanced` in the disk settings and enable `SSD emulation`.

Example VM configuration:

| Setting | Value |
| --- | --- |
| VM ID | `222` |
| Name | `arr` |
| OS | Ubuntu Server 24.04 |
| CPU type | `host` |
| vCPU | 2 |
| RAM | 4096 MiB |
| Disk | 900 GB |
| Storage | `hdd1tb` |
| Network bridge | `vmbr0` |
| Start at boot | Enabled |

---

## Prepare Ubuntu

Start the VM and install Ubuntu Server.

Example values:

| Item | Value |
| --- | --- |
| VM hostname | `arr` |
| VM IP | `192.168.1.222` |
| User | `<VM_USERNAME>` |
| Password | `<VM_USER_PASSWORD>` |

During the Ubuntu installation, edit the IPv4 settings for `ens18`.

Example network configuration:

```text
Subnet:      192.168.1.0/24
Address:     192.168.1.222
Gateway:     192.168.1.1
Name server: 1.1.1.1
```

After Ubuntu is installed, connect to the VM from your main PC:

```bash
ssh <VM_USERNAME>@192.168.1.222
```

Install basic tools:

```bash
sudo apt update
sudo apt install -y curl wget git nano tree ca-certificates gnupg
```

Check the disk:

```bash
df -h /
lsblk
```

Expected example:

```text
/dev/sda2  885G  28G  813G  4% /
```

---

## Install Docker and Docker Compose

Add Docker's official GPG key:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the Docker repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Update the package list:

```bash
sudo apt update
```

Install Docker and Docker Compose:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check that Docker is running:

```bash
sudo systemctl status docker
```

---

## Folder Structure

The stack uses a single shared `/data` path so downloads and media stay on the same filesystem. This helps hardlinks work correctly.

Create the base folder structure:

```bash
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}
sudo mkdir -p /docker/appdata/{radarr,sonarr,lidarr,bazarr,prowlarr,qbittorrent,homarr,seerr,jellyfin}
sudo mkdir -p /docker/appdata/{seerr-ts,jellyfin-ts}
```

Set ownership and permissions:

```bash
sudo chown -R 1000:1000 /data /docker
sudo chmod -R a=,a+rX,u+w,g+w /data /docker
```

> Note: `1000:1000` should match the user ID and group ID of your Ubuntu user. This is usually correct for the first user created during installation.

Check the folder structure:

```bash
tree -L 3 /data
tree -L 2 /docker
ls -ln /data
```

Expected `/data` layout:

```text
/data
+-- media
|   +-- movies
|   +-- music
|   +-- tv
+-- torrents
    +-- movies
    +-- music
    +-- tv
```

Use `/data` as the stack directory:

```bash
cd /data
sudo nano .env
```

---

## Tailscale Setup

[Tailscale](https://tailscale.com/) is used for remote access to selected services.

To prepare Tailscale:

1. Create a [Tailscale](https://tailscale.com/) account.
2. Open the Tailscale admin settings.
3. Generate a reusable auth key.
4. Go back to the Ubuntu terminal.
5. Store the key in `/data/.env`.

Add this line to `.env`:

```env
TS_AUTHKEY=<TAILSCALE_AUTH_KEY>
```

Save and exit Nano:

```text
Ctrl + O
Enter
Ctrl + X
```
---

## Deploy the ARR Stack

Generate a secret key for Homarr:

```bash
openssl rand -hex 32
```

Edit `.env`:

```bash
cd /data
sudo nano .env
```

Your `.env` file should look like this:

```env
TS_AUTHKEY=<TAILSCALE_AUTH_KEY>
HOMARR_SECRET_ENCRYPTION_KEY=<HOMARR_SECRET_ENCRYPTION_KEY>
```

Create the Docker Compose file:

```bash
sudo nano docker-compose.yml
```

Paste the following configuration:

> Note: The example uses `TZ: Europe/London`. Replace it with your own timezone if needed.

```yaml
###############################################
# Common keys
###############################################

x-common-env: &common-env
  PUID: 1000
  PGID: 1000
  TZ: Europe/London

x-common-service: &common-service
  restart: unless-stopped
  logging:
    driver: json-file
  environment:
    <<: *common-env
  dns:
    - 1.1.1.1
    - 1.0.0.1

configs:
  jellyfin-serve:
    content: |
      {"TCP":{"443":{"HTTPS":true}},
      "Web":{"$${TS_CERT_DOMAIN}:443":{"Handlers":{"/":{"Proxy":"http://127.0.0.1:8096"}}}},
      "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":false}}

  seerr-serve:
    content: |
      {"TCP":{"443":{"HTTPS":true}},
      "Web":{"$${TS_CERT_DOMAIN}:443":{"Handlers":{"/":{"Proxy":"http://127.0.0.1:5055"}}}},
      "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":false}}

services:
###############################################
# RADARR - Movies
###############################################

  radarr:
    <<: *common-service
    container_name: radarr
    image: ghcr.io/hotio/radarr:latest
    ports:
      - 7878:7878
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/radarr:/config
      - /data:/data

###############################################
# SONARR - TV Shows
###############################################

  sonarr:
    <<: *common-service
    container_name: sonarr
    image: ghcr.io/hotio/sonarr:latest
    ports:
      - 8989:8989
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/sonarr:/config
      - /data:/data

###############################################
# LIDARR - Music
###############################################

  lidarr:
    <<: *common-service
    container_name: lidarr
    image: ghcr.io/hotio/lidarr:latest
    ports:
      - 8686:8686
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/lidarr:/config
      - /data:/data

###############################################
# BAZARR - Subtitles
###############################################

  bazarr:
    <<: *common-service
    container_name: bazarr
    image: ghcr.io/hotio/bazarr:latest
    ports:
      - 6767:6767
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/bazarr:/config
      - /data/media:/data/media

###############################################
# PROWLARR - Indexer Manager
###############################################

  prowlarr:
    <<: *common-service
    container_name: prowlarr
    image: ghcr.io/hotio/prowlarr:latest
    ports:
      - 9696:9696
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/prowlarr:/config

###############################################
# FLARESOLVERR - Cloudflare Solver
###############################################

  flaresolverr:
    <<: *common-service
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports:
      - 8191:8191
    environment:
      <<: *common-env
      LOG_LEVEL: info
      LOG_FILE: none
      LOG_HTML: "false"
      CAPTCHA_SOLVER: none
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/flaresolver:/config

###############################################
# QBITTORRENT - Downloader
###############################################

  qbittorrent:
    <<: *common-service
    container_name: qbittorrent
    image: ghcr.io/hotio/qbittorrent:latest
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    environment:
      <<: *common-env
      WEBUI_PORT: 8080
      TORRENTING_PORT: 6881
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/qbittorrent:/config
      - /data:/data

###############################################
# HOMARR - Homelab Dashboard
###############################################

  homarr:
    <<: *common-service
    container_name: homarr
    image: ghcr.io/homarr-labs/homarr:latest
    ports:
      - 7575:7575
    environment:
      <<: *common-env
      SECRET_ENCRYPTION_KEY: ${HOMARR_SECRET_ENCRYPTION_KEY}
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/homarr:/appdata
      - /var/run/docker.sock:/var/run/docker.sock

###############################################
# SEERR TAILSCALE - Remote Access
###############################################

  seerr-ts:
    image: tailscale/tailscale:latest
    container_name: seerr-ts
    hostname: seerr
    environment:
      TS_AUTHKEY: ${TS_AUTHKEY}
      TS_AUTH_ONCE: "true"
      TS_STATE_DIR: /var/lib/tailscale
      TS_SERVE_CONFIG: /config/serve.json
    volumes:
      - /docker/appdata/seerr-ts:/var/lib/tailscale
    configs:
      - source: seerr-serve
        target: /config/serve.json
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    ports:
      - 5055:5055
    restart: unless-stopped

###############################################
# SEERR - Media Requests
###############################################

  seerr:
    init: true
    restart: unless-stopped
    logging:
      driver: json-file
    container_name: seerr
    image: ghcr.io/seerr-team/seerr:latest
    network_mode: service:seerr-ts
    depends_on:
      - seerr-ts
    environment:
      TZ: Europe/London
      LOG_LEVEL: info
      PORT: 5055
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/seerr:/app/config

###############################################
# JELLYFIN TAILSCALE - Remote Access
###############################################

  jellyfin-ts:
    image: tailscale/tailscale:latest
    container_name: jellyfin-ts
    hostname: jellyfin
    environment:
      TS_AUTHKEY: ${TS_AUTHKEY}
      TS_AUTH_ONCE: "true"
      TS_STATE_DIR: /var/lib/tailscale
      TS_SERVE_CONFIG: /config/serve.json
    volumes:
      - /docker/appdata/jellyfin-ts:/var/lib/tailscale
    configs:
      - source: jellyfin-serve
        target: /config/serve.json
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    ports:
      - 8096:8096
    restart: unless-stopped

###############################################
# JELLYFIN - Media Server
###############################################

  jellyfin:
    <<: *common-service
    container_name: jellyfin
    image: ghcr.io/hotio/jellyfin:latest
    network_mode: service:jellyfin-ts
    depends_on:
      - jellyfin-ts
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/jellyfin:/config
      - /data/media:/data/media:ro

###############################################
# ARR Stack Dedicated Network
###############################################

networks:
  default:
    name: arr_network
```

Save and exit Nano:

```text
Ctrl + O
Enter
Ctrl + X
```

Validate the Compose file and start the stack:

```bash
cd /data
sudo mkdir -p /docker/appdata/homarr
sudo chown -R 1000:1000 /docker/appdata/homarr
sudo docker compose config
sudo docker compose up -d
```

Check the containers:

```bash
sudo docker compose ps
```

Your `/data` folder should now contain:

```text
/data
+-- .env
+-- docker-compose.yml
+-- media
+-- torrents
```

The ARR stack is now running inside the Ubuntu VM.

---

## Service URLs and Ports

Use the VM IP address followed by the service port:

```text
192.168.1.222
```

| Service | URL | Port |
| --- | --- | --- |
| Radarr | `http://192.168.1.222:7878` | `7878` |
| Sonarr | `http://192.168.1.222:8989` | `8989` |
| Lidarr | `http://192.168.1.222:8686` | `8686` |
| Bazarr | `http://192.168.1.222:6767` | `6767` |
| Prowlarr | `http://192.168.1.222:9696` | `9696` |
| qBittorrent | `http://192.168.1.222:8080` | `8080` |
| Jellyfin | `http://192.168.1.222:8096` | `8096` |
| Homarr | `http://192.168.1.222:7575` | `7575` |
| Seerr | `http://192.168.1.222:5055` | `5055` |

Internal Docker URLs:

| App | Internal URL |
| --- | --- |
| Prowlarr | `http://prowlarr:9696` |
| Radarr | `http://radarr:7878` |
| Sonarr | `http://sonarr:8989` |
| Lidarr | `http://lidarr:8686` |
| qBittorrent | `http://qbittorrent:8080` |

---

## Initial App Configuration

### qBittorrent

Get the temporary login details:

```bash
cd /data
sudo docker logs qbittorrent
```

The logs should show the username, usually `admin`, and a temporary password.

Open qBittorrent:

```text
http://192.168.1.222:8080
```

Then:

1. Log in with the temporary credentials.
2. Go to `Tools` -> `Options` -> `Web UI`.
3. Change the password.
4. Save the settings.

Create these categories:

| Category | Save path |
| --- | --- |
| `movies` | `movies` |
| `tv` | `tv` |
| `music` | `music` |

Then go to `Tools` -> `Options` -> `Downloads` and configure:

| Setting | Value |
| --- | --- |
| Default Torrent Management Mode | `Automatic` |
| When Torrent Category changed | `Relocate torrent` |
| When Default Save Path changed | `Switch affected torrents to Manual Mode` |
| When Category Save Path changed | `Switch affected torrents to Manual Mode` |
| Use Subcategories | Enabled |
| Use Category paths in Manual Mode | Enabled |
| Default Save Path | `/data/torrents` |

Saving `.torrent` files is optional. You can leave that field blank.

### Prowlarr

Open Prowlarr:

```text
http://192.168.1.222:9696
```

Create authentication:

| Setting | Value |
| --- | --- |
| Method | `Forms` |
| Username | Your choice |
| Password | Your choice |

Add qBittorrent as a download client:

1. Go to `Settings` -> `Download Clients`.
2. Click `+`.
3. Select `qBittorrent`.
4. Disable `Use SSL`, unless you configured SSL in qBittorrent.
5. Set `Host` to `qbittorrent`.
6. Set `Port` to `8080`.
7. Enter the qBittorrent username and password.
8. Click `Test`.
9. If the test succeeds, click `Save`.

### Radarr

Open Radarr:

```text
http://192.168.1.222:7878
```

Configure media management:

1. Go to `Settings` -> `Media Management`.
2. Add the root folder:

```text
/data/media/movies
```

3. Enable `Show Advanced`.
4. Under `Importing`, enable `Use Hardlinks instead of Copy`.

Optional settings:

| Setting | Suggested value |
| --- | --- |
| Rename Movies | Enabled |
| Delete empty movie folders during disk scan | Enabled |
| Import Extra Files | Enabled |
| Import Extra Files extensions | `srt,sub,nfo` |

Add qBittorrent as a download client:

1. Go to `Settings` -> `Download Clients`.
2. Click `+`.
3. Select `qBittorrent`.
4. Set `Host` to `qbittorrent`.
5. Set `Port` to `8080`.
6. Make sure `Use SSL` is disabled.
7. Enter the qBittorrent username and password.
8. Set `Category` to `movies`.
9. Click `Test`.
10. If the test succeeds, click `Save`.

Connect Radarr to Prowlarr:

1. In Radarr, go to `Settings` -> `General`.
2. Copy the API key.
3. In Prowlarr, go to `Settings` -> `Apps`.
4. Click `+`.
5. Select `Radarr`.
6. Paste the Radarr API key.
7. Set `Prowlarr Server` to:

```text
http://prowlarr:9696
```

8. Set `Radarr Server` to:

```text
http://radarr:7878
```

9. Click `Test`.
10. If the test succeeds, click `Save`.

### Sonarr

Open Sonarr:

```text
http://192.168.1.222:8989
```

Configure media management:

1. Go to `Settings` -> `Media Management`.
2. Add the root folder:

```text
/data/media/tv
```

3. Enable `Show Advanced`.
4. Under `Importing`, enable `Use Hardlinks instead of Copy`.

Optional settings:

| Setting | Suggested value |
| --- | --- |
| Rename Episodes | Enabled |
| Delete empty series and season folders during disk scan | Enabled |
| Import Extra Files | Enabled |
| Import Extra Files extensions | `srt,sub,nfo` |

Add qBittorrent as a download client:

1. Go to `Settings` -> `Download Clients`.
2. Click `+`.
3. Select `qBittorrent`.
4. Set `Host` to `qbittorrent`.
5. Set `Port` to `8080`.
6. Make sure `Use SSL` is disabled.
7. Enter the qBittorrent username and password.
8. Set `Category` to `tv`.
9. Click `Test`.
10. If the test succeeds, click `Save`.

Connect Sonarr to Prowlarr:

1. In Sonarr, go to `Settings` -> `General`.
2. Copy the API key.
3. In Prowlarr, go to `Settings` -> `Apps`.
4. Click `+`.
5. Select `Sonarr`.
6. Paste the Sonarr API key.
7. Set `Prowlarr Server` to:

```text
http://prowlarr:9696
```

8. Set `Sonarr Server` to:

```text
http://sonarr:8989
```

9. Click `Test`.
10. If the test succeeds, click `Save`.

### Lidarr

Open Lidarr:

```text
http://192.168.1.222:8686
```

Configure media management:

1. Go to `Settings` -> `Media Management`.
2. Add the root folder:

```text
/data/media/music
```

3. Give it a name, for example `Music`.
4. Save the root folder.

Add qBittorrent as a download client:

1. Go to `Settings` -> `Download Clients`.
2. Click `+`.
3. Select `qBittorrent`.
4. Set `Host` to `qbittorrent`.
5. Set `Port` to `8080`.
6. Make sure `Use SSL` is disabled.
7. Enter the qBittorrent username and password.
8. Set `Category` to `music`.
9. Click `Test`.
10. If the test succeeds, click `Save`.

Connect Lidarr to Prowlarr:

1. In Lidarr, go to `Settings` -> `General`.
2. Copy the API key.
3. In Prowlarr, go to `Settings` -> `Apps`.
4. Click `+`.
5. Select `Lidarr`.
6. Paste the Lidarr API key.
7. Set `Prowlarr Server` to:

```text
http://prowlarr:9696
```

8. Set `Lidarr Server` to:

```text
http://lidarr:8686
```

9. Click `Test`.
10. If the test succeeds, click `Save`.

### Bazarr

Open Bazarr:

```text
http://192.168.1.222:6767
```

Configure Bazarr:

1. Go to `Settings` -> `Languages`.
2. Create a language profile, for example `English` or `Any`.
3. Go to `Settings` -> `Providers`.
4. Add your subtitle providers.
5. Connect Radarr and Sonarr.
6. Go to the `Series` or `Movies` tab.
7. Click `Update` to pull in your existing library.

Some subtitle providers require a free account.

### Restart Services

After the initial configuration, restart the stack:

```bash
cd /data
sudo docker compose down
sudo docker compose up -d
```

Check that the containers are running:

```bash
sudo docker compose ps
```

### Jellyfin

Open Jellyfin:

```text
http://192.168.1.222:8096
```

Create a user and password.

Add these libraries:

| Library | Path |
| --- | --- |
| Movies | `/data/media/movies` |
| TV Shows | `/data/media/tv` |
| Music | `/data/media/music` |

### Homarr

Open Homarr:

```text
http://192.168.1.222:7575
```

Create an account, then connect your services.

Recommended setup:

- Add dashboard widgets.
- Add service tiles.
- Add API integrations.

### Seerr

Open Seerr:

```text
http://192.168.1.222:5055
```

Configure Seerr:

- Connect Jellyfin.
- Connect Radarr.
- Connect Sonarr.
- Configure request permissions.

### Tailscale

I recommend watching this [video](https://www.youtube.com/watch?v=E34gf3GC7xw) from Tailscale on how to set up Jellyfin. You can do the same step for Seerr as well after watching this guide.

After watching this guide you should be able to set up Tailscale properly so you can have access remotely to Seerr and Jellyfin.

You should ultimately get something like this: 

```text
https://jellyfin.tailXXXXX.ts.net
https://seerr.tailXXXXXX.ts.net
```

---

## References

- [Automation Avenue ARR guide](https://github.com/automation-avenue/arr-new)
- [Trash Guides](https://trash-guides.info)
- [Servarr Wiki](https://wiki.servarr.com)
- [Proxmox documentation](https://pve.proxmox.com/wiki/Main_Page)
- [Docker documentation](https://docs.docker.com)
- [Tailscale documentation](https://tailscale.com/kb)
