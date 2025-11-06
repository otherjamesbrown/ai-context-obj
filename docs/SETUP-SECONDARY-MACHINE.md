# Secondary Machine Setup Guide

**Quick setup guide for your second laptop or any additional development machines.**

---

## 🎯 Overview

Since you've already set up secrets on your primary machine and stored them in 1Password, setting up additional machines is quick and easy!

**Time required: ~5 minutes**

---

## ✅ Prerequisites

- 1Password installed and signed in
- Git installed
- Access to the GitHub repository

---

## 🚀 Setup Steps

### **Step 1: Clone the Repository (2 minutes)**

```bash
# Navigate to your projects directory
cd ~/Documents/GitHub

# Clone the repo (use SSH or HTTPS)
# Option A: SSH (if you have SSH key set up)
git clone git@github.com:otherjamesbrown/ai-context-obj.git ai-as-a-service

# Option B: HTTPS
git clone https://github.com/otherjamesbrown/ai-context-obj.git ai-as-a-service

# Navigate into the project
cd ai-as-a-service
```

---

### **Step 2: Retrieve Secrets from 1Password (2 minutes)**

#### **Method A: 1Password App (Easiest)**

1. Open 1Password app
2. Navigate to **"IAS-Development"** vault
3. Find **"IAS Local .env"** secure note
4. Copy the entire contents
5. Create `.env` file in project root:
   ```bash
   nano .env
   # Paste the contents
   # Save: Ctrl+O, Enter, Ctrl+X
   ```

#### **Method B: 1Password CLI (Fastest)**

```bash
# Install 1Password CLI (if not installed)
brew install 1password-cli

# Sign in
eval $(op signin)

# Retrieve the .env file directly
op document get "IAS Local .env" --vault IAS-Development > .env

# Verify it worked
cat .env | head -5
```

---

### **Step 3: Verify Setup (1 minute)**

```bash
# Check that .env exists and has content
ls -la .env
# Should show: -rw-r--r--  1 username  staff  938 ...

# Verify it's gitignored (Git should NOT show it)
git status
# Should NOT list .env as untracked

# Verify secrets look correct
cat .env | grep JWT_SECRET
# Should show: JWT_SECRET=<long hex string>
```

---

## ✅ **That's It!**

Your second machine is now ready to use. The secrets are identical to your primary machine.

---

## 🔄 **If Secrets Change**

If you regenerate secrets on your primary machine:

1. **On primary machine:**
   ```bash
   # Update 1Password with new .env
   cat .env | pbcopy
   # Paste into 1Password "IAS Local .env" note
   ```

2. **On secondary machine:**
   ```bash
   # Retrieve updated .env
   eval $(op signin)
   op document get "IAS Local .env" --vault IAS-Development > .env
   
   # Restart services if running
   docker-compose restart
   ```

---

## 🛠️ **Additional Tools (Optional)**

### **Install GitHub CLI (for managing secrets)**

```bash
brew install gh
gh auth login
```

### **Install Docker Desktop (for local development)**

```bash
# Download from: https://www.docker.com/products/docker-desktop
# Or via Homebrew:
brew install --cask docker
```

### **Install Development Tools**

```bash
# Go (for building services)
brew install go

# Node.js (for web portal)
brew install node

# Terraform (for infrastructure)
brew install terraform

# kubectl (for Kubernetes)
brew install kubectl
```

---

## 📋 **Setup Verification Checklist**

- [ ] Repository cloned
- [ ] `.env` file exists in project root
- [ ] `.env` contains secrets (not CHANGEME placeholders)
- [ ] `git status` does NOT show `.env`
- [ ] Can view secrets: `cat .env | grep SECRET`

---

## 🆘 **Troubleshooting**

### **Problem: "Can't find .env in 1Password"**

**Solution:**
```bash
# On primary machine, verify it's stored:
# Open 1Password → IAS-Development vault → Should see "IAS Local .env"

# If missing, store it:
cat .env | pbcopy
# Then create secure note in 1Password
```

### **Problem: ".env has CHANGEME values"**

**Solution:**
You copied `.env.example` instead of the real `.env` from 1Password.

```bash
# Delete the incorrect file
rm .env

# Retrieve the correct one from 1Password
eval $(op signin)
op document get "IAS Local .env" --vault IAS-Development > .env
```

### **Problem: "Git shows .env as untracked"**

**Solution:**
The `.gitignore` might not be working. Verify:

```bash
# Check .gitignore exists
cat .gitignore | grep "^.env$"
# Should show: .env

# If .gitignore is missing, pull it:
git pull origin main
```

### **Problem: "1Password CLI not working"**

**Solution:**
```bash
# Reinstall 1Password CLI
brew reinstall 1password-cli

# Sign in again
eval $(op signin)

# Test retrieval
op document list --vault IAS-Development
```

---

## 🎯 **Quick Reference Commands**

```bash
# Clone repo
git clone git@github.com:otherjamesbrown/ai-context-obj.git ai-as-a-service

# Get secrets from 1Password
eval $(op signin)
op document get "IAS Local .env" --vault IAS-Development > .env

# Verify setup
ls -la .env
git status
cat .env | head

# Start development
docker-compose up -d
```

---

## 🔐 **Security Reminders**

- ✅ `.env` is protected by `.gitignore` (never committed)
- ✅ Secrets sync via 1Password (encrypted)
- ✅ Each machine has identical secrets
- ✅ No secrets in chat/email/Slack

---

## 📚 **Next Steps**

After setup is complete:

1. **Start local development:**
   ```bash
   docker-compose up -d
   ```

2. **Begin coding:**
   - See: `GETTING-STARTED.md` for local dev setup
   - See: `specs/` for implementation specs
   - See: `docs/` for documentation

---

## 💡 **Pro Tips**

### **Automated Setup Script**

Create `~/scripts/ias-setup-machine.sh`:

```bash
#!/bin/bash
# Quick setup script for new machines

set -e

echo "🚀 Setting up IAS development environment..."

# Check prerequisites
command -v op >/dev/null 2>&1 || { echo "❌ Install 1Password CLI first"; exit 1; }
command -v git >/dev/null 2>&1 || { echo "❌ Install Git first"; exit 1; }

# Clone if needed
if [ ! -d ~/Documents/GitHub/ai-as-a-service ]; then
    echo "📦 Cloning repository..."
    cd ~/Documents/GitHub
    git clone git@github.com:otherjamesbrown/ai-context-obj.git ai-as-a-service
fi

# Get secrets
cd ~/Documents/GitHub/ai-as-a-service
echo "🔐 Retrieving secrets from 1Password..."
eval $(op signin)
op document get "IAS Local .env" --vault IAS-Development > .env

# Verify
if [ -f .env ] && grep -q "JWT_SECRET" .env; then
    echo "✅ Setup complete!"
    echo "📁 Project: ~/Documents/GitHub/ai-as-a-service"
    echo "🔑 Secrets: .env file ready"
else
    echo "❌ Setup failed - check 1Password"
    exit 1
fi
```

**Usage:**
```bash
chmod +x ~/scripts/ias-setup-machine.sh
~/scripts/ias-setup-machine.sh
```

---

## 🎉 **You're All Set!**

Your secondary machine is now configured identically to your primary machine. All secrets are synced via 1Password. Happy coding! 🚀

---

**Questions?** Check the main documentation:
- `docs/QUICK-SETUP-MACHINE-2.md` - Quick 5-minute setup
- `docs/SETUP-SECRETS.md` - Complete setup guide  
- `docs/KEY-MANAGEMENT.md` - Secret management reference
- `README.md` - Project overview

