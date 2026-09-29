# My-Home-Server

A collection of Docker Compose stacks and host setup notes for a self-hosted home server. The
stacks cover a Traefik reverse proxy, core infrastructure (DNS filtering, a mail relay, MariaDB,
MQTT, NTP), Home Assistant and other smart-home tools, media streaming, a download/seedbox stack,
AI/computer-vision services, and a few bots and APIs. The `Install scripts/` folder holds the
commands used to prepare a fresh Linux host.

This is a personal homelab configuration, published as a reference. It is not a turnkey product:
many values are left blank in the compose files and `.env` files and must be filled in before use.

## Table of contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Repository layout](#repository-layout)
- [Stacks and services](#stacks-and-services)
  - [Core (`docker/`)](#core-docker)
  - [Infrastructure (`docker/infrastructure`)](#infrastructure-dockerinfrastructure)
  - [Traefik (`docker/infrastructure/traefik.docker-compose.yaml`)](#traefik-dockerinfrastructuretraefikdocker-composeyaml)
  - [Home Assistant (`docker/homeassistant`)](#home-assistant-dockerhomeassistant)
  - [IoT (`docker/iot`)](#iot-dockeriot)
  - [Media (`docker/media`)](#media-dockermedia)
  - [Seedbox (`docker/seedbox`)](#seedbox-dockerseedbox)
  - [AI (`docker/ai`)](#ai-dockerai)
  - [Applications (`docker/applications`)](#applications-dockerapplications)
  - [Bots (`docker/bots`)](#bots-dockerbots)
  - [API (`docker/api`)](#api-dockerapi)
- [Networks](#networks)
- [Environment variables](#environment-variables)
- [Setup and deployment order](#setup-and-deployment-order)
- [Traefik notes](#traefik-notes)
- [Automatic updates (Ouroboros)](#automatic-updates-ouroboros)
- [Install scripts](#install-scripts)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Architecture

Every folder under `docker/` is an independent Compose project with its own
`docker-compose.yaml` (and, for some, its own `.env`). The top-level `docker/docker-compose.yaml`
does **not** include the other stacks (there is no `include:` or `extends:`). The stacks are
linked only through shared, named Docker networks:

- `docker`: created by the top-level stack and joined (as `external`) by the AI, media and
  seedbox stacks.
- `internal`: an external network that Traefik uses to reach containers. Also used by
  phpMyAdmin, OpenSpeedTest and the SonarQube pair.
- `mysql` and `mail`: external networks used inside the infrastructure stack.

Several services use `network_mode: host` (Home Assistant, Broadlink Manager, Plex, AdGuard Home,
Pi.Alert) so they can see the LAN directly.

```mermaid
flowchart LR
    internet((Internet)) -->|80 / 443| traefik[Traefik]
    traefik -->|file provider| plex[Plex :32400]
    traefik -->|file provider| tvh[TVHeadend :9981]
    traefik -->|api@internal| dash[Traefik dashboard]
    cf[(Cloudflare DNS)] -. DNS-01 challenge .- traefik

    subgraph net_internal[network: internal]
        traefik
        pma[phpMyAdmin]
        ost[OpenSpeedTest]
        sq[SonarQube + Postgres]
    end

    subgraph net_docker[network: docker]
        portainer[Portainer]
        ouro[Ouroboros]
        organizr[Organizr]
        media[Tautulli / Conreq]
        seed[Seedbox: Deluge, qBittorrent, *arr, JOAL, ...]
        ai[DeepStack / DeepStack UI / Trainer]
    end

    subgraph net_mysql[network: mysql]
        mariadb[(MariaDB)]
    end
    pma --- mariadb

    subgraph host[host network]
        ha[Home Assistant]
        adguard[AdGuard Home]
        pialert[Pi.Alert]
        plex
        blm[Broadlink Manager]
    end

    ouro -. updates labelled containers .-> portainer
```

## Requirements

- A Linux host (the install scripts assume a Debian/Ubuntu-based system).
- Docker Engine and Docker Compose (v2). See [`Install scripts/install-docker`](Install%20scripts/install-docker).
- An NVIDIA GPU with the NVIDIA Container Toolkit for the services that reserve a GPU
  (`deoldify`, `plex`). The `deepstack` service uses the `:gpu` image tag, and `CodeProject.AI`
  uses a CUDA image. Their GPU reservations are commented out.
- A domain managed in Cloudflare, plus Cloudflare API credentials, for Traefik's DNS-01
  certificates and for the Cloudflare DDNS container.
- An upstream SMTP provider (host, username, password) for the mail relay.
- Host paths that the stacks bind-mount, for example `/mnt/media/...`, `/mnt/downloads/` and
  `/mnt/hass`. Adjust them to your storage layout.
- Tokens and accounts for the individual services you enable (Telegram bots, Plex claim token,
  Green API, Xiaomi cloud, MQTT broker, and so on).

## Repository layout

```text
.
├── docker/
│   ├── docker-compose.yaml            # Core stack: Portainer, Ouroboros, Organizr (creates the "docker" network)
│   ├── .env
│   ├── ai/docker-compose.yaml
│   ├── api/docker-compose.yaml
│   ├── applications/{docker-compose.yaml,.env}
│   ├── bots/docker-compose.yaml
│   ├── homeassistant/{docker-compose.yaml,.env}
│   ├── infrastructure/
│   │   ├── docker-compose.yaml        # Mail relay, Pi.Alert, MariaDB, phpMyAdmin, Chrony, AdGuard, ...
│   │   ├── traefik.docker-compose.yaml
│   │   ├── .env
│   │   └── traefik/
│   │       ├── acme/acme.json         # Let's Encrypt certificate store (mounted at /acme.json)
│   │       ├── rules/                 # Traefik file-provider config (mounted at /rules)
│   │       │   ├── middlewares.toml
│   │       │   ├── middleware-chains.toml
│   │       │   └── routes.toml
│   │       └── shared/.htpasswd       # Basic-auth users file (mounted at /shared)
│   ├── iot/docker-compose.yaml
│   ├── media/{docker-compose.yaml,.env}
│   └── seedbox/
│       ├── docker-compose.yaml
│       ├── .env
│       └── joal/                      # JOAL config.json, client profiles and a JOAL .jar
└── Install scripts/
    ├── change-locale
    ├── geo-ip.txt
    ├── go.txt
    ├── install-docker
    ├── oracle-firewall
    ├── Python/requirements.txt
    └── Repositories/repos.sh
```

Relative volume paths such as `./plex` or `./sonarr` are created next to each stack's compose
file the first time the stack starts.

## Stacks and services

Legend: "host" means `network_mode: host`, so the service listens directly on the host's ports.
"default" means the Compose project's own default network.

### Core (`docker/`)

File: `docker/docker-compose.yaml`. Uses `docker/.env`.

| Service | Image | Purpose | Ports (host:container) | Key volumes | Network |
|---|---|---|---|---|---|
| `portainer` | `portainer/portainer-ee:latest` | Docker management UI (Business Edition) | `9000:9000` | `./portainer/data`, Docker socket | `docker` |
| `auto-updater` (container `ouroboros`) | `pyouroboros/ouroboros:latest` | Automatically updates containers labelled `com.ouroboros.enable=true` | none | Docker socket | `docker` |
| `organizr` | `organizr/organizr` | Homelab dashboard / tab organizer | `8089:80` | `./organizr`, `./shared` | `docker` |

This file defines the `docker` network (not external), so it creates it.

### Infrastructure (`docker/infrastructure`)

File: `docker/infrastructure/docker-compose.yaml`. Uses `docker/infrastructure/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `mail-relay` | `mwader/postfix-relay:latest` | Postfix SMTP relay that forwards mail to an upstream SMTP provider with SASL auth | `25:25` | `./mail-relay/*` (queue, spool, DKIM keys, SASL map), host CA bundle | `mail` |
| `pialert` | `jokobsk/pi.alert:latest` | LAN device presence / intruder detection | host | `./pialert/{db,config,logs}` | host |
| `mysql` | `mariadb:latest` | MariaDB database server | `3306:3306` | `./mysql` | `mysql` |
| `phpmyadmin` | `phpmyadmin/phpmyadmin:latest` | Web UI for MariaDB | `8081:80` | none | `mysql`, `internal` |
| `chrony` (container `Chrony`) | `publicarray/chrony:latest` | NTP server (`SYS_TIME` capability) | `123:123/udp` | `./chrony/chrony.conf` | `bridge` |
| `adguard-home` | `adguard/adguardhome` | DNS server with ad and tracker blocking | host | `./adguard/{work,conf}` | host |
| `openspeedtest` | `openspeedtest/latest` | Self-hosted network speed test | `3004:3000`, `3005:3001` | none | `internal` |
| `mosquitto` | `eclipse-mosquitto:latest` | MQTT broker | `1883:1883`, `9001:9001` (WebSockets) | `./mosquitto`, `./mosquitto/mosquitto.conf` | default |
| `cloudflare-ddns` | `oznu/cloudflare-ddns:latest` | Keeps a Cloudflare DNS record pointed at the host's public IP (`SUBDOMAIN=*`, `PROXIED=false`) | none | none | default |

The mail relay writes the upstream host, username and password to a SASL password map at start-up
(see its `command:`), then runs Postfix. It accepts mail from loopback, `172.0.0.0/8` and
`192.168.0.0/16`. Set `POSTFIX_myhostname` in the compose file to your own mail host name
(for example `mail.example.com`).

`mosquitto` and `chrony` bind-mount single files (`./mosquitto/mosquitto.conf` and
`./chrony/chrony.conf`) that are not in the repository. If a file is missing, Docker creates a
directory at that path instead, and the container fails to start. Create both files before the
first start.

`cloudflare-ddns` reads `API_KEY` and `ZONE` from inline, empty values in the compose file. Fill
them in there (or change them to `${...}` references to your `.env`).

### Traefik (`docker/infrastructure/traefik.docker-compose.yaml`)

A separate compose file in the same folder, so it also reads `docker/infrastructure/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `traefik` | `traefik:latest` | Reverse proxy and TLS termination (Let's Encrypt via Cloudflare DNS-01) | `80`, `443`, `8080` (dashboard/API), `8899` (Prometheus metrics) | `./traefik/rules:/rules`, `./traefik/acme/acme.json:/acme.json`, `./traefik/logs:/var/logs`, `./traefik/shared:/shared`, Docker socket (read-only) | `internal` |

See [Traefik notes](#traefik-notes) for entrypoints, resolver, middlewares and routes.

### Home Assistant (`docker/homeassistant`)

File: `docker/homeassistant/docker-compose.yaml`. Uses `docker/homeassistant/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `homeassistant` | `homeassistant/home-assistant` | Home Assistant (privileged) | host | `/mnt/hass:/config`, Docker socket | host |
| `broadlinkmanager` | `techblog/broadlinkmanager` | Web UI for learning and sending Broadlink IR/RF codes | host | none | host |
| `tasmota-admin` | `ghcr.io/tasmoadmin/tasmoadmin:latest` | Management UI for Tasmota devices | `8111:80` | `./tasmota-admin/to/data` | default |
| `xiaomi_token_extractor` | `techblog/xiaomi_token_extractor:latest` | Web UI that retrieves Xiaomi device tokens from the Xiaomi cloud | `8888:8080` | none | default |
| `zanzito-thingsboard-exporter` | `techblog/zanzito-thingsboard-exporter` | Forwards location and battery data from the Zanzito app (via MQTT) to ThingsBoard | none | `./zanzito-thingsboard-exporter/config` | default |
| `adb_api` (container `adb-api`) | `techblog/adb-api` | REST API and web remote for controlling Android TV streamers over ADB (privileged) | `9876:80` | `./adb-api/config` | default |
| `tasmota-thingsboard-bridge` | `techblog/tasmota-thingsboard-bridge` | Reports Tasmota device data to ThingsBoard every `REPORT_INTERVAL` seconds | none | `./tasmota-tb/config` | default |

The file declares the external `internal` network, but no service joins it.

### IoT (`docker/iot`)

File: `docker/iot/docker-compose.yaml`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `esphome` | `ghcr.io/imagegenius/esphome:latest` | ESPHome dashboard for building and flashing ESP firmware | `6052:6052` | `./esphome/config` | default |
| `espcam-secserver` | `techblog/espcam-secserver` | Backend for an ESP32-CAM door camera: on a door-open event it fetches snapshots and sends them to WhatsApp via Green API | `8200:80` | none | default |

### Media (`docker/media`)

File: `docker/media/docker-compose.yaml`. Uses `docker/media/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `plex` | `linuxserver/plex:latest` | Plex Media Server with NVIDIA GPU transcoding | host | `./plex`, `./plex/transcode`, `/mnt/media/Kids`, `/mnt/media/Parents` | host |
| `tautulli` | `linuxserver/tautulli` | Plex monitoring and statistics | `8181:8181` | `./tautulli/config`, `./tautulli/logs` | `docker` |
| `conreq` | `ghcr.io/roxedus/conreq:latest` | Content request portal | `8884:8000` | `./conreq/config` | `docker` |

### Seedbox (`docker/seedbox`)

File: `docker/seedbox/docker-compose.yaml`. Uses `docker/seedbox/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `deluge` | `linuxserver/deluge` | BitTorrent client | `8112:8112` (web UI), `58846:58846`, `58946:58946` | `./deluge`, `/mnt/downloads`, `/mnt/media/...`, `./joal/torrents` | `docker` |
| `qbittorrent` | `lscr.io/linuxserver/qbittorrent:latest` | BitTorrent client | `6882:8080` (web UI), `6881:6881` tcp+udp | `./qbittorrent/config`, `./downloads`, `/mnt/media`, `./joal/torrents` | `docker` |
| `telegram-download-daemon` (container `Telegram_Downloader`) | `alfem/telegram-download-daemon` | Downloads files posted to a Telegram channel | none | `./downloads`, `./tbdownloader/sessions`, `/tmp` | `docker` |
| `joal` | `anthonyraymond/joal` | See note below | `15538:15538` | `./joal:/data` | `docker` |
| `sonarr` | `linuxserver/sonarr` | TV series manager | `8989:8989` | `./sonarr`, `/mnt/downloads`, `/mnt/media/...` | `docker` |
| `jackett` | `linuxserver/jackett` | Indexer proxy for the *arr apps | `9117:9117` | `./jackett`, `/mnt/downloads`, `/mnt/media/...` | `docker` |
| `radarr` | `linuxserver/radarr` | Movie manager | `7878:7878` | `./radarr`, `/mnt/downloads`, `/mnt/media/...` | `docker` |
| `lidarr` | `volikon/lidarr` | Music manager | `8686:8686` | `./lidarr/config`, `/mnt/downloads`, `/mnt/media/...` | `docker` |
| `bazarr` | `linuxserver/bazarr` | Subtitle manager | `6767:6767` | `./bazarr/config`, `/mnt/downloads`, `/mnt/media/...` | `docker` |

**About JOAL.** JOAL ("Jack of all trades") is a tool that emulates a BitTorrent client and
reports simulated upload and download statistics to torrent trackers. The `joal/` folder holds
its `config.json`, a set of client profiles (`joal/clients/*.client`) and a JOAL `.jar`. The
compose file runs its web UI on port `15538`, with `joal.ui.path.prefix` and
`joal.ui.secret-token` left empty for you to set.

> [!WARNING]
> Reporting simulated statistics to trackers may violate the rules of private trackers and can
> get your account banned. Check the rules of any tracker you use.

### AI (`docker/ai`)

File: `docker/ai/docker-compose.yaml`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `deepstack` | `deepquestai/deepstack:gpu` | DeepStack AI server with face, object-detection and scene APIs enabled | `5002:5000` | `./deepstack:/datastore` | `docker` |
| `deepstack_trainer` | `techblog/deepstack-trainer` | Web UI for training DeepStack face recognition (privileged) | `5003:8080` | none | `docker` |
| `deepstack_ui` | `robmarkcole/deepstack-ui:latest` | Streamlit UI for testing DeepStack | `8501:8501` | none | `docker` |
| `deoldify` | `techblog/deoldify` | Telegram bot that colorizes old photos with DeOldify (NVIDIA GPU) | none | `./deoldify/models` | default |
| `CodeProject.AI` | `codeproject/ai-server:cuda12_2` | CodeProject.AI server (CUDA image) | `32168:32168` | `./codeproject/ai/etc`, `./codeproject/ai/modules` | default |

### Applications (`docker/applications`)

File: `docker/applications/docker-compose.yaml`. Uses `docker/applications/.env`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `sonarqube` | `sonarqube` | Code quality analysis | `8091:9000` | `./sonarqube/{conf,data,extensions,temp}` | `internal` |
| `sonarqube-db` | `postgres` | PostgreSQL database for SonarQube | none | `./sonarqube/postgresql`, `./sonarqube/postgresql_data` | `internal` |
| `mealie` | `ghcr.io/mealie-recipes/mealie:latest` | Recipe manager (1000 MB memory limit) | `9925:9000` | `./mealie/data` | default |

The SonarQube database user and password are set inline in the compose file. Change them before
use (see [Security](#security)). Set Mealie's `BASE_URL` and `TZ` to your own values.

### Bots (`docker/bots`)

File: `docker/bots/docker-compose.yaml`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `botami` | `techblog/botami4:latest` | Telegram bot for controlling a Tami4 Edge water bar | none | `./botami/tokens` | default |
| `redalert` | `techblog/redalert` | Reads Israeli Home Front Command (Oref) "Red Alert" alerts and publishes them over MQTT | none | none | default |

### API (`docker/api`)

File: `docker/api/docker-compose.yaml`.

| Service | Image | Purpose | Ports | Key volumes | Network |
|---|---|---|---|---|---|
| `wapi` | `chrishubert/whatsapp-web-api:latest` | REST API wrapper around WhatsApp Web | `3033:3000` | `./wapi/sessions` | default |
| `wapi_dev` | `chrishubert/whatsapp-web-api:latest` | Second instance for development | `3034:3000` | `./wapi_dev/sessions` | default |

Both instances enable the Swagger endpoint (`/api-docs`) and the local callback example, and
disable the `message_ack` callback. The `API_KEY` setting is commented out, so the API runs
without authentication until you set one.

## Networks

| Network | Created by | Used by |
|---|---|---|
| `docker` | `docker/docker-compose.yaml` (core stack) | core, ai, media, seedbox |
| `internal` | Must be created manually (`external: true` everywhere) | traefik, phpmyadmin, openspeedtest, sonarqube, sonarqube-db |
| `mysql` | Must be created manually | mysql, phpmyadmin |
| `mail` | Must be created manually | mail-relay |

Traefik is configured with `--providers.docker.network=internal`, so any container you want to
route through Traefik must also join `internal`.

## Environment variables

Docker Compose reads the `.env` file in the same folder as the compose file. Values must not be
quoted (the `.env` files note that quotes become part of the value).

### Variables referenced by the compose files

| Variable | Stack(s) | Required | Description |
|---|---|---|---|
| `TZ` | core, media, seedbox | yes | Time zone, e.g. `<Region/City>` |
| `PUID` / `PGID` | core, homeassistant, media (tautulli), seedbox | yes | User/group ID the containers run as |
| `UPSTREAM_SMTP_HOST` | infrastructure | yes (compose fails if unset) | Upstream SMTP host and port, e.g. `smtp.example.com:587` |
| `UPSTREAM_SMTP_USERNAME` | infrastructure | yes (compose fails if unset) | Upstream SMTP user |
| `UPSTREAM_SMTP_PASSWORD` | infrastructure | yes (compose fails if unset) | Upstream SMTP password |
| `MYSQL_ROOT_PASSWORD` | infrastructure | yes | MariaDB root password. Also used as `MYSQL_PASSWORD` and by phpMyAdmin |
| `PMA_HOST` | infrastructure | yes (compose fails if unset) | Database host for phpMyAdmin (`mysql`) |
| `DOMAINNAME` | traefik | yes | Base domain, e.g. `example.com` (dashboard at `traefik.example.com`, wildcard cert `*.example.com`) |
| `CLOUDFLARE_EMAIL` | traefik | yes | ACME account e-mail for Let's Encrypt |
| `PLEX_HOSTNAME` | media | optional | Plex server name |
| `PLEX_CLAIM` | media | first run | Plex claim token |
| `PLEX_UID` / `PLEX_GID` | media | yes | User/group for Plex |
| `PLEX_ADVERTISE_IP` | media | optional | URL(s) Plex advertises to clients |

### Variables present in `.env` files but not referenced by any compose file

These are kept in the `.env` templates but currently have no effect: `UID`, `DUCKDNS_SUBDOMAIN`,
`DUCKDNS_TOKEN`, `REGISTERED_DOMAIN`, `TRAEFIK_DASHBOARD_USERS_PASSWORD_FILE`,
`TRAEFIK_DASHBOARD_DOMAIN`, `NOTIFIERS`, `DOMAINNAME` (in `docker/.env`), `FROM_ADDRESS`,
`MYSQL_PASSWORD`, `CFDDNS_EMAIL`, `CFDDNS_API_KEY`, `CFDDNS_ZONE`, `CFDDNS_SUBDOMAIN`,
`CLOUDFLARE_API_KEY`, `SONARQUBE_JDBC_USERNAME`, `SONARQUBE_JDBC_PASS`, and the Red Alert values
`MQTT_HOST`, `MQTT_USER`, `MQTT_PASS`, `DEBUG_MODE` in `docker/homeassistant/.env` (the
`redalert` service lives in the bots stack and sets these inline). `TZ` is also unused in the
infrastructure, applications and homeassistant `.env` files: those stacks hard-code the time zone
in the compose files or leave it commented out.

### Settings set inline in the compose files

Many services take their settings as inline, empty `environment:` entries. Fill these in the
compose file itself (or change them to `${VAR}` references):

| Stack | Service | Settings |
|---|---|---|
| infrastructure | `cloudflare-ddns` | `API_KEY`, `ZONE`, `SUBDOMAIN`, `PROXIED` |
| infrastructure | `mail-relay` | `POSTFIX_myhostname` |
| traefik | `traefik` | `CF_API_EMAIL`, `CF_API_KEY` (Cloudflare credentials for the DNS challenge) |
| core | `auto-updater` | `NOTIFIERS`, `INTERVAL`, `CRON`, `LOG_LEVEL` |
| ai | `deepstack` | `VISION-FACE`, `VISION-DETECTION`, `VISION-SCENE`, optional `API-KEY` |
| ai | `deepstack_trainer` | `DEEPSTACK_HOST_ADDRESS`, `MIN_CONFIDANCE` |
| ai | `deepstack_ui` | `DEEPSTACK_IP`, `DEEPSTACK_PORT`, `DEEPSTACK_API_KEY`, `DEEPSTACK_TIMEOUT` |
| ai | `deoldify` | `BOT_TOKEN`, `RENDER_FACTOR` |
| bots | `botami` | `BOT_TOKEN`, `ALLOWED_IDS` |
| bots | `redalert` | `MQTT_HOST`, `MQTT_USER`, `MQTT_PASS`, `DEBUG_MODE`, `REGION`, `NOTIFIERS` |
| homeassistant | `xiaomi_token_extractor` | `XIA_USER`, `XIA_PASS`, `XIA_SRV` (optional region) |
| homeassistant | `zanzito-thingsboard-exporter` | `TB_SERVER_ADDRESS`, `MQTT_BROKER_ADDRESS`, `MQTT_BROKER_PORT`, `MQTT_BROKER_USER`, `MQTT_BROKER_PASSWORD` |
| homeassistant | `tasmota-thingsboard-bridge` | `TB_SERVER_ADDRESS`, `REPORT_INTERVAL` |
| iot | `espcam-secserver` | `GREEN_API_INSTANCE_ID`, `GREEN_API_TOKEN`, `TARGET`, `MESSAGE` |
| seedbox | `telegram-download-daemon` | `TELEGRAM_DAEMON_API_ID`, `TELEGRAM_DAEMON_API_HASH`, `TELEGRAM_DAEMON_CHANNEL` |
| seedbox | `joal` | `joal.ui.path.prefix`, `joal.ui.secret-token` |
| api | `wapi`, `wapi_dev` | `API_KEY` (commented out in both), `BASE_WEBHOOK_URL` (commented out in `wapi`, set only in `wapi_dev`) |
| applications | `mealie` | `BASE_URL`, `ALLOW_SIGNUP`, `TZ` |

Example `.env` for the infrastructure stack (placeholders only):

```dotenv
UPSTREAM_SMTP_HOST=<smtp.example.com:587>
UPSTREAM_SMTP_USERNAME=<smtp-user>
UPSTREAM_SMTP_PASSWORD=<smtp-password>
MYSQL_ROOT_PASSWORD=<strong-password>
PMA_HOST=mysql
DOMAINNAME=example.com
CLOUDFLARE_EMAIL=<acme-email>
```

## Setup and deployment order

1. Prepare the host (see [Install scripts](#install-scripts)): install Docker and Docker Compose,
   set the time zone, and open the firewall if needed.
2. Clone the repository and review every compose file and `.env` file. Fill in the variables
   and the inline settings listed above, and adjust host paths such as `/mnt/media` and
   `/mnt/hass`.
3. Create the external networks:

   ```bash
   docker network create internal
   docker network create mysql
   docker network create mail
   ```

4. Run the following commands from the repository root. Start the core stack first, because it
   creates the `docker` network that the AI, media and seedbox stacks join:

   ```bash
   docker compose -f docker/docker-compose.yaml up -d
   ```

5. Before starting Traefik, replace the committed `.htpasswd` in
   `docker/infrastructure/traefik/shared/` with your own users file (for example generated with
   `htpasswd -nB <user>`), and make sure `docker/infrastructure/traefik/acme/acme.json` exists and
   is readable only by its owner (`chmod 600`). Then start the infrastructure stack and Traefik:

   ```bash
   docker compose -p infrastructure -f docker/infrastructure/docker-compose.yaml up -d
   docker compose -p traefik -f docker/infrastructure/traefik.docker-compose.yaml up -d
   ```

   Both files sit in the same folder, so without `-p` Compose gives them the same project name
   (`infrastructure`) and warns that the other file's containers are orphans. Use a separate
   project name as shown, and don't pass `--remove-orphans`, or one file's `up` removes the
   other's containers.

6. Start the remaining stacks in any order, for example:

   ```bash
   (cd docker/homeassistant && docker compose up -d)
   (cd docker/media && docker compose up -d)
   ```

## Traefik notes

- **Entrypoints:** `http` (`:80`), `https` (`:443`), `traefik` (`:8080`, dashboard/API) and
  `metrics` (`:8899`, Prometheus). MQTT and WebSocket entrypoints are present but commented out.
- **HTTP to HTTPS:** a catch-all router on `http` redirects every host to HTTPS with the
  `redirect-to-https` middleware.
- **Certificates:** resolver `dns-cloudflare` uses Let's Encrypt with the Cloudflare DNS-01
  challenge, stores certificates in `/acme.json`, and requests `DOMAINNAME` plus the wildcard
  `*.DOMAINNAME`. A commented line switches to the Let's Encrypt staging server for testing.
- **Providers:**
  - Docker provider with `exposedByDefault=false`, default rule
    `Host(<compose service name>.DOMAINNAME)`, and network `internal`.
  - File provider loading `/rules` (`docker/infrastructure/traefik/rules`) with `watch=true`.
- **Middlewares** (`rules/middlewares.toml`):
  - `middlewares-basic-auth`: basic auth using the users file `/shared/.htpasswd`.
  - `middlewares-rate-limit`: average 100, burst 50.
  - `middlewares-secure-headers`: HSTS (2 years, subdomains, preload), SSL redirect,
    `nosniff`, XSS filter, `same-origin` referrer policy, a restrictive feature policy,
    `X-Robots-Tag: none,...`, an empty `Server` header, and a custom frame-options value that
    contains a `[my_domain]` placeholder to replace.
- **Chains** (`rules/middleware-chains.toml`):
  - `chain-no-auth`: rate limit + secure headers.
  - `chain-basic-auth`: rate limit + secure headers + basic auth.
- **What is exposed:**
  - The Traefik dashboard at `traefik.example.com` on `https`, protected by
    `chain-basic-auth@file`.
  - `plex.example.com` and `tvh.example.com` via the file provider (`rules/routes.toml`), which
    forward to `http://<server_ip>:32400` (Plex) and `http://<server_ip>:9981` (TVHeadend). Replace
    the `[my_domain]` and `[server_ip]` placeholders in that file. TVHeadend is not defined in
    any stack in this repository.
  - Prometheus metrics at `/metrics` on the `metrics` entrypoint (port `8899`). The `prometheus`
    router references a middleware `my-basic-auth` that is not defined anywhere, so that router is
    broken. Traefik still serves `/metrics` on port `8899` without authentication.
  - No other container sets `traefik.enable=true`. Every other service is reached directly on
    its published port.
- **Ping:** `--ping.entryPoint=web` points to an entrypoint named `web`, which is not defined
  (the entrypoints are `http`, `https`, `traefik` and `metrics`).
- **Logging:** log level `DEBUG`; JSON access log written to `./traefik/logs/traefik.json`.

## Automatic updates (Ouroboros)

The `auto-updater` service (Ouroboros) runs with `LABEL_ENABLE=true` and `LABELS_ONLY=true`, so it
only updates containers labelled `com.ouroboros.enable=true`. It is meant to check every 30
minutes (`CRON="*/30 * * * *"`, otherwise `INTERVAL=300`), remove old images (`CLEANUP=true`) and update itself. See the Ouroboros item in
[Troubleshooting](#troubleshooting) for caveats about how these values are written. Services labelled
`com.ouroboros.enable=false` (for example `deepstack`, `chrony`, `sonarqube`) or with no label are
not updated.

## Install scripts

The files in `Install scripts/` are command notes for preparing a host. None of them has a shebang
or the executable bit, so read them and run the commands you need by hand.

| File | What it does |
|---|---|
| `install-docker` | Installs Docker Engine with the official `get.docker.com` script, then downloads the standalone `docker-compose` v2.24.1 binary to `/usr/local/bin`. It has separate sections for amd64 and arm64: run only the one that matches your CPU. |
| `change-locale` | Shows the current time settings (`timedatectl`) and relinks `/etc/localtime` to a hard-coded time zone. Despite the name, it changes the time zone, not the locale. Edit the zone before running. |
| `go.txt` | Downloads Go 1.21.6 (linux-amd64) into `/usr/local/go` and adds `/usr/local/go/bin` to `PATH` for the current shell. |
| `geo-ip.txt` | Downloads freegeoip 3.4.1 to `/opt/freegeoip` and a GeoLite2-City database, followed by a sample systemd unit that runs freegeoip on `:8080` (with an internal server on `:8888`). The unit text is meant to be copied into a `.service` file; it is not a runnable script. |
| `oracle-firewall` | Installs `firewalld` and opens TCP ports 80, 443 and 8080 in the public zone (for Ubuntu hosts on Oracle Cloud). |
| `Python/requirements.txt` | A mostly unpinned list of Python packages used across the author's projects (FastAPI, Flask, MQTT, Telegram, OpenCV, scikit-learn and more). Install with `pip install -r "Install scripts/Python/requirements.txt"`. |
| `Repositories/repos.sh` | Clones the author's related GitHub repositories (and one third-party repository) over SSH. Requires an SSH key registered with GitHub. |

## Security

- **Never commit secrets.** Keep `.env` files, basic-auth users files (`.htpasswd`) and ACME
  certificate stores (`acme.json`) out of git with a `.gitignore`. Commit only templates with
  empty or placeholder values (for example `.env.example`).
- **Prefer env files or Docker secrets** over hard-coded credentials in compose files. Replace
  inline passwords, tokens and API keys with `${VAR}` references or Compose `secrets:`.
- **Rotate anything that was ever committed.** A value that has been pushed to a public
  repository stays in its history even after the file is changed. Treat it as compromised,
  rotate it, and if needed rewrite history.
- **Change default credentials** (for example database users and passwords) before first start.
- **Limit exposure.** Many services publish ports directly on the host and have no
  authentication in front of them. Keep them on the LAN, or put them behind Traefik with the
  `chain-basic-auth` chain or a VPN. In particular:
  - Traefik runs with `--api.insecure=true`, which serves the dashboard without authentication
    on port `8080`. Don't expose that port to the internet (note that `oracle-firewall` opens it).
  - Prometheus metrics are served without authentication on port `8899`.
  - The WhatsApp Web API instances run without an API key and with Swagger enabled.
  - Portainer, phpMyAdmin, MariaDB (`3306`), Mosquitto (`1883`/`9001`) and the mail relay (`25`)
    are published on all host interfaces.
- Several containers mount the Docker socket (`portainer`, `ouroboros`, `homeassistant`,
  `traefik`) or run `privileged` (`homeassistant`, `deepstack_trainer`, `adb_api`). Both give
  root-level access to the host.

## Troubleshooting

- **`network docker declared as external, but could not be found`**: start the core stack
  (`docker/docker-compose.yaml`) first, since it creates the `docker` network.
- **`network internal/mysql/mail ... could not be found`**: create them with
  `docker network create <name>` (see [Setup](#setup-and-deployment-order)).
- **Compose fails with "Please copy template.env to .env and provide provide a value for ..."**: the infrastructure stack
  requires `UPSTREAM_SMTP_HOST`, `UPSTREAM_SMTP_USERNAME`, `UPSTREAM_SMTP_PASSWORD` and
  `PMA_HOST`. There is no `template.env` in the repository; set them in
  `docker/infrastructure/.env`.
- **Traefik can't reach a container**: the container must join the `internal` network and set
  `traefik.enable=true`, because `exposedByDefault=false`.
- **Traefik doesn't issue certificates**: `CF_API_EMAIL` and `CF_API_KEY` are empty in
  `traefik.docker-compose.yaml`. Traefik also refuses an `acme.json` whose permissions are too
  open; set it to `600`.
- **Port conflicts**: Traefik publishes `8080`, and the freegeoip unit from `geo-ip.txt` also
  listens on `:8080`. That unit also binds `:8888`, which clashes with `xiaomi_token_extractor`
  (`8888:8080`). AdGuard Home, Pi.Alert, Plex, Home Assistant and Broadlink Manager use host
  networking and bind their ports directly on the host. <!-- TODO: verify which host ports AdGuard Home and Pi.Alert bind, and whether they clash with Traefik on 80/443 -->
- **Mosquitto or Chrony fails because a config path is a directory**: `./mosquitto/mosquitto.conf`
  and `./chrony/chrony.conf` are not in the repository, so Docker created directories in their
  place. Stop the container, remove the directories, create the files, and start it again.
- **Ouroboros settings don't behave as expected**: the service uses list-form `environment:`, so the
  double quotes around `CRON` and `DOCKER_SOCKETS` become part of the values. Also,
  `TZ=TZ=${TZ}` sets `TZ` to a literal string that starts with `TZ=`. Remove the quotes and fix
  the `TZ` line in `docker/docker-compose.yaml` if the schedule or time zone is wrong.
- **`version` is obsolete warning**: Docker Compose v2 ignores the top-level `version:` key in
  these files. The warning is harmless.
- **GPU services fail to start**: `deoldify` and `plex` reserve an NVIDIA GPU. Install the NVIDIA
  Container Toolkit, or remove the `deploy.resources.reservations` block.

## Contributing

This repository documents a personal setup. Issues and pull requests with fixes or suggestions
are welcome.

## License

This repository does not include a license file, so no license is granted by default.
<!-- TODO: verify - add a LICENSE file if the author wants to allow reuse -->
