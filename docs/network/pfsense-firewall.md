# Firewall — pfSense CE
# blerdmh Lab | Glovary i3-N355 6-port | Management: 10.0.10.1

## Overview

Firewall/router: **pfSense CE** (Community Edition)
Hardware: Glovary i3-N355 6-port mini firewall appliance
Management UI: https://10.0.10.1 (VLAN 10 only)
Automation: `pfsensible.core` Ansible collection

---

## Interface Map

| pfSense Interface | Physical Port | Role |
|-------------------|--------------|------|
| igb0 | Port 1 | WAN (ISP uplink) |
| igb1 | Port 2 | LAN trunk → MokerLink Port 1 |
| igb1.10 | VLAN tag | MGMT — 10.0.10.0/24 — GW 10.0.10.1 |
| igb1.20 | VLAN tag | SERVERS — 10.0.20.0/24 — GW 10.0.20.1 |
| igb1.30 | VLAN tag | CEPH_PUB — 10.0.30.0/24 — GW 10.0.30.1 |
| igb1.31 | VLAN tag | CEPH_CLUSTER — 10.0.31.0/24 — GW 10.0.31.1 |
| igb1.40 | VLAN tag | SECURITY — 10.0.40.0/24 — GW 10.0.40.1 |
| igb1.50 | VLAN tag | IOT_HONEYPOT — 10.0.50.0/24 — GW 10.0.50.1 |
| igb1.60 | VLAN tag | PRODUCTION — 10.0.60.0/24 — GW 10.0.60.1 |
| igb1.61 | VLAN tag | RED_TEAM — 10.0.61.0/24 — GW 10.0.61.1 |
| igb1.62 | VLAN tag | PURPLE_LAB — 10.0.62.0/24 — GW 10.0.62.1 |
| igb1.70 | VLAN tag | GUEST_DMZ — 10.0.70.0/24 — GW 10.0.70.1 |

---

## Firewall Aliases

| Alias | Type | Value | Purpose |
|-------|------|-------|---------|
| RFC1918 | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges |
| MGMT_HOSTS | Host | 10.0.10.10 | Skynet workstation (admin) |
| K3S_NODES | Host | 10.0.20.10–.13 | k3s cluster nodes |
| SECURITY_VMS | Host | 10.0.40.10–.13 | Wazuh/IDS/OpenVAS/Grafana |
| NAS_HOST | Host | 10.0.20.5 | ZimaBoard TrueNAS |
| DNS_SERVERS | Host | 10.0.10.1, 10.0.10.3 | pfSense Unbound + secondary |
| PROXMOX_NODES | Host | 10.0.10.20, 10.0.10.21 | AMDPVE + N1 Mini |
| TAILSCALE_SUBNET | Network | 100.64.0.0/10 | Tailscale CGNAT |

> **Note:** K3S_NODES and SECURITY_VMS are `Host` type aliases (individual IPs),
> not `Network` type. This is intentional — they contain specific host addresses,
> not CIDR ranges. Using `Network` type for host IPs technically works but creates
> audit confusion as the ruleset grows.

---

## Inter-VLAN Policy Summary

| Source VLAN | Destination | Action | Notes |
|-------------|-------------|--------|-------|
| MGMT (MGMT_HOSTS) | Any | ✅ Allow | Admin workstation full access |
| MGMT (other hosts) | RFC1918 | 🚫 Block | Non-admin MGMT hosts isolated |
| SERVERS (K3S_NODES) | NAS_HOST :2049 | ✅ Allow | NFS mounts |
| SERVERS (K3S_NODES) | Internet | ✅ Allow | Image pulls, updates |
| SERVERS (PROXMOX) | NAS_HOST | ✅ Allow | PBS backups |
| SERVERS | RFC1918 | 🚫 Block | Default deny lateral |
| SECURITY | RFC1918 :514/1514 | ✅ Allow | Wazuh log ingest |
| SECURITY | Internet | ✅ Allow | Threat intel feeds |
| SECURITY | RFC1918 (other) | 🚫 Block | Security VMs don't initiate |
| IOT_HONEYPOT | RFC1918 | 🚫 Block | Strict isolation |
| IOT_HONEYPOT | Internet | ✅ Allow | IoT cloud calls only |
| PRODUCTION | K3S_NODES :80/443 | ✅ Allow | Access k3s services |
| PRODUCTION | RFC1918 | 🚫 Block | Default deny lateral |
| RED_TEAM | RFC1918 | 🚫 Block | Fully isolated |
| RED_TEAM | Internet | ✅ Allow | Tool downloads (monitor closely) |
| PURPLE_LAB | SECURITY_VMS :1514 | ✅ Allow | Log shipping to Wazuh |
| PURPLE_LAB | RFC1918 | 🚫 Block | Isolated except log shipping |
| GUEST_DMZ | RFC1918 | 🚫 Block | Guests cannot reach internal |
| GUEST_DMZ | Internet | ✅ Allow | Guest internet only |

---

## Automation

All of the above is managed via Ansible using the `pfsensible.core` collection.

```bash
# Install collection (run once on Skynet)
ansible-galaxy collection install pfsensible.core

# Apply full pfSense config
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/06-pfsense.yml

# Apply only firewall rules
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/06-pfsense.yml --tags rules

# Apply only DHCP reservations
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/06-pfsense.yml --tags dhcp
```

Before running, ensure vault.yml contains:
- `vault_pfsense_password`
- `vault_skynet_mac`, `vault_amdpve_mac`, `vault_n1mini_mac`
- `vault_zimaboard_mac`, `vault_pi4_mac`, `vault_pi5_mac`

---

## pfSense Backup

pfSense config is XML-based. Export and commit after any manual change:

1. Diagnostics → Backup & Restore → Download configuration as XML
2. Save to: `switch/pfsense-backup-YYYY-MM-DD.xml`
3. Commit to repo (XML contains no plaintext passwords — they are hashed)

> Treat the XML backup as a recovery artifact, not the source of truth.
> The Ansible playbooks are the source of truth.
