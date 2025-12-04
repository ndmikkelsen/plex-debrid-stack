# Migration Guide: Usenet to Debrid + Zurg + Plex

## Overview

This guide walks you through migrating from a Usenet-based setup (NZBGet, Sonarr, Radarr, Plex) to a Real-Debrid + Zurg streaming setup with full automation and multi-user request management.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER EXPERIENCE                              │
├─────────────────────────────────────────────────────────────────────┤
│  Users browse & request    →    Overseerr (localhost:5055)          │
│  content via nice UI                    │                            │
│                                         ▼                            │
├─────────────────────────────────────────────────────────────────────┤
│                         AUTOMATION LAYER                             │
├─────────────────────────────────────────────────────────────────────┤
│  TV Shows                  →    Sonarr (localhost:8989)             │
│  Movies                    →    Radarr (localhost:7878)             │
│                                         │                            │
│  Indexer Management        →    Prowlarr (localhost:9696)           │
│  (finds torrents)                       │                            │
│                                         ▼                            │
│  Download Client           →    RDTClient (localhost:6500)          │
│  (sends to Real-Debrid)                 │                            │
│                                         ▼                            │
├─────────────────────────────────────────────────────────────────────┤
│                         DEBRID LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Cloud Torrent Service     →    Real-Debrid (cloud)                 │
│                                         │                            │
│  WebDAV Server             →    Zurg (localhost:9999)               │
│  (organizes RD content)                 │                            │
│                                         ▼                            │
│  Filesystem Mount          →    rclone                              │
│                                         │                            │
│                                         ▼                            │
├─────────────────────────────────────────────────────────────────────┤
│                         MEDIA SERVER                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Stream to devices         →    Plex (localhost:32400)              │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **User** requests a movie/show in **Overseerr**
2. **Overseerr** sends request to **Sonarr** (TV) or **Radarr** (Movies)
3. **Sonarr/Radarr** searches **Prowlarr** for torrents
4. **Sonarr/Radarr** sends torrent to **RDTClient**
5. **RDTClient** adds torrent to **Real-Debrid** cloud
6. **Zurg** sees new content and organizes it
7. **rclone** mounts Zurg as local filesystem
8. **Plex** sees new files and adds to library
9. **User** watches content on **Plex**

---

## Prerequisites

1. **Real-Debrid Account**: Sign up at https://real-debrid.com (~€16/180 days)
2. **Docker & Docker Compose**: Already installed
3. **FUSE support**: Required for rclone mount (`/dev/fuse` must exist)

---

## Step 1: Copy Example Configuration Files

```bash
# Copy the example configs to create your own
cp config/zurg/config.yml.example config/zurg/config.yml
cp config/rclone/rclone.conf.example config/rclone/rclone.conf
cp .env.example .env
```

---

## Step 2: Get Your Real-Debrid API Token

1. Go to https://real-debrid.com/apitoken
2. Copy your API token
3. Edit `config/zurg/config.yml` and replace `YOUR_REAL_DEBRID_API_TOKEN_HERE` with your token

---

## Step 3: Configure Your Environment

### Find Your PUID/PGID

```bash
id $(whoami)
# Output: uid=1000(username) gid=1000(username) ...
```

Update all `PUID` and `PGID` values in `docker-compose.yml` to match.

### Update Timezone

Change `TZ=America/New_York` to your timezone throughout `docker-compose.yml`.

Common timezones:
- `America/Los_Angeles` (Pacific)
- `America/Chicago` (Central)
- `America/New_York` (Eastern)
- `Europe/London`
- `Europe/Paris`

### Plex Claim Token (New Installation Only)

If this is a new Plex installation:
1. Go to https://plex.tv/claim
2. Copy the claim token (valid for 4 minutes)
3. Uncomment and set `PLEX_CLAIM` in docker-compose.yml

---

## Step 4: Migrate Existing Configurations (Optional)

If you want to keep your existing Sonarr/Radarr/Plex configurations:

1. Stop your current containers
2. Copy config directories or update volume mounts:

```yaml
plex:
  volumes:
    - /path/to/existing/plex/config:/config

sonarr:
  volumes:
    - /path/to/existing/sonarr/config:/config

radarr:
  volumes:
    - /path/to/existing/radarr/config:/config
```

