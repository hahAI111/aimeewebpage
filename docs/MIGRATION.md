# Migration & Self-Deployment Guide

What to do when your company Azure account is no longer available — how to back up everything, migrate to a new platform, and deploy the website yourself.

---

## Table of Contents

- [What Will You Lose?](#what-will-you-lose)
- [Pre-Departure Checklist](#pre-departure-checklist)
  - [1. Backup PostgreSQL Data](#1-backup-postgresql-data)
  - [2. Export Environment Variables](#2-export-environment-variables)
  - [3. Clone Code to Personal Device](#3-clone-code-to-personal-device)
  - [4. Confirm GitHub Account Ownership](#4-confirm-github-account-ownership)
- [How Code Becomes a Website](#how-code-becomes-a-website)
- [Method 1: Run Locally](#method-1-run-locally)
- [Method 2: Deploy to Railway (Recommended)](#method-2-deploy-to-railway-recommended)
- [Method 3: Deploy to Render](#method-3-deploy-to-render)
- [Method 4: Personal Azure Account](#method-4-personal-azure-account)
- [Platform Comparison](#platform-comparison)
- [Architecture: Before vs After](#architecture-before-vs-after)
- [FAQ](#faq)

---

## What Will You Lose?

| Component | What Happens | Need Backup? | Recovery Plan |
|-----------|-------------|-------------|---------------|
| **App Service** | Website goes offline | No — code is on GitHub, just redeploy | Deploy to new platform |
| **PostgreSQL** | All data lost (visitors, blog posts, messages, analytics) | **YES — CRITICAL** | `pg_dump` before leaving |
| **Redis** | Cache data lost | No — Redis is purely a cache layer | Rebuilds automatically on first request |
| **GitHub Actions** | Azure deploy pipeline stops working | No — workflow file is in the repo | Update for new platform or delete |
| **Code** | Safe on GitHub | No (already backed up) | `git clone` to any computer |

### Why Redis Doesn't Need Backup

Your code treats Redis as a **read-through cache**. When Redis is empty (or unavailable):

```
Request → cache_get(key) → Returns None (miss)
       → Query PostgreSQL → Get fresh data
       → cache_set(key, data) → Redis rebuilds itself
```

Every cache entry auto-rebuilds on the next visit. The `cache_get` / `cache_set` / `cache_delete` functions all have try/except — if Redis doesn't exist at all, the website still works, just slightly slower.

---

## Pre-Departure Checklist

### 1. Backup PostgreSQL Data

This is the **most important step**. Your database contains irreplaceable data.

**Option A: Full database dump (recommended)**

```powershell
pg_dump "host=aimeelan-server.postgres.database.azure.com dbname=aimeelan-database user=prjxaadsjr password=YOUR_PASSWORD sslmode=require" > backup.sql
```

This creates a single `.sql` file containing all table schemas + all data. You can restore it on any PostgreSQL server with:

```bash
psql "YOUR_NEW_DATABASE_URL" < backup.sql
```

**Option B: CSV export via the website's admin API**

```
GET https://aimeelan.azurewebsites.net/api/admin/export/visitors
GET https://aimeelan.azurewebsites.net/api/admin/export/click_logs
GET https://aimeelan.azurewebsites.net/api/admin/export/messages
GET https://aimeelan.azurewebsites.net/api/admin/export/page_views
```

Note: CSV export only covers 4 tables. `pg_dump` covers all 9 tables including blog posts, tags, and projects.

**Save the backup to**: Personal email, personal cloud drive, or USB — anywhere your company can't revoke access.

### 2. Export Environment Variables

Run this command to see all current settings:

```powershell
az webapp config appsettings list --name aimeelan --resource-group aimee-test-env --output table
```

Save these values to a file on your personal device:

```env
# Flask
SECRET_KEY=your-secret-key-value

# Admin
ADMIN_USER=admin
ADMIN_PASS_HASH=your-bcrypt-hash

# Email notifications
OWNER_EMAIL=your-email@example.com
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-gmail@gmail.com
SMTP_PASS=your-16-char-app-password

# GitHub integration
GITHUB_USERNAME=hahAI111
GITHUB_TOKEN=your-github-token

# Database & Redis (these will change on new platform)
AZURE_POSTGRESQL_CONNECTIONSTRING=...
AZURE_REDIS_CONNECTIONSTRING=...
```

### 3. Clone Code to Personal Device

```bash
git clone https://github.com/hahAI111/aimeewebpage.git
```

### 4. Confirm GitHub Account Ownership

Go to [github.com/settings/emails](https://github.com/settings/emails):
- Is the primary email your **personal** email? If it's a company email, add a personal one and set it as primary.
- Enable 2FA with a personal phone number, not a company device.

---

## How Code Becomes a Website

```
Your code (app.py + HTML/CSS/JS)
        │
        ▼
A computer runs Flask server (python app.py or gunicorn)
        │
        ▼
Server listens on a port (e.g., port 8000)
        │
        ▼
User's browser visits http://that-computer's-address:8000
        │
        ▼
Flask receives request → Returns HTML → Browser renders → User sees the website
```

**In one sentence:** A computer runs `app.py` 24/7, and anyone who knows its address can visit the website.

| Who runs it? | Address | Who can see it? |
|-------------|---------|----------------|
| Your laptop | `localhost:5000` | Only you |
| Azure App Service | `aimeelan.azurewebsites.net` | Everyone |
| Railway | `yourapp.up.railway.app` | Everyone |
| Any cloud server | `your-domain.com` | Everyone |

### Current Setup: How Azure Deploys Automatically

```
You: git push origin main
        │
        ▼
GitHub Actions triggers (.github/workflows/main_aimeelan.yml)
        │
        ▼
Actions: pip install -r requirements.txt → zip the code
        │
        ▼
Actions: Upload zip to Azure App Service
        │
        ▼
Azure: Runs gunicorn --bind=0.0.0.0:8000 --timeout 600 app:app
        │
        ▼
Users visit https://aimeelan.azurewebsites.net → Flask handles requests
```

---

## Method 1: Run Locally

The simplest way — good for development and testing.

```powershell
# 1. Clone the repo
git clone https://github.com/hahAI111/aimeewebpage.git
cd aimeewebpage

# 2. Create virtual environment
python -m venv venv
.\venv\Scripts\Activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set environment variables
$env:SECRET_KEY = "any-random-string-here"
$env:ADMIN_PASS = "admin123"

# 5. Start PostgreSQL locally (if you have Docker)
docker run -d --name pg -p 5432:5432 -e POSTGRES_DB=portfoliodb -e POSTGRES_PASSWORD=postgres postgres:15

# 6. Run Flask
python app.py
```

Open browser → `http://localhost:5000` → Website is live!

**Limitation:** Only you can see it. Stops when you close the terminal.

---

## Method 2: Deploy to Railway (Recommended)

Railway gives you Flask + PostgreSQL + Redis for free (within $5/month credit).

### Step 1: Sign Up

Go to [railway.app](https://railway.app) → Sign in with GitHub.

### Step 2: Create Project

1. Click **New Project**
2. Select **Deploy from GitHub Repo**
3. Choose `hahAI111/aimeewebpage`
4. Railway auto-detects Python, installs `requirements.txt`, starts the app

### Step 3: Add Databases

1. In your project, click **+ New → Database → PostgreSQL**
2. Click **+ New → Database → Redis**
3. Railway auto-generates connection URLs

### Step 4: Set Environment Variables

Go to your web service → **Variables** tab → Add:

```
SECRET_KEY=generate-a-random-string
ADMIN_USER=admin
ADMIN_PASS_HASH=your-saved-bcrypt-hash
OWNER_EMAIL=your@email.com
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-gmail@gmail.com
SMTP_PASS=your-app-password
GITHUB_USERNAME=hahAI111
```

For database connections, Railway provides environment variables automatically. Link them:
- `AZURE_POSTGRESQL_CONNECTIONSTRING` → Reference Railway's `DATABASE_URL`
- `AZURE_REDIS_CONNECTIONSTRING` → Reference Railway's `REDIS_URL`

### Step 5: Set Start Command

Go to **Settings → Start Command**:

```bash
gunicorn --bind=0.0.0.0:$PORT --timeout 600 app:app
```

### Step 6: Import Data

Get your Railway PostgreSQL URL from the **Variables** tab, then:

```bash
psql "postgresql://user:pass@host:port/railway" < backup.sql
```

### Step 7: Done!

Railway gives you a URL like `aimeewebpage-production.up.railway.app`.

**Auto-deploy:** Every `git push` to `main` automatically redeploys — same as Azure.

---

## Method 3: Deploy to Render

### Step 1: Sign Up

Go to [render.com](https://render.com) → Sign in with GitHub.

### Step 2: Create Web Service

1. Click **New → Web Service**
2. Connect `hahAI111/aimeewebpage`
3. Settings:
   - **Runtime:** Python
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `gunicorn --bind=0.0.0.0:$PORT --timeout 600 app:app`

### Step 3: Add PostgreSQL

1. Click **New → PostgreSQL**
2. Copy the **Internal Database URL**
3. Add as environment variable `AZURE_POSTGRESQL_CONNECTIONSTRING`

### Step 4: Add Redis

1. Click **New → Redis**
2. Copy the **Internal Redis URL**
3. Add as environment variable `AZURE_REDIS_CONNECTIONSTRING`

### Step 5: Set Other Environment Variables

Same as Railway — `SECRET_KEY`, `ADMIN_USER`, etc.

### Step 6: Import Data & Go Live

```bash
psql "render-postgresql-url" < backup.sql
```

---

## Method 4: Personal Azure Account

If you want to keep the exact same setup:

1. Register a personal Azure account at [azure.microsoft.com/free](https://azure.microsoft.com/en-us/free/) (12 months free tier)
2. Create a new Resource Group
3. Create App Service (Linux, Python)
4. Create PostgreSQL Flexible Server
5. Create Redis Cache (optional — can skip to save money)
6. Import data: `psql < backup.sql`
7. Set environment variables in App Service → Configuration
8. Update GitHub Actions: replace the `publish-profile` secret with the new App Service's publish profile
9. `git push` → auto-deploys

---

## Platform Comparison

| Feature | Azure (Current) | Railway | Render | Personal Azure |
|---------|----------------|---------|--------|---------------|
| **Flask hosting** | App Service ~$13/mo | Free ($5 credit) | Free (750 hrs/mo) | Free tier 12 months |
| **PostgreSQL** | Flexible Server ~$15/mo | Free (1GB) | Free (1GB, 90 days) | ~$15/mo after free tier |
| **Redis** | Cache ~$16/mo | Free (25MB) | Free (25MB) | ~$16/mo |
| **Monthly cost** | ~$44/mo | **$0** (within free tier) | **$0** (within free tier) | $0 → ~$44/mo |
| **Auto-deploy from GitHub** | Yes (GitHub Actions) | Yes (built-in) | Yes (built-in) | Yes (GitHub Actions) |
| **Custom domain** | Yes | Yes | Yes | Yes |
| **SSL/HTTPS** | Auto | Auto | Auto | Auto |
| **Code changes needed** | — | None or minimal | None or minimal | None |

### Why Your Code Works Everywhere

The connection string parsers in `app.py` handle multiple formats:

```python
def _parse_pg_conn(raw):
    if not raw or raw.startswith("host="):
        return raw                    # Standard libpq format (local dev)
    # ... converts ADO.NET format    # Azure format
    
def _parse_redis_conn(raw):
    if not raw or raw.startswith("redis"):
        return raw                    # Standard redis:// URL (Railway, Render)
    # ... converts Azure format      # Azure format
```

Railway and Render provide standard URL formats (`postgresql://...`, `redis://...`) — your parsers already handle these. **Zero code changes needed.**

---

## Architecture: Before vs After

```
====== Current: Azure (Company Account) ======

Browser → Azure App Service (aimeelan)
               ├→ Azure PostgreSQL (aimeelan-server)    ← Data here
               └→ Azure Redis (aimee-cache)             ← Just cache

====== After: Railway (Personal Account) ======

Browser → Railway Web Service
               ├→ Railway PostgreSQL                    ← Data restored from backup.sql
               └→ Railway Redis                         ← Auto-rebuilds from DB

Same code. Same features. Different infrastructure.
```

---

## FAQ

**Q: Do I need to change any code to switch platforms?**
A: Almost certainly not. Your `_parse_pg_conn()` and `_parse_redis_conn()` already handle standard URL formats used by Railway/Render. Just set the right environment variables.

**Q: What if I don't set up Redis on the new platform?**
A: The website works perfectly without Redis. All `cache_get()` calls return `None`, so every request goes directly to PostgreSQL. The site is slightly slower but 100% functional.

**Q: How do I restore my blog posts and visitor data?**
A: `psql "new-database-url" < backup.sql` — this restores all 9 tables (visitors, posts, tags, projects, etc.) with all their data.

**Q: Will the GitHub auto-sync for projects still work?**
A: Yes. The background sync thread (`_github_sync_loop`) runs inside the Flask process. As long as `GITHUB_USERNAME` is set, it syncs every 6 hours automatically.

**Q: What about the email notifications?**
A: Gmail SMTP works from anywhere. Just keep the same `SMTP_USER` and `SMTP_PASS` (App Password). Make sure 2FA stays enabled on your Google account.

**Q: Can I use a custom domain name?**
A: Yes. All platforms (Railway, Render, Azure) support custom domains. Buy a domain (~$12/year) from Namecheap/Cloudflare, point DNS to your new platform, and it's done.

**Q: What is the absolute minimum I need to run this website?**
A: Python + PostgreSQL + the code. That's it. Redis is optional. Email is optional. GitHub sync is optional. The core website works with just Flask + a PostgreSQL database.
