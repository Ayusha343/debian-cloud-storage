# Hardware Specification

## Overview
A repurposed early-2000s desktop PC from postal office clearance sale, upgraded with additional RAM and hard drives for cloud storage duties.

## System Specifications

| Component | Specification | Source |
|-----------|--------------|--------|
| **CPU** | Intel Core 2 Duo (early generation) | Facebook Marketplace (₹3,500 bundle) |
| **RAM** | 4 GB DDR2 (2×2 GB sticks) | ₹150 (2 sticks) + original 2×1 GB |
| **System Drive** | 128 GB SSD | Included in PC bundle |
| **Live Storage - Drive 1** | 1 TB Toshiba 2.5" HDD | Facebook Marketplace (₹300) |
| **Live Storage - Drive 2** | 500 GB 3.5" HDD | Included in PC |
| **Backup Drive** | 1 TB (Cold backup, weekly rotation) | Facebook Marketplace (₹300)|
| **Wi-Fi Adapter** | TP-Link TL-WN725N (RTL8188EUS) | Separate purchase (₹600) |
| **PSU** | 350W ATX | Included in PC bundle |
| **UPS** | Standard UPS unit | Included in PC bundle |
| **Accessories** | Monitor, Keyboard, Mouse | Included in PC bundle |

## Storage Layout

```
┌─────────────────────────────────────┐
│   128 GB SSD (/dev/sda)             │
│   ├─ 100 GB / (root)                │
│   ├─ 3 GB [SWAP]                    │
│   └─ 23 GB /home                    │
└─────────────────────────────────────┘
         ↓
  Debian 12.6 Bookworm
  Apache2, PHP, MariaDB
  Redis, Nextcloud
         ↓
┌─────────────────────────────────────┐
│   MergerFS Pool (/srv/data)         │
│   ├─ 1 TB HDD A (DATA1)             │
│   ├─ 500 GB HDD B (DATA2)           │
│   └─ Total: ~1.4 TB usable          │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│   Cold Backup (/srv/backup)         │
│   └─ 1 TB HDD C (BACKUP1)           │
│      (Mounted weekly for rsync)     │
└─────────────────────────────────────┘
```

## Hard Drive Details

### System Drive (128 GB SSD)
- **Model**: Included in bundle
- **Mount Point**: /
- **Purpose**: OS, applications, databases, cache
- **Partition Scheme**:
  - `/` (root): 100 GB ext4
  - `[SWAP]`: 3 GB
  - `/home`: 23 GB ext4

### Live Storage Drives
- **DATA1**: 1 TB Toshiba 2.5" HDD
  - **Label**: DATA1
  - **UUID**: [Run `blkid` to confirm]
  - **Mount Point**: /srv/dataA
  
- **DATA2**: 500 GB 3.5" HDD
  - **Label**: DATA2
  - **UUID**: [Run `blkid` to confirm]
  - **Mount Point**: /srv/dataB

- **Pool**: Both drives merged via MergerFS at `/srv/data`

### Backup Drive
- **BACKUP1**: 1 TB 3.5" HDD (cold backup)
- **Label**: BACKUP1
- **Mount Point**: /srv/backup (mounted only during weekly backup)
- **Schedule**: Saturdays at 02:00 (SystemD timer)

## Network Interfaces

### Primary: Wi-Fi Adapter
- **Model**: TP-Link TL-WN725N
- **Chipset**: Realtek RTL8188EUS
- **USB ID**: 0bda:8179
- **Driver**: rtl8188eus (DKMS from aircrack-ng)
- **Speed**: N150 (2.4 GHz, single-stream)
- **Throughput**: 25–60 Mb/s typical
- **USB Speed**: 480M (USB 2.0)

### Secondary (Not in Use): Wired Ethernet
- **Model**: Integrated Realtek r8169/RTL8105e
- **Status**: Disabled (unstable, flaps between 100/10 Mb/s)
- **Reason for Disabling**: PHY-level compatibility issue with router

## Power Management

- **PSU**: 350W ATX (adequate for system + 3 HDDs)
- **UPS**: Standard unit (enables graceful shutdown during power loss)
- **Current Draw**: ~80–120W under load (estimated)

## Procurement Cost

| Item | Cost |
|------|------|
| PC Bundle (Facebook Marketplace) | ₹3,500 |
| 2× 1 TB Hard Drives | ₹1200 |
| DDR2 2 GB RAM (2 sticks) | ₹300 |
| TP-Link TL-WN725N Wi-Fi Adapter | ₹600 |
| Domain (storagepaglu.pp.ua, 1 year) | ₹120 |
| Miscellaneous (cables, screws) | ₹0 |
| **Total** | **~₹6,000** |

## Performance Notes

- **CPU**: Adequate for Nextcloud with 4–8 concurrent users on local network
- **RAM**: 4 GB with Redis caching demonstrates solid performance; swap usage is minimal
- **Storage**: MergerFS pool shows ~1.8 TB usable after formatting overhead
- **Network**: Wi-Fi provides 25–60 Mb/s, sufficient for family file sync and photo backup
- **Power**: Runs continuously 24/7 with stable thermal operation

## Known Issues & Resolutions

1. **Ethernet flapping**: Realtek NIC incompatibility with router; replaced with USB Wi-Fi adapter
2. **Initial HDD failure**: One purchased drive failed; replaced by seller at half price
3. **USB speed fallback**: First USB port negotiated at USB 1.1; moved to rear port for proper USB 2.0 operation

See [updating-network-driver.md](../Commands/updating-network-driver.md) for detailed network troubleshooting.
