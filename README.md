# Plex Debrid Stack

A fully automated media server stack using Real-Debrid for cloud torrenting, with Plex for streaming and Overseerr for user requests.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  USERS                                                           │
│  Friends access Overseerr via Cloudflare Tunnel                  │
│  https://requests.yourdomain.com                                 │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  OVERSEERR                                                       │
│  Browse, search, and request movies/TV shows                     │
└──────────────────────┬───────────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
┌─────────────────┐       ┌─────────────────┐
│  SONARR         │       │  RADARR         │
│  TV Shows       │       │  Movies         │
└────────┬────────┘       └────────┬────────┘
         │                         │
         └────────────┬────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│  PROWLARR                                                        │
│  Indexer management - finds torrents                             │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  RDTCLIENT                                                       │
│  Sends torrents to Real-Debrid (qBittorrent API emulator)        │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  REAL-DEBRID (Cloud)                                             │
│  Downloads torrents in the cloud                                 │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  ZURG                                                            │
│  Organizes Real-Debrid content via WebDAV                        │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  RCLONE                                                          │
│  Mounts Zurg as local filesystem                                 │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  PLEX                                                            │
│  Streams media to all devices                                    │
└──────────────────────────────────────────────────────────────────┘
```

## Services

| Service | Port | Description |
|---------|------|-------------|
| Plex | 32400 | Media server |
| Overseerr | 5055 | Request management UI |
| Sonarr | 8989 | TV show automation |
| Radarr | 7878 | Movie automation |
| Prowlarr | 9696 | Indexer management |
| RDTClient | 6500 | Real-Debrid download client |
| Zurg | 9999 | Real-Debrid WebDAV server |
| Cloudflared | - | Secure tunnel for external access |

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Real-Debrid account (~$3/month)
- Cloudflare account (free) + domain for external access

### 1. Copy Example Configs

```bash
# Copy config examples
cp config/zurg/config.yml.example config/zurg/config.yml
cp config/rclone/rclone.conf.example config/rclone/rclone.conf
cp .env.example .env
```

### 2. Configure Zurg

Edit `config/zurg/config.yml` and add your Real-Debrid API token:

```yaml
token: YOUR_REAL_DEBRID_API_TOKEN
```

Get your token from: https://real-debrid.com/apitoken

### 3. Configure Environment

Edit `.env` and add your Cloudflare tunnel token (see [MIGRATION.md](MIGRATION.md) for setup).

### 4. Update docker-compose.yml

Update the following values:
- `PUID` / `PGID` - Run `id $(whoami)` to find your values
- `TZ` - Your timezone (e.g., `America/New_York`)
- `PLEX_CLAIM` - Get from https://plex.tv/claim (new installs only)

### 5. Start the Stack

```bash
# Create directories
mkdir -p mnt data/zurg config/rdtclient

# Start services
docker-compose up -d

# View logs
docker-compose logs -f
```

### 6. Configure Services

See [MIGRATION.md](MIGRATION.md) for detailed configuration steps for each service.

## Directory Structure

```
.
├── docker-compose.yml              # Service definitions
├── .env.example                    # Environment template (copy to .env)
├── .gitignore                      # Git ignore rules
├── README.md                       # This file
├── MIGRATION.md                    # Detailed setup guide
└── config/
    ├── zurg/
    │   └── config.yml.example      # Zurg config template (copy to config.yml)
    └── rclone/
        └── rclone.conf.example     # rclone config template (copy to rclone.conf)

# Created after setup (gitignored):
├── .env                            # Your environment variables
├── config/
│   ├── zurg/config.yml             # Your Zurg config
│   ├── rclone/rclone.conf          # Your rclone config
│   ├── plex/                       # Plex config (auto-generated)
│   ├── sonarr/                     # Sonarr config (auto-generated)
│   ├── radarr/                     # Radarr config (auto-generated)
│   ├── prowlarr/                   # Prowlarr config (auto-generated)
│   ├── overseerr/                  # Overseerr config (auto-generated)
│   └── rdtclient/                  # RDTClient config (auto-generated)
├── mnt/zurg/                       # Mounted Real-Debrid content
└── data/zurg/                      # Zurg data
```

## User Access

### For You (Admin)

Access all services locally:
- Plex: http://localhost:32400/web
- Overseerr: http://localhost:5055
- Sonarr: http://localhost:8989
- Radarr: http://localhost:7878
- Prowlarr: http://localhost:9696
- RDTClient: http://localhost:6500

### For Friends

Share these URLs:
- **Request content**: https://requests.yourdomain.com (Overseerr)
- **Watch content**: https://app.plex.tv (Plex)

Friends need:
1. A Plex account
2. An invitation to your Plex server
3. Sign into Overseerr with their Plex account

## Common Commands

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f plex
docker-compose logs -f zurg

# Restart a service
docker-compose restart plex

# Update all images
docker-compose pull && docker-compose up -d

# Check service status
docker-compose ps
```

## Troubleshooting

### Zurg not connecting
```bash
docker logs zurg
# Verify token: curl -H "Authorization: Bearer YOUR_TOKEN" https://api.real-debrid.com/rest/1.0/user
```

### rclone mount not working
```bash
docker logs rclone
ls -la /dev/fuse  # Verify FUSE is available
```

### Plex not seeing media
```bash
ls mnt/zurg/  # Check if mount is working
docker restart plex
```

### Cloudflare tunnel not connecting
```bash
docker logs cloudflared
# Verify CLOUDFLARE_TUNNEL_TOKEN is set in .env
```

## Costs

| Service | Cost |
|---------|------|
| Real-Debrid | ~€16/180 days (~$3/month) |
| Cloudflare | Free |
| Domain | ~$10/year |

## Security Notes

- **Never expose** Sonarr, Radarr, Prowlarr, RDTClient, or Zurg externally
- **Safe to expose**: Overseerr (via tunnel), Plex (has own auth)
- Keep your `.env` file secret (contains tunnel token)
- Real-Debrid API token is stored in `config/zurg/config.yml`

## Documentation

- [MIGRATION.md](MIGRATION.md) - Detailed setup and configuration guide
- [Real-Debrid](https://real-debrid.com)
- [Zurg](https://github.com/debridmediamanager/zurg)
- [Overseerr](https://overseerr.dev)
- [Sonarr](https://sonarr.tv)
- [Radarr](https://radarr.video)

## License

This configuration is provided as-is for personal use.
