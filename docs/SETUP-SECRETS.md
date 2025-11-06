# Secret Management Setup Guide

**Quick start guide for setting up secrets on your two development machines.**

---

## 📋 Prerequisites

- ✅ 1Password installed on both laptops
- ✅ GitHub CLI installed: `brew install gh`
- ✅ Repository cloned on both machines

---

## 🚀 Part 1: Local Development Setup (1Password)

### **Step 1: Create 1Password Vault**

**On Machine 1:**

1. Open 1Password
2. Click "+" → New Vault
3. Name: **"IAS-Development"**
4. Click "Create"

### **Step 2: Generate Local Development Secrets**

```bash
# Navigate to your project
cd ~/Documents/GitHub/otherjamesbrown/ai-as-a-service

# Generate JWT secret
JWT_SECRET=$(openssl rand -hex 32)
echo "JWT_SECRET=$JWT_SECRET"

# Generate database password
DB_PASSWORD=$(openssl rand -base64 16)
echo "DATABASE_URL=postgres://ias_user:$DB_PASSWORD@localhost:5432/ias_dev?sslmode=disable"

# Generate Redis password
REDIS_PASSWORD=$(openssl rand -base64 16)
echo "REDIS_PASSWORD=$REDIS_PASSWORD"
```

### **Step 3: Create .env File**

```bash
# Copy the template
cp .env.example .env

# Edit with your generated values
nano .env
```

**Your .env should look like:**
```bash
# Database
DATABASE_URL=postgres://ias_user:YOUR_GENERATED_PASSWORD@localhost:5432/ias_dev?sslmode=disable

# Redis
REDIS_PASSWORD=YOUR_GENERATED_PASSWORD

# JWT Secret
JWT_SECRET=YOUR_GENERATED_HEX_STRING

# Local settings
LOG_LEVEL=debug
ENABLE_DEBUG_ENDPOINTS=true
```

### **Step 4: Store in 1Password**

**Via 1Password App:**

1. Click "+" → Secure Note
2. Title: **"IAS Local .env"**
3. Vault: **IAS-Development**
4. Copy entire contents of `.env` file into note
5. Save

**Or via 1Password CLI:**

```bash
# Install 1Password CLI if needed
brew install 1password-cli

# Sign in
eval $(op signin)

# Store .env as a document
op document create .env --vault IAS-Development --title "IAS Local .env"

# Or store individual secrets
op item create \
  --category=password \
  --title="IAS JWT Secret" \
  --vault="IAS-Development" \
  "password=$JWT_SECRET"
```

### **Step 5: Verify .env is Gitignored**

```bash
# This should show .env is ignored
git status

# Should NOT show .env as untracked
# If it does, something is wrong!
```

---

## 💻 Part 2: Setup Machine 2

**On your second laptop:**

### **Option A: 1Password App (Easiest)**

1. Open 1Password
2. Navigate to **IAS-Development** vault
3. Open **"IAS Local .env"** note
4. Copy contents
5. Create `.env` file in project root
6. Paste contents
7. Save

### **Option B: 1Password CLI**

```bash
# Sign in to 1Password
eval $(op signin)

# Retrieve .env file
op document get "IAS Local .env" --vault IAS-Development > .env

# Verify
cat .env
```

### **Option C: Shell Script (Automated)**

Create: `~/scripts/ias-load-secrets.sh`

```bash
#!/bin/bash
# Load IAS secrets from 1Password

PROJECT_DIR="$HOME/Documents/GitHub/otherjamesbrown/ai-as-a-service"

if [ ! -d "$PROJECT_DIR" ]; then
    echo "❌ Project directory not found: $PROJECT_DIR"
    exit 1
fi

# Sign in to 1Password if needed
if ! op account get &> /dev/null; then
    eval $(op signin)
fi

# Retrieve .env file
echo "📥 Retrieving .env from 1Password..."
op document get "IAS Local .env" --vault IAS-Development > "$PROJECT_DIR/.env"

if [ $? -eq 0 ]; then
    echo "✅ .env file loaded successfully"
    echo "📁 Location: $PROJECT_DIR/.env"
else
    echo "❌ Failed to retrieve .env"
    exit 1
fi
```

**Make executable:**
```bash
chmod +x ~/scripts/ias-load-secrets.sh
```

**Run on Machine 2:**
```bash
~/scripts/ias-load-secrets.sh
```

---

## 🔐 Part 3: GitHub Secrets for CI/CD

### **Step 1: Generate GitHub Personal Access Token**

1. Go to: https://github.com/settings/tokens
2. Click **"Generate new token (classic)"**
3. Name: **"IAS Container Registry"**
4. Expiration: **90 days**
5. Select scopes:
   - ✅ `write:packages`
   - ✅ `read:packages`
   - ✅ `delete:packages`
6. Click **"Generate token"**
7. **Copy the token** (starts with `ghp_`)

### **Step 2: Store PAT in 1Password**

```bash
# Via 1Password CLI
op item create \
  --category=password \
  --title="GitHub GHCR Token" \
  --vault="IAS-Development" \
  "password=ghp_YOUR_TOKEN_HERE"

# Or manually in 1Password app:
# New Item → Password → Title: "GitHub GHCR Token"
```

### **Step 3: Add to GitHub Repository Secrets**

```bash
# Authenticate GitHub CLI
gh auth login

# Set the secret
gh secret set GHCR_TOKEN --body "ghp_YOUR_TOKEN_HERE"

# Verify
gh secret list
# Should show: GHCR_TOKEN    Updated YYYY-MM-DD
```

### **Step 4: Add Additional CI/CD Secrets**

