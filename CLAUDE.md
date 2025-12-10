# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker Compose-based media server stack that uses Real-Debrid for cloud torrenting instead of local downloads. The stack automates media requests and streaming through Plex.

## Architecture

Data flows through these services in sequence:
1. **Overseerr** (port 5055) - Users request content
2. **Sonarr** (port 8989) / **Radarr** (port 7878) - TV/Movie automation
3. **Prowlarr** (port 9696) - Indexer management, finds torrents
4. **RDTClient** (port 6500) - qBittorrent API emulator, sends to Real-Debrid
5. **Zurg** (port 9999) - Organizes Real-Debrid content as WebDAV
6. **rclone** - Mounts Zurg as local filesystem at `./mnt/zurg`
7. **Plex** (port 32400) - Streams media to users

External access is provided via **Cloudflare Tunnel** (cloudflared) - only Overseerr should be exposed externally.

## Commands

```bash
# Start all services
docker compose up -d

# Start with Cloudflare tunnel
docker compose --profile tunnel up -d

# Stop all services
docker compose down

# View logs (all services)
docker compose logs -f

# View specific service logs
docker compose logs -f zurg
docker compose logs -f rclone
docker compose logs -f plex

# Restart a single service
docker compose restart <service>

# Update all images
docker compose pull && docker compose up -d

# Check service status
docker compose ps
```

## Configuration Files

Sensitive configs must be created from examples before first run:
- `config/zurg/config.yml` - Real-Debrid API token goes here
- `config/rclone/rclone.conf` - WebDAV mount config (no changes needed)
- `.env` - Environment variables (PUID, PGID, TZ, PLEX_CLAIM, CLOUDFLARE_TUNNEL_TOKEN)

Create from examples:
```bash
cp config/zurg/config.yml.example config/zurg/config.yml
cp config/rclone/rclone.conf.example config/rclone/rclone.conf
cp .env.example .env
```

## Key Environment Variables

All set in `.env` file:
- `PUID`/`PGID` - User/group IDs (find with `id $(whoami)`)
- `TZ` - Timezone (e.g., `America/New_York`)
- `PLEX_CLAIM` - Only needed for new Plex installs (get from https://plex.tv/claim)
- `CLOUDFLARE_TUNNEL_TOKEN` - For external access via Cloudflare Tunnel

## Service Dependencies

The services have startup dependencies:
- `rclone` waits for `zurg` to be healthy
- `plex` depends on `rclone`
- `overseerr` depends on `sonarr`, `radarr`, and `plex`
- `cloudflared` depends on `overseerr` (only starts with `--profile tunnel`)

## Inter-Service Communication

Services communicate using container names as hostnames:
- Sonarr/Radarr connect to RDTClient at `rdtclient:6500`
- Prowlarr connects to Sonarr at `sonarr:8989` and Radarr at `radarr:7878`
- Overseerr connects to Sonarr/Radarr/Plex using their container names
- rclone connects to Zurg at `http://zurg:9999/dav`
- Plex uses `network_mode: host` so it's accessed at `localhost:32400`

## Mount Points

- `./mnt/zurg` - rclone mount point for Real-Debrid content
- `./config/<service>` - Persistent config for each service
- `./data/zurg` - Zurg data directory

Inside containers, the Real-Debrid content is available at `/debrid`.

## Troubleshooting

```bash
# Verify Real-Debrid API token
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.real-debrid.com/rest/1.0/user

# Check if rclone mount is working
ls mnt/zurg/

# Verify FUSE is available (required for rclone)
ls -la /dev/fuse
```
