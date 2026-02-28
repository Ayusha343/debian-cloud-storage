# Debian Cloud Storage Server

This repository documents the setup and configuration of a Debian-based cloud storage server built entirely with free software and recycled hardware.

## Key Features

- **Free & Open-Source**: Debian, Nextcloud, Apache, MariaDB, Redis
- **Low-Cost Hardware**: Recycled PC + second-hand drives (~₹5,000)
- **24/7 Operation**: Always-on cloud storage with UPS backup
- **Multi-User Support**: Individual family member accounts with separate quotas
- **Auto Photo Backup**: Nextcloud Android app for automatic DCIM uploads
- **Global Access**: Cloudflare Tunnel for secure, port-forward-free Internet access
- **Weekly Backups**: Automated rsync to cold backup drive (Saturdays 02:00)
- **High Availability**: MergerFS pool combining 2 TB of storage

## Server Details

| Element | Value |
|---------|-------|
| **Domain** | storagepaglu.pp.ua |
| **Local IP** | 192.168.1.12 |
| **SSH User** | homeserver |
| **SSH Port** | 2222 |
| **OS** | Debian 12.6 Bookworm |
| **CPU** | Intel Core 2 Duo |
| **RAM** | 4 GB DDR2 |
| **Storage** | 1.5 TB (MergerFS) + 1 TB Backup |

## Quick Start

### Local Network Access
```
http://192.168.1.12/nextcloud
```

### Global Internet Access
```
https://storagepaglu.pp.ua/nextcloud
```

### SSH Remote Access
```bash
ssh -p 2222 homeserver@192.168.1.12
```

### Web Management (Cockpit)
```
https://192.168.1.12:9090  # Start service first
```

## Table of Contents

- [Hardware.md](./Hardware.md) – Specs, procurement, storage layout
- [Commands/installing-OS.md](./Commands/installing-OS.md) – Debian 12.6 installation steps
- [Commands/updating-network-driver.md](./Commands/updating-network-driver.md) – Network troubleshooting & WiFi driver setup
- [Docs/Server-guide.md](./Docs/Server-guide.md) – Complete technical reference
- [Docs/Project-report.md](./Docs/Project-report.md) – Full project documentation with cost analysis

## Common Commands

```bash
# Check storage
df -h

# Nextcloud file scan
sudo -u homeserver php /srv/data/nextcloud/occ files:scan homeserver

# Manual backup
sudo systemctl start run-backup.service

# View system health
journalctl -p err -n 20
```

## Services Running

| Service | Port | Auto-Start |
|---------|------|-----------|
| Apache2 (Nextcloud) | 80/443 | ✓ |
| MariaDB | 3306 | ✓ |
| Redis Cache | 6379 | ✓ |
| SSH | 2222 | ✓ |
| Cloudflared Tunnel | - | ✓ |
| Cockpit (optional) | 9090 | ○ Manual |

## Troubleshooting

For detailed troubleshooting, network diagnostics, and advanced commands, see [Docs/Server-guide.md](./Commands/Server-guide.md).

## License

Open-source licensed. Individual components (Debian, Nextcloud, Apache, MariaDB, Redis) retain their respective licenses (GPL, AGPL, Apache 2.0, etc.).