```bash
# Linode API token (for Terraform)
# Get from: https://cloud.linode.com/profile/tokens
gh secret set LINODE_TOKEN --body "YOUR_LINODE_TOKEN"

# Database URL for CI tests (non-production)
gh secret set CI_DATABASE_URL --body "postgres://test_user:test_pass@localhost:5432/ias_test"
```

---

## 🏭 Part 4: Production Secrets (Kubernetes)

**⚠️ Do this AFTER infrastructure is deployed**

### **Step 1: Generate Production Secrets**

```bash
# Generate strong production secrets
PROD_JWT_SECRET=$(openssl rand -hex 32)
PROD_DB_PASSWORD=$(openssl rand -base64 32)

# Print (save these to 1Password immediately!)
echo "Production JWT Secret: $PROD_JWT_SECRET"
echo "Production DB Password: $PROD_DB_PASSWORD"
```

### **Step 2: Store in 1Password**

```bash
# Create Production vault
# 1Password → New Vault → "IAS-Production"

# Store production secrets
op item create \
  --category=password \
  --title="IAS Production JWT Secret" \
  --vault="IAS-Production" \
  "password=$PROD_JWT_SECRET"

op item create \
  --category=password \
  --title="IAS Production Database Password" \
  --vault="IAS-Production" \
  "password=$PROD_DB_PASSWORD"
```

### **Step 3: Create Kubernetes Secrets**

```bash
# Get production kubeconfig
# (This will be set up when you deploy infrastructure)

# Create namespace
kubectl create namespace ias-production

# Create JWT secret
kubectl create secret generic ias-jwt-secret \
  --from-literal=secret=$PROD_JWT_SECRET \
  --namespace=ias-production

# Create database credentials secret
kubectl create secret generic ias-db-credentials \
  --from-literal=url="postgres://ias_user:$PROD_DB_PASSWORD@postgres.ias-production.svc.cluster.local:5432/ias" \
  --from-literal=password="$PROD_DB_PASSWORD" \
  --namespace=ias-production

# Verify
kubectl get secrets -n ias-production
```

---

## ✅ Verification Checklist

### **Machine 1:**
- [ ] 1Password vault "IAS-Development" created
- [ ] `.env` file created in project root
- [ ] `.env` file stored in 1Password
- [ ] `git status` does NOT show `.env` as untracked
- [ ] Can run `docker-compose up` successfully

### **Machine 2:**
- [ ] 1Password synced (can see "IAS-Development" vault)
- [ ] Retrieved `.env` from 1Password
- [ ] `.env` file exists in project root
- [ ] Can run `docker-compose up` successfully

### **GitHub:**
- [ ] Personal Access Token created
- [ ] PAT stored in 1Password
- [ ] `GHCR_TOKEN` added to repository secrets
- [ ] `gh secret list` shows secrets

### **Production (Later):**
- [ ] Production secrets generated
- [ ] Production secrets stored in 1Password vault "IAS-Production"
- [ ] Kubernetes secrets created in cluster
- [ ] Helm charts reference secrets correctly

---

## 🔄 Daily Workflow

### **Starting Development:**

```bash
# On either machine
cd ~/Documents/GitHub/otherjamesbrown/ai-as-a-service

# If .env is missing, load from 1Password
if [ ! -f .env ]; then
    eval $(op signin)
    op document get "IAS Local .env" --vault IAS-Development > .env
fi

# Start local development
docker-compose up -d

# Start coding!
```

### **Updating Secrets:**

**If you regenerate a secret:**

1. Update `.env` locally
2. Update in 1Password (so Machine 2 gets it)
3. Restart services: `docker-compose restart`

---

## 🆘 Common Issues

### **Problem: "Permission denied" on .env**

```bash
# Fix permissions
chmod 600 .env
```

### **Problem: 1Password CLI not working**

```bash
# Reinstall
brew reinstall 1password-cli

# Sign in again
eval $(op signin)
```

### **Problem: .env committed to Git accidentally**

```bash
# IMMEDIATELY remove from Git
git rm --cached .env
git commit -m "Remove .env from Git"

# Regenerate ALL secrets in .env
# Update 1Password with new secrets
```

### **Problem: GitHub secret not working in Actions**

```bash
# Verify secret exists
gh secret list

# Check workflow file uses correct name
# ${{ secrets.GHCR_TOKEN }}  ← must match exactly
```

---

## 🎯 Quick Reference

### **1Password CLI Commands**

```bash
# Sign in
eval $(op signin)

# Get document
op document get "filename" --vault "vault-name"

# Get item field
op item get "item-name" --vault "vault-name" --fields password

# Create item
op item create --category=password --title="name" --vault="vault" "password=value"
```

### **GitHub CLI Commands**

```bash
# Set secret
gh secret set SECRET_NAME --body "value"

# Set from file
gh secret set SECRET_NAME < file.txt

# List secrets
gh secret list

# Delete secret
gh secret remove SECRET_NAME
```

### **Generate Secrets**

```bash
# JWT Secret (256-bit hex)
openssl rand -hex 32

# Password (base64)
openssl rand -base64 32

# API Key (custom format)
echo "ias_$(openssl rand -hex 16)"
```

---

## 🎉 You're All Set!

Your secret management is now:
- ✅ Secure (never in Git)
- ✅ Synced (1Password across both machines)
- ✅ Automated (GitHub Actions can access secrets)
- ✅ Production-ready (Kubernetes secrets)

**Next steps:**
1. Set up local development environment (see GETTING-STARTED.md)
2. Configure GitHub Actions workflows (automatic with secrets)
3. Deploy infrastructure (Terraform will use GitHub secrets)

Need help? Check the troubleshooting section or open an issue! 🚀