**Important**: You'll need to reconfigure download clients even if migrating configs.

---

## Step 5: Start the Stack

```bash
# Create necessary directories
mkdir -p mnt data/zurg config/rdtclient

# Start the services
docker-compose up -d

# Watch logs (Ctrl+C to exit)
docker-compose logs -f
```

Wait for all services to be healthy before proceeding.

---

## Step 6: Configure RDTClient

1. Access RDTClient at http://localhost:6500
2. Create an account (first-time setup)
3. Go to **Settings**
4. Under **Download Client**, select **Real-Debrid**
5. Enter your Real-Debrid API token
6. Configure download path: `/data/downloads`
7. Save settings

---

## Step 7: Configure Prowlarr (Indexers)

1. Access Prowlarr at http://localhost:9696
2. Go to **Indexers** → **Add Indexer**
3. Add torrent indexers. Recommended public indexers:
   - 1337x
   - The Pirate Bay
   - EZTV (for TV)
   - TorrentGalaxy
   - LimeTorrents

4. Go to **Settings** → **Apps**
5. Add **Sonarr**:
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key: (get from Sonarr → Settings → General)
6. Add **Radarr**:
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: (get from Radarr → Settings → General)
7. Click **Sync App Indexers**

---

## Step 8: Configure Sonarr

1. Access Sonarr at http://localhost:8989
2. Go to **Settings** → **Download Clients** → **+**
3. Select **qBittorrent** (RDTClient emulates this)
4. Configure:
   - Name: `RDTClient`
   - Host: `rdtclient`
   - Port: `6500`
   - Username: (your RDTClient username)
   - Password: (your RDTClient password)
5. Test and Save

6. Go to **Settings** → **Media Management**
7. Add Root Folder: `/media/shows`

---

## Step 9: Configure Radarr

1. Access Radarr at http://localhost:7878
2. Go to **Settings** → **Download Clients** → **+**
3. Select **qBittorrent**
4. Configure same as Sonarr:
   - Name: `RDTClient`
   - Host: `rdtclient`
   - Port: `6500`
   - Username/Password: (your RDTClient credentials)
5. Test and Save

6. Go to **Settings** → **Media Management**
7. Add Root Folder: `/media/movies`

---

## Step 10: Configure Plex

1. Access Plex at http://localhost:32400/web
2. Complete initial setup if new installation
3. Go to **Settings** → **Libraries** → **Add Library**
4. Add libraries:
   - **Movies**: `/media/movies`
   - **TV Shows**: `/media/shows`
   - **Anime**: `/media/anime` (optional)

---

## Step 11: Configure Overseerr

1. Access Overseerr at http://localhost:5055
2. Sign in with your Plex account
3. Connect to Plex:
   - Server: Select your Plex server
   - Libraries: Select your movie and TV libraries
4. Connect to Sonarr:
   - Hostname: `sonarr`
   - Port: `8989`
   - API Key: (from Sonarr → Settings → General)
   - Root Folder: `/media/shows`
   - Quality Profile: Select preferred quality
5. Connect to Radarr:
   - Hostname: `radarr`
   - Port: `7878`
   - API Key: (from Radarr → Settings → General)
   - Root Folder: `/media/movies`
   - Quality Profile: Select preferred quality
6. Configure users and permissions as desired

---

## Step 12: Set Up Cloudflare Tunnel (External Access)

This allows friends to access Overseerr from anywhere without exposing your home IP or opening ports.

### Prerequisites
- A Cloudflare account (free)
- A domain name (can be purchased through Cloudflare or elsewhere)

### Create the Tunnel

1. Go to https://one.dash.cloudflare.com (Cloudflare Zero Trust)
2. Navigate to **Networks** → **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the connector
5. Name your tunnel (e.g., "plex-media-server")
6. Copy the tunnel token (starts with `eyJ...`)

### Configure the Environment

1. Copy the example environment file:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and add your tunnel token:
   ```
   CLOUDFLARE_TUNNEL_TOKEN=eyJ...your_token_here
   ```

### Configure Public Hostname

