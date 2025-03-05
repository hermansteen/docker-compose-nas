# Docker Compose NAS

After searching for the perfect NAS solution, I found AdrienPoupa's [docker-compose-nas](https://github.com/AdrienPoupa/docker-compose-nas), it's a brilliant starting point but it had a bit too many bells and whistles for a newbie such as myself. So I created my own version with some of the features of his config scaled away. The result is an opinionated Docker Compose configuration capable of 
browsing indexers to retrieve media resources and downloading them through a WireGuard VPN, remote access through Tailscale and a request handler run as a Discord bot.

Requirements: Any Docker-capable recent Linux box with Docker Engine and Docker Compose V2.
I am running it in NixOS.

## Table of Contents

<!-- TOC -->
- [Docker Compose NAS](#docker-compose-nas)
  - [Table of Contents](#table-of-contents)
  - [Applications](#applications)
  - [Quick Start](#quick-start)
  - [Environment Variables](#environment-variables)
  - [NordLynx](#nordlynx)
  - [Sonarr \& Radarr](#sonarr--radarr)
    - [File Structure](#file-structure)
    - [Download Client](#download-client)
  - [Prowlarr](#prowlarr)
    - [Trackers / Indexers](#trackers--indexers)
  - [qBittorrent](#qbittorrent)
  - [Jellyfin](#jellyfin)
  - [Traefik](#traefik)
    - [Accessing from the outside with Tailscale](#accessing-from-the-outside-with-tailscale)
  - [User Permissions (IMPORTANT)](#user-permissions-important)
  - [Use Separate Paths for Torrents and Storage](#use-separate-paths-for-torrents-and-storage)
<!-- TOC -->

## Applications

| **Application**                                             | **Description**                                                                               | **Image**                                                                                | **URL**      |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------ |
| [Sonarr](https://sonarr.tv)                                 | PVR for newsgroup and bittorrent users                                                        | [linuxserver/sonarr](https://hub.docker.com/r/linuxserver/sonarr)                        | /sonarr      |
| [Radarr](https://radarr.video)                              | Movie collection manager for Usenet and BitTorrent users                                      | [linuxserver/radarr](https://hub.docker.com/r/linuxserver/radarr)                        | /radarr      |
| [Bazarr](https://www.bazarr.media/)                         | Companion application to Sonarr and Radarr that manages and downloads subtitles               | [linuxserver/bazarr](https://hub.docker.com/r/linuxserver/bazarr)                        | /bazarr      |
| [Prowlarr](https://github.com/Prowlarr/Prowlarr)            | Indexer aggregator for Sonarr and Radarr                                                      | [linuxserver/prowlarr:latest](https://hub.docker.com/r/linuxserver/prowlarr)             | /prowlarr    |
| [NordLynx VPN](https://github.com/bubuntux/nordlynx)        | Encapsulate qBittorrent traffic in [Nordlynx](https://www.nordvpn.com/)                       | [bubuntux/nordlynx](https://ghcr.io/bubuntux/nordlynx)                          |              |
| [qBittorrent](https://www.qbittorrent.org)                  | Bittorrent client with a complete web UI<br/>Uses VPN network<br/>Using Libtorrent 1.x        | [linuxserver/qbittorrent:libtorrentv1](https://hub.docker.com/r/linuxserver/qbittorrent) | /qbittorrent |
| [Unpackerr](https://github.com/ManiMatter/decluttarr)       | Automated Archive Extractions                                                                 | [manimatter/decluttarr](https://ghcr.io/manimatter/decluttarr)                           |              |
| [Decluttarr](https://unpackerr.zip)                         | Automated removal of stalled, failed, missing files in the queue                            | [golift/unpackerr](https://hub.docker.com/r/golift/unpackerr)                            |              |
| [Jellyfin](https://jellyfin.org)                            | Media server designed to organize, manage, and share digital media files to networked devices | [linuxserver/jellyfin](https://hub.docker.com/r/linuxserver/jellyfin)                    | /jellyfin    |
| [Requestrr](https://github.com/thomst08/requestrr)          | Discord chatbot requester service for TV and Movies                                           | [thomst08/requestrr](https://hub.docker.com/r/thomst08/requestrr)                        |              |
| [Traefik](https://traefik.io)                               | Reverse proxy                                                                                 | [traefik](https://hub.docker.com/_/traefik)                                              |              |
| [Watchtower](https://containrrr.dev/watchtower/)            | Automated Docker images update                                                                | [containrrr/watchtower](https://hub.docker.com/r/containrrr/watchtower)                  |              |
| [Autoheal](https://github.com/willfarrell/docker-autoheal/) | Monitor and restart unhealthy Docker containers                                               | [willfarrell/autoheal](https://hub.docker.com/r/willfarrell/autoheal)                    |              |

## Quick Start

`cp .env.example .env`, edit to your needs then `docker compose up -d`.

For the first time, run `./update-config.sh` to update the applications base URLs and set the API keys in `.env`.

If you want to show Jellyfin information in the homepage, create it in Jellyfin settings and fill `JELLYFIN_API_KEY`.

## Environment Variables

| Variable                       | Description                                                                                                                                                                                            | Default                                          |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| `COMPOSE_FILE`                 | Docker compose files to load                                                                                                                                                                           |                                                  |
| `COMPOSE_PROFILES`             | Docker compose profiles to load (`flaresolverr`, `adguardhome`, `sabnzbd`)                                                                                                                             |                                                  |
| `USER_ID`                      | ID of the user to use in Docker containers                                                                                                                                                             | `1000`                                           |
| `GROUP_ID`                     | ID of the user group to use in Docker containers                                                                                                                                                       | `1000`                                           |
| `TIMEZONE`                     | TimeZone used by the container.                                                                                                                                                                        | `Europe/Helsinki`                               |
| `CONFIG_ROOT`                  | Host location for configuration files                                                                                                                                                                  | `.`                                              |
| `DATA_ROOT`                    | Host location of the data files                                                                                                                                                                        | `/data`                                      |
| `DOWNLOAD_ROOT`                | Host download location for qBittorrent, should be a subfolder of `DATA_ROOT`                                                                                                                           | `/data/torrents`                             |
| `HOSTNAME`                     | Hostname of the NAS, could be a local IP or a domain name                                                                                                                                              | `localhost`                                      |
| `QBITTORRENT_USERNAME`         | qBittorrent username to access the web UI                                                                                                                                                              | `admin`                                          |
| `QBITTORRENT_PASSWORD`         | qBittorrent password to access the web UI                                                                                                                                                              | `adminadmin`                                     |
| `SONARR_API_KEY`               | Sonarr API key to show information in the homepage                                                                                                                                                     |                                                  |
| `RADARR_API_KEY`               | Radarr API key to show information in the homepage                                                                                                                                                     |                                                  |
| `PROWLARR_API_KEY`             | Prowlarr API key to show information in the homepage                                                                                                                                                   |                                                  |
| `BAZARR_API_KEY`               | Bazarr API key to show information in the homepage                                                                                                                                                     |                                                  |
| `JELLYFIN_API_KEY`             | Jellyfin API key to show information in the homepage                                                                                                                                                   |                                                  |
## NordLynx

I chose NordLynx since its fast and I already had a subscription, it's based on WireGuard, but you could use other providers:

- OpenVPN: [linuxserver/openvpn-as](https://hub.docker.com/r/linuxserver/openvpn-as)
- WireGuard: [linuxserver/wireguard](https://hub.docker.com/r/linuxserver/wireguard)
- Private Internet Access: [wiorca/docker-pia](https://github.com/wiorca/docker-pia)
- NordVPN + OpenVPN: [bubuntux/nordvpn](https://hub.docker.com/r/bubuntux/nordvpn/dockerfile)
- NordVPN + WireGuard (NordLynx): [bubuntux/nordlynx](https://hub.docker.com/r/bubuntux/nordlynx)

For PIA + WireGuard, fill `.env` and fill it with your PIA credentials.

The location of the server it will connect to is set by `LOC=ca`, defaulting to Montreal - Canada.

You need to fill the credentials in the `PIA_*` environment variable, 
otherwise the VPN container will exit and qBittorrent will not start.

## Sonarr & Radarr

### File Structure

Sonarr and Radarr must be configured to support hardlinks, to allow instant moves and prevent using twice the storage
(Bittorrent downloads and final file). The trick is to use a single volume shared by the Bittorrent client and the *arrs.
Subfolders are used to separate the TV shows from the movies.

The configuration is well explained by [this guide](https://trash-guides.info/Hardlinks/How-to-setup-for/Docker/).

In summary, the final structure of the shared volume will be as follows:

```
data
├── torrents = shared folder qBittorrent downloads
│  ├── movies = movies downloads tagged by Radarr
│  └── tv = movies downloads tagged by Sonarr
└── media = shared folder for Sonarr and Radarr files
   ├── movies = Radarr
   └── tv = Sonarr
```

Go to Settings > Management.
In Sonarr, set the Root folder to `/data/media/tv`.
In Radarr, set the Root folder to `/data/media/movies`.

### Download Client

Then qBittorrent can be configured at Settings > Download Clients. Because all the networking for qBittorrent takes
place in the VPN container, the hostname for qBittorrent is the hostname of the VPN container, ie `vpn`, and the port is `8080`:

## Prowlarr

The indexers are configured through Prowlarr. They synchronize automatically to Radarr and Sonarr.

Radarr and Sonarr may then be added via Settings > Apps. The Prowlarr server is `http://prowlarr:9696/prowlarr`, the Radarr server is `http://radarr:7878/radarr` Sonarr `http://sonarr:8989/sonarr`.

Their API keys can be found in Settings > Security > API Key.

### Trackers / Indexers
Private trackers are all the rage, but as the name indicates: they are not accessible to the general public. I am running only public trackers as of right now. And it's working okay, but after running publics for a while and maintaining a decent ratio you should probably try to move to a private tracker.

## qBittorrent

Running `update-config.sh` will set qBittorrent's password to `adminadmin`. If you wish to update the password manually,
since qBittorrent v4.6.2, a temporary password is generated on startup. Get it with `docker compose logs qbittorrent`:
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: <some_password>
```

Use this password to access the UI, then go to Settings > Web UI and set your own password, 
then set it in `.env`'s `QBITTORRENT_PASSWORD` variable.

The login page can be disabled on for the local network in by enabling `Bypass authentication for clients`.

```
192.168.0.0/16
127.0.0.0/8
172.17.0.0/16
```

Set the default save path to `/data/torrents` in Settings, and restrict the network interface to WireGuard (`wg0`).

To use the VueTorrent WebUI just go to `qBittorrent`, `Options`, `Web UI`, `Use Alternative WebUI`, and enter `/vuetorrent`. Special thanks to gabe565 for the easy enablement with (https://github.com/gabe565/linuxserver-mod-vuetorrent).

## Jellyfin

To enable [hardware transcoding](https://jellyfin.org/docs/general/administration/hardware-acceleration/),
depending on your system, you may need to add the following block:

```yml    
deploy:
  resources:
    reservations:
      devices:
      - driver: cdi
        device_ids:
        - nvidia.com/gpu=all
        capabilities: [gpu]
environment:
      - NVIDIA_VISIBLE_DEVICES=all
```

This is for using an Nvidia GPU with passthrough through the docker container. Requires [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to be installed on your system.

## Traefik

Why surf to an array of different IP addresses when you can simply surf to [localhost/jellyfin]()? Traefik make this simple by exposing a number of different entrypoints for the different services. For this to work, authentication must be disabled for local addresses in sonarr, radarr, etc. If a service is not accessible by going to [localhost/servicename](), you can always reach it by going to [localhost:8080]() and finding the ip address in the Traefik dashboard.

### Accessing from the outside with Tailscale

If we want to make it reachable from outside the network without opening ports or exposing it to the internet, I found
[Tailscale](https://tailscale.com) to be a great solution: create a network, run the client on both the NAS and the device you are connecting from, and they will see each other. For example I set my clients to connect to the NAS at [http://hostname/jellyfin]() thanks to the lookup baked into Tailscale.

See [here](https://tailscale.com/kb/installation) for installation instructions.

This way, when connected to the local network, the NAS is accessible directly from the private IP,
and from the outside you need to connect to Tailscale first, then the NAS will be accessible.

## User Permissions (IMPORTANT)
By default, the user and groups are set to `1000` as it is the default on Ubuntu and many other Linux distributions.
However, that is not the case in NixOS; the first user should have an ID of `1000` and a group of `100`. 
You may check yours with `id`. 
Update the `USER_ID` and `GROUP_ID` in `.env` with your IDs.
Not updating them will result in a lot of [permission issues](https://github.com/AdrienPoupa/docker-compose-nas/issues/10) and subsequent errors which will make your NAS grind to a halt and present many strange issues.

```
USER_ID=1000
GROUP_ID=100
```

## Use Separate Paths for Torrents and Storage

If you want to use separate paths for torrents download and long term storage, to use different disks for example,
set your `docker-compose.override.yml` to:

```yml
services:
  sonarr:
    volumes:
      - ./sonarr:/config
      - ${DATA_ROOT}/media/tv:/data/media/tv
      - ${DOWNLOAD_ROOT}/tv:/data/torrents/tv
  radarr:
    volumes:
      - ./radarr:/config
      - ${DATA_ROOT}/media/movies:/data/media/movies
      - ${DOWNLOAD_ROOT}/movies:/data/torrents/movies
```

Note you will lose the hard link ability, ie your files will be duplicated.

In Sonarr and Radarr, go to `Settings` > `Importing` > Untick `Use Hardlinks instead of Copy`