# Homelab

Self-hosted infrastructure running on a headless Linux server on my home network. Primary services: Nextcloud (photos, files, sync), SMB shares for LAN clients, Tailscale for remote access, qBittorrent behind a kill switch.

## Stack

| Layer        | Tool                                                |
| ------------ | --------------------------------------------------- |
| Host OS      | Ubuntu server, static IP `192.168.0.115`            |
| Containers   | Docker + Docker Compose, Watchtower for auto-update |
| Cloud / sync | Nextcloud (custom image)                            |
| File sharing | Samba (SMB) for Nautilus / Android clients          |
| Remote       | Tailscale mesh VPN                                  |
| Power        | Wake-on-LAN from phone (Android WoL app, LAN only)  |
| Maintenance  | `unattended-upgrades` for security patches          |

## Hardware

- HP slim desktop
- Storage: 1Tb hard disk
- Wired to main router over Cat6

## Notable bits

### Custom Nextcloud image

The stock Nextcloud Docker image doesn't ship `ffmpeg` (needed by the Memories app for video previews) and doesn't run cron inside the main container. I build a custom image that:

- Adds `ffmpeg` for video thumbnails and transcoding
- Runs a cron sidecar so background jobs (`php occ` schedules, Preview Generator pre-rendering) actually fire
- Pre-downloads the Recognize ML models so the container is reproducible across rebuilds
- Loads ~660 K geometries for offline reverse geocoding so Memories can show place names on photos without hitting an external API

See [`nextcloud/Dockerfile`](./nextcloud/Dockerfile) and [`docker-compose.yml`](./docker-compose.yml).

### Wake-on-LAN from phone

Server sleeps when idle. I wake it from my phone over LAN using a standard WoL Android app — works fine, just slower than I'd like to enumerate and send the packet. No remote wake when I'm off-network since I don't run an always-on device to relay the magic packet; Tailscale handles access once the server is already up.

## Layout

```
.
├── docker-compose.yml        # all services
├── nextcloud/
│   ├── Dockerfile            # ffmpeg + cron + Recognize
│   └── config/               # config.php overrides
├── samba/
│   └── smb.conf
├── scripts/
│   └── backup.sh             # nightly rsync to external drive
└── docs/
    └── debugging-notes.md    # things that broke and how I fixed them
```

## Things that broke (and how I fixed them)

- **Recognize models silently failing to download** — the install script times out behind some networks; mirror them into the image at build time instead of relying on runtime download.
- **Preview Generator running forever on first pass** — schedule it in batches via cron (`occ preview:pre-generate`) instead of one giant job.