1. In the Cloudflare tunnel configuration, go to **Public Hostnames**
2. Add a new public hostname:
   - **Subdomain**: `requests` (or whatever you prefer)
   - **Domain**: Select your domain
   - **Service Type**: `HTTP`
   - **URL**: `overseerr:5055`
3. Save the configuration

### Start the Tunnel

```bash
# Restart the stack to start cloudflared
docker-compose up -d

# Verify tunnel is running
docker logs cloudflared
```

### Share with Friends

Your friends can now access Overseerr at:
```
https://requests.yourdomain.com
```

They'll need to:
1. Create a Plex account (if they don't have one)
2. You invite them to your Plex server
3. They sign into Overseerr with their Plex account
4. Request content!

---

## Step 13: Decommission Old Services

Once everything is working:

1. Stop NZBGet (no longer needed)
2. Remove NZBGet from your old docker-compose
3. Keep old media accessible during transition
4. Eventually remove old local media storage

---

## Directory Structure

After setup, your mount will look like:
```
mnt/
└── zurg/
    ├── movies/
    │   └── Movie Name (Year)/
    │       └── Movie.Name.Year.mkv
    ├── shows/
    │   └── Show Name/
    │       └── Season 01/
    │           └── Show.Name.S01E01.mkv
    └── anime/
        └── ...
```

---

## Service URLs

### External (Share with Friends)

| Service | URL | Purpose |
|---------|-----|---------|
| Overseerr | https://requests.yourdomain.com | User requests |
| Plex | https://app.plex.tv | Media streaming |

### Internal (Admin Only)

| Service | URL | Purpose |
|---------|-----|---------|
| Overseerr | http://localhost:5055 | User requests |
| Plex | http://localhost:32400/web | Media streaming |
| Sonarr | http://localhost:8989 | TV show management |
| Radarr | http://localhost:7878 | Movie management |
| Prowlarr | http://localhost:9696 | Indexer management |
| RDTClient | http://localhost:6500 | Download client |
| Zurg | http://localhost:9999 | Debrid WebDAV |

**For your users**: Share only the Overseerr external URL. They'll use app.plex.tv to watch content.

---

## Troubleshooting

### Mount not working
```bash
# Check rclone logs
docker logs rclone

# Verify FUSE is available
ls -la /dev/fuse

# Test mount manually
docker exec -it rclone ls /data/zurg
```

### Zurg not connecting
```bash
# Check zurg logs
docker logs zurg

# Verify API token is correct
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.real-debrid.com/rest/1.0/user
```

### Plex not seeing files
1. Ensure rclone mount is healthy: `ls mnt/zurg/`
2. Check Plex library paths are correct
3. Restart Plex after mount is ready: `docker restart plex`
4. Trigger library scan in Plex

### RDTClient not connecting to Real-Debrid
1. Verify API token in RDTClient settings
2. Check RDTClient logs: `docker logs rdtclient`
3. Test Real-Debrid API directly

### Sonarr/Radarr can't connect to RDTClient
1. Ensure you're using `rdtclient` as hostname (not `localhost`)
2. Verify RDTClient credentials
3. Check that RDTClient is running: `docker ps`

---

## Security Notes

- **Overseerr**: Safe to expose to users, has its own auth
- **Plex**: Safe to expose, has its own auth
- **Sonarr/Radarr/Prowlarr**: Keep admin-only, have minimal auth
- **RDTClient/Zurg**: Keep internal, contain API tokens

Consider using a reverse proxy (Traefik, Nginx Proxy Manager) for external access.

---

## Benefits of This Setup

1. **No local storage needed** - Media streams from Real-Debrid
2. **Instant availability** - No download wait times
3. **Multi-user requests** - Overseerr lets everyone request content
4. **Full automation** - Sonarr/Radarr handle everything
5. **Lower bandwidth** - Only streams what you watch
6. **Cheaper** - ~$3/month vs Usenet subscriptions + indexers

---

## Costs

| Service | Cost |
|---------|------|
| Real-Debrid | ~€16/180 days (~$3/month) |
| Usenet (old) | ~$10-15/month + indexers |

**Savings**: ~$7-12/month
