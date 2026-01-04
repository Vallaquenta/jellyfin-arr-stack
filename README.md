# Automated Home Media Server
A complete guide to setting up a home media server with automated requests/downloads.
This guide uses a lot of information from the [TRaSH Guides](https://trash-guides.info/). We will be using their:
- File and folder structure
- User setup and permissions
- Quality profiles
Of course you can adjust anything you'd like.

## Table of Contents
[The Stack](#the-stack)  
[Emphasis](#emphasis)   

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

# Installing Docker & Docker Compose


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
