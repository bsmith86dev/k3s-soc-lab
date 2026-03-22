# blerdmh-lab

Enterprise-grade cybersecurity home lab — infrastructure as code, security tooling, and portfolio projects.

## Purpose

This lab serves two goals running in parallel:

1. **Self-hosted infrastructure platform** — personal cloud, media, home automation, and supporting services managed entirely through code
2. **Cybersecurity skills lab** — realistic detection, threat simulation, and incident response practice targeting SOC Analyst, Security Engineer, and Penetration Tester roles

## Key Features

- Infrastructure as Code (Ansible-driven deployment)
- Enterprise VLAN segmentation (10+ networks)
- Full SOC stack (Wazuh, Suricata, Zeek, OpenVAS)
- Kubernetes platform (k3s with secure workloads)
- CI/CD pipelines (GitHub Actions)
- Observability stack (Prometheus, Grafana, Loki)

## Hardware

| Host | Hardware | Role |
|------|----------|------|
| AMDPVE | Ryzen 9 7900X / 128GB DDR5 | Primary Proxmox node — security stack VMs, GPU worker |
| N1 Mini PVE | Ryzen 7 5825U / 16GB DDR4 | Secondary Proxmox node — Gitea LXC, PBS LXC |
| ZimaBoard 832 | Intel N3350 / 8GB | TrueNAS Scale NAS |
| Raspberry Pi 4 8GB | ARM Cortex-A72 | k3s control plane + Tailscale subnet router |
| Raspberry Pi 5 8GB | ARM Cortex-A76 | k3s worker node |
| Skynet | Ryzen 9 9950X3D / RTX 5090 | Workstation — Ansible control node |
| Glovary i3-N355 | 6-port mini PC | pfSense CE firewall/router |
| MokerLink POE-G0800GM | 8-port managed PoE | L2 switch |

## Network

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | MGMT | 10.0.10.0/24 | Management — Proxmox, switch, workstation |
| 20 | SERVERS | 10.0.20.0/24 | k3s cluster, TrueNAS |
| 30 | CEPH_PUB | 10.0.30.0/24 | Ceph public network |
| 31 | CEPH_CLUSTER | 10.0.31.0/24 | Ceph replication network |
| 40 | SECURITY | 10.0.40.0/24 | Wazuh, IDS/IPS, OpenVAS |
| 50 | IOT_HONEYPOT | 10.0.50.0/24 | IoT devices, honeypots — isolated |
| 60 | PRODUCTION | 10.0.60.0/24 | k3s workloads, personal services |
| 61 | RED_TEAM | 10.0.61.0/24 | Attack tooling — fully isolated |
| 62 | PURPLE_LAB | 10.0.62.0/24 | Attack/defense exercises |
| 70 | GUEST_DMZ | 10.0.70.0/24 | Guest WiFi — internet only |

## Stack

**Virtualization:** Proxmox VE (clustered), TrueNAS Scale
**Orchestration:** k3s (Raspberry Pi control plane + VM workers)
**Firewall/Routing:** pfSense CE
**IaC/Automation:** Ansible, GitHub Actions, Sealed Secrets
**Remote Access:** Tailscale (subnet router), Cloudflare Tunnel
**SIEM:** Wazuh (Manager + Dashboard + OpenSearch)
**IDS/NSM:** Suricata + Zeek
**Vulnerability Scanning:** Greenbone/OpenVAS
**Monitoring:** Prometheus + Grafana + Loki + Promtail
**Services:** Nextcloud, Jellyfin (AMD GPU VAAPI), Home Assistant, Radarr/Sonarr/Lidarr
**Source Control:** GitHub (primary) + self-hosted Gitea (mirror)
**Documentation:** MkDocs Material → GitHub Pages

## Repository Structure

