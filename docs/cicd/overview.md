# CI/CD Pipeline Overview

## Design Goals

- Every push to `main` is validated before it can break anything
- Deployments to the lab require explicit human approval
- No credentials ever appear in logs, artifacts, or workflow files
- The runner connects to the lab via Tailscale — no exposed ports required

---

## Pipeline Architecture

```
git push → GitHub
    │
    ├── 01-ci.yml (automatic on every push + PR)
    │   ├── Secret scan (Gitleaks + detect-secrets)
    │   ├── YAML lint
    │   ├── Ansible lint
    │   └── k8s manifest validation (kubeconform)
    │
    ├── 03-docs.yml (automatic when docs/ changes)
    │   ├── MkDocs build
    │   └── Deploy to GitHub Pages
    │
    └── 02-cd-deploy.yml (manual trigger only)
        ├── Human approval gate (GitHub Environment protection)
        ├── Tailscale ephemeral connection to lab network
        ├── SSH key injection
        ├── Ansible playbook run
        └── Tailscale disconnect + credential cleanup
```

---

## Workflow Files

| File | Trigger | Purpose |
|------|---------|---------|
| `.github/workflows/01-ci.yml` | Push, PR | Lint + security scan |
| `.github/workflows/02-cd-deploy.yml` | Manual dispatch | Deploy to lab |
| `.github/workflows/03-docs.yml` | Push to `docs/` | Docs site |

---

## Tailscale Connection Strategy

The GitHub Actions runner has no persistent network access to your lab.
Instead of exposing SSH ports through your firewall, the CD workflow:

1. Uses a Tailscale **ephemeral auth key** (reusable, expires after use)
2. Connects the runner as a temporary Tailscale node
3. The runner gains access to all advertised subnets via the Pi 4 subnet router
4. After the playbook completes, the runner disconnects and the ephemeral node expires

This means your lab never has an exposed port and the auth key cannot be reused
after the workflow exits.

### Generating the Tailscale auth key

1. Go to https://login.tailscale.com/admin/settings/keys
2. Generate key — select: **Ephemeral**, **Reusable**, expiry 90 days
3. Add as GitHub secret: `TAILSCALE_AUTH_KEY`
4. Rotate every 90 days

---

## GitHub Environment Protection

The CD workflow targets a GitHub Environment named `lab-production`.
Configure it in: **Settings → Environments → lab-production**

Recommended protection rules:
- Required reviewers: yourself (forces manual approval before any deploy)
- Wait timer: 0 minutes (approval is sufficient)
- Deployment branches: `main` only

---

## Running a Deployment

```
GitHub repo → Actions → CD Deploy → Run workflow

Inputs:
  playbook:  01-baseline.yml     ← which playbook to run
  tags:      common,ssh          ← optional tag filter
  limit:     k3s-cp              ← optional host/group limit
  dry_run:   false               ← set true to run --check --diff first
```

**Always run with `dry_run: true` first** when deploying to production hosts
for the first time or after significant changes. Review the diff output before
setting dry_run to false.

---

## Required GitHub Secrets

| Secret | Description | Where to Get |
|--------|-------------|-------------|
| `ANSIBLE_VAULT_PASSWORD` | Decrypts vault.yml | Your vault password |
| `TAILSCALE_AUTH_KEY` | Ephemeral Tailscale key | Tailscale admin console |
| `ANSIBLE_SSH_PRIVATE_KEY` | SSH private key | `cat ~/.ssh/id_ed25519` on Skynet |
