# Homelab

Self-hosted infrastructure running on a headless Linux server on my home network. Primary services: Nextcloud (photos, files, sync), SMB shares for LAN clients, Tailscale for remote access, qBittorrent behind a kill switch.

## Stack

| Layer        | Tool                                                |
| ------------ | --------------------------------------------------- |
| Host OS      | Debian-based, static IP `192.168.0.115`             |
| Containers   | Docker + Docker Compose, Watchtower for auto-update |
| Cloud / sync | Nextcloud (custom image)                            |
| File sharing | Samba (SMB) for Nautilus / Android clients          |
| Remote       | Tailscale mesh VPN with HTTPS certs                 |
| Power        | Wake-on-LAN triggered from phone via Termux         |
| Maintenance  | `unattended-upgrades` for security patches          |

## Hardware

- SFF chassis, repurposed desktop
- Storage: 1 TB + 512 GB + 120 GB (planning migration to ~12 TB across 2–3 drives in a custom 3D-printed NAS case with dual PSU)
- Wired to main switch over Cat6 (17–25 m run, 1 GbE)

## Notable bits

### Custom Nextcloud image

The stock Nextcloud Docker image doesn't ship `ffmpeg` (needed by the Memories app for video previews) and doesn't run cron inside the main container. I build a custom image that:

- Adds `ffmpeg` for video thumbnails and transcoding
- Runs a cron sidecar so background jobs (`php occ` schedules, Preview Generator pre-rendering) actually fire
- Pre-downloads the Recognize ML models so the container is reproducible across rebuilds
- Loads ~660 K geometries for offline reverse geocoding so Memories can show place names on photos without hitting an external API

See [`nextcloud/Dockerfile`](./nextcloud/Dockerfile) and [`docker-compose.yml`](./docker-compose.yml).

### Wake-on-LAN from phone

Server sleeps when idle. A small Python script in Termux on my phone sends the magic packet over the LAN (or over Tailscale when away):

```python
# wol.py — see scripts/wol.py
send_magic_packet("9c:7b:ef:57:69:dd", ip_address="192.168.0.115")
```

### Tailscale HTTPS

The Nextcloud Memories Android app refuses to talk to a server with a self-signed cert. Fix: enable Tailscale HTTPS (`tailscale cert`) and point Nextcloud's `trusted_domains` and `overwrite.cli.url` at the `*.ts.net` hostname. The app is happy, and the server stays unreachable from the public internet.

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
│   ├── wol.py                # Termux wake-on-LAN
│   └── backup.sh             # nightly rsync to external drive
└── docs/
    └── debugging-notes.md    # things that broke and how I fixed them
```

## Things that broke (and how I fixed them)

- **Recognize models silently failing to download** — the install script times out behind some networks; mirror them into the image at build time instead of relying on runtime download.
- **Preview Generator running forever on first pass** — schedule it in batches via cron (`occ preview:pre-generate`) instead of one giant job.
- **TLS mismatch on Memories Android app** — switched from self-signed to Tailscale-issued certs.
- **GPU stuck at 30 W on the workstation** (separate machine, but same homelab toolchain) — `nvidia-powerd` was inactive; `sudo systemctl enable --now nvidia-powerd` restored full TGP.

## Roadmap

- [ ] Migrate to ~12 TB storage in a custom 3D-printed NAS case
- [ ] Add Prometheus + Grafana for power / temp / disk monitoring
- [ ] Move qBittorrent behind a Gluetun VPN container with kill switch
- [ ] Off-site encrypted backup (rclone → cheap S3-compatible)
