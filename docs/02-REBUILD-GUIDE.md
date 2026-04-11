# Rebuild Guide

IAC-first phased rebuild sequence — from bare metal to fully operational lab.

**Rule:** Do not start a phase until all prerequisites for that phase are confirmed complete.
Skipping ahead causes dependency failures that are harder to debug than doing it right.

---

## Phase 0 — Workstation and Repository (Skynet)

Complete this before touching any hardware.

### 0.1 Clone repo and install tools

```bash
# Clone
git clone https://github.com/techinb/blerdmh-lab.git ~/CSLAB/SOC/blerdmh-lab
cd ~/CSLAB/SOC/blerdmh-lab

# Python tooling
pip install ansible ansible-lint pre-commit detect-secrets

# Ansible collections
ansible-galaxy collection install \
  community.general \
  ansible.posix \
  pfsensible.core

# Pre-commit hooks
pre-commit install
detect-secrets scan \
  --exclude-files '.*-sealed\.yml' \
  --exclude-files 'vault\.yml' \
  > .secrets.baseline
```

### 0.2 Generate SSH key (if not already done)

```bash
ssh-keygen -t ed25519 -C "skynet-ansible" -f ~/.ssh/id_ed25519
# One key pair — distribute public key to all hosts
```

### 0.3 Set up vault.yml

```bash
cp ansible/inventory/group_vars/all/vault.yml.example \
   ansible/inventory/group_vars/all/vault.yml

# Edit vault.yml — fill in ALL values marked CHANGEME
# Then encrypt:

```

### 0.4 Configure GitHub Actions secrets

In GitHub repo → Settings → Secrets → Actions, add:

| Secret | Value |
|--------|-------|
| `ANSIBLE_VAULT_PASSWORD` | Your vault password |
| `TAILSCALE_AUTH_KEY` | Ephemeral + reusable key from Tailscale admin |
| `ANSIBLE_SSH_PRIVATE_KEY` | Contents of `~/.ssh/id_ed25519` |

**Checkpoint:** `pre-commit run --all-files` passes with no errors.

---

## Phase 1 — Network Foundation

### 1.1 pfSense — initial setup (manual)

1. Install pfSense CE on Glovary i3-N355 via USB
2. Assign interfaces: `igb0` = WAN, `igb1` = LAN
3. Set LAN IP to `10.0.10.1/24`
4. Access web UI at `http://10.0.10.1` from Skynet
5. Complete setup wizard — set admin password, configure WAN

### 1.2 MokerLink switch — VLAN configuration (manual)

Follow `switch/mokerlink-vlan-config.md` exactly.
Critical: configure management VLAN migration **last**.

### 1.3 pfSense — DHCP reservations (manual, before any host boots)

Set static DHCP reservations by MAC for every device:

| Host | VLAN | IP |
|------|------|----|
| Skynet | 10 | 10.0.10.10 |
| AMDPVE | 10 | 10.0.10.20 |
| N1 Mini | 10 | 10.0.10.21 |
| ZimaBoard | 20 | 10.0.20.5 |
| Pi 4 | 20 | 10.0.20.10 |
| Pi 5 | 20 | 10.0.20.11 |

### 1.4 pfSense — automate via Ansible

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/06-pfsense.yml \
  --ask-vault-pass \
  --tags aliases,vlans,rules,dns
# Run DHCP tag separately after confirming all MACs in vault.yml
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/06-pfsense.yml \
  --ask-vault-pass \
  --tags dhcp
```

**Checkpoint:** All VLAN gateways pingable from Skynet. Switch management at 10.0.10.2.

---

## Phase 2 — Proxmox Cluster

### 2.1 Install Proxmox VE (manual — both nodes)

1. Flash Proxmox VE ISO to USB
2. Install on AMDPVE — set IP `10.0.10.20`, hostname `amdpve`
3. Install on N1 Mini — set IP `10.0.10.21`, hostname `n1minipve`
4. Deploy SSH public key to root on both: `ssh-copy-id root@10.0.10.20`

### 2.2 Form the cluster (manual — one time only)

```bash
# On AMDPVE
pvecm create blerdmh-lab

# On N1 Mini
pvecm add 10.0.10.20

