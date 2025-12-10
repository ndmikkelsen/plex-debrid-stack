# Plex Debrid Stack

A fully automated media server stack using Real-Debrid for cloud torrenting, with Plex for streaming and Overseerr for user requests.

## How It Works

Instead of downloading torrents locally, this stack uses [Real-Debrid](https://real-debrid.com) to download content in the cloud. The content is then streamed directly to Plex - no local storage needed for new content.

```
User Request → Overseerr → Sonarr/Radarr → Prowlarr → RDTClient → Real-Debrid
                                                                      ↓
                              Plex ← rclone ← Zurg ← Real-Debrid Library
```

## Services

| Service | Port | Description |
|---------|------|-------------|
| **Plex** | 32400 | Media server - streams to all your devices |
| **Overseerr** | 5055 | Request UI - users browse and request content |
| **Sonarr** | 8989 | TV show automation |
| **Radarr** | 7878 | Movie automation |
| **Prowlarr** | 9696 | Indexer management - finds torrents |
| **RDTClient** | 6500 | Sends torrents to Real-Debrid |
| **Zurg** | 9999 | Serves Real-Debrid library via WebDAV |
| **Cloudflared** | - | Secure tunnel for external access (optional) |

## Prerequisites

- Docker and Docker Compose
- [Real-Debrid](https://real-debrid.com) account (~$3/month)
- (Optional) Cloudflare account + domain for external access

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/plex-debrid-stack.git
cd plex-debrid-stack
```

### 2. Copy Configuration Files

```bash
# Required configs
cp .env.example .env
cp config/zurg/config.yml.example config/zurg/config.yml
cp config/rclone/rclone.conf.example config/rclone/rclone.conf

# Create required directories
mkdir -p mnt data/zurg
```

### 3. Configure Real-Debrid

Get your API token from https://real-debrid.com/apitoken

Edit `config/zurg/config.yml`:
```yaml
token: "YOUR_REAL_DEBRID_API_TOKEN"
```

### 4. Configure Environment

Edit `.env` with your settings:
```bash
# Find your user/group IDs
id $(whoami)

# Edit .env
PUID=1000          # Your user ID
PGID=1000          # Your group ID
TZ=America/New_York  # Your timezone
```

### 5. Start the Stack

```bash
docker compose up -d
```

### 6. Configure Services

Access each service and complete initial setup:

1. **Plex** (http://localhost:32400/web)
   - Sign in with your Plex account
   - Add library pointing to `/debrid/movies` and `/debrid/shows`

2. **Sonarr** (http://localhost:8989)
   - Settings → Media Management → Add root folder: `/debrid/shows`
   - Settings → Download Clients → Add qBittorrent (host: `rdtclient`, port: `6500`)

3. **Radarr** (http://localhost:7878)
   - Settings → Media Management → Add root folder: `/debrid/movies`
   - Settings → Download Clients → Add qBittorrent (host: `rdtclient`, port: `6500`)

4. **Prowlarr** (http://localhost:9696)
   - Add indexers (torrent sites)
   - Settings → Apps → Add Sonarr and Radarr

5. **RDTClient** (http://localhost:6500)
   - Settings → Add Real-Debrid provider with your API token

6. **Overseerr** (http://localhost:5055)
   - Connect to Plex
   - Add Sonarr and Radarr servers

## External Access (Optional)

To let friends access Overseerr from outside your network:

### Using Cloudflare Tunnel

1. Create a tunnel in [Cloudflare Zero Trust](https://one.dash.cloudflare.com/)
2. Add your tunnel token to `.env`:
   ```
   CLOUDFLARE_TUNNEL_TOKEN=your_token_here
   ```
3. Start with the tunnel profile:
   ```bash
   docker compose --profile tunnel up -d
   ```

### For Friends

Share these with your friends:
- **Request content**: https://requests.yourdomain.com (Overseerr)
- **Watch content**: https://app.plex.tv or your Plex apps

They'll need:
1. A Plex account (free)
2. An invitation to your Plex server
3. Sign into Overseerr with their Plex account

## Commands

```bash
# Start all services
docker compose up -d

# Start with Cloudflare tunnel
docker compose --profile tunnel up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f

# View specific service logs
docker compose logs -f zurg
docker compose logs -f plex

# Restart a service
docker compose restart plex

# Update all images
docker compose pull && docker compose up -d

# Check status
docker compose ps
```

## Directory Structure

```
.
├── docker-compose.yml          # Service definitions
├── .env.example                # Environment template
├── config/
│   ├── zurg/
│   │   └── config.yml.example  # Zurg config (Real-Debrid token)
│   ├── rclone/
│   │   └── rclone.conf.example # rclone config (no changes needed)
│   ├── plex/                   # Plex config (auto-generated)
│   ├── sonarr/                 # Sonarr config (auto-generated)
│   ├── radarr/                 # Radarr config (auto-generated)
│   ├── prowlarr/               # Prowlarr config (auto-generated)
│   ├── overseerr/              # Overseerr config (auto-generated)
│   └── rdtclient/              # RDTClient config (auto-generated)
├── mnt/
│   └── zurg/                   # Mounted Real-Debrid content
└── data/
    └── zurg/                   # Zurg cache data
```

## Troubleshooting

### Zurg not starting
```bash
docker compose logs zurg
# Verify your Real-Debrid token:
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.real-debrid.com/rest/1.0/user
```

### Mount not working
```bash
docker compose logs rclone
# Check if FUSE is available:
ls -la /dev/fuse
# Check if mount is working:
ls mnt/zurg/
```

### Plex not seeing media
```bash
# Verify mount is working first
ls mnt/zurg/
# Restart Plex
docker compose restart plex
# In Plex, scan library or add libraries pointing to /debrid
```

### Services can't connect to each other
Services use container names as hostnames:
- RDTClient: `rdtclient:6500`
- Sonarr: `sonarr:8989`
- Radarr: `radarr:7878`
- Plex: `plex:32400` (or `localhost:32400` since it uses host networking)

## Costs

| Service | Cost |
|---------|------|
| Real-Debrid | ~€16/180 days (~$3/month) |
| Cloudflare | Free |
| Domain | ~$10-15/year |

## Security

- **Never expose** Sonarr, Radarr, Prowlarr, RDTClient, or Zurg to the internet
- **Safe to expose**: Overseerr (via tunnel), Plex (has its own auth)
- Keep `.env` and config files with tokens private
- The `.gitignore` excludes sensitive files from version control

## Links

- [Real-Debrid](https://real-debrid.com)
- [Zurg](https://github.com/debridmediamanager/zurg-testing)
- [Overseerr](https://overseerr.dev)
- [Sonarr](https://sonarr.tv)
- [Radarr](https://radarr.video)
- [Prowlarr](https://prowlarr.com)

## License

MIT License - See [LICENSE](LICENSE) for details.
