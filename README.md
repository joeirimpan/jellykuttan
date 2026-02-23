# Jellyfin Media Server Stack — Setup Guide

## Overview

This sets up a complete **automated media server** using Docker Compose.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                   YOU (Browser)                      │
│                                                      │
│   Jellyfin :8096        Jellyseerr :5055             │
│   (Watch media)         (Request movies/shows)       │
└──────┬──────────────────────────┬────────────────────┘
       │                          │
       │                          ▼
       │               ┌─────────────────────┐
       │               │  Radarr :7878       │ ← Movies
       │               │  Sonarr :8989       │ ← TV Shows
       │               └────────┬────────────┘
       │                        │
       │                        ▼
       │               ┌─────────────────────┐
       │               │  Prowlarr :9696     │ ← Torrent indexer manager
       │               └────────┬────────────┘
       │                        │
       │                        ▼
       │               ┌─────────────────────┐
       │               │ qBittorrent :8080   │ ← Torrent downloads
       │               └────────┬────────────┘
       │                        │
       ▼                        ▼
┌──────────────────────────────────────┐
│         $MEDIA_DIR/                  │
│   ├── movies/    (Radarr manages)   │
│   ├── tv/        (Sonarr manages)   │
│   └── downloads/ (qBittorrent)      │
└──────────────────────────────────────┘
```

### What Each Service Does

- **Radarr** — Movie manager. You tell it "I want Inception", it searches torrents, picks the best quality, downloads it, and organizes it into `/movies/Inception (2010)/`.
- **Sonarr** — TV show manager. Same as Radarr but for TV. Can auto-download new episodes as they air.
- **Prowlarr** — Indexer manager. Connects to torrent indexers once, and syncs them to both Radarr & Sonarr automatically.
- **qBittorrent** — Torrent download client. Does the actual downloading.
- **Jellyfin** — Media server. Streams your content with a Netflix-like UI.
- **Jellyseerr** — Request UI. Discover & request movies/shows with a clean interface.

### How It Works (The Flow)

```
You request "Inception" in Jellyseerr
        ↓
Jellyseerr tells Radarr "get Inception"
        ↓
Radarr asks Prowlarr "search all indexers for Inception"
        ↓
Prowlarr searches configured indexers → returns results
        ↓
Radarr picks the best torrent → sends to qBittorrent
        ↓
qBittorrent downloads it
        ↓
Radarr moves & renames it → /movies/Inception (2010)/
        ↓
Jellyfin detects it → ready to watch!
```

1. **You** open **Jellyseerr** (`:5055`) and request a movie or TV show
2. Jellyseerr sends the request to **Radarr** (movies) or **Sonarr** (TV shows)
3. Radarr/Sonarr search for torrents via **Prowlarr** (which manages torrent indexer sites)
4. Radarr/Sonarr send the best torrent to **qBittorrent** for downloading
5. Once downloaded, Radarr/Sonarr **move & rename** the file into the media library
6. **Jellyfin** detects the new file and makes it available to watch

### Services

| Service        | Port  | Purpose                                      |
|----------------|-------|----------------------------------------------|
| **Jellyfin**   | 8096  | Media server — watch your content            |
| **Jellyseerr** | 5055  | Request UI — discover & request movies/shows |
| **Radarr**     | 7878  | Movie management & automation                |
| **Sonarr**     | 8989  | TV show management & automation              |
| **Prowlarr**   | 9696  | Torrent indexer manager                      |
| **qBittorrent**| 8080  | Torrent download client                      |

---

## Step 0: Set Up Paths

Define where your media and project will live. Create a `.env` file in the project directory:

```bash
# Clone/create the project directory
PROJECT_DIR="$HOME/jellyfin"     # or wherever you cloned this
MEDIA_DIR="$HOME/media"          # where media files will be stored

# Create .env for docker-compose
cat > "$PROJECT_DIR/.env" <<EOF
MEDIA_DIR=$MEDIA_DIR
EOF
```

## Step 1: Create Directory Structure

```bash
# Media directories
sudo mkdir -p "$MEDIA_DIR"/{movies,tv,downloads/complete,downloads/incomplete}
sudo chown -R $(id -u):$(id -g) "$MEDIA_DIR"

