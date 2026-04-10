# TrueNAS Scale — Storage Architecture

## Hardware

**Platform:** ZimaBoard 832 (Intel N3350, 8GB RAM)
**OS:** TrueNAS Scale — installed to internal eMMC (`mmcblk1`, 29.12 GiB, boot-pool)
The eMMC is the OS drive only — all SATA/NVMe slots are available for data.

---

## Physical Disks

| Device | Serial | Size | Role |
|--------|--------|------|------|
| sdb | 2BHV2WWN | 14.55 TiB | `tank` pool — primary NAS |
| sda | ZR161K8E | 7.28 TiB | `archive` — ZFS replication target |
| nvme0n1 | 25285182F769 | 931.51 GiB | `fast` pool — NVMe workloads |
| sdc | ZAD8Y181 | 5.46 TiB | `backup` pool — mirror member |
| sdd | ZAD15NFW | 5.46 TiB | `backup` pool — mirror member |
| mmcblk1 | 0x77224766 | 29.12 GiB | boot-pool (OS — not a data pool member) |

---

## Storage Pools

### `tank` — Primary NAS pool

| Device | Capacity | Role |
|--------|----------|------|
| sdb | 14.55 TiB | Single drive |

**Type:** Single drive (no mirror — sda is a different size, not suitable as mirror peer)
**Usable:** ~14.55 TiB
**Recordsize:** 1M for media datasets, 128K for arr/iso
**Compression:** lz4
**Atime:** off

> If a matching 16TB drive is added later, tank can be converted to a mirror with `zpool attach tank sdb <new-disk>` without downtime.

### `backup` — Mirrored backup pool

| Slot | Device | Capacity | Role |
|------|--------|----------|------|
| SATA 3 | sdc (ZAD8Y181) | 5.46 TiB | Mirror member |
| SATA 4 | sdd (ZAD15NFW) | 5.46 TiB | Mirror member |

**Type:** ZFS Mirror (equivalent to RAID-1)
**Usable:** ~5.46 TiB
**Recordsize:** 128K
**Compression:** lz4
**Atime:** off

Backups are on the mirrored pool intentionally — they are the most critical data tier and
warrant redundancy even at the cost of reduced raw capacity.

### `fast` — NVMe pool

| Device | Capacity | Role |
|--------|----------|------|
| nvme0n1 | 931.51 GiB | Single NVMe |

**Type:** Single drive
**Recordsize:** 16K (database/Nextcloud workloads)
**Compression:** lz4

### `archive` — Replication target

| Device | Capacity | Role |
|--------|----------|------|
| sda (ZR161K8E) | 7.28 TiB | Standalone ZFS replication target |

**Important:** `archive` is a replication target ONLY — never used as an NFS export or active
data store. It receives `zfs send` streams from `tank`, `backup`, and `fast`.
This replaces the previous USB external drives (b1/b2) with a more reliable internal SATA drive.

---

## Dataset Structure

```
tank/
├── media/          → NFS export → k3s media PVCs (Jellyfin library)
├── arr/            → NFS export → k3s ARR stack PVCs (downloads, configs)
└── iso/            → NFS export → Proxmox ISO store

backup/
└── proxmox/        → NFS export → Proxmox Backup Server

fast/
└── nextcloud/      → NFS export → Nextcloud data directory

archive/            → ZFS replication target only (no exports)
```

---

## NFS Exports

| Export Path | Consumers | Notes |
|-------------|-----------|-------|
| `/mnt/tank/media` | Jellyfin, Radarr, Sonarr, Lidarr, Bazarr | Read/write for ARR stack |
| `/mnt/tank/arr` | Radarr, Sonarr, Lidarr, SABnzbd, qBittorrent | Downloads landing zone |
| `/mnt/tank/iso` | Proxmox ISO store | ISO and LXC template storage |
| `/mnt/backup/proxmox` | Proxmox Backup Server | Backup destination (mirrored pool) |
| `/mnt/fast/nextcloud` | Nextcloud | User data directory |

### NFS export settings

```
Network: 10.0.20.0/24 (SERVERS VLAN)
Options: vers=4, rw, sync, no_subtree_check
```

---

## Backup Architecture

```
tank / backup / fast datasets
    │
    ├── ZFS snapshots (hourly — auto, keep 24)
    │                (daily  — auto, keep 30)
    │
    ├── ZFS replication → archive (sda, internal SATA, 7.28 TiB)
    │
    └── Proxmox Backup Server (on backup/proxmox via NFS)
         └── VM/LXC backups from AMDPVE + N1 Mini
              └── Backblaze B2 offsite (via Nextcloud external storage)
```

---

## Pool Creation (Manual — One Time)

> Run these commands on the TrueNAS Shell or CLI.
> Verify device names first: `lsblk -o NAME,SIZE,SERIAL,TYPE`

```bash
# tank — single 14.55 TiB drive (primary NAS)
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  tank /dev/sdb

# backup — mirrored pair of matched 5.46 TiB drives
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=128k \
  -O xattr=sa \
  backup mirror /dev/sdc /dev/sdd

# fast — single NVMe
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=16k \
  -O xattr=sa \
  fast /dev/nvme0n1

# archive — replication target only
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  archive /dev/sda

# Create datasets
zfs create -o recordsize=1M tank/media
zfs create -o recordsize=128k tank/arr
zfs create -o recordsize=128k tank/iso
zfs create -o recordsize=128k backup/proxmox
zfs create -o recordsize=16k fast/nextcloud
```

After pool creation, configure NFS shares via the TrueNAS web UI or API.
Pool creation is manual and intentional — a scripting error here destroys data.

> **Future migration:** When a second 16TB drive is available, convert `tank` to a mirror:
> `zpool attach tank sdb /dev/sdX` (no downtime required)
