# Architecture Overview

## Design Philosophy

**IAC-first** — Nothing runs until it is in this repository. Every VM, LXC, firewall rule, Kubernetes manifest, and service configuration is expressed as code before it is applied.

**Security-first segmentation** — All inter-VLAN traffic is default-deny at the pfSense layer. Explicit allow rules exist only for documented, justified flows.

**Visibility everywhere** — Every host ships logs to Wazuh. Every network interface is monitored by Suricata/Zeek. Nothing runs dark.

---

## Physical Topology

```
ISP
 │
 └── pfSense (Glovary i3-N355)
      igc0 — WAN
      igc1 — Trunk → MokerLink POE-G0800GM
               │
               ├── Port 1  pfSense trunk uplink
               ├── Port 2  AMDPVE        (VLAN trunk: 10,20,30,31,40,60)
               ├── Port 3  N1 Mini PVE   (VLAN trunk: 10,20,30,31,60)
               ├── Port 4  ZimaBoard     (VLAN trunk: 20,30)
               ├── Port 5  TP-Link EAP720 AP (VLAN trunk: 10,60,70)
               ├── Port 6  Skynet        (VLAN 10 access)
               ├── Port 7  Pi 4          (VLAN 20 access)
               └── Port 8  Pi 5          (VLAN 20 access)
```

---

## VLAN Architecture

| VLAN | Name | Subnet | Gateway | Purpose |
|------|------|--------|---------|---------|
| 10 | MGMT | 10.0.10.0/24 | 10.0.10.1 | Proxmox, switch, workstation, Gitea LXC |
| 20 | SERVERS | 10.0.20.0/24 | 10.0.20.1 | k3s cluster, TrueNAS, PBS LXC |
| 30 | CEPH_PUB | 10.0.30.0/24 | 10.0.30.1 | Ceph public network (future) |
| 31 | CEPH_CLUSTER | 10.0.31.0/24 | 10.0.31.1 | Ceph replication (future) |
| 40 | SECURITY | 10.0.40.0/24 | 10.0.40.1 | Wazuh, IDS, OpenVAS |
| 50 | IOT_HONEYPOT | 10.0.50.0/24 | 10.0.50.1 | IoT devices, honeypots — isolated |
| 60 | PRODUCTION | 10.0.60.0/24 | 10.0.60.1 | k3s workloads and personal services |
| 61 | RED_TEAM | 10.0.61.0/24 | 10.0.61.1 | Offensive tooling — fully isolated |
| 62 | PURPLE_LAB | 10.0.62.0/24 | 10.0.62.1 | Attack/defense exercises |
| 70 | GUEST_DMZ | 10.0.70.0/24 | 10.0.70.1 | Guest WiFi — internet only |

---

## Compute Layer

### AMDPVE — Primary Proxmox Node
- **Hardware:** Ryzen 9 7900X, 128GB DDR5, local NVMe
- **Role:** Hosts all security stack VMs and the k3s GPU worker VM
- **IOMMU:** Enabled — RX 5700 XT passed through to k3s-w2-amd for Jellyfin VAAPI

| VMID | Name | VLAN | IP | Purpose |
|------|------|------|----|---------|
| 200 | k3s-w2-amd | 20 | 10.0.20.12 | k3s worker + AMD GPU passthrough |
| 400 | wazuh-vm | 40 | 10.0.40.10 | Wazuh Manager + OpenSearch + Dashboard |
| 401 | ids-vm | 40 | 10.0.40.11 | Suricata IDS + Zeek NSM |
| 402 | openvas-vm | 40 | 10.0.40.12 | Greenbone/OpenVAS vulnerability scanner |

### N1 Mini PVE — Secondary Proxmox Node
- **Hardware:** Ryzen 7 5825U, 16GB DDR4
- **Role:** Hosts service LXC containers

| VMID | Name | VLAN | IP | Purpose |
|------|------|------|----|---------|
| 302 | gitea | 10 | 10.0.10.50 | Self-hosted Gitea — GitHub mirror |
| 310 | pbs | 20 | 10.0.20.30 | Proxmox Backup Server |

