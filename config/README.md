# Configuration Directory

This directory contains configuration files for all services in the stack.

## Setup Required

Before first run, copy the example config files:

```bash
cp config/zurg/config.yml.example config/zurg/config.yml
cp config/rclone/rclone.conf.example config/rclone/rclone.conf
cp .env.example .env
```

Then edit the files with your credentials.

## Service Configuration

### Zurg (Manual Setup Required)
- **File:** `zurg/config.yml`
- **Setup:** Copy from `config.yml.example` and add your Real-Debrid API token
- Get your token from: https://real-debrid.com/apitoken

### Rclone (No Changes Needed)
- **File:** `rclone/rclone.conf`
- **Setup:** Copy from `rclone.conf.example` - no modifications needed
- Connects to Zurg via Docker network automatically

### Plex (Auto-Generated or Example)
- **File:** `plex/Library/Application Support/Plex Media Server/Preferences.xml`
- **Example:** `Preferences.xml.example` provided for reference
- **Setup:** Auto-generates on first run with Plex account details
- **Web UI:** http://localhost:32400/web
- For new installs, get claim token from https://plex.tv/claim and set `PLEX_CLAIM` in `.env`

### Sonarr (Auto-Generated or Example)
- **File:** `sonarr/config.xml`
- **Example:** `config.xml.example` provided for reference
- **Setup:** Auto-generates on first run, or copy from example
- **Web UI:** http://localhost:8989
- API key is auto-generated - needed for Prowlarr/Overseerr integration

### Radarr (Auto-Generated or Example)
- **File:** `radarr/config.xml`
- **Example:** `config.xml.example` provided for reference
- **Setup:** Auto-generates on first run, or copy from example
- **Web UI:** http://localhost:7878
- API key is auto-generated - needed for Prowlarr/Overseerr integration

### Prowlarr (Auto-Generated or Example)
- **File:** `prowlarr/config.xml`
- **Example:** `config.xml.example` provided for reference
- **Setup:** Auto-generates on first run, or copy from example
- **Web UI:** http://localhost:9696
- Connects to Sonarr/Radarr to sync indexers automatically

### RDTClient (Auto-Generated)
- **Directory:** `rdtclient/`
- **Setup:** Configured via web UI at http://localhost:6500
- Stores settings in SQLite database
- Add Real-Debrid credentials through the web interface

### Overseerr (Auto-Generated or Example)
- **File:** `overseerr/settings.json`
- **Example:** `settings.json.example` provided for reference
- **Setup:** Auto-generates on first run, or copy from example
- **Web UI:** http://localhost:5055
- Setup wizard guides you through Plex/Sonarr/Radarr integration
- Requires Plex, Sonarr, and Radarr API keys

## Security Notes

- Never commit actual config files with credentials to git
- The `.gitignore` file excludes sensitive files like `config.yml`, `rclone.conf`, etc.
- Only `.example` files and `.gitkeep` placeholders are tracked in version control