# Verify
pvecm status
pvecm nodes
```

### 2.3 Configure Proxmox via Ansible

```bash
# Full config — both nodes
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/05-proxmox.yml \
  --ask-vault-pass

# IOMMU only on AMDPVE (requires reboot)
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/05-proxmox.yml \
  --ask-vault-pass --tags iommu --limit amdpve
```

> After IOMMU task: reboot AMDPVE, then run `lspci -nn | grep -i amd` to get
> RX 5700 XT PCI IDs and populate `proxmox_vfio_pci_ids` in host_vars/amdpve/main.yml.

**Checkpoint:** Both nodes visible in Proxmox cluster web UI. Storage backends registered.

---

## Phase 3 — TrueNAS Scale

See `docs/04b-REBUILD-PHASE3-TRUENAS.md` for the full TrueNAS setup guide.

**Checkpoint:** NFS shares mountable from AMDPVE: `showmount -e 10.0.20.5`

---

## Phase 4 — k3s Cluster

### 4.1 Flash OS to Raspberry Pis (manual)

Flash Ubuntu 24.04 LTS (64-bit) to both Pis using Raspberry Pi Imager.
In Imager advanced settings:
- Enable SSH, add your public key
- Set hostname (`k3s-cp`, `k3s-w1`)
- Set username `techinb`

Boot both Pis. Verify SSH: `ssh techinb@10.0.20.10`

### 4.2 Run k3s playbook

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/02-k3s.yml \
  --ask-vault-pass
```

### 4.3 Approve Tailscale subnet routes

After playbook completes:
1. Go to https://login.tailscale.com/admin/machines
2. Find `k3s-cp-blerdmh`
3. Edit route settings → approve all advertised subnets

**Checkpoint:** `kubectl --kubeconfig ~/.kube/config-blerdmh-lab get nodes` shows all nodes Ready.

---

## Phase 5 — Baseline All Hosts

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/01-baseline.yml \
  --ask-vault-pass
```

**Checkpoint:** All hosts in inventory reachable, hardened, shipping logs.

---

## Phase 6 — Security Stack

```bash
# Step 1: Wazuh manager
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/03-security.yml \
  --ask-vault-pass --tags wazuh,manager

# Step 2: Verify dashboard at https://10.0.40.10

# Step 3: Deploy agents
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/03-security.yml \
  --ask-vault-pass --tags wazuh,agents

# Step 4: IDS stack + OpenVAS
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/03-security.yml \
  --ask-vault-pass --tags security
```

**Checkpoint:** Wazuh dashboard shows all agents connected. Suricata EVE log populating.

---

## Phase 7 — k3s Workloads

### 7.1 Seal secrets before applying manifests

```bash
# Install kubeseal on Skynet
brew install kubeseal  # or: curl ... kubeseal binary

# Seal each secret (example — Nextcloud DB password)
kubectl create secret generic nextcloud-db \
  --namespace personal \
  --from-literal=password=YOURPASSWORD \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > k3s/personal/nextcloud/secret-sealed.yml
```

### 7.2 Apply all manifests

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/04-k3s-workloads.yml \
  --ask-vault-pass
```

**Checkpoint:** All pods running. Nextcloud, Jellyfin, and Grafana accessible.

---

## Phase 8 — Gitea

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  ansible/playbooks/07-gitea.yml \
  --ask-vault-pass
```

**Checkpoint:** Gitea accessible at http://10.0.10.50:3000. Mirror syncing from GitHub.

---

## Verification Checklist

After all phases complete, verify end-to-end:

- [ ] All VLAN gateways respond to ping from Skynet
- [ ] Proxmox cluster shows both nodes — no quorum warnings
- [ ] TrueNAS NFS shares mounted on Proxmox nodes
- [ ] k3s cluster: all nodes Ready, all pods Running
- [ ] Wazuh: all agents connected, custom rules loaded
- [ ] Suricata: EVE JSON log updating (`tail -f /var/log/suricata/eve.json`)
- [ ] Zeek: logs appearing in `/var/log/zeek/`
- [ ] Gitea: mirror shows current commit from GitHub
- [ ] CI pipeline: push a commit and verify all CI checks pass
- [ ] Tailscale: access lab services from phone on cellular
