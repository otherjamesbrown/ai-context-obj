# 🚀 Quick Setup for Machine 2

**5-minute setup guide for your other laptop.**

---

## Steps:

### 1. Clone Repository
```bash
cd ~/Documents/GitHub
git clone https://github.com/otherjamesbrown/ai-context-obj.git ai-as-a-service
cd ai-as-a-service
```

### 2. Get Secrets from 1Password

**Option A: 1Password App**
1. Open 1Password
2. Go to "IAS-Development" vault
3. Open "IAS Local .env"
4. Copy contents
5. Create `.env` file and paste

**Option B: 1Password CLI**
```bash
brew install 1password-cli
eval $(op signin)
op document get "IAS Local .env" --vault IAS-Development > .env
```

### 3. Verify
```bash
ls -la .env          # Should exist
git status           # Should NOT show .env
cat .env | head      # Should show secrets
```

---

## ✅ Done!

Your second machine is ready. Secrets are identical to Machine 1.

**Full guide:** See `SETUP-SECONDARY-MACHINE.md` (in this directory)
