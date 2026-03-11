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
│  │   POST /api/verify   → validate email → insert visitor │  │
│  │   POST /api/track    → log click event                 │  │
│  │   POST /api/contact  → save message → send email       │  │
│  │   GET  /api/admin/stats → return aggregated stats      │  │
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

### Entity Relationship

```
visitors (1) ──→ (N) click_logs
    │
    └──────────→ (N) messages
```

每个 visitor 可以有多条 click_logs 和多条 messages。

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
5. 用户浏览页面，每次点击带 data-track 属性的元素
   → script.js 发送 POST /api/track → 写入 click_logs 表
   │
   ▼
6. 用户填写 Contact 表单 → POST /api/contact
   ├── 写入 messages 表
   ├── 清除 Redis 缓存 (stats:*)
   └── 发送 Gmail 通知给网站主人
   │
   ▼
7. 网站主人访问 /api/admin/stats → 查看访客统计
   └── 先查 Redis 缓存 → 没有则查 PostgreSQL → 结果写入 Redis (60s TTL)
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
