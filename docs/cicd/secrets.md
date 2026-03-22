# Secrets Management

No plaintext secret ever touches Git. This is enforced at three layers.

---

## The Three Layers

```
Layer 1 — Workstation (pre-commit)
  Gitleaks + detect-secrets scan on every commit
  Blocks the commit if a secret pattern is detected

Layer 2 — Ansible secrets (ansible-vault)
  vault.yml encrypted with AES-256
  Used for: infrastructure credentials, API tokens, passwords
  Decrypted only at playbook runtime

Layer 3 — Kubernetes secrets (Sealed Secrets)
  kubeseal encrypts secrets with the cluster's public key
  Sealed YAML files are safe to commit — only the cluster can decrypt them
  Used for: app passwords, API keys needed by k3s workloads
```

---

## Layer 1 — Pre-commit

### Setup (run once on Skynet)

```bash
pip install pre-commit detect-secrets
pre-commit install

# Create baseline — marks intentional false positives as allowed
detect-secrets scan \
  --exclude-files '.*-sealed\.yml' \
  --exclude-files 'vault\.yml' \
  > .secrets.baseline

git add .secrets.baseline
git commit -m "chore: initialize detect-secrets baseline"
```

From this point, any commit containing a detected secret is **blocked locally**
before it can be pushed. The CI pipeline runs the same check as a second gate.

### What gets scanned

- AWS-style key patterns
- Private key headers (BEGIN RSA PRIVATE KEY, etc.)
- High-entropy strings
- Common secret variable names (password=, token=, secret=)

### Handling false positives

```bash
# If detect-secrets flags something that is not a secret:
detect-secrets audit .secrets.baseline
# Follow prompts to mark as false positive
git add .secrets.baseline
```

---

## Layer 2 — Ansible Vault

### File location

```
ansible/inventory/group_vars/all/vault.yml   ← encrypted, committed
ansible/inventory/group_vars/all/vault.yml.example  ← template, committed
```

### Workflow

```bash
# Initial setup
cp ansible/inventory/group_vars/all/vault.yml.example \
   ansible/inventory/group_vars/all/vault.yml
# Edit vault.yml — fill in all CHANGEME values
ansible-vault encrypt ansible/inventory/group_vars/all/vault.yml

# Editing later
ansible-vault edit ansible/inventory/group_vars/all/vault.yml

# Running playbooks
ansible-playbook ... --ask-vault-pass
# Or store password in a file (chmod 600, listed in .gitignore):
echo "yourpassword" > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook ... --vault-password-file ~/.vault_pass
```

### All vault keys

See `vault.yml.example` for the complete list with descriptions.
Key categories:
- pfSense admin password + device MACs
- Proxmox API token
- k3s cluster token
- Tailscale auth key
- Cloudflare API token
- Gitea credentials + GitHub PAT
- Wazuh enrollment + API passwords
- PBS connection details
- LXC root password
- TrueNAS API key

---

## Layer 3 — Sealed Secrets (Kubernetes)

Sealed Secrets lets you commit encrypted Kubernetes Secret objects to Git.
The Bitnami Sealed Secrets controller in the cluster holds the private key
and is the only thing that can decrypt them.

### Setup on Skynet

```bash
# Install kubeseal CLI
brew install kubeseal
# or on Linux:
curl -sSL https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.3/kubeseal-0.26.3-linux-amd64.tar.gz \
  | tar -xz -C /usr/local/bin kubeseal
```

### Creating a sealed secret

```bash
# Step 1: Create a standard Kubernetes secret (dry-run, don't apply)
kubectl create secret generic my-app-secret \
  --namespace my-namespace \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > /tmp/my-secret.yml

# Step 2: Seal it with the cluster's public key
kubeseal \
  --kubeconfig ~/.kube/config-blerdmh-lab \
  --format yaml \
  < /tmp/my-secret.yml \
  > k3s/personal/my-app/secret-sealed.yml

# Step 3: Clean up the plaintext
rm /tmp/my-secret.yml

# Step 4: Commit the sealed file — it's safe
git add k3s/personal/my-app/secret-sealed.yml
git commit -m "feat: add sealed secret for my-app"
```

### Important notes

- Sealed secrets are **cluster-specific** — a secret sealed with one cluster's key
  cannot be decrypted by a different cluster
- If you rebuild the cluster, you must re-seal all secrets
- Back up the Sealed Secrets controller key:
  ```bash
  kubectl get secret \
    -n kube-system \
    -l sealedsecrets.bitnami.com/sealed-secrets-key \
    -o yaml > sealed-secrets-master-key-BACKUP.yml
  # Store this somewhere safe and OFF the lab network
  ```

---

## Secret Rotation Checklist

Run quarterly or after any suspected compromise:

- [ ] Rotate Tailscale auth key (90-day expiry anyway)
- [ ] Rotate Cloudflare API token
- [ ] Rotate GitHub PAT (gitea_github_token)
- [ ] Rotate Wazuh API password
- [ ] Rotate Proxmox API token
- [ ] Update vault.yml → re-encrypt → push
- [ ] Re-seal any Kubernetes secrets that use rotated credentials
- [ ] Update GitHub Actions secrets (TAILSCALE_AUTH_KEY, ANSIBLE_VAULT_PASSWORD)
