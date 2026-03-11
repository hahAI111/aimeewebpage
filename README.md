# Aimee's Portfolio Website

A full-stack personal portfolio website with visitor verification, click tracking, contact form, and email notifications.

**Live Site:** https://aimeelan.azurewebsites.net

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Backend | Python Flask | Web framework, API routes, session management |
| Database | Azure PostgreSQL Flexible Server | Persistent data: visitors, click logs, messages |
| Cache | Azure Cache for Redis | Cache admin stats, reduce DB queries |
| Frontend | HTML / CSS / JavaScript | Dark-theme portfolio UI, SPA-like experience |
| Hosting | Azure App Service (Linux) | Production hosting with managed SSL |
| CI/CD | GitHub Actions | Auto build & deploy on `git push` |
| Email | Gmail SMTP | Notify site owner when visitors send messages |

## Project Structure

```
my-website/
├── app.py                  # Flask backend (all API routes + DB logic)
├── requirements.txt        # Python dependencies
├── .gitignore
├── static/
│   ├── verify.html         # Entry gate — name + email verification
│   ├── index.html          # Main portfolio page (after verification)
│   ├── style.css           # Dark theme with purple accents
│   └── script.js           # Click tracking, contact form, animations
└── .github/
    └── workflows/
        └── main_aimeelan.yml   # GitHub Actions CI/CD pipeline
```

## Features

- **Visitor Verification** — Visitors must enter name + email before viewing the portfolio
- **Anti-Phishing** — Disposable/temporary email domains (mailinator, yopmail, etc.) are blocked
- **Click Tracking** — Every navigation click is logged to PostgreSQL with visitor ID
- **Contact Form** — Visitors can send messages, saved to DB + email notification to owner
- **Admin Stats API** — `/api/admin/stats` returns visitor count, click count, recent messages
- **Redis Caching** — Admin stats are cached for 60s, auto-invalidated on new messages

## Quick Start (Local Development)

```bash
# 1. Clone
git clone https://github.com/hahAI111/aimeewebpage.git
cd aimeewebpage

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run (uses local PostgreSQL by default)
python app.py
# → http://localhost:5000
```

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | — | Serves verify.html or index.html based on session |
| POST | `/api/verify` | — | Verify visitor (name + email) |
| POST | `/api/track` | Verified | Log a click event |
| POST | `/api/contact` | Verified | Send a message to site owner |
| GET | `/api/admin/stats` | — | View visitor/click/message stats |

## Deployment

Code is auto-deployed via GitHub Actions. Just push to `main`:

```bash
git add -A
git commit -m "your change"
git push
```

GitHub Actions will build and deploy to Azure App Service automatically.

## Related Docs

- [ARCHITECTURE.md](ARCHITECTURE.md) — System architecture, database design, data flow diagrams
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — Issues we encountered during development and how we solved them
