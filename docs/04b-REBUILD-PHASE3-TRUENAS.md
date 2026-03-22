# Phase 3 — TrueNAS Scale Setup

Execution guide for standing up the ZimaBoard NAS.
Read `04-TRUENAS-STORAGE.md` for the architectural reasoning behind every decision here.

---

## Prerequisites

- ZimaBoard 832 powered off
- Both WD 16TB HDDs physically installed in SATA bays
- Crucial P3 Plus 2TB NVMe in M.2 Slot 1 only (Slot 2 empty)
- USB externals connected (Seagate 8TB + 6TB)
- Network cable connected to MokerLink Port 4

---

## Step 1 — Flash TrueNAS Scale to eMMC

The ZimaBoard has 32GB internal eMMC. Flash TrueNAS Scale to it directly
so the SATA ports are fully available for storage drives.

```bash
# On Skynet — download latest TrueNAS Scale ISO
# https://www.truenas.com/download-truenas-scale/

# Flash to USB installer
sudo dd if=TrueNAS-SCALE-*.iso of=/dev/sdX bs=4M status=progress
sync
```

1. Boot ZimaBoard from USB installer
2. In installer: select eMMC as install target (`/dev/mmcblk0`)
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

### tank (mirrored HDDs)

1. Pool name: `tank`
2. Layout: Mirror
3. Add disks: both WD 16TB drives
4. Advanced → ashift: 12
5. Create

### fast (single NVMe)

1. Pool name: `fast`
2. Layout: Stripe (single disk)
3. Add disk: Crucial P3 Plus NVMe (Slot 1)
4. Advanced → ashift: 12
5. Create

> **Do not add the Slot 2 NVMe** — leave it empty per the architecture doc.

---

## Step 4 — Create Datasets

**Storage → tank → Add Dataset** for each:

| Dataset | Recordsize | Sync | Compression |
|---------|-----------|------|-------------|
| `tank/media` | 1M | Standard | lz4 |
| `tank/arr` | 128K | Standard | lz4 |
| `tank/backups` | 128K | Always | lz4 |
| `tank/iso` | 128K | Standard | lz4 |

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
| `/mnt/tank/backups` | `10.0.20.0/24` | Proxmox Backup Server |
| `/mnt/tank/iso` | `10.0.20.0/24` | Proxmox ISO store |
| `/mnt/fast/nextcloud` | `10.0.20.0/24` | Nextcloud data |

Enable NFS service: **System → Services → NFS → Start Automatically → ON**

---

## Step 6 — Configure ZFS Snapshots

**Data Protection → Periodic Snapshot Tasks → Add**

| Dataset | Schedule | Keep | Purpose |
|---------|----------|------|---------|
| `tank` (recursive) | Hourly | 24 | Short-term recovery |
| `tank` (recursive) | Daily | 30 | Medium-term recovery |
| `fast` (recursive) | Daily | 30 | Nextcloud data |

---

## Step 7 — Configure Replication to USB Drives

**Data Protection → Replication Tasks → Add**

| Source | Destination | Schedule |
|--------|-------------|----------|
| `tank` | `b1` (Seagate 8TB) | Daily 02:00 |
| `fast` | `b2` (Seagate 6TB) | Daily 03:00 |

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

# Test mount
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs 10.0.20.5:/mnt/tank/media /mnt/test-nfs
df -h /mnt/test-nfs
sudo umount /mnt/test-nfs
```

From AMDPVE, verify Proxmox can see NFS storage:
```bash
pvesm status
# Should show nfs-truenas-backups and nfs-truenas-iso as active
```