# Docker config directories
mkdir -p config/{jellyfin,jellyseerr,radarr,sonarr,prowlarr,qbittorrent}
```

## Step 2: Create Docker Compose File

The `docker-compose.yml` file is in this same directory. Launch with:

```bash
cd "$PROJECT_DIR"
docker compose up -d
```

## Step 3: Configure qBittorrent

1. Open **http://localhost:8080**
2. Check container logs for the temporary password:
   ```bash
   docker compose logs qbittorrent | grep -i password
   ```
3. Login with `admin` / (password from logs)
4. Go to **Tools → Options → Downloads**:
   - Default Save Path: `/downloads/complete`
   - Keep incomplete in: `/downloads/incomplete` (enable this)
5. Go to **Tools → Options → Web UI**:
   - Change the default password to something you'll remember
6. Go to **Tools → Options → BitTorrent**:
   - Enable "When seeding goal is reached: Pause torrent" (optional)

## Step 4: Configure Prowlarr (Indexer Manager)

1. Open **http://localhost:9696**
2. Set up authentication (username/password) when prompted
3. Add indexers:
   - Go to **Indexers → Add Indexer**
   - Search for and add your preferred public or private indexers
   - Most public indexers need no configuration
4. Add Apps (connect to Radarr & Sonarr):
   - Go to **Settings → Apps → Add Application**
   - Add **Radarr**:
     - Prowlarr Server: `http://prowlarr:9696`
     - Radarr Server: `http://radarr:7878`
     - API Key: (get from Radarr → Settings → General → API Key)
   - Add **Sonarr**:
     - Prowlarr Server: `http://prowlarr:9696`
     - Sonarr Server: `http://sonarr:8989`
     - API Key: (get from Sonarr → Settings → General → API Key)
   - Click **Sync App Indexers** — this pushes all indexers to Radarr/Sonarr automatically

## Step 5: Configure Radarr (Movies)

1. Open **http://localhost:7878**
2. Set up authentication when prompted
3. **Add Download Client**:
   - Settings → Download Clients → Add → **qBittorrent**
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: `admin`
   - Password: (your qBittorrent password)
   - Category: `radarr`
   - Test & Save
4. **Add Root Folder**:
   - Settings → Media Management → Root Folders → Add Root Folder
   - Path: `/movies`
5. **Quality Profile** (optional):
   - Settings → Quality → pick your preferred quality (e.g., "HD-1080p")
   - Recommend keeping "Any" or "HD-1080p" to start
6. Note the **API Key** from Settings → General (needed for Prowlarr & Jellyseerr)

## Step 6: Configure Sonarr (TV Shows)

1. Open **http://localhost:8989**
2. Set up authentication when prompted
3. **Add Download Client** (same as Radarr):
   - Settings → Download Clients → Add → **qBittorrent**
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: `admin`
   - Password: (your qBittorrent password)
   - Category: `sonarr`
   - Test & Save
4. **Add Root Folder**:
   - Settings → Media Management → Root Folders → Add Root Folder
   - Path: `/tv`
5. Note the **API Key** from Settings → General

## Step 7: Configure Jellyfin

1. Open **http://localhost:8096**
2. Follow the setup wizard:
   - Create admin user
   - Add Media Libraries:
     - **Movies** → Folder: `/data/movies`
     - **TV Shows** → Folder: `/data/tv`
   - Enable **hardware transcoding** (Settings → Playback → Transcoding):
     - Hardware acceleration: **Video Acceleration API (VAAPI)**
     - Device: `/dev/dri/renderD128`
     - Enable hardware decoding for: H.264, HEVC, VP9, etc.

> **Note:** The docker-compose file passes `/dev/dri` to Jellyfin for Intel VAAPI hardware transcoding. If your system doesn't have an Intel iGPU, remove the `devices` section from the Jellyfin service.

## Step 8: Configure Jellyseerr (Request UI)

1. Open **http://localhost:5055**
2. Sign in with Jellyfin:
   - Select **Jellyfin** as the media server
   - Jellyfin URL: `http://jellyfin:8096`
   - Enter your Jellyfin admin credentials
3. Add Radarr:
   - Default Server: Yes
   - Server Name: `Radarr`
   - Hostname: `radarr`
   - Port: `7878`
   - API Key: (from Radarr Step 5)
   - Quality Profile: pick one (e.g., "HD-1080p")
   - Root Folder: `/movies`
   - Test & Save
4. Add Sonarr:
   - Default Server: Yes
   - Server Name: `Sonarr`
   - Hostname: `sonarr`
   - Port: `8989`
   - API Key: (from Sonarr Step 6)
   - Quality Profile: pick one
   - Root Folder: `/tv`
   - Test & Save

> **TMDB / IPv6 issue**: TMDB (The Movie Database) is the API Jellyseerr
> uses for all movie/show metadata — posters, descriptions, ratings, search
> results, and trending content. Its calls may fail inside Docker.
> Docker's bridge network can't route IPv6 (`ENETUNREACH`), and if Node.js
> tries IPv6 first (depends on DNS resolver ordering), TMDB requests fail.
> Setting `forceIpv4First` ensures IPv4 is always preferred. Fix this by
> editing `config/jellyseerr/settings.json` and setting:
> ```json
> "network": { "forceIpv4First": true, ... }
> ```
> Then restart Jellyseerr (`docker compose restart jellyseerr`).

## Step 9: Configure Caddy Reverse Proxy (URL Base Paths)

Caddy serves all services under a single port (`:80`) using subpaths. Some apps handle base URLs natively, others need Caddy to strip the prefix.

### Radarr, Sonarr, Prowlarr — Set `UrlBase` in config

