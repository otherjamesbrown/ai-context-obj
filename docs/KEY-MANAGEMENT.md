# Key & Secret Management Guide

**⚠️ CRITICAL: This document explains how to manage secrets securely across multiple development machines and environments.**

---

## 🎯 Our Setup

This project uses a **three-tier secret management strategy**:

1. **🔐 1Password** → Local development (both laptops)
2. **🔐 GitHub Secrets** → CI/CD (GitHub Actions)
3. **🔐 Kubernetes Secrets** → Production runtime (managed by Helm)

**Key Principle:** Secrets never live in Git. Ever.

```
┌──────────────┐
│  1Password   │  ← Local dev secrets (your 2 laptops)
└──────┬───────┘
       │
       ├─→ Machine 1: Auto-sync
       └─→ Machine 2: Auto-sync
       
┌──────────────┐
│ GitHub       │  ← CI/CD secrets (GitHub Actions)
│ Secrets      │
└──────────────┘
       
┌──────────────┐
│ Kubernetes   │  ← Production secrets (K8s cluster)
│ Secrets      │
└──────────────┘
```

---

## 📁 Where Keys Live (By Environment)

### **1. Development Machines (Your 2 Laptops)**

#### **Option A: 1Password / LastPass / Bitwarden (Recommended)**

**Structure:**
```
Vault: IAS-Development
├── 📁 Local Development
│   ├── .env file (entire file as secure note)
│   ├── JWT_SECRET
│   ├── DATABASE_URL
│   └── REDIS_PASSWORD
│
├── 📁 GitHub Tokens
│   ├── GHCR_TOKEN (Personal Access Token)
│   ├── SSH Private Key
│   └── Deploy Keys
│
├── 📁 Cloud Providers
│   ├── Linode API Token
│   └── AWS Access Keys (if used)
│
└── 📁 Production Secrets (System Admin only)
    ├── Production DATABASE_URL
    ├── Production JWT_SECRET
    └── Kubernetes Admin Token
```

**Workflow:**
```bash
# Machine 1: Save .env to 1Password
op document create .env --vault IAS-Development

# Machine 2: Retrieve .env from 1Password
op document get .env --vault IAS-Development > .env

# Or use 1Password CLI in scripts
eval $(op signin)
export JWT_SECRET=$(op read "op://IAS-Development/JWT_SECRET/credential")
```

---

#### **Option B: Encrypted File in Private Repo (Team Approach)**

**Setup:**
```bash
# 1. Create a PRIVATE repository (separate from main repo)
# Repository: otherjamesbrown/ias-secrets (PRIVATE!)

# 2. Use git-crypt to encrypt secrets
cd ias-secrets
git-crypt init
git-crypt add-gpg-user your-email@example.com

# 3. Create .gitattributes
echo "*.env filter=git-crypt diff=git-crypt" >> .gitattributes
echo "*.secret filter=git-crypt diff=git-crypt" >> .gitattributes

# 4. Add secrets
cp ~/path/to/.env development.env
git add development.env
git commit -m "Add development secrets"
git push

# 5. On second machine, clone and unlock
git clone git@github.com:otherjamesbrown/ias-secrets.git
cd ias-secrets
git-crypt unlock
```

**Structure:**
```
ias-secrets/ (PRIVATE repo)
├── development.env           # Encrypted by git-crypt
├── github-tokens.secret      # Encrypted
├── linode-token.secret       # Encrypted
├── ssh-keys/
│   ├── gitops_deploy_key     # Encrypted
│   └── gitops_deploy_key.pub
└── README.md                 # Instructions (not encrypted)
```

---

#### **Option C: Local Encrypted Directory (Solo Developer)**

**Setup:**
```bash
# Create encrypted directory (macOS)
hdiutil create -size 100m -encryption AES-256 \
  -volname "IAS-Secrets" -fs HFS+ ~/ias-secrets.dmg

# Mount it
hdiutil attach ~/ias-secrets.dmg
# Opens as /Volumes/IAS-Secrets

# Store secrets
cp .env /Volumes/IAS-Secrets/development.env
cp ~/.ssh/github_key /Volumes/IAS-Secrets/ssh-keys/

# Unmount when done
hdiutil detach /Volumes/IAS-Secrets

# Sync to second machine (encrypted file is safe)
scp ~/ias-secrets.dmg user@machine2:~/
```

---

### **2. CI/CD (GitHub Actions)**

**Location:** Repository → Settings → Secrets and variables → Actions

