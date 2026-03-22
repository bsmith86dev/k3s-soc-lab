# TrueNAS Scale — Storage Architecture

## Hardware

**Platform:** ZimaBoard 832 (Intel N3350, 8GB RAM, internal eMMC)
**OS:** TrueNAS Scale — installed to internal eMMC (not a pool member)

---

## Storage Pools

### `tank` — Primary NAS pool

| Slot | Device | Capacity | Role |
|------|--------|----------|------|
| SATA 1 | WD HC550 16TB | 16TB | Mirror member |
| SATA 2 | WD HC550 16TB | 16TB | Mirror member |

**Type:** ZFS Mirror (equivalent to RAID-1)
**Usable:** ~16TB
**Recordsize:** 128K (NFS workloads)
**Compression:** lz4
**Atime:** off

### `fast` — NVMe pool

| Slot | Device | Capacity | Role |
|------|--------|----------|------|
| M.2 Slot 1 | Crucial P3 Plus 2TB | 2TB | Single drive — full x4 bandwidth |
| M.2 Slot 2 | Empty | — | Left empty intentionally |

**Type:** Single drive
**Why not stripe:** The ZimaBoard dual-NVMe adapter splits PCIe 2.0 x4 lanes between slots.
For NAS workloads (small random NFS I/O), a single drive with full x4 bandwidth
outperforms two drives sharing x4 lanes. Slot 2 stays empty.
**Recordsize:** 16K (database/Nextcloud workloads)
**Compression:** lz4

### `b1` / `b2` — USB replication targets

| Name | Device | Capacity | Role |
|------|--------|----------|------|
| b1 | Seagate 8TB USB | 8TB | ZFS replication target |
| b2 | Seagate 6TB USB | 6TB | ZFS replication target |

**Important:** USB drives are replication targets ONLY — never pool members.
TrueNAS has unreliable SMART passthrough over USB, and unexpected disconnects
can corrupt ZFS pools. These exist solely to receive `zfs send` streams.

---

## Dataset Structure

```
tank/
├── media/          → NFS export → k3s media PVCs (Jellyfin library)
├── arr/            → NFS export → k3s ARR stack PVCs (downloads, configs)
├── backups/        → NFS export → Proxmox Backup Server
├── iso/            → NFS export → Proxmox ISO store
└── nextcloud/      → NFS export → Nextcloud data (on fast pool)

fast/
└── nextcloud/      → NFS export → Nextcloud data directory
```

---

## NFS Exports

| Export Path | Consumers | Notes |
|-------------|-----------|-------|
| `/mnt/tank/media` | Jellyfin, Radarr, Sonarr, Lidarr, Bazarr | Read/write for ARR stack |
| `/mnt/tank/arr` | Radarr, Sonarr, Lidarr, SABnzbd, qBittorrent | Downloads landing zone |
| `/mnt/tank/backups` | Proxmox Backup Server | Backup destination |
| `/mnt/tank/iso` | Proxmox ISO store | ISO and LXC template storage |
| `/mnt/fast/nextcloud` | Nextcloud | User data directory |

### NFS export settings

```
Network: 10.0.20.0/24 (SERVERS VLAN)
Options: vers=4, rw, sync, no_subtree_check
```

---

## Backup Architecture

```
TrueNAS tank datasets
    │
    ├── ZFS snapshots (hourly — auto, keep 24)
    │                (daily  — auto, keep 30)
    │
    ├── ZFS replication → b1 (Seagate 8TB USB)
    │                   → b2 (Seagate 6TB USB)
    │
    └── Proxmox Backup Server (on tank/backups via NFS)
         └── VM/LXC backups from AMDPVE + N1 Mini
              └── Backblaze B2 offsite (via Nextcloud external storage)
```

---

## Pool Creation (Manual — One Time)

> ⚠️ Run these commands on the ZimaBoard TrueNAS CLI or Shell.
> Verify disk device names first: `lsblk`

```bash
# tank — mirrored pair
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=128k \
  -O xattr=sa \
  tank mirror /dev/sda /dev/sdb

# fast — single NVMe (Slot 1 only)
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=16k \
  -O xattr=sa \
  fast /dev/nvme0n1

# Create datasets
zfs create tank/media
zfs create tank/arr
zfs create tank/backups
zfs create tank/iso
zfs create fast/nextcloud
```

After pool creation, configure NFS shares via the TrueNAS web UI or API.
Pool creation is manual and intentional — a scripting error here destroys data.
