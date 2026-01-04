# Automated Home Media Server
A complete guide to setting up a home media server with automated requests/downloads.
This guide uses a lot of information from the [TRaSH Guides](https://trash-guides.info/). We will be using their:
- File and folder structure
- User setup and permissions
- Quality profiles
Of course you can adjust anything you'd like.

## Table of Contents
- [Automated Home Media Server](#automated-home-media-server)
  - [Table of Contents](#table-of-contents)
- [The Stack](#the-stack)
  - [Core software](#core-software)
  - [Downloading](#downloading)
  - [*arr Stack](#arr-stack)
  - [Optional software](#optional-software)
- [Basic installation](#basic-installation)
  - [Docker & Docker compose](#docker--docker-compose)
  - [User setup](#user-setup)
  - [File & Folder setup](#file--folder-setup)     

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

## Optional software
**[NPMPlus](github.com/ZoeyVid/NPMplus/)** is an improved fork of nginx Proxy Manager; a webui that allows you to run reverse proxies with automatic TLS certificate creation and renewal via Let's Encrypt

# Basic installation
## Docker & Docker compose
You need to install [Docker](https://docs.docker.com/engine/install/) & [Docker Compose]([https://docs.docker.com/compose](https://docs.docker.com/compose/install/)) for your setup. Preferably use Docker Compose V2.
Both guides on the Docker website should be sufficient for you to install it.

## User & Group setup
An often overlooked part of the *arr stack is that it works best with individual users per app; that way you can give folder permissions (chown) only where they're needed. For our stack we will be starting our ID's for users at 13000. We will be be creating a new user called `Mediaserver` that we will use to create all folders and to run Docker Compose. 

Run the following code to create all `users` and a usergroup for them; `mediacenter`
```
sudo useradd mediauser -u 13000
sudo useradd qbittorrent -u 13001
sudo useradd sabnzbd -u 13002
sudo useradd sonarr -u 13003
sudo useradd radarr -u 13004
sudo useradd prowlarr -u 13005
sudo useradd bazarr -u 13006
sudo useradd seerr -u 13007
sudo useradd homepage -u 13008
sudo useradd unmanic -u 13009
sudo useradd npmplus -u 13010
sudo groupadd mediacenter -g 13000
```

Then we want to add all users to the mediacenter

## File & Folder setup
We are using [TRaSH Guides' setup on file & folder structure](https://trash-guides.info/File-and-Folder-Structure/), as well as using individual users for each Docker container.
The folder structure will look something like: 
```
data
├── torrents
│   ├── books
│   ├── movies
│   ├── music
│   └── tv
├── usenet
│   ├── incomplete
│   └── complete
│       ├── books
│       ├── movies
│       ├── music
│       └── tv
└── media
    ├── books
    ├── movies
    ├── music
    └── tv
```






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