```
blerdmh-lab/
├── .github/workflows/    # CI lint + CD deploy via Tailscale
├── ansible/
│   ├── inventory/        # Hosts, host_vars, vault.yml
│   ├── playbooks/        # 00-site through 07-gitea
│   └── roles/            # common, k3s, proxmox, pfsense, gitea, wazuh, security
├── k3s/                  # Kubernetes manifests
├── switch/               # pfSense + MokerLink config docs
└── docs/                 # MkDocs source
```

## Architecture Overview

```mermaid
flowchart TB

%% =========================
%% USER / INTERNET LAYER
%% =========================
User[User / Internet]

Cloudflare[Cloudflare Tunnel]
Tailscale[Tailscale VPN]

User --> Cloudflare
User --> Tailscale

%% =========================
%% EDGE NETWORK
%% =========================
Cloudflare --> pfSense
Tailscale --> pfSense

pfSense[pfSense Firewall / Router]

%% =========================
%% VLAN SEGMENTS
%% =========================
subgraph VLAN10["VLAN 10 - MGMT"]
    MGMT[Proxmox UI / Admin Access]
end

subgraph VLAN20["VLAN 20 - SERVERS"]
    K3SCP[k3s Control Plane (Pi4)]
    K3SW1[k3s Worker (Pi5)]
    NAS[TrueNAS Scale]
end

subgraph VLAN40["VLAN 40 - SECURITY"]
    Wazuh[Wazuh SIEM]
    Suricata[Suricata IDS]
    Zeek[Zeek NSM]
    OpenVAS[OpenVAS Scanner]
end

subgraph VLAN50["VLAN 50 - IOT / HONEYPOT"]
    Honeypot[Honeypots / IoT Devices]
end

subgraph VLAN60["VLAN 60 - PRODUCTION"]
    Apps[Nextcloud / Jellyfin / Home Assistant / ARR Stack]
end

subgraph VLAN61["VLAN 61 - RED TEAM"]
    Attacker[Attack VM / Tools]
end

subgraph VLAN62["VLAN 62 - PURPLE LAB"]
    Purple[Attack + Detection Testing]
end

subgraph VLAN70["VLAN 70 - GUEST"]
    Guest[Guest Network]
end

%% =========================
%% INFRASTRUCTURE LAYER
%% =========================
pfSense --> VLAN10
pfSense --> VLAN20
pfSense --> VLAN40
pfSense --> VLAN50
pfSense --> VLAN60
pfSense --> VLAN61
pfSense --> VLAN62
pfSense --> VLAN70

%% =========================
%% PROXMOX CLUSTER
%% =========================
subgraph ProxmoxCluster["Proxmox Cluster"]
    AMDPVE[AMDPVE Node]
    N1Mini[N1 Mini Node]
end

VLAN10 --> ProxmoxCluster

%% =========================
%% K3S WORKLOAD FLOW
%% =========================
K3SCP --> K3SW1
K3SW1 --> Apps

%% =========================
%% STORAGE
%% =========================
NAS --> Apps
NAS --> ProxmoxCluster

%% =========================
%% SECURITY TELEMETRY FLOW
%% =========================
Suricata --> Wazuh
Zeek --> Wazuh
OpenVAS --> Wazuh
Apps --> Wazuh
ProxmoxCluster --> Wazuh

%% =========================
%% ATTACK SIMULATION
%% =========================
Attacker --> VLAN60
Attacker --> VLAN40

%% =========================
%% MONITORING STACK
%% =========================
subgraph Monitoring["Monitoring Stack"]
    Prometheus
    Grafana
    Loki
end

Apps --> Prometheus
ProxmoxCluster --> Prometheus
Wazuh --> Grafana
Loki --> Grafana

## Architecture (Simplified View)

```mermaid
flowchart LR

User --> Edge

subgraph Edge["Secure Access Layer"]
    CF[Cloudflare Tunnel]
    TS[Tailscale VPN]
end

Edge --> Firewall

subgraph Network["pfSense + VLAN Segmentation"]
    Firewall[pfSense Firewall]

    MGMT[Mgmt VLAN]
    PROD[Production VLAN]
    SEC[Security VLAN]
    RED[Red Team VLAN]
