# 🚀 Universal CI/CD Playbook (Reusable & Public-Safe)

> **Maintainer:** Uzair  
> **Audience:** Future me + team members  
> **Goal:** One CI/CD reference usable across **all projects** without leaking secrets  
> **Works with:** Angular, React, Vue, Node, Docker / Non-Docker

---

## 🧠 Core Rules (Do Not Break)

1. **CI = verify & build**
2. **CD = deploy only**
3. **Production servers never build frontend**
4. **SSH access must be verified explicitly**
5. **Disk issues come before code issues**
6. **Uploads are runtime data, not code**

---

## 🌿 Branch Strategy (Standardized)

```text
dev   → development branch
main  → production branch
```

| Action | Branch |
|--------|--------|
| Feature work | dev |
| CI runs | dev, PR → main |
| Deployment | main ONLY |

- ❌ No CD on dev
- ❌ No CD on PR
- ✅ Merge to main = deploy

---

## 📁 Repository Layout (Generic)

```text
project-root/
├── backend/                # API / Server
│   └── src/
├── frontend/               # Angular / React / Vue
│   ├── src/
│   └── dist/               # Generated in CI
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
└── README.md
```

---

## 🌐 SERVER ACCESS PRE-CHECK (MANDATORY)

### 1. Check if server is reachable

```bash
ping <SERVER_IP>
nc -zv <SERVER_IP> 22
```

If port 22 fails → SSH is restricted (firewall / security group).

### 2. If firewall exists, allow SSH + GitHub traffic

```bash
sudo ufw allow ssh
sudo ufw reload
```

If cloud firewall → allow GitHub Actions IP ranges manually.

---

## 🔑 SSH KEY STRATEGY (REUSABLE & PROJECT-SAFE)

❗ **DO NOT reuse one key name for all projects**  
Each project must have its own identity, not `github-actions`.

### 🔐 Naming Convention (IMPORTANT)

Use project-specific naming:

```text
<project-name>-deploy
<project-name>-ci
```

**Examples:**
- `growth-frontend-deploy`
- `crm-backend-ci`

---

### OPTION A — SERVER → GITHUB (Deploy Keys)

Recommended when server pulls code or artifacts

**1. Login to server**

```bash
ssh <SERVER_USER>@<SERVER_IP>
```

**2. Generate project-specific key**

```bash
ssh-keygen -t ed25519 -C "<PROJECT_NAME>-deploy"
```

Press ENTER for default path.

**3. Add PUBLIC key to GitHub**

```
Repo → Settings → Deploy Keys
Title: <PROJECT_NAME>-server-key
✔ Allow write access
```

**4. Test from server**

```bash
ssh -T git@github.com
```

Expected:

```text
You've successfully authenticated
```

---

### OPTION B — GITHUB ACTIONS → SERVER (Most Used)

**1. Generate key (locally or temp machine)**

```bash
ssh-keygen -t ed25519 -C "<PROJECT_NAME>-ci"
```

**2. Add PUBLIC key to server**

```bash
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**3. Add PRIVATE key to GitHub Secrets**

```text
SSH_PRIVATE_KEY   = <PRIVATE KEY CONTENT>
SSH_HOST          = <SERVER_IP>
SSH_USER          = <SERVER_USER>
DEPLOY_PATH       = /var/www/<PROJECT_NAME>
```

⚠ **Never commit secrets.**

---

## ⚙️ CI WORKFLOW (Generic but Real)

### Purpose

- Install dependencies
- Validate builds
- Fail fast
- No SSH
- No deploy

### Example CI (`.github/workflows/ci.yml`)

```yaml
name: CI – Verify Builds

on:
  push:
    branches: [dev]
  pull_request:
    branches: [main]

jobs:
  backend:
    name: Backend check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - working-directory: <BACKEND_PATH>
        run: npm ci

  frontend:
    name: Frontend build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - working-directory: <FRONTEND_PATH>
        run: |
          npm ci
          npm run build
```

🔁 **Replace:**
- `<BACKEND_PATH>` → e.g. `api.project.com`
- `<FRONTEND_PATH>` → e.g. `admin.project.com`

---

## 🚀 CD WORKFLOW (Growth-Style, Safe & Clean)

### Rules

- Runs only on main
- Uses SSH
- No builds
- Deploys already built files

### Example CD (`.github/workflows/deploy.yml`)

```yaml
name: CD – Deploy to Server

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H $SSH_HOST >> ~/.ssh/known_hosts

      - name: Deploy frontend build
        run: |
          rsync -avz --delete \
          <FRONTEND_BUILD_PATH>/ \
          $SSH_USER@$SSH_HOST:$DEPLOY_PATH
```

🔁 **Replace:**
- `<FRONTEND_BUILD_PATH>` → `frontend/dist/<project-name>`

---

## 📦 BUILD OUTPUT VERIFICATION (CRITICAL)

### Angular

```text
dist/<project-name>/
  ├── index.html
  ├── assets/
```

### React / Vue

```text
dist/
  ├── index.html
  ├── assets/
```

- ❌ Never deploy the parent dist incorrectly
- ❌ Never deploy `dist/<project>` folder itself unless nginx expects it

---

## 💾 SERVER DISK MANAGEMENT

### Check disk

```bash
df -h
```

❌ If `/` > 85% → **STOP**

### Reclaim space

```bash
sudo apt clean
sudo journalctl --vacuum-size=100M
rm -rf node_modules dist
docker system prune -af --volumes
```

---

## 📁 UPLOADS RULE (ABSOLUTE)

**Uploads are runtime data.**

```gitignore
uploads/
public/uploads/
```

- ❌ Never commit
- ❌ Never deploy
- ❌ Never delete in CI/CD

---

## ⚠️ MERGE CONFLICT MARKERS

```text
<<<<<<< branchA
=======
>>>>>>> branchB
```

**Meaning:**
- Same lines modified in both branches
- Must be resolved manually

❌ **Never deploy unresolved conflicts**

---

## ✅ PRE-DEPLOY CHECKLIST

- [ ] SSH access verified
- [ ] Disk space OK
- [ ] Secrets added
- [ ] CI passing
- [ ] Correct build folder
- [ ] Nginx root verified

---

## 🏁 FINAL RULES

1. **Disk → SSH → CI → CD**
2. **CI builds, CD deploys**
3. **Server never compiles frontend**
4. **Every project has its own identity**
5. **Public repo = zero secrets**