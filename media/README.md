# Self-Hosted Media

This is currently a work in progress. If you have any issues please submit or suggest any edits.

# Navigation
- [Apps](apps/)
- [Media Server](media/)
  - [Arr Apps](#arr-apps)
    - [Included Applications](#included-applications)
    - [How It Works](#how-it-works)
    - [Features](#features)
  - [Data Folders](#data-folders)
  - [Download Clients](#download-clients)
    - [NZBGet](#nzbget)
    - [qBittorrent](#qbittorrent)
  - [Gluetun](#gluetun-vpn)
    - [Testing Gluetun Connectivity](#testing-gluetun=connectivity)
    - [Passing Through Containers](#passing-through-containers)
    - [External Container to Gluetun](#external-container-to-gluetun)
    - [Container in another compose.yml](#container-in-another-docker-compose.yaml)
    - [Arr apps wont connect to prowlarr](#arr-apps-wont-connect-to-prowlarr)
    - [Gluetun Proxmox Fix](#gluetun-proxmox-fix)
    - [Reduce Gluetun Ram Usage](#reduce-gluetun-ram-usage)
    - [Testing Containers](#testing-other-containers)
  - [Jellyfin Companion Stack](#jellyfin-companion-stack)
    - [How They Work Together](#how-they-work-together)
- [Remote Access](access/)


# Data Folders
My setup uses the same folder structure from TechHutTV. 
```
data
├── downloads
│   ├── qbittorrent
│   │   ├── completed
│   │   ├── incomplete
│   │   └── torrents
│   └── nzbget
│       ├── completed
│       ├── intermediate
│       ├── nzb
│       ├── queue
│       └── tmp
├── movies
├── music
├── shows
└── youtube
```

Create the directory:
```
mkdir -p downloads/qbittorrent/{completed,incomplete,torrents} && mkdir -p downloads/nzbget/{completed,intermediate,nzb,queue,tmp}
```

# Gluetun VPN
### Testing Gluetun Connectivity
Once your containers are up, you can test your connection to your provider.
```
docker run --rm --network=container:gluetun alpine:3.18 sh -c "apk add wget && wget -qO- https://ipinfo.io"
```

### Passing Through Containers
When containers are in the same docker compose, all you need to add is a network_mode: `service:container_name` and open the ports through the the gluetun container. See example from the arr-compose.yaml
```
services:
  gluetun: # This config is for wireguard tested with Mullvad
    image: qmcgaw/gluetun
    container_name: gluetun
    ...
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ...
    network_mode: service:gluetun
```
### External Container to Gluetun
Add `--network=container:gluetun` when launching a container from outside the stack.

### Container in another docker-compose.yaml
Add `network_mode: "container:gluetun"` to your docker-compose.yaml. Ensure you open the ports through the gluetun container.

### Arr apps wont connect to prowlarr
This setup will run Prowlarr though a VPN as it is the service that will fetch magnet links and skim torrent sites. If you have issues with connections between arr apps and prowlarr you need to allow gluetun to access your lan. You can do this by adding the following to your docker compose file under the glutun enviromental varibles.

`- FIREWALL_OUTBOUND_SUBNETS=192.168.1.0/24 #change to your specific subnet`

### Gluetun Proxmox Fix
"cannot Unix Open TUN device file: operation not permitted and cannot create TUN device file node: operation not permitted" May happen if you're running this on LXC containers.

Find your container number, mine is 110

edit `/etc/pve/lxc/110.conf` and add:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Make sure you pass through the tun device (/dev/net/tun:/dev/net/tun) as shown in arr-compose.yaml.

### Reduce Gluetun Ram Usage
Gluetun bundles a recursive caching DNS resolver called unbound for handling domain name requests securely. Over time the cache size, which rests in RAM, can balloon to gigabytes.

You can reduce this by adding the following to your docker compose file under the glutun enviromental varibles.

`BLOCK_MALICIOUS=off`
This may not be an issue as DNS over HTTPS in Go to replace Unbound is implemented.

### Testing Other Containers
Jump into the Exec console and run the wget command below. Tested with qbittorrent and prowlarr. Ensure you open the ports through the the gluetun container.
```
docker exec -it conatiner_name bash
wget -qO- https://ipinfo.io
```

# Download Clients

### NZBGet
**Fix directory does not appear to exist inside the container error**
This error may appear within Sonarr and Radarr. Once NZBGet is setup go to settings and under INCOMING NZBS change the AppendCategoryDir to No. This will prevent some potential mapping issues and save on unnessesary directories.

### qBittorrent
**qBittorrent Stalls with VPN Timeout**
I noticed that qBittorrent stalls out if there is a timeout or any type of interuption on the VPN. This is good because it drops connection, but I need it to start back up when the connection is restored without manually restarting the container.

**Solution #1:** Within the WebUI of qbittorrent head over to advanced options and select `tun0` as the networking interface. See image below.

![image](https://github.com/user-attachments/assets/cb6ea317-db4a-4af2-a46a-52e1d43296f0)

**Solution #2:** Deunhealth container - automatically restart qbittorrent and prowlarr when they give an unheathly status.
Add the deunhealth service to your stack. This is already included in arr-compose.yaml
```
  deunhealth:
    image: qmcgaw/deunhealth
    container_name: deunhealth
    network_mode: "none"
    environment:
      - LOG_LEVEL=info
      - HEALTH_SERVER_ADDRESS=127.0.0.1:9999
      - TZ=America/Halifax
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
Next add a healthcheck and label to the container(s). Add `deunhealth.restart.on.unhealthy=true` as a label and a simple ping health check as shown below.
```
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    labels:
      deunhealth.restart.on.unhealthy=true # Label for deunhealth
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /docker/qbittorrent:/config
      - /data:/data
    network_mode: service:gluetun
    healthcheck:
        test: ping -c 1 www.google.com || exit 1
        interval: 60s
        retries: 3
        start_period: 20s
        timeout: 10s
```
Resources: [DBTech Video on Deunhealth](https://www.youtube.com/watch?v=Oeo-mrtwRgE)

# Arr apps
The **ARR Stack** is a powerful collection of self-hosted applications designed to automate and organize your entire media library—from movies and TV shows to music and subtitles. This stack simplifies the process of searching, downloading, renaming, organizing, and even subtitle syncing—all while seamlessly integrating with your preferred torrent or Usenet clients.

## Included Applications

| App       | Role                                         | Description |
|-----------|----------------------------------------------|-------------|
| **Radarr** | 🎥 Movie Management                         | Automates the discovery, downloading, and organization of movies. Supports custom quality profiles, scheduled searches, and renaming logic. |
| **Sonarr** | 📺 TV Show Management                       | Tracks your favorite series, monitors for new episodes, downloads automatically, and organizes them with your preferred naming scheme. |
| **Lidarr** | 🎵 Music Management                         | Designed for managing music collections. Lidarr can monitor artists, albums, and tracks—automating downloads and organizing your music library. |
| **Bazarr** | 📝 Subtitle Management                      | Fetches and syncs subtitles for your TV shows and movies using the metadata from Radarr and Sonarr. Supports multiple languages and providers. |
| **Prowlarr** | 🔍 Indexer Aggregator & Manager         | Acts as a central indexer manager for all other *ARR* apps. Supports over 500 indexers and integrates directly with Radarr, Sonarr, Lidarr, and others for seamless search and filtering. |

## How It Works

1. **Prowlarr** manages your indexers and passes metadata to Radarr/Sonarr/Lidarr.
2. **Radarr**, **Sonarr**, and **Lidarr** monitor for media content you want—whether it's movies, shows, or music.
3. They communicate with your download clients (like qBittorrent, NZBGet, or SABnzbd) to fetch the files.
4. Once downloaded, the apps automatically rename and move them to the correct folders.
5. **Bazarr** then detects the new media and fetches the best available subtitles for them.
6. The result: a fully organized, ready-to-watch/listen media library—hands-free.

## Features

- Automated downloading with custom quality and language profiles
- Intelligent media organization and renaming
- Subtitles fetched and synced automatically
- Supports both Torrent and Usenet
- API support and full integration between components
- Easily extensible with third-party apps and downloaders

# Jellyfin Companion Stack
This stack extends Jellyfin's capabilities with modern interfaces and insightful statistics. It includes multiple apps working in harmony to enhance your media server experience, while still offering API compatibility for third-party dashboards and integrations.

| App             | Role                      | Description                                                                                                                                                   |
|------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Jellyseerr**   | 🌟 Request Management     | A fork of Overseerr tailored for Jellyfin. Lets users request movies and TV shows via a sleek web interface, with admin approval workflows and notifications. |
| **StreamyStats** | 📊 Live Statistics UI     | A visually polished dashboard to view real-time and historical playback statistics from Jellyfin. Preferred for its elegant UI and detailed user insights.    |
| **Jellystat**    | ⚙️ Stats API Backend      | Provides a backend API to expose Jellyfin statistics. While not as visually refined as StreamyStats, it is still vital for supporting external tools.         |

### How They Work Together

- **Jellyseerr** handles user requests and approval queues for your Jellyfin media.
- **StreamyStats** connects to your Jellyfin server to display current viewing sessions, history, bandwidth usage, and user activity in a modern dashboard.
- **Jellystat** silently runs in the background to provide an API layer, ensuring compatibility with Grafana dashboards and other tools that rely on Jellyfin metrics.

> You get the best of both worlds: a beautiful UI from StreamyStats, and broad API support from Jellystat.


# No VPN arr-compose (NOT RECOMMENDED)
```
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - 8080:8080 # web interface
      - 6881:6881 # torrent port
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /docker/qbittorrent:/config
      - /data:/data

  nzbget:
    image: lscr.io/linuxserver/nzbget:latest
    container_name: nzbget
    ports:
      - 6789:6789
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
      - NZBGET_USER=user
      - NZBGET_PASS=password
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/nzbget:/config
      - /data:/data
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    ports:
      - 9696:9696
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/prowlarr:/config
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/sonarr:/config
      - /data:/data
    ports:
      - 8989:8989

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/radarr:/config
      - /data:/data
    ports:
      - 7878:7878

  lidarr:
    container_name: lidarr
    image: lscr.io/linuxserver/lidarr:latest
    restart: unless-stopped
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/lidarr:/config
      - /data:/data
    environment:
     - PUID=1000
     - PGID=1000
     - TZ=America/Halifax
    ports:
      - 8686:8686

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Halifax
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/bazarr:/config
      - /data:/data
    ports:
      - 6767:6767

  flaresolverr:
    # DockerHub mirror flaresolverr/flaresolverr:latest
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=America/Halifax
    ports:
      - 8191:8191
    restart: unless-stopped
```
