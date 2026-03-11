# Architecture — Aimee's Portfolio

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                             │
│                  (https://aimeelan.azurewebsites.net)       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Azure App Service (Linux, Python 3.14)          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │   gunicorn  →  Flask (app.py)                          │  │
│  │                                                        │  │
│  │   Routes:                                              │  │
│  │   GET  /             → verify.html / index.html        │  │
│  │   GET  /blog         → blog listing                    │  │
│  │   GET  /blog/<slug>  → individual post                 │  │
│  │   GET  /projects     → GitHub project showcase         │  │
│  │   GET  /admin        → admin dashboard (login req.)    │  │
│  │   POST /api/verify   → validate email → insert visitor │  │
│  │   POST /api/pageview → record page view analytics      │  │
│  │   POST /api/track    → log click event                 │  │
│  │   POST /api/contact  → save message → send email       │  │
│  │   GET  /api/posts    → blog posts (pagination, tags)   │  │
│  │   GET  /api/projects → synced GitHub repos             │  │
│  │   POST /api/admin/*  → login, stats, export, retention │  │
│  └────────────┬──────────────────┬────────────────────────┘  │
│               │  VNet Integration│                            │
└───────────────┼──────────────────┼────────────────────────────┘
                │                  │
        ┌───────▼───────┐  ┌──────▼────────┐
        │   PostgreSQL   │  │  Redis Cache   │
        │  (Flexible     │  │  (Basic SKU)   │
        │   Server)      │  │               │
        │  Port 5432     │  │  Port 6380    │
        │  SSL required  │  │  SSL required │
        └───────────────┘  └───────────────┘
             ▲                    ▲
             │                    │
         Private Endpoint      Private Endpoint
         (VNet 内网访问)        (VNet 内网访问)

                                        ┌──────────────┐
                Flask send_notification ─►  Gmail SMTP   │
                                        │ smtp.gmail.com│
                                        │   Port 587    │
                                        └──────────────┘
```

## Azure Resource List

| Resource | Name | Type | Region | Purpose |
|----------|------|------|--------|---------|
| Web App | `aimeelan` | App Service (Linux, Python 3.14) | Canada Central | Hosts Flask application |
| PostgreSQL | `aimeelan-server` | Flexible Server | Canada Central | Persistent storage |
| Database | `aimeelan-database` | PostgreSQL DB | — | App database on the above server |
| Redis | `aimee-cache` | Azure Cache for Redis (Basic) | Canada Central | Response caching |
| VNet | `vnet-zqopjmgp` | Virtual Network | Canada Central | Private networking |
| Resource Group | `aimee-test-env` | — | — | Contains all resources |

## Networking Architecture

```
┌─── Azure VNet (vnet-zqopjmgp) ─────────────────────────┐
│                                                          │
│  ┌─ Subnet: web ──────────────────────────────────────┐  │
│  │  App Service (VNet Integration)                    │  │
│  │  ← 出站流量走 VNet                                  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌─ Subnet: db ───────────────────────────────────────┐  │
│  │  PostgreSQL Flexible Server (Private Endpoint)     │  │
│  │  ← 只接受 VNet 内部连接，公网不可访问                  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌─ Subnet: redis ────────────────────────────────────┐  │
│  │  Redis Cache (Private Endpoint)                    │  │
│  │  ← 只接受 VNet 内部连接                              │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Key Concepts:**
- **VNet Integration (出站)**: App Service 的出站流量（连数据库、连 Redis）走 VNet 内网，不走公网
- **Private Endpoint (入站)**: PostgreSQL 和 Redis 通过 Private Endpoint 暴露在 VNet 内，外部网络无法直接访问
- **公网只暴露 Web App**: 只有 `aimeelan.azurewebsites.net` 对外开放（Azure 自带 SSL）

---

## Database Design (PostgreSQL)

### Tables

#### `visitors` — 访客信息
记录每一个通过验证页面进入网站的访客。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `name` | TEXT | Visitor's name |
| `email` | TEXT | Visitor's email (validated, blocked domains rejected) |
| `verified` | INTEGER | 1 = verified |
| `token` | TEXT | Unique session token |
| `created_at` | TIMESTAMP | Registration time |

#### `click_logs` — 点击追踪
记录已验证用户在页面上的每次点击行为（比如点了导航栏的 "About"、点了 LinkedIn 链接等）。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `visitor_id` | INTEGER FK → visitors(id) | Who clicked |
| `element` | TEXT | What was clicked (e.g. `nav-about`, `social-linkedin`) |
| `page` | TEXT | Which page they were on |
| `clicked_at` | TIMESTAMP | When they clicked |

#### `messages` — 留言/联系
访客通过 Contact 表单发送给网站主人的消息。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `visitor_id` | INTEGER FK → visitors(id) | Who sent it |
| `name` | TEXT | Sender name |
| `email` | TEXT | Sender email |
| `message` | TEXT | Message content |
| `sent_at` | TIMESTAMP | When it was sent |

#### `page_views` — 页面浏览追踪
记录每次页面访问的详细信息，用于分析访客行为。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `visitor_id` | INTEGER FK → visitors(id) | Who viewed (nullable for anonymous) |
| `page` | TEXT | URL path visited |
| `referrer` | TEXT | Where they came from |
| `user_agent` | TEXT | Browser/device info |
| `ip_hash` | TEXT | SHA-256 hash of IP (privacy) |
| `duration_sec` | INTEGER | Time spent on page |
| `screen_width` | INTEGER | Screen width in px (device classification) |
| `created_at` | TIMESTAMP | When the page was viewed |

#### `visitor_sessions` — 访客会话
追踪单次访问的会话生命周期。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `visitor_id` | INTEGER FK → visitors(id) | Session owner |
| `session_token` | TEXT UNIQUE | Random session identifier |
| `started_at` | TIMESTAMP | Session start |
| `ended_at` | TIMESTAMP | Last activity |
| `page_count` | INTEGER | Pages viewed in session |

#### `posts` — 博客文章
存储 Markdown 格式的博客文章。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `slug` | TEXT UNIQUE | URL-friendly identifier |
| `title` | TEXT | Post title |
| `summary` | TEXT | Short description |
| `content` | TEXT | Full Markdown content |
| `status` | TEXT | `published` or `draft` |
| `views` | INTEGER | View counter |
| `published_at` | TIMESTAMP | Publish date |
| `updated_at` | TIMESTAMP | Last edit |
| `created_at` | TIMESTAMP | Creation time |

#### `tags` — 标签

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `name` | TEXT UNIQUE | Tag name (e.g. `Azure`, `Python`) |

#### `post_tags` — 文章标签关联 (多对多)

| Column | Type | Description |
|--------|------|-------------|
| `post_id` | INTEGER FK → posts(id) | Post reference |
| `tag_id` | INTEGER FK → tags(id) | Tag reference |

#### `projects` — GitHub 项目展示
从 GitHub API 同步的仓库信息。

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL PK | Auto-increment ID |
| `github_repo` | TEXT UNIQUE | Full repo name (e.g. `hahAI111/aimeewebpage`) |
| `name` | TEXT | Repo name |
| `description` | TEXT | Repo description |
| `language` | TEXT | Primary language |
| `stars` | INTEGER | Star count |
| `forks` | INTEGER | Fork count |
| `open_issues` | INTEGER | Open issue count |
| `homepage` | TEXT | Homepage URL |
| `last_commit_at` | TIMESTAMP | Last push time |
| `featured` | BOOLEAN | Highlighted project |
| `synced_at` | TIMESTAMP | Last sync time |

### Entity Relationship

```
visitors (1) ──→ (N) click_logs
    │
    ├──────────→ (N) messages
    │
    ├──────────→ (N) page_views
    │
    └──────────→ (N) visitor_sessions

posts (N) ←──→ (N) tags    (via post_tags)

projects (standalone, synced from GitHub API)
```

每个 visitor 可以有多条 click_logs、messages、page_views 和 visitor_sessions。  
posts 和 tags 是多对多关系，通过 post_tags 中间表连接。  
projects 独立于访客系统，通过 admin 触发 GitHub API 同步。

---

## Redis Cache — 作用与策略

Redis 在这个项目里的作用是 **缓存 Admin Stats API 的结果**，减少不必要的数据库查询。

### 工作流程

```
GET /api/admin/stats
        │
        ▼
  cache_get("stats:overview")
        │
   ┌────┴────┐
   │ 有缓存？ │
   └────┬────┘
    Yes │      No
    ┌───▼──┐  ┌──▼────────────┐
    │返回   │  │查询 PostgreSQL │
    │缓存   │  │  COUNT(*)×3   │
    │数据   │  │  + 最近10条    │
    └──────┘  └──────┬────────┘
                     │
                     ▼
              cache_set("stats:overview", data, ttl=60)
                     │
                     ▼
                  返回数据
```

### 缓存失效
- **TTL**: 60 秒后自动过期
- **主动清除**: 当有新 message 写入时，调用 `cache_delete("stats:*")` 清除所有 stats 缓存
- **降级**: 如果 Redis 连不上，直接查库，不影响功能

### 为什么用 Redis 而不是内存缓存？
- Azure App Service 可能有多个 worker 进程，内存缓存不共享
- Redis 是独立服务，所有 worker 共享同一份缓存
- 重启 App Service 后缓存依然在（直到 TTL 过期）

---

## Data Flow — 完整用户旅程

```
1. 用户访问 https://aimeelan.azurewebsites.net
   │
   ▼
2. Flask 检查 session → 未验证 → 返回 verify.html
   │
   ▼
3. 用户输入 Name + Email → POST /api/verify
   │
   ├── 邮箱格式验证 (regex)
   ├── 屏蔽域名检查 (BLOCKED_DOMAINS)
   ├── 写入 PostgreSQL visitors 表
   └── 设置 session: verified=True, visitor_id, name, email
   │
   ▼
4. 页面跳转到 / → Flask 检查 session → 已验证 → 返回 index.html
   │
   ▼
5. script.js 自动发送 POST /api/pageview（页面、referrer、屏幕宽度、UA）
   → 写入 page_views 表 + 创建/更新 visitor_sessions
   │
   ▼
6. 用户浏览页面，每次点击带 data-track 属性的元素
   → script.js 发送 POST /api/track → 写入 click_logs 表
   │
   ▼
7. 用户可以访问 /blog → 浏览博客文章（5 篇预置文章关于 Azure AI Support）
   → GET /api/posts 支持标签筛选和分页
   → GET /api/posts/<slug> 获取完整 Markdown 内容，前端用 marked.js 渲染
   │
   ▼
8. 用户可以访问 /projects → 查看从 GitHub 同步的项目
   → GET /api/projects 返回项目列表（语言、stars、forks 等）
   │
   ▼
9. 用户填写 Contact 表单 → POST /api/contact
   ├── 写入 messages 表
   ├── 清除 Redis 缓存 (stats:*)
   └── 发送 Gmail 通知给网站主人
   │
   ▼
10. 网站主人访问 /admin → 登录后查看完整数据仪表盘
    ├── POST /api/admin/login (bcrypt 密码验证)
    ├── GET /api/admin/stats → KPI 卡片 + 6 个 Chart.js 图表
    ├── GET /api/admin/visitors → 分页访客列表 + 域名筛选
    ├── GET /api/admin/retention → 留存分析（Day 0/1/7/30 cohorts）
    ├── GET /api/admin/export/<table> → CSV 下载
    └── POST /api/projects/sync → 一键同步 GitHub 仓库
```

---

## Environment Variables

| Variable | Source | Description |
|----------|--------|-------------|
| `AZURE_POSTGRESQL_CONNECTIONSTRING` | Azure auto-set (Connection Strings) | PostgreSQL connection info |
| `REDISCACHECONNSTR_azure_redis_cache` | Azure auto-set (Connection Strings) | Redis connection info |
| `OWNER_EMAIL` | Manual (App Settings) | Email to receive notifications |
| `SMTP_SERVER` | Manual (App Settings) | `smtp.gmail.com` |
| `SMTP_PORT` | Manual (App Settings) | `587` |
| `SMTP_USER` | Manual (App Settings) | Gmail address |
| `SMTP_PASS` | Manual (App Settings) | Gmail App Password (16-char) |
| `SECRET_KEY` | Manual (App Settings) | Flask session encryption key |
| `ADMIN_USER` | Manual (App Settings) | Admin login username (default: `admin`) |
| `ADMIN_PASS` or `ADMIN_PASS_HASH` | Manual (App Settings) | Admin password (plain text or bcrypt hash) |
| `GITHUB_USERNAME` | Manual (App Settings) | GitHub user for project sync (default: `hahAI111`) |
| `GITHUB_TOKEN` | Manual (App Settings) | GitHub PAT for higher API rate limit (optional) |

**Note:** Azure Connection Strings are injected as environment variables with prefixes:
- Custom type → `CUSTOMCONNSTR_` prefix
- Redis type → `REDISCACHECONNSTR_` prefix

Our code checks both prefixed and unprefixed names for compatibility.

---

## CI/CD Pipeline

```
git push origin main
        │
        ▼
GitHub Actions (.github/workflows/main_aimeelan.yml)
        │
        ├── 1. Checkout code
        ├── 2. Setup Python 3.14
        ├── 3. pip install -r requirements.txt (验证构建)
        ├── 4. Upload artifact (排除 venv)
        │
        ▼
        ├── 5. Azure Login (OIDC, federated credential)
        └── 6. Deploy to Azure App Service (Oryx build)
                │
                ▼
            Oryx 在 Azure 上再次 pip install
                │
                ▼
            gunicorn --bind=0.0.0.0:8000 app:app
                │
                ▼
            Site live at https://aimeelan.azurewebsites.net
```