These apps have a built-in base URL setting. Set it in each app's `config.xml` so they know they're being served under a subpath:

```bash
for app in sonarr radarr prowlarr; do
  docker exec $app sed -i "s|<UrlBase></UrlBase>|<UrlBase>/$app</UrlBase>|" /config/config.xml
done

# Restart for changes to take effect
docker compose restart sonarr radarr prowlarr
```

Caddy passes requests through **without stripping** the prefix (the apps handle it themselves):
```
handle /radarr*   { reverse_proxy radarr:7878 }
handle /sonarr*   { reverse_proxy sonarr:8989 }
handle /prowlarr* { reverse_proxy prowlarr:9696 }
```

### Jellyfin, Jellyseerr, qBittorrent — Caddy strips the prefix

These apps don't have a native base URL setting, so Caddy strips the subpath before forwarding:
```
handle /jellyfin/*    { uri strip_prefix /jellyfin;    reverse_proxy jellyfin:8096 }
handle /jellyseerr/*  { uri strip_prefix /jellyseerr;  reverse_proxy jellyseerr:5055 }
handle /qbittorrent/* { uri strip_prefix /qbittorrent; reverse_proxy qbittorrent:8080 }
```

### Result

All services accessible from a single IP on port 80:

| URL Path        | Service      |
|-----------------|--------------|
| `/`             | Jellyseerr (default) |
| `/radarr`       | Radarr       |
| `/sonarr`       | Sonarr       |
| `/prowlarr`     | Prowlarr     |
| `/jellyfin`     | Jellyfin     |
| `/jellyseerr`   | Jellyseerr   |
| `/qbittorrent`  | qBittorrent  |

---

## Usage

### Requesting a Movie/Show
1. Go to **http://localhost:5055** (Jellyseerr)
2. Search for a movie or TV show
3. Click **Request**
4. It gets automatically downloaded and added to Jellyfin!

### Watching
1. Go to **http://localhost:8096** (Jellyfin)
2. Your media appears automatically once downloaded

### Monitoring Downloads
- **qBittorrent**: http://localhost:8080 — see active downloads
- **Radarr**: http://localhost:7878 → Activity — see movie queue
- **Sonarr**: http://localhost:8989 → Activity — see TV queue

---

## Management Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f              # all services
docker compose logs -f jellyfin     # specific service

# Restart a service
docker compose restart radarr

# Update all containers
docker compose pull && docker compose up -d

# Check status
docker compose ps
```

---

## Folder Layout

```
$PROJECT_DIR/
├── docker-compose.yml          # Stack definition
├── .env                        # MEDIA_DIR path (not committed)
├── setup.md                    # This file
└── config/                     # Persistent config for all services
    ├── jellyfin/
    ├── jellyseerr/
    ├── radarr/
    ├── sonarr/
    ├── prowlarr/
    └── qbittorrent/

$MEDIA_DIR/                     # Media storage
├── movies/                     # Radarr-managed movie files
├── tv/                         # Sonarr-managed TV files
└── downloads/                  # qBittorrent downloads
    ├── complete/
    └── incomplete/
```

## Cloudflare WARP (Optional — Bypass ISP Blocks)

If your ISP blocks torrent indexer sites, you can install **Cloudflare WARP** on the host machine to route traffic through Cloudflare's network.

### Installation (Fedora)

```bash
# Add Cloudflare WARP repo
curl -fsSl https://pkg.cloudflareclient.com/cloudflare-warp-ascii.repo \
  | sudo tee /etc/yum.repos.d/cloudflare-warp.repo

# Install
sudo dnf install -y cloudflare-warp
```

### Setup

```bash
# Register (one-time, --accept-tos must come before the subcommand)
warp-cli --accept-tos registration new

# Connect
warp-cli --accept-tos connect

# Verify it's working
curl https://www.cloudflare.com/cdn-cgi/trace/ | grep warp
# Should show: warp=on
```

### Useful Commands

```bash
warp-cli status        # Check connection status
warp-cli connect       # Connect to WARP
warp-cli disconnect    # Disconnect from WARP
```

Shell aliases (added to `~/.zshrc`):
```bash
alias cf-warp-on='warp-cli connect && warp-cli status'
alias cf-warp-off='warp-cli disconnect && warp-cli status'
```

> **Note:** WARP runs on the host, so all Docker containers (Prowlarr, FlareSolverr, etc.) benefit from it automatically since they share the host's network route. The `dns: 1.1.1.1` entries in `docker-compose.yml` complement this by ensuring Cloudflare DNS resolution.

---

## Troubleshooting

- **Can't connect between services?** They're all on the `media-net` Docker network and refer to each other by container name.
- **Permission denied on media dir?** Run: `sudo chown -R $(id -u):$(id -g) "$MEDIA_DIR"`
- **qBittorrent password?** Check logs: `docker compose logs qbittorrent | grep password`
- **Indexers not working in Radarr/Sonarr?** Make sure Prowlarr synced them (Prowlarr → Settings → Apps → Sync).