### k3s Cluster
| Node | Hardware | IP | Role |
|------|----------|----|------|
| k3s-cp | Raspberry Pi 4 8GB | 10.0.20.10 | Control plane + Tailscale subnet router |
| k3s-w1 | Raspberry Pi 5 8GB | 10.0.20.11 | Worker node |
| k3s-w2-amd | AMDPVE VM | 10.0.20.12 | Worker node — AMD GPU for Jellyfin VAAPI |

---

## Storage Layer

### TrueNAS Scale — ZimaBoard 832
| Pool | Devices | Type | Purpose |
|------|---------|------|---------|
| `tank` | sdb 14.55 TiB | Single | Primary NAS — media, arr, ISO |
| `backup` | sdc + sdd 5.46 TiB each | Mirror | Proxmox Backup Server destination |
| `fast` | nvme0n1 931.51 GiB | Single | High-speed dataset for Nextcloud, DB |
| `archive` | sda 7.28 TiB | Single | ZFS replication target only (no exports) |

### NFS Exports (to Proxmox + k3s)
| Export | Mount | Consumers |
|--------|-------|-----------|
| `/mnt/backup/proxmox` | `nfs-truenas-backups` | Proxmox Backup Server |
| `/mnt/tank/iso` | `nfs-truenas-iso` | Proxmox ISO store |
| `/mnt/tank/media` | k3s PVCs | Jellyfin, Radarr, Sonarr, Lidarr, etc. |
| `/mnt/tank/arr` | k3s PVCs | ARR stack + downloads |
| `/mnt/fast/nextcloud` | k3s PVC | Nextcloud data |

---

## Security Stack

```
All lab hosts
    │  Wazuh agent (port 1514/1515)
    ▼
wazuh-vm (10.0.40.10)
    Wazuh Manager
    OpenSearch
    Wazuh Dashboard (:443)

ids-vm (10.0.40.11)
    Suricata — signature detection → EVE JSON → Wazuh
    Zeek — deep protocol analysis → conn/dns/http logs → Wazuh

openvas-vm (10.0.40.12)
    Greenbone/OpenVAS — scheduled vulnerability scanning
```

### Detection Coverage
| Technique | MITRE ID | Detection Source |
|-----------|----------|-----------------|
| SSH Brute Force | T1110 | Wazuh rule 100100 |
| Lateral Movement via SSH | T1021.004 | Wazuh rule 100200 |
| Privilege Escalation (sudo) | T1548.003 | Wazuh rule 100300 |
| Account Creation | T1136.001 | Wazuh rule 100400 |
| Scheduled Task Persistence | T1053.003 | Wazuh rule 100402 |
| Network Port Scan | T1046 | Wazuh rule 100500 |

---

## Remote Access

| Method | Use Case |
|--------|----------|
| Tailscale subnet router (Pi 4) | Primary remote access — all VLANs reachable |
| Cloudflare Tunnel | Public-facing services (Nextcloud, etc.) |
| Cloudflare DNS-01 | Wildcard TLS certs for `*.blerdmh.com` via cert-manager |

---

## Service Map

| Service | Namespace | Internal URL | External URL |
|---------|-----------|-------------|-------------|
| Nextcloud | personal | nextcloud.personal.svc | nextcloud.blerdmh.com |
| Jellyfin | media | jellyfin.media.svc | jellyfin.blerdmh.com |
| Home Assistant | homeauto | homeassistant.homeauto.svc | ha.blerdmh.com |
| Grafana | monitoring | grafana.monitoring.svc | grafana.blerdmh.com |
| Wazuh Dashboard | — | https://10.0.40.10 | — |
| Gitea | — | http://10.0.10.50:3000 | — |
| Proxmox AMDPVE | — | https://10.0.10.20:8006 | — |
| TrueNAS | — | https://10.0.20.5 | — |