end

Firewall --> MGMT
Firewall --> PROD
Firewall --> SEC
Firewall --> RED

subgraph Compute["Compute Layer"]
    Proxmox[Proxmox Cluster]
    K3S[k3s Cluster]
end

MGMT --> Proxmox
PROD --> K3S

subgraph Apps["Applications"]
    Apps1[Nextcloud / Jellyfin / HA]
end

K3S --> Apps1

subgraph Security["Security Stack"]
    Wazuh
    Suricata
    Zeek
    OpenVAS
end

SEC --> Security

subgraph Observability["Monitoring"]
    Grafana
    Prometheus
    Loki
end

Apps1 --> Observability
Security --> Observability
Proxmox --> Observability

## Attack & Detection Flow (Red vs Blue)

```mermaid
flowchart LR

Attacker[Red Team VM] --> Target[Target System (Prod VLAN)]

Target --> Logs[System Logs]
Target --> NetworkTraffic[Network Traffic]

NetworkTraffic --> Suricata[Suricata IDS]
NetworkTraffic --> Zeek[Zeek NSM]

Logs --> Wazuh
Suricata --> Wazuh
Zeek --> Wazuh

Wazuh --> Alert[Alert Triggered]

Alert --> Response[Active Response]
Response --> FirewallBlock[pfSense Block / Isolation]

Alert --> Analyst[SOC Analyst / Dashboard (Grafana)]

## SOC Data Pipeline

```mermaid
flowchart LR

Endpoints[Servers / Apps / Nodes] --> Logs[Logs + Events]

Logs --> Agents[Wazuh Agents]

Agents --> Manager[Wazuh Manager]

Network --> Suricata
Network --> Zeek

Suricata --> Manager
Zeek --> Manager

Manager --> Indexer[OpenSearch Indexer]

Indexer --> Dashboard[Wazuh Dashboard / Grafana]

Dashboard --> Analyst[SOC Analyst]

Manager --> Response[Active Response Engine]

Response --> Firewall[pfSense Actions]

## Quickstart

```bash
# 1. Clone the repo
git clone https://github.com/techinb/blerdmh-lab.git
cd blerdmh-lab

# 2. Install pre-commit hooks
pip install pre-commit detect-secrets
pre-commit install

# 3. Install Ansible collections
ansible-galaxy collection install community.general ansible.posix pfsensible.core

# 4. Copy and fill in vault.yml
cp ansible/inventory/group_vars/all/vault.yml.example \
   ansible/inventory/group_vars/all/vault.yml
# Edit vault.yml with real values, then encrypt:
ansible-vault encrypt ansible/inventory/group_vars/all/vault.yml

# 5. Run baseline across all hosts
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/01-baseline.yml \
  --ask-vault-pass
```

See [docs/02-REBUILD-GUIDE.md](docs/02-REBUILD-GUIDE.md) for the full phased rebuild sequence.

## Portfolio Projects

Five portfolio-grade cybersecurity projects built from this lab infrastructure:

1. **Detection Engineering** — Custom Sigma/Wazuh rules mapped to MITRE ATT&CK
2. **Active Directory Attack & Defense** — Full kill chain with detections
3. **Honeypot Threat Intel Pipeline** — OpenCanary → MISP → pfSense auto-block
4. **IoT Security Assessment** — ESP32 firmware + traffic analysis
5. **SOAR Automation Pipeline** — Wazuh → Shuffle → TheHive → auto-response

See [docs/portfolio-projects.md](docs/portfolio-projects.md) for full project specs.

## What I Learned

- Designing segmented enterprise networks
- Building reproducible infrastructure with IaC
- Implementing detection engineering workflows
- Integrating multiple security tools into a cohesive SOC pipeline

---

*Target roles: SOC Analyst · Security Engineer · Junior Penetration Tester*
*Domain: blerdmh.com (Cloudflare) | Internal: lab.blerdmh.local*
