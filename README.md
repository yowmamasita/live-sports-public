# Live Sports M3U Server

Multi-source live sports streaming server that aggregates channels from multiple providers, health-checks them, and serves an M3U playlist with a streaming HLS proxy — designed for Apple TV (UHF app) and Stremio.

![Dashboard](https://raw.githubusercontent.com/yowmamasita/live-sports-public/main/screenshot.png)

## How it works

1. **Fetches channels** from all sources on startup and every 30 minutes
2. **Discovers DaddyLive player endpoints** — each channel maps to a specific embed endpoint (`daddy2`, `daddy3`, or `daddy5`); the server rotates through them on health-check failure until the correct one is found, then caches it
3. **Health-checks** every channel — DaddyLive channels get a lightweight playlist validation; other sources get a full segment download + `ffprobe` to verify video + audio
4. **Serves an M3U playlist** with only working channels at `/playlist.m3u`
5. **Streams HLS** through a proxy that rewrites playlist URLs to relative paths, forwards byte-range requests, and preserves upstream `Content-Type` — works behind reverse proxies and tunnels without configuration

## Docker

Pre-built multi-arch images (`amd64`, `arm64`, `arm/v7`) are available on GHCR:

```bash
docker run -d -p 8765:8765 ghcr.io/yowmamasita/live-sports:latest
```

Or with Docker Compose:

```yaml
services:
  live-sports:
    image: ghcr.io/yowmamasita/live-sports:latest
    ports:
      - "8765:8765"
    restart: unless-stopped
```

Images are automatically built and pushed on every commit to `main`.

### Environment Variables

| Variable | Description |
|----------|-------------|
| `BASE_URL` | Override the base URL used in M3U playlist entries (e.g. `http://myserver:8765`). Without this, the server auto-detects its LAN IP. Set this when running behind a reverse proxy or accessing via a hostname. |

## Local Setup

### Requirements

- Python 3.10+
- `aiohttp`
- `ffprobe` (part of ffmpeg)

### Run

```bash
pip install -r requirements.txt
python3 server.py
```

### Options

```
--port PORT              Server port (default: 8765)
--host HOST              Bind address (default: 0.0.0.0)
--check-interval SECS    Health check interval (default: 180)
```

## Endpoints

| Endpoint | Description |
|----------|-------------|
| `/` | Web UI with status dashboard |
| `/playlist.m3u` | M3U playlist (only alive channels) |
| `/stream/{channel_id}` | HLS proxy for a specific channel |
| `/proxy/seg.m3u8` | Proxied playlist segments |
| `/proxy/seg.ts` | Proxied video segments |
| `/api/channels` | JSON API with all channels and status |
| `/manifest.json` | Stremio addon manifest |
| `/catalog/tv/all.json` | Stremio catalog (all alive channels) |
| `/catalog/tv/all/genre={Genre}.json` | Stremio catalog filtered by genre |
| `/meta/tv/{id}.json` | Stremio channel metadata |
| `/stream/tv/{id}.json` | Stremio stream info |
| `/poster/{name}.svg` | Dynamic SVG poster for Stremio catalog |

## Categories

Channels are organized into groups for IPTV player navigation:

- `Live: Soccer`, `Live: Baseball`, etc. — active event streams
- `Sports: Tennis`, `Sports: Basketball`, etc. — sport-specific 24/7 networks
- `Sports` — general sports networks (ESPN, Sky Sports, beIN, DAZN, etc.)
- `TV: United States`, `TV: Germany`, etc. — TV channels by country

## Stremio

Install as a Stremio addon from the dashboard or manually:

```
http://<server-ip>:8765/manifest.json
```

Each channel gets a dynamically generated SVG poster in the Stremio catalog.

## Apple TV

Use the [UHF](https://apps.apple.com/app/uhf-iptv-player/id6502846354) app. Add the playlist URL:

```
http://<server-ip>:8765/playlist.m3u
```