**How to add:**
```bash
# Via GitHub CLI (recommended)
gh secret set GHCR_TOKEN --body "ghp_xxxxxxxxxxxx"
gh secret set DATABASE_URL --body "postgres://..."
gh secret set JWT_SECRET --body "$(openssl rand -hex 32)"

# Or via GitHub UI
# Settings → Secrets → New repository secret
```

**Secrets needed:**
```yaml
# Container Registry
GHCR_TOKEN              # GitHub PAT with write:packages

# Cloud Provider
LINODE_TOKEN            # Linode API token for Terraform

# Kubernetes
KUBECONFIG              # Base64-encoded kubeconfig

# Deployment
ARGOCD_AUTH_TOKEN       # ArgoCD API token
GITOPS_DEPLOY_KEY       # SSH key for GitOps repo
```

---

### **3. Kubernetes (Production)**

**Storage:** Kubernetes Secrets (via Helm)

**How it works:**
```yaml
# values-production.yaml (checked into Git)
userService:
  env:
    DATABASE_URL: 
      valueFrom:
        secretKeyRef:
          name: ias-db-credentials
          key: url
    JWT_SECRET:
      valueFrom:
        secretKeyRef:
          name: ias-jwt-secret
          key: secret

# Secrets created separately (NOT in Git)
# Created via Helm during initial setup
```

**How to create (one-time setup):**
```bash
# Generate production secrets
JWT_SECRET=$(openssl rand -hex 32)
DATABASE_PASSWORD=$(openssl rand -base64 32)

# Create Kubernetes secret
kubectl create secret generic ias-jwt-secret \
  --from-literal=secret=$JWT_SECRET \
  --namespace=ias-production

kubectl create secret generic ias-db-credentials \
  --from-literal=url="postgres://user:$DATABASE_PASSWORD@postgres:5432/ias" \
  --namespace=ias-production

# Store the secrets in 1Password for recovery
op item create \
  --category=password \
  --title="IAS Production JWT Secret" \
  --vault="IAS-Production" \
  JWT_SECRET=$JWT_SECRET
```

---

## 🔐 Multi-Machine Sync Strategy

### **Recommended: 1Password CLI + Shell Script**

Create: `~/scripts/ias-env.sh`

```bash
#!/bin/bash
# Load IAS development secrets from 1Password

if ! command -v op &> /dev/null; then
    echo "Error: 1Password CLI not installed"
    echo "Install: brew install 1password-cli"
    exit 1
fi

# Sign in to 1Password (if not already)
eval $(op signin)

# Export environment variables from 1Password
export DATABASE_URL=$(op read "op://IAS-Development/Local/.env/DATABASE_URL")
export REDIS_PASSWORD=$(op read "op://IAS-Development/Local/.env/REDIS_PASSWORD")
export JWT_SECRET=$(op read "op://IAS-Development/Local/.env/JWT_SECRET")
export LINODE_TOKEN=$(op read "op://IAS-Development/Cloud/LINODE_TOKEN")

echo "✅ Environment variables loaded from 1Password"
```

**Usage on both machines:**
```bash
# Run before development
source ~/scripts/ias-env.sh

# Or add to .zshrc / .bashrc
alias ias-env='source ~/scripts/ias-env.sh'
```

---

## 🛡️ Git Protection (Prevent Accidental Commits)

### **1. Pre-Commit Hook (Installed Automatically)**

Create: `.git/hooks/pre-commit`

```bash
#!/bin/bash
# Pre-commit hook to prevent secrets from being committed

# Check for common secret patterns
if git diff --cached --name-only | grep -E '\.(env|secret|key|pem)$'; then
    echo "❌ ERROR: Attempting to commit secret files!"
    echo "Files matched:"
    git diff --cached --name-only | grep -E '\.(env|secret|key|pem)$'
    exit 1
fi

# Check for secret patterns in file contents
if git diff --cached | grep -E '(password|secret|token|api_key).*=.*[A-Za-z0-9]{20,}'; then
    echo "❌ ERROR: Possible secret detected in file content!"
    echo "Please review your changes and use environment variables instead."
    exit 1
fi

echo "✅ Pre-commit checks passed"
exit 0
```

**Install:**
```bash
chmod +x .git/hooks/pre-commit
```

---

### **2. Git Secrets Tool (Automatic Scanning)**

```bash
# Install
brew install git-secrets

# Initialize in repo
cd ~/path/to/ai-as-a-service
git secrets --install
git secrets --register-aws

# Add custom patterns
git secrets --add 'ghp_[0-9a-zA-Z]{36}'          # GitHub PAT
git secrets --add 'ias_[0-9a-zA-Z]{32}'          # IAS API keys
git secrets --add 'postgres://.*:.*@'            # Database URLs
git secrets --add '[0-9a-f]{64}'                 # SHA-256 secrets

# Scan existing history (one-time)
git secrets --scan-history
```

