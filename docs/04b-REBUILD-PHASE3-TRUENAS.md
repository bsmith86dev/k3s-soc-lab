# Phase 3 — TrueNAS Scale Setup

Execution guide for standing up the ZimaBoard NAS.
Read `04-TRUENAS-STORAGE.md` for the architectural reasoning behind every decision here.

---

## Prerequisites

- ZimaBoard 832 powered off
- Drives physically installed:
  - SATA: sdb (14.55 TiB), sda (7.28 TiB), sdc (5.46 TiB), sdd (5.46 TiB)
  - M.2: nvme0n1 (931.51 GiB NVMe)
- Network cable connected to MokerLink Port 4
- eMMC (mmcblk1, 29.12 GiB) used as TrueNAS OS boot drive

---

## Step 1 — Flash TrueNAS Scale to eMMC

The ZimaBoard has internal eMMC. Flash TrueNAS Scale to it directly
so all SATA ports are available for storage drives.

```bash
# On Skynet — download latest TrueNAS Scale ISO
# https://www.truenas.com/download-truenas-scale/

# Flash to USB installer
sudo dd if=TrueNAS-SCALE-*.iso of=/dev/sdX bs=4M status=progress
sync
```

1. Boot ZimaBoard from USB installer
2. In installer: select eMMC as install target (`/dev/mmcblk1`)
3. Set admin password during install
4. Remove USB, reboot

---

## Step 2 — Initial Network Configuration

On first boot, TrueNAS will show the console menu.

1. Select option 1 — Configure Network Interfaces
2. Assign the NIC to VLAN 20 (SERVERS) — IP `10.0.20.5/24`, gateway `10.0.20.1`
3. Confirm DNS: `10.0.10.1`

Verify from Skynet: `ping 10.0.20.5`
Access web UI: `https://10.0.20.5`

---

## Step 3 — Create Storage Pools

In TrueNAS web UI: **Storage → Create Pool**

Verify disk device assignments first: **Storage → Disks**

Expected disk layout:
| Device | Size | Pool |
|--------|------|------|
| sdb | 14.55 TiB | `tank` |
| sda | 7.28 TiB | `archive` |
| nvme0n1 | 931.51 GiB | `fast` |
| sdc | 5.46 TiB | `backup` mirror |
| sdd | 5.46 TiB | `backup` mirror |
| mmcblk1 | 29.12 GiB | boot-pool (already set) |

### tank (single 14.55 TiB drive — primary NAS)

1. Pool name: `tank`
2. Layout: Stripe (single disk)
3. Add disk: sdb (14.55 TiB)
4. Advanced → ashift: 12
5. Create

> Note: A second 16TB drive can be added later to mirror tank non-destructively.

### backup (mirrored 6TB pair)

1. Pool name: `backup`
2. Layout: Mirror
3. Add disks: sdc and sdd (both 5.46 TiB — matched pair)
4. Advanced → ashift: 12
5. Create

### fast (single NVMe)

1. Pool name: `fast`
2. Layout: Stripe (single disk)
3. Add disk: nvme0n1 (931.51 GiB)
4. Advanced → ashift: 12
5. Create

### archive (replication target)

1. Pool name: `archive`
2. Layout: Stripe (single disk)
3. Add disk: sda (7.28 TiB)
4. Advanced → ashift: 12
5. Create

> `archive` receives ZFS replication streams only — no NFS exports, no active workloads.

---

## Step 4 — Create Datasets

**Storage → tank → Add Dataset** for each:

| Dataset | Recordsize | Sync | Compression |
|---------|-----------|------|-------------|
| `tank/media` | 1M | Standard | lz4 |
| `tank/arr` | 128K | Standard | lz4 |
| `tank/iso` | 128K | Standard | lz4 |

**Storage → backup → Add Dataset:**

| Dataset | Recordsize | Sync | Compression |
|---------|-----------|------|-------------|
| `backup/proxmox` | 128K | Always | lz4 |

**Storage → fast → Add Dataset:**

| Dataset | Recordsize | Sync | Compression |
|---------|-----------|------|-------------|
| `fast/nextcloud` | 16K | Always | lz4 |

---

## Step 5 — Configure NFS Shares

**Shares → NFS → Add** for each export:

| Path | Allowed Networks | Notes |
|------|-----------------|-------|
| `/mnt/tank/media` | `10.0.20.0/24` | Jellyfin + ARR stack |
| `/mnt/tank/arr` | `10.0.20.0/24` | Downloads directory |
| `/mnt/tank/iso` | `10.0.20.0/24` | Proxmox ISO store |
| `/mnt/backup/proxmox` | `10.0.20.0/24` | Proxmox Backup Server |
| `/mnt/fast/nextcloud` | `10.0.20.0/24` | Nextcloud data |

Enable NFS service: **System → Services → NFS → Start Automatically → ON**

---

## Step 6 — Configure ZFS Snapshots

**Data Protection → Periodic Snapshot Tasks → Add**

| Dataset | Schedule | Keep | Purpose |
|---------|----------|------|---------|
| `tank` (recursive) | Hourly | 24 | Short-term recovery |
| `tank` (recursive) | Daily | 30 | Medium-term recovery |
| `backup` (recursive) | Daily | 30 | Backup pool snapshots |
| `fast` (recursive) | Daily | 30 | Nextcloud data |

---

## Step 7 — Configure ZFS Replication to archive

**Data Protection → Replication Tasks → Add**

| Source | Destination | Schedule | Notes |
|--------|-------------|----------|-------|
| `tank` (recursive) | `archive/tank` | Daily 02:00 | Primary NAS to internal replication target |
| `backup` (recursive) | `archive/backup` | Daily 03:00 | Backup pool to archive |
| `fast` (recursive) | `archive/fast` | Daily 04:00 | Nextcloud to archive |

The `archive` pool on sda (7.28 TiB) replaces the previous USB external drives (b1/b2).
Create the destination datasets on `archive` before setting up replication tasks:
- `archive/tank`
- `archive/backup`
- `archive/fast`

---

## Step 8 — Generate API Key

**System → API Keys → Add**

Name: `ansible-automation`
Copy the key → add to `vault.yml` as `vault_truenas_api_key`

---

## Verification

From Skynet, verify NFS is working:

```bash
# Check what's exported
showmount -e 10.0.20.5

# Test mount tank
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs 10.0.20.5:/mnt/tank/media /mnt/test-nfs
df -h /mnt/test-nfs
sudo umount /mnt/test-nfs

# Test mount backup
sudo mount -t nfs 10.0.20.5:/mnt/backup/proxmox /mnt/test-nfs
df -h /mnt/test-nfs
sudo umount /mnt/test-nfs
```

From AMDPVE, verify Proxmox can see NFS storage:
```bash
pvesm status
# Should show nfs-truenas-backups and nfs-truenas-iso as active
```
