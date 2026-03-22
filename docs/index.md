# blerdmh Lab

Enterprise-grade cybersecurity home lab — infrastructure as code, detection engineering, and portfolio projects.

## What This Is

This lab runs two workloads in parallel on shared hardware:

**Cybersecurity platform** — SIEM, IDS/NSM, vulnerability scanning, red/blue team VLANs, and detection engineering practice targeting SOC Analyst and Security Engineer roles.

**Self-hosted infrastructure** — personal cloud, media server, home automation, and supporting services managed entirely through Ansible and Kubernetes manifests.

Everything is IAC-first. Nothing runs until it exists in this repo.

---

## Quick Links

| | |
|---|---|
| [Architecture Overview](00-ARCHITECTURE.md) | Full lab design — hardware, VLANs, services |
| [Rebuild Guide](02-REBUILD-GUIDE.md) | Phased rebuild sequence from bare metal |
| [pfSense Firewall](network/pfsense-firewall.md) | Interface map, aliases, inter-VLAN rules |
| [TrueNAS Storage](04-TRUENAS-STORAGE.md) | ZimaBoard pool design and NFS exports |
| [CI/CD Pipeline](cicd/overview.md) | GitHub Actions + Tailscale deploy workflow |
| [Secrets Management](cicd/secrets.md) | Vault + Sealed Secrets strategy |
| [Portfolio Projects](portfolio-projects.md) | Five employer-facing security projects |

---

## Hardware at a Glance

| Host | Hardware | Role |
|------|----------|------|
| AMDPVE | Ryzen 9 7900X / 128GB DDR5 | Primary Proxmox — security VMs, GPU worker |
| N1 Mini PVE | Ryzen 7 5825U / 16GB DDR4 | Secondary Proxmox — Gitea, PBS |
| ZimaBoard 832 | Intel N3350 / 8GB | TrueNAS Scale NAS |
| Raspberry Pi 4 8GB | ARM Cortex-A72 | k3s control plane + Tailscale subnet router |
| Raspberry Pi 5 8GB | ARM Cortex-A76 | k3s worker node |
| Skynet | Ryzen 9 9950X3D / RTX 5090 | Workstation — Ansible control node |

---

## VLAN Summary

| VLAN | Name | Subnet |
|------|------|--------|
| 10 | MGMT | 10.0.10.0/24 |
| 20 | SERVERS | 10.0.20.0/24 |
| 40 | SECURITY | 10.0.40.0/24 |
| 50 | IOT_HONEYPOT | 10.0.50.0/24 |
| 60 | PRODUCTION | 10.0.60.0/24 |
| 61 | RED_TEAM | 10.0.61.0/24 |
| 62 | PURPLE_LAB | 10.0.62.0/24 |
| 70 | GUEST_DMZ | 10.0.70.0/24 |