---

### **3. GitHub Secret Scanning (Automatic)**

**Enable:** Repository → Settings → Code security and analysis

- ✅ Enable: Secret scanning
- ✅ Enable: Push protection

**What it catches:**
- GitHub Personal Access Tokens
- AWS keys
- Google API keys
- Slack tokens
- 100+ other secret types

---

## 📋 Setup Checklist (For Your 2 Machines)

### **Machine 1 (Primary Development Machine)**

- [ ] Install 1Password CLI: `brew install 1password-cli`
- [ ] Create 1Password vault: "IAS-Development"
- [ ] Generate SSH key for GitHub: `ssh-keygen -t ed25519`
- [ ] Add SSH key to GitHub account
- [ ] Clone repository
- [ ] Copy `.env.example` to `.env` and fill in values
- [ ] Store `.env` in 1Password as secure note
- [ ] Generate GitHub PAT (write:packages)
- [ ] Store PAT in 1Password
- [ ] Install git-secrets: `brew install git-secrets`
- [ ] Initialize git-secrets in repo: `git secrets --install`

### **Machine 2 (Secondary Machine)**

- [ ] Install 1Password CLI: `brew install 1password-cli`
- [ ] Sign in to 1Password account
- [ ] Clone repository
- [ ] Retrieve `.env` from 1Password: `op read "op://IAS-Development/env-file"`
- [ ] Save to `.env` in project root
- [ ] Generate SSH key (separate from Machine 1)
- [ ] Add to GitHub account (label: "Machine 2")
- [ ] Install git-secrets
- [ ] Initialize git-secrets in repo

---

## 🚨 What to Do If Secrets Are Committed

### **If You Catch It BEFORE Pushing:**

```bash
# Remove from last commit
git reset --soft HEAD~1
rm .env  # or other secret file
git add .
git commit -m "Add feature X"

# Or amend the commit
git rm --cached .env
git commit --amend --no-edit
```

### **If You Already Pushed:**

```bash
# 1. IMMEDIATELY revoke the exposed secrets
#    - Regenerate JWT_SECRET
#    - Rotate API keys
#    - Change database passwords

# 2. Remove from Git history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch .env' \
  --prune-empty --tag-name-filter cat -- --all

# 3. Force push (after revoking secrets!)
git push origin --force --all

# 4. Notify team if secrets were shared
```

**⚠️ Better: Assume it's compromised, rotate everything immediately!**

---

## 📊 Secret Rotation Schedule

| Secret Type | Rotation Frequency | Method |
|-------------|-------------------|--------|
| GitHub PATs | Every 90 days | Regenerate in GitHub Settings |
| JWT Secret | Every 6 months | Update K8s secret + restart pods |
| Database Passwords | Every 6 months | Coordinate with team |
| Linode API Token | Every 90 days | Regenerate in Linode Cloud Manager |
| SSH Keys | Every year | Generate new, remove old from GitHub |

---

## ✅ Quick Reference: Command Cheat Sheet

```bash
# Generate secrets
openssl rand -hex 32              # 256-bit random (JWT secret)
openssl rand -base64 32           # Base64 random
uuidgen                           # UUID

# Base64 encode/decode (for K8s)
echo -n "my-secret" | base64      # Encode
echo "bXktc2VjcmV0" | base64 -d   # Decode

# 1Password CLI
op signin                         # Sign in
op item list                      # List all items
op read "op://vault/item/field"   # Read specific value
op document create file.txt       # Store a file

# GitHub CLI
gh secret set NAME --body "value" # Add GitHub secret
gh secret list                    # List all secrets
gh secret remove NAME             # Remove secret

# Git secrets
git secrets --scan                # Scan staged changes
git secrets --scan-history        # Scan entire history
git secrets --list                # List patterns
```

---

## 🎓 Best Practices Summary

1. ✅ **Never commit secrets to Git** - Use .gitignore + pre-commit hooks
2. ✅ **Use password managers** - 1Password / Bitwarden for team sharing
3. ✅ **Separate dev from prod** - Different secrets for each environment
4. ✅ **Rotate regularly** - Follow the rotation schedule
5. ✅ **Principle of least privilege** - Only share what's needed
6. ✅ **Encrypt at rest** - Use git-crypt or encrypted vaults
7. ✅ **Audit access** - Review who has access to what (monthly)
8. ✅ **Have a breach plan** - Know what to rotate if secrets leak

---

**Need help setting up 1Password or git-secrets?** Let me know! 🔐

