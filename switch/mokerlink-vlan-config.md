# MokerLink POE-G0800GM — VLAN Configuration
# blerdmh Lab | Switch IP: 10.0.10.2 | Management VLAN: 10

## Overview

8-port L2 managed PoE switch.
All inter-VLAN routing is handled by pfSense — the switch is Layer 2 only.
Management access: http://10.0.10.2 (VLAN 10 only)

---

## VLAN Database

Create all VLANs before configuring ports.
Location in UI: Advanced → 802.1Q VLAN → VLAN Config

| VLAN ID | Name |
|---------|------|
| 10 | MGMT |
| 20 | SERVERS |
| 30 | CEPH_PUB |
| 31 | CEPH_CLUSTER |
| 40 | SECURITY |
| 50 | IOT_HONEYPOT |
| 60 | PRODUCTION |
| 61 | RED_TEAM |
| 62 | PURPLE_LAB |
| 70 | GUEST_DMZ |

---

## Port Configuration

### Port 1 — pfSense igc1 (TRUNK uplink)
**Mode:** Tagged trunk
**Native/PVID:** 1 (default — leave at 1, do NOT change to 10)
**Tagged VLANs:** 10, 20, 30, 31, 40, 50, 60, 61, 62, 70

All VLANs including MGMT (10) are tagged on this port. pfSense's igc1.10
sub-interface handles tagged VLAN 10 and serves 10.0.10.1/24. igc1 itself
has no IP and is a pure trunk parent. Changing Port 1 PVID to 10 will
make VLAN 10 arrive untagged on igc1, bypassing igc1.10 and causing lockout.

### Port 2 — AMDPVE (Proxmox primary)
**Mode:** Tagged trunk
**Native/PVID:** 10
**Tagged VLANs:** 10, 20, 30, 31, 40, 60

AMDPVE needs MGMT for Proxmox UI, SERVERS for VM networking,
CEPH VLANs for cluster storage, SECURITY for security stack VMs,
PRODUCTION for workload VMs.

### Port 3 — N1 Mini PVE (Proxmox secondary)
**Mode:** Tagged trunk
**Native/PVID:** 10
**Tagged VLANs:** 10, 20, 30, 31, 60

N1 Mini hosts Gitea LXC and k3s worker VM. Needs MGMT, SERVERS, CEPH, PRODUCTION.

### Port 4 — ZimaBoard (TrueNAS Scale)
**Mode:** Tagged trunk
**Native/PVID:** 20
**Tagged VLANs:** 20, 30

NAS primary traffic on SERVERS VLAN. CEPH_PUB for potential future Ceph integration.
No MGMT access for NAS — management is via SERVERS VLAN IP.

### Port 5 — TP-Link EAP720 (WiFi 7 AP)
**Mode:** Tagged trunk
**Native/PVID:** 10
**Tagged VLANs:** 10, 60, 70

AP bridges SSIDs to VLANs:
- Management SSID → VLAN 10
- Main home SSID → VLAN 60
- Guest SSID → VLAN 70

### Port 6 — Skynet (workstation)
**Mode:** Access (untagged)
**Native/PVID:** 10
**Tagged VLANs:** none

Skynet is the admin workstation. Untagged VLAN 10 (MGMT).
Skynet has firewall rules permitting full lab access from MGMT_HOSTS alias.

### Port 7 — Raspberry Pi 4 (k3s control plane + Tailscale)
**Mode:** Access (untagged)
**Native/PVID:** 20
**Tagged VLANs:** none

Pi 4 is a Linux host — it doesn't process 802.1Q tags.
Placed directly on SERVERS VLAN 20 as an access port.

### Port 8 — Raspberry Pi 5 (k3s worker)
**Mode:** Access (untagged)
**Native/PVID:** 20
**Tagged VLANs:** none

Same as Pi 4 — untagged SERVERS VLAN 20.

---

## Configuration Sequence (Critical Order)

> ⚠️ Always configure management VLAN migration LAST.
> Changing the management VLAN mid-sequence causes lockout.

1. Create all VLANs in the VLAN database
2. Configure Port 1 (pfSense trunk) — tagged all VLANs
3. Configure Ports 2–5 (trunks for Proxmox, ZimaBoard, AP)
4. Configure Ports 7–8 (access ports, Pi 4 and Pi 5)
5. Configure Port 6 (Skynet — access VLAN 10)
6. **LAST:** Migrate switch management interface to VLAN 10, set IP to 10.0.10.2

If you lose connectivity at step 6, connect a laptop directly to any port
on VLAN 1 (default) with IP 192.168.0.x and access the switch at its
factory default IP to recover.

---

## Post-Configuration Verification

From Skynet (10.0.10.10), verify each VLAN gateway is reachable:

```bash
for gw in 10.0.10.1 10.0.20.1 10.0.30.1 10.0.40.1 10.0.60.1; do
  echo -n "Pinging $gw: "
  ping -c 1 -W 1 $gw > /dev/null 2>&1 && echo "OK" || echo "FAIL"
done
```

Verify switch management:
```bash
ping -c 2 10.0.10.2 && echo "Switch reachable"
```
