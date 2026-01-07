# Automated Home Media Server
## Features
A complete guide to setting up a home media server with automated requests/downloads
we will set up the following flow:
1. Allow users to request media via Seerr
2. Looking up indexers/trackers/RSS feeds via Prowlarr
3. Compare results to quality profiles imported via Configarr
4. Select best matching result (ie. Bluray 1080p) and add to download client via Sonarr/Radarr
5. Download movie/series via Qbittorrent/SABNZBD
6. Automatically look up subtitles & synchronise them with Bazarr
7. Move to Jellyfin library & add to Jellyfin
8. Give public access to Jellyfin with a secure reverse proxy via NPMplus + Crowdsec
9. Manage unwatched files & unwatched requests with Maintainarr

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
**[Sonarr](https://sonarr.tv/)** is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them, and moving them to your library. It will also be able to check RSS feeds for information

**[Radarr](https://radarr.video/)** is a fork of Sonarr that does all the same stuff but for Movies.

**[Prowlarr](https://prowlarr.com/)** is a tool that Sonarr and Radarr use to search indexers and trackers for torrents and newsgroups.

**FlareSolverr** can optionally be installed alongside Prowlarr to bypass cloudflare protections. Required for certain indexers (ex. 1337).

**[Configarr](https://configarr.de/)** allows you to automatically synchronise quality profiles from TRaSH or create your own.

**[Bazarr](https://www.bazarr.media/)** automatically downloads and synchronises subtitles in your preferred language.

**[Seerr](https://github.com/seerr-team/seerr)** allows users to request media and automatically add it to Radarr/Sonarr.

**[Homepage](https://github.com/gethomepage/homepage)** is a dashboard for keeping track all of the services we're running.

**[Maintainarr](https://github.com/Maintainerr/Maintainerr)** is a great tool for removing media that hasn't been watched in a long while, or ones that were requested but never watched.

**[Unmanic](https://docs.unmanic.app/docs/)** is a tool to organise your library and remove unused subtitles and other things.

## Optional software
**[NPMPlus](https://github.com/ZoeyVid/NPMplus/)** is an improved fork of nginx Proxy Manager; a webui that allows you to run reverse proxies with automatic TLS certificate creation and renewal via Let's Encrypt

# Basic installation
## Docker & Docker compose
You need to install [Docker](https://docs.docker.com/engine/install/) & [Docker Compose](https://docs.docker.com/compose/install/) for your setup. Preferably use Docker Compose V2.
Both guides on the Docker website should be sufficient for you to install it.

## User & Group setup
An often overlooked part of the *arr stack is that it works best with individual users per app; that way you can give folder permissions (chown) only where they're needed. For our stack we will be starting our ID's for users at 13000. We will be be creating a new user called `Mediaserver` that we will use to create all folders and to run Docker Compose. 

Run the following code to create all `users` and a usergroup for them; `mediacenter`
```
sudo groupadd mediacenter -g 13000
sudo useradd mediauser -u 13000
sudo useradd qbittorrent -u 13001
sudo useradd sabnzbd -u 13002
sudo useradd sonarr -u 13003
sudo useradd radarr -u 13004
sudo useradd prowlarr -u 13005
sudo useradd configarr -u 13006
sudo useradd bazarr -u 13007
sudo useradd seerr -u 13008
sudo useradd homepage -u 13009
sudo useradd maintainarr -u 13010
sudo useradd unmanic -u 13011
```

Then we want to add all users to the mediacenter group and add our mediauser to the docker group, so it can access the docker repositories:
```
sudo usermod -a -G mediacenter mediauser
sudo usermod -a -G docker mediauser
sudo usermod -a -G mediacenter qbittorrent
sudo usermod -a -G mediacenter sabnzbd
sudo usermod -a -G mediacenter sonarr
sudo usermod -a -G mediacenter radarr
sudo usermod -a -G mediacenter prowlarr
sudo usermod -a -G mediacenter configarr
sudo usermod -a -G mediacenter bazarr
sudo usermod -a -G mediacenter seerr
sudo usermod -a -G mediacenter homepage
sudo usermod -a -G mediacenter maintainarr
sudo usermod -a -G mediacenter unmanic
```
## Configure mediauser
Now we want to create a password for `mediauser`, do: `sudo passwd mediauser`. This will prompt you for a new password.
Then we want to allow the `mediauser` to do sudo so do: `sudo adduser mediauser sudo`.

Finally we want to create a home folder for the `mediauser`: `sudo mkhomedir_helper mediauser`

## File & Folder setup
For the file and folder structure, we are using [TRaSH Guides' setup on file & folder structure](https://trash-guides.info/File-and-Folder-Structure/), the folder structure will look something like: 
```
config
├── qbittorrent
├── sabnznbd
├── etc.
data
├── torrents
│   ├── movies
│   └── tv
├── usenet
│   ├── incomplete
│   └── complete
│       ├── movies
│       └── tv
└── media
    ├── movies
    └── tv
```

## Creating Folders
First we need to decide where you want the Docker containers and their config files to live. I would personally recommend either `/home` or `/opt`. In this guide we will be using `/home`.
**Please note** if you're using an external NAS, you will have to edit your `/etc/fstab` first and permanently mount the volumes. Use that mountpoint for the /data/ folders. I would personally recommend keeping the config files and transcode cache on an SSD ([as per Jellyfin recommendations](https://jellyfin.org/docs/general/administration/hardware-selection#storage))

Make sure you're logged in as the user `mediauser` we've previously set up by doing: `login mediauser`. Also make sure you are using `bash` for all commands to work.

Create the folder structure by entering the following commands:
```
sudo mkdir -pv /home/mediauser/config/{jellyfin,qbittorrent,sabnzbd,sonarr,radarr,prowlarr,configarr,bazarr,seerr,homepage,maintainarr,unmanic}
sudo mkdir -pv /home/mediauser/data/{torrents,media}/{movies,tv}
sudo mkdir -pv /home/mediauser/data/usenet/incomplete
sudo mkdir -pv /home/mediauser/data/usenet/complete/{movies,tv}
```
### Other media location
If you're using an external mount point, you will have to adjust the /data/ folders to the mountpoint you've specified in your `/etc/fstab`. For example, we're using the `tank` mounted in `mnt` here, while omitting the data folder.
```
sudo mkdir -pv /mnt/tank/{torrents,media}/{movies,tv}
sudo mkdir -pv /mnt/tank/usenet/incomplete
sudo mkdir -pv /mnt/tank/usenet/complete/{movies,tv}
```

## Folder permissions
Remember to adjust the first line to your actual data location if you're using separate volumes/mountpoints
```
sudo chmod -R 775 /home/mediauser/data/
sudo chmod -R 775 /home/mediauser/config/
sudo chown -R mediauser:mediacenter /home/mediauser/data/
sudo chown -R mediauser:mediacenter /home/mediauser/config/
sudo chown -R mediauser:mediacenter /home/mediauser/config/jellyfin
sudo chown -R qbittorrent:mediacenter /home/mediauser/config/qbittorrent
sudo chown -R sabnzbd:mediacenter /home/mediauser/config/sabnzbd
sudo chown -R sonarr:mediacenter /home/mediauser/config/sonarr
sudo chown -R radarr:mediacenter /home/mediauser/config/radarr
sudo chown -R prowlarr:mediacenter /home/mediauser/config/prowlarr
sudo chown -R configarr:mediacenter /home/mediauser/config/configarr
sudo chown -R bazarr:mediacenter /home/mediauser/config/bazarr
sudo chown -R seerr:mediacenter /home/mediauser/config/seerr
sudo chown -R homepage:mediacenter /home/mediauser/config/homepage
sudo chown -R maintainarr:mediacenter /home/mediauser/config/maintainarr
sudo chown -R unmanic:mediacenter /home/mediauser/config/unmanic
```

## Docker Compose File
Now that all file and folder permissions are set up we want to set up our docker compose file. This we will put in the default mediauser home folder `/home/mediauser` for ease of use. 

# Application installation
Launch all our Docker containers by doing `docker compose up -d`. It will now start pulling all images automatically. We will move on to configure every app individually

## Qbittorrent
Access the WebUI with the ports you've set up in the docker compose file we've created earlier. It will automatically create a password for you that you can access by doing `docker logs qbittorrent`. The output will have your password in it.
<details>
  <summary>Screenshot</summary>
  ![Screenshot of a comment on a GitHub issue showing an image, added in the Markdown, of an Octocat smiling and raising a tentacle.](https://myoctocat.com/assets/images/base-octocat.svg)
</details>
![Screenshot of a comment on a GitHub issue showing an image, added in the Markdown, of an Octocat smiling and raising a tentacle.](https://myoctocat.com/assets/images/base-octocat.svg)


Change your default login details



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
