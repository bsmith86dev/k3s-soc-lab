# blerdmh-lab — Repository Tree
# 91 files | pfSense CE edition | Updated: 2026-03
# Authoritative reference for repo structure
# Use to verify your local clone has everything in the right place

blerdmh-lab/
│
├── README.md                                        # Project overview, hardware, quickstart
├── REPO-TREE.md                                     # This file
├── .gitignore                                       # Never-commit rules
├── .pre-commit-config.yaml                          # detect-secrets, ansible-lint, yamllint
├── .yamllint.yml                                    # YAML lint rules (CI + pre-commit)
├── mkdocs.yml                                       # MkDocs Material → GitHub Pages
│
├── .github/
│   └── workflows/
│       ├── 01-ci.yml                                # Secret scan, YAML lint, Ansible lint, k8s validate
│       ├── 02-cd-deploy.yml                         # Ansible deploy via Tailscale (manual trigger)
│       └── 03-docs.yml                              # MkDocs build + GitHub Pages deploy
│
├── ansible/
│   ├── inventory/
│   │   ├── hosts.yml                                # All hosts + groups
│   │   ├── group_vars/all/
│   │   │   └── vault.yml.example                    # Template — all required vault keys
│   │   └── host_vars/
│   │       ├── amdpve/main.yml                      # IOMMU on, 64GB ARC, provisions VMs
│   │       ├── n1minipve/main.yml                   # 8GB ARC, provisions LXCs
│   │       ├── k3s-cp/main.yml                      # Tailscale subnet router enabled
│   │       ├── k3s-w1/main.yml                      # Worker UFW rules
│   │       └── k3s-w2-amd/main.yml                  # GPU flag + worker rules
│   │
│   ├── playbooks/
│   │   ├── 00-site.yml                              # Master — all phases in order
│   │   ├── 01-baseline.yml                          # Common hardening → all linux_hosts
│   │   ├── 02-k3s.yml                               # Control plane then workers (serial: 1)
│   │   ├── 03-security.yml                          # Wazuh → agents → IDS → OpenVAS
│   │   ├── 04-k3s-workloads.yml                     # kubectl apply all manifests from Skynet
│   │   ├── 05-proxmox.yml                           # Proxmox post-install config
│   │   ├── 06-pfsense.yml                           # pfSense aliases/VLANs/rules/DNS/DHCP
│   │   └── 07-gitea.yml                             # Gitea on N1 Mini LXC 302
│   │
│   └── roles/
│       │
│       ├── common/
│       │   ├── defaults/main.yml                    # Packages, SSH settings, UFW, fail2ban config
│       │   ├── handlers/main.yml                    # sshd, fail2ban, ufw, unattended-upgrades, reboot
│       │   ├── tasks/
│       │   │   ├── main.yml
│       │   │   ├── system.yml                       # Hostname, timezone, locale, kernel params, MOTD
│       │   │   ├── packages.yml                     # Install baseline, remove insecure packages
│       │   │   ├── ssh.yml                          # Key deploy, sshd_config template, remove weak keys
│       │   │   ├── ufw.yml                          # Default deny, MGMT SSH, Tailscale, host rules
│       │   │   ├── fail2ban.yml                     # jail.d config from template
│       │   │   ├── upgrades.yml                     # Unattended security upgrades, hold k3s/docker
│       │   │   ├── ntp.yml                          # Chrony — critical for Wazuh correlation + TLS
│       │   │   └── logging.yml                      # rsyslog → Wazuh, logrotate
│       │   └── templates/
│       │       ├── sshd_config.j2
│       │       └── fail2ban-jail.j2
│       │
│       ├── k3s/
│       │   ├── defaults/main.yml                    # Version pins, CIDRs, MetalLB pool, Tailscale routes
│       │   ├── handlers/main.yml                    # k3s server/agent, tailscaled, systemd reload
│       │   ├── tasks/
│       │   │   ├── main.yml
│       │   │   ├── preflight.yml                    # OS/RAM/disk, cgroups v2, swap off, kernel modules
│       │   │   ├── install-server.yml               # k3s server, MetalLB, cert-manager, Sealed Secrets
│       │   │   ├── install-agent.yml                # Worker join, GPU node label, readiness wait
│       │   │   ├── tailscale.yml                    # Install, subnet router, route advertisement
│       │   │   └── kubeconfig.yml                   # Fetch kubeconfig to Skynet
│       │   └── templates/
│       │       └── k3s-config.j2                    # Server + agent config per node
│       │
│       ├── proxmox/
│       │   ├── defaults/main.yml                    # API creds, cluster nodes, ZFS ARC, VM/LXC defs
│       │   ├── handlers/main.yml                    # pveproxy, pvedaemon, initramfs, grub, reboot
│       │   ├── tasks/
│       │   │   ├── main.yml
│       │   │   ├── repos.yml                        # Remove enterprise nag, add no-subscription
│       │   │   ├── packages.yml                     # ZFS, NFS, python3-proxmoxer, Ceph tools
│       │   │   ├── network.yml                      # VLAN-aware bridge via interfaces.j2
│       │   │   ├── iommu.yml                        # AMD IOMMU, VFIO, GPU blacklist (AMDPVE only)
│       │   │   ├── zfs.yml                          # Datasets, ARC tuning, pvesm register, snapshots
│       │   │   ├── storage-nfs.yml                  # TrueNAS NFS (backups, iso) as pvesm backends
│       │   │   ├── storage-pbs.yml                  # PBS with prune policy
│       │   │   ├── datacenter.yml                   # Cluster settings, API token, HA, CPU governor
│       │   │   ├── hardening.yml                    # SSH, sysctl, PVE firewall, auditd, fail2ban
│       │   │   ├── lxc.yml                          # Gitea LXC 302 + PBS LXC 310 on N1 Mini
│       │   │   └── vms.yml                          # Security VMs + k3s GPU worker on AMDPVE
│       │   └── templates/
│       │       └── interfaces.j2                    # /etc/network/interfaces per node
│       │
│       ├── pfsense/
│       │   ├── defaults/main.yml                    # pfsense_host, interface names, MAC vars
│       │   └── tasks/
│       │       ├── main.yml
│       │       ├── aliases.yml                      # RFC1918, MGMT_HOSTS, K3S_NODES, SECURITY_VMS…
│       │       ├── vlans.yml                        # VLANs 10–70 on igb1
│       │       ├── rules.yml                        # Full inter-VLAN ruleset all 8 VLANs
│       │       ├── dns.yml                          # Unbound + host registrations
│       │       └── dhcp.yml                         # DHCP pools + static MAC reservations
│       │
│       ├── gitea/
│       │   ├── defaults/main.yml                    # Version, port, admin user, mirror interval
│       │   ├── handlers/main.yml                    # gitea/nginx restart and reload
│       │   ├── tasks/
│       │   │   ├── main.yml
│       │   │   ├── install.yml                      # Binary, systemd service, wait for ready
│       │   │   ├── configure.yml                    # Admin user, org, UFW rules via API
│       │   │   └── mirror.yml                       # GitHub → Gitea mirror + sync trigger
│       │   └── templates/
│       │       └── app.ini.j2                       # Full Gitea config (SQLite, private)
│       │
│       ├── wazuh/
│       │   ├── defaults/main.yml                    # Version, manager IP, active response settings
│       │   ├── handlers/main.yml                    # manager, agent, dashboard, filebeat
│       │   └── tasks/
│       │       ├── main.yml
│       │       ├── manager.yml                      # Manager + OpenSearch + Dashboard + UFW
│       │       ├── agents.yml                       # Install, auto-enroll, start, verify
│       │       ├── rules.yml                        # Custom MITRE-mapped rules (T1110, T1021,
│       │       │                                    #   T1548, T1136, T1053, T1046)
│       │       └── active-response.yml              # firewall-drop on brute force + port scan
│       │
│       └── security/
│           ├── defaults/main.yml                    # suricata_capture_interface
│           ├── handlers/main.yml                    # suricata, zeek, gvmd, gsad, rule reloads
│           ├── tasks/
│           │   ├── main.yml
│           │   ├── suricata.yml                     # Install, ET Open rules, EVE JSON, daily updates
│           │   ├── zeek.yml                         # Install, node.cfg, local.zeek, systemd service
│           │   └── openvas.yml                      # Greenbone CE via Docker Compose
│           └── templates/
│               └── suricata.yaml.j2                 # AF_PACKET, EVE JSON, app layer protocols
│
├── k3s/
│   ├── namespaces/                                  # Namespace definitions
│   ├── ingress/                                     # cert-manager ClusterIssuer, Traefik, CF Tunnel
│   ├── network-policy/                              # Default deny + allow-monitoring + allow-dns
│   ├── security/                                    # Sealed Secrets controller, AMD GPU plugin
│   ├── personal/
│   │   ├── nextcloud/                               # Deployment, service, ingress, PVC, sealed secret
│   │   ├── jellyfin/                                # GPU resource request, service, ingress, PVC
│   │   ├── homeassistant/                           # Deployment, service, ingress
│   │   └── media/                                   # Radarr, Sonarr, Lidarr
│   └── monitoring/
│       ├── prometheus/                              # Deployment, service, configmap
│       ├── grafana/                                 # Deployment, service, ingress, PVC
│       ├── loki/                                    # Deployment, service
│       └── promtail/                               # DaemonSet, configmap
│
├── switch/
│   ├── mokerlink-vlan-config.md                    # Port map, VLAN table, config sequence
│   └── pfsense-backup-TEMPLATE.xml                 # Placeholder — commit real XML backups here
│
└── docs/
    ├── index.md
    ├── 00-ARCHITECTURE.md
    ├── 02-REBUILD-GUIDE.md
    ├── 04-TRUENAS-STORAGE.md
    ├── 04b-REBUILD-PHASE3-TRUENAS.md
    ├── portfolio-projects.md
    ├── network/
    │   └── pfsense-firewall.md                     # Interface map, aliases, rule matrix
    └── cicd/
        ├── overview.md
        └── secrets.md

# ── Counts ──────────────────────────────────────────────────────────────────
# Root level              6   (.gitignore, .yamllint, .pre-commit, mkdocs, README, REPO-TREE)
# .github/workflows       3
# ansible/inventory       8   (hosts.yml, vault example, 5x host_vars)
# ansible/playbooks       8   (00–07)
# ansible/roles/common   15   (defaults, handlers, 9x tasks, 2x templates)
# ansible/roles/k3s      10   (defaults, handlers, 6x tasks, 1x template)
# ansible/roles/proxmox  17   (defaults, handlers, 12x tasks, 1x template)
# ansible/roles/pfsense   7   (defaults, 6x tasks)
# ansible/roles/gitea     8   (defaults, handlers, 4x tasks, 1x template)
# ansible/roles/wazuh     7   (defaults, handlers, 5x tasks)
# ansible/roles/security  7   (defaults, handlers, 4x tasks, 1x template)
# switch                  2
# docs                    9   (mix of generated + previously created)
# k3s manifests          ~20  (stubs — built during lab execution phases)
# ─────────────────────────────────────────────────────────────────────────────
# TOTAL GENERATED        91 files
