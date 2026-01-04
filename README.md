# Automated Home Media Server
A complete guide to setting up a home media server with automated requests/downloads

## Table of Contents
[The Stack](#the_stack)  
[Emphasis](#emphasis)   
<a name="headers"/>

# The Stack
## Core software
**Docker** lets us run and isolate each of our services into a container. Everything for each of these services will live in the container except the configuration files which will live on our host.

**Jellyfin** is an open source media server. Clients aren't as flashed out compared to Plex, but it doesn't lock anyting (like transcoding) behind a paywall. There is a high variety of clients for Android TV / iOS / Android / PC

## Downloading
**qBittorrent** is a torrent client. Transmission and Deluge are also popular choices but I chose qBittorrent because you can easily configure it to only operate over the VPN connection.

**SABNZBD** is a usenet client. Usenet has my personal preference over torrent clients due to more consistent quality, unless you're using private trackers.

## *arr Stack
The *arr stack is full of apps that integrate with eachother very well. They will completely automate the process of:
1. Allowing users to request media via Seerr
2. Looking up indexers/trackers/RSS feeds
3. Compare results to quality profiles imported via Configarr

**[Sonarr](https://sonarr.tv/)** is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them, and moving them to your library. It will also be able to check RSS feeds for information

**[Radarr](https://radarr.video/)** is a fork of Sonarr that does all the same stuff but for Movies.

**[Prowlarr](https://prowlarr.com/)** is a tool that Sonarr and Radarr use to search indexers and trackers for torrents and newsgroups.

**FlareSolverr** can optionally be installed alongside Prowlarr to bypass cloudflare protections. Required for certain indexers (ex. 1337).

**[Bazarr](https://www.bazarr.media/)** automatically downloads and synchronises subtitles in your preferred language.

**[Seerr](https://github.com/Fallenbagel/jellyseerr)** is a fork of Overseerr that's now merging codebase with Jellyseerr itself to automate

**[Homepage](https://github.com/gethomepage/homepage)** is a dashboard for keeping track all of these web services

**[Unmanic](https://docs.unmanic.app/)** is a great tool for optimising media files. For example, you can use it to remove unneccessary subtitles/audio tracks and transcode media to your desired format.

**[nginx-proxy-manager](https://nginxproxymanager.com/guide/#quick-setup)** is a simple reverse proxy service for making Jellyfin accessible outside of your local network

**Update 10/25/25** I've personally switched from nginx-proxy-manager to [Caddy](https://caddyserver.com/). But this is strictly a personal preference.
## Optional software
**[NPMPlus](github.com/ZoeyVid/NPMplus/)** is an improved fork of nginx Proxy Manager; a webui that allows you to serve 

# Installing Docker

Installation steps will vary based on distro. I recommend following the instructions in the [Docker documentation](https://docs.docker.com/engine/install/).

You will also need to install Docker Compose. Again I recommend following the official [Docker documentation](https://docs.docker.com/compose/install).

I like to keep my configs in one easy place, so let’s create a folder to hold our docker container configs and create our docker-compose.yml file

```
mkdir ~/docker
touch ~/docker/docker-compose.yml
```

And add this to the top of your docker-compose file. We will be filling in the services in the coming steps. If you are confused on how to add the services or how your file should look, [here is a good resource on docker-compose]([https://docs.docker.com/compose/compose-file/compose-file-v2/](https://docs.docker.com/reference/compose-file/)).

```
services:
```

# Jellyfin Docker Config

Add the following lines to ~/docker/docker-compose.yml under "services:" Pay close attention to the indentations.

```
  jellyfin:
    image: ghcr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ./jellyfin:/config
      - /path/to/media:/media
    ports:
      - 8096:8096
    restart: unless-stopped
```

This will start your Jellyfin server on port 8096, and add the volumes ~/docker/jellyfin and /path/to/media onto the container.

Replace /path/to/media to your media directory. Your media directory should contain subdirectories labelled "tv" and "movies".

Make sure those PUID and GUID match the ID for your user and group. You can find these by simply typing "id" into terminal. If they don't match replace the PUID and GUID in the docker compose script with the ones you found in terminal.

# VPN Docker Config

We will use Gluetun to setup your VPN connection in a Docker container. This way, services that rely on the VPN such as qbittorrent can access it while the host and public facing services do not.

The exact configuration will vary based on which VPN provider you use. I recommend one that allows for port forwarding, as it will allow you to seed more reliably. This may not be important if you only use public trackers, but most private trackers are strict about maintaining a good seeding ratio. I will leave this shell for you to fill in with your VPN info. For more information on how to set it up, please see the [Gluetun documentation](https://github.com/qdm12/gluetun).

```
  gluetun:
    image: qmcgaw/gluetun
    # container_name: gluetun
    # line above must be uncommented to allow external containers to connect.
    # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/connect-a-container-to-gluetun.md#external-container-to-gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 8888:8888/tcp # HTTP proxy
      - 8388:8388/tcp # Shadowsocks
      - 8388:8388/udp # Shadowsocks
      - 8080:8080 # qbittorrent
      - 9696:9696 # prowlarr
    volumes:
      - ./gluetun:/gluetun
    environment:
      # See https://github.com/qdm12/gluetun-wiki/tree/main/setup#setup
      - PUID=1000
      - PGID=1000
      - VPN_SERVICE_PROVIDER=ivpn
      - VPN_TYPE=openvpn
      # OpenVPN:
      - OPENVPN_USER=
      - OPENVPN_PASSWORD=
      # Wireguard:
      # - WIREGUARD_PRIVATE_KEY=wOEI9rqqbDwnN8/Bpp22sVz48T71vJ4fYmFWujulwUU=
      # - WIREGUARD_ADDRESSES=10.64.222.21/32
      # Timezone for accurate log times
      - TZ=
      # Server list updater
      # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/servers.md#update-the-vpn-servers-list
      - UPDATER_PERIOD=
```

# qBittorrent Docker Config

```
qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent:/config
      - /path/to/qbittorrent-downloads:/downloads
    network_mode: service:gluetun
    restart: unless-stopped
 ```

Replace /path/to/qbittorrent-downloads with the directory you would like qBittorrent to store downloads.

We added network_mode: service:gluetun here so that all qbittorrent traffic gets routed through the VPN container.

Notice that we did not add the ports: section. When routing network traffic through another container, the ports have to be defined in the VPN container instead.

# Prowlarr Docker Config

```
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:develop
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./prowlarr:/config
    network_mode: service:gluetun
    ports:
      - 9696:9696
    restart: unless-stopped
```

This is super basic and just boots your Prowlarr service on port 9696. Doesn’t need much else!

# Flaresolverr
```
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    ports:
      - 8191:8191
    volumes:
      - ./flaresolver:/config
    restart: unless-stopped
```

# Sonarr & Radarr Docker Config

```
  sonarr:
    image: ghcr.io/linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ./sonarr:/config
      - /path/to/media/tv:/tv
      - /path/to/qbittorrent-downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
  radarr:
    image: ghcr.io/linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ./radarr:/config
      - /path/to/media/movies:/movies
      - /path/to/qbittorrent-downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
 ```
Replace /path/to/media/tv & /path/to/media/movies with the directories you created during the Plex docker setup. Replace /path/to/qbittorrent-downloads with the directory you created during the qBittorrent docker setup.

Make sure that user has read-write permissions for the media directories. Sonarr and Radarr are going to try to be creating folders and files in there when they copy or hard link files over. You can do this with the following command: ```chmod -r +rw /path/to/media```

# Start it up!

Run this command to boot up all your services! Remember to go back and update your qBittorrent settings after this.

```
cd ~/docker
docker compose up -d
```

You now have these services running locally. Go to their web UI’s and configure everything.

```
sonarr       @ http://localhost:8989
radarr       @ http://localhost:7878
prowlarr     @ http://localhost:9696
qbittorrent  @ http://localhost:8080
jellyfin     @ http://localhost:8096
```

# Configuration

## Jellyfin

1. Navigate to http://localhost:8096
2. Follow the setup wizard
3. Add libraries for TV and Movies. The library folders should be located at /media/movies and /media/tv respectively.

See [documentation](https://jellyfin.org/docs/general/quick-start.html) for more information

## qBittorrent

### Set Download Directory

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tools > Options
3. Under the "Downloads" tab change "Default Save Path" to /downloads/completed
4. Check the box next to "Keep incomplete torrents in:" Make sure it's set to /downloads/incomplete

### Bind qBittorrent to VPN

This will ensure that qBittorrent only downloads over your VPN connection.

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tool > Options
3. Go to the Advanced tab
4. Change Network interface from Any to the interface corresponding to your VPN (probably something like tun0)

### Change default login (Optional)

By default the login to the web UI is just "admin" "adminadmin". Personally I don't find it worth changing since qBittorrent is only accessible through my home network. I wouldn't recommend making it publically available unless you have a good reason, but if you do you should at least change the default login.

1. Navigate to Tools > Options > WebUI > Authentication
3. Enter a secure username and password.

## Update 10/25/25:
Newer installs of qBittorrent reportedly generate a random password on install. This password can be found by running `docker logs qbittorrent`.

## Prowlarr

See [Prowlarr Quick Start Guide](https://wiki.servarr.com/prowlarr/quick-start-guide)

## Sonarr / Radarr

### Connect to qBittorrent

1. Navigate to Settings > Downloads Clients > Add
2. Select "qBittorrent"
3. Enter "qBittorrent" in the "Name" field
4. Set host and port to the qBittorrent Web UI. If you followed this guide correctly the default values should be correct
5. Enter the Username and Password for the webui
6. Click "Save"


### Set root directory

1. Navigate to Settings > Media Management
2. Scroll to the bottom of the page and click on "Add Root Folder"
3. Sonarr: Select /tv
4. Radarr: Select /movies

### Create quality profile

You will want to create a quality profile to specify what resolution you want your TV Shows and Movies to be.

1. Navigate to Settings > Profiles
3. Select the Any profile
4. Check all qualities you would like to allow and uncheck all qualities you would like to disable. For example if you want your movie quality to cap out at 1080p to save disk space uncheck everything above Bluray-1080p. Sonarr/Radarr will prioritize the highest allowed resolution but will download lower allowed ones if it can't find it.
5. (Optional) Allow upgrades by checking the "Upgrades Allowed" Checkbox. You can then change the "Upgrade Until" drop down to your prefered maximum resolution. This is useful for if you want your library to be entirely 1080p but the only torrent available for a specific show or movie is 720p. This way it will still download the 720p torrent but if a 1080p torrent ever comes along it will automatically download it and replace the 720p version.


### File renaming (Optional)

I like having this setting enabled to keep my media folders nice and organized.

1. Navigate to Settings > Media Management
2. Check the checkbox under "Rename Episodes/Movies"
3. (Optional) Configure episode format by preference

### Connect Sonarr and Radarr

**Sonarr:**

1. Navigate to Settings > TV > Sonarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:8989
4. Enter API key. You can find this in Sonarr under Settings > General
5. Leave base URL blank unless you configured one in Sonarr
6. Under Sonarr Interface click on "Load Qualities", "Load Folders", and "Load Languages"
7. Select "Any" under Quality Profiles
8. Select /tv under Default Root Folders
9. Select your prefered languages under Language Profiles


**Radarr:**


1. Navigate to Settings > Movies > Radarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:7878
4. Enter API key. You can find this in Radarr under Settings > General
5. Leave base URL blank unless you configured one in Radarr
6. Under Radarr Interface click on "Load Profiles" and "Load Root Folders"
7. Select "Any" under Quality Profiles
8. Select /movies under Default Root Folders
9. Select Physical / Web under Default Minimum Availability. Optionally you could select and earlier setting in case a movie gets leaked before being released to DVD but you will more often than not probably just get cam recordings.

# Advanced Configuration

**Multiple Hard Drives**

As your media library grows, you will eventually run into the issue of storage space. The obvious solution is to get more hard drives, but it's less obvious how to distribute files across multiple drives with this setup. The best solution in my experience is to use [MergerFS](https://github.com/trapexit/mergerfs). MergerFS essentially combines multiple hard drives into a single folder on your filesystem. Then you can point your '/path/to/media' folder to the MergerFS folder. As opposed to traditional solutions such as RAID, MergerFS allows any combination of drives regardless of brand or storage capacity. However, it does not provide any redundancy. It is possible to combine MergerFS with [SnapRAID](https://www.snapraid.it/) if you'd like some extra redundancy, but personally I have never used it.

**NPMPlus + Crowdsec**
Use the following whitelist:
```
name: crowdsecurity/jellyfin-whitelist
description: "Whitelist events from jellyfin"
filter: "evt.Meta.service == 'http' && evt.Meta.log_type in ['http_access-log', 'http_error-log']"
whitelist:
  reason: "Jellyfin whitelist"
  expression:
   - evt.Meta.http_status == '403' && evt.Meta.http_verb == 'POST' && evt.Meta.http_path matches '^/Sessions/Playing/Progress$' # When playing videos
   - evt.Meta.http_status == '404' && evt.Meta.http_verb == 'GET' && evt.Meta.http_path matches '(?i)^/items/([a-z0-9\\-]+)/images/(thumb|primary)' # When brow>
   - evt.Meta.http_verb == 'GET' && evt.Meta.http_path contains '/HomeScreen/CachedImage/' # When using Custom Home Sections plugin with Jellyseerr integration
```
