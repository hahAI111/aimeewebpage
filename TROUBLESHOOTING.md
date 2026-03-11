# Troubleshooting — 开发过程中遇到的问题与解决方案

本文档记录了这个项目从零到部署上线过程中遇到的所有重要问题，以及对应的解决方法。

---

## Issue #1: SQLite → PostgreSQL 语法不兼容

### 问题
项目最初使用 SQLite 开发，后来改为 Azure PostgreSQL。直接替换连接代码后，SQL 语句全部报错。

### 原因
| | SQLite | PostgreSQL (psycopg2) |
|---|--------|----------------------|
| 占位符 | `?` | `%s` |
| 执行方式 | `conn.execute(sql)` | `cur = conn.cursor(); cur.execute(sql)` |
| 获取自增 ID | `cur.lastrowid` | `INSERT ... RETURNING id` + `cur.fetchone()[0]` |
| 字典行 | `conn.row_factory = sqlite3.Row` | `cursor_factory=psycopg2.extras.RealDictCursor` |

### 解决
逐个修改所有 route 函数中的 SQL 语法：
- `?` → `%s`
- `conn.execute()` → `cur = conn.cursor()` + `cur.execute()` + `cur.close()`
- `cur.lastrowid` → `RETURNING id` + `cur.fetchone()[0]`
- Admin stats 用 `RealDictCursor` 获取字典格式结果

---

## Issue #2: Azure Connection String 格式不匹配 → 504 Gateway Timeout

### 问题
代码推送到 Azure 后，网站返回 **504 Gateway Timeout**。

### 原因
Azure "Web App + Database" 自动生成的 PostgreSQL Connection String 是 **ADO.NET 格式**（给 C#/.NET 用的）：
```
Database=aimeelan-database;Server=aimeelan-server.postgres.database.azure.com;User Id=prjxaadsjr;Password=xxx
```

但 Python 的 psycopg2 需要 **libpq 格式**：
```
host=aimeelan-server.postgres.database.azure.com dbname=aimeelan-database user=prjxaadsjr password=xxx
```

同样，Azure Redis Connection String 格式也不兼容：
```
# Azure 给的格式
aimee-cache.redis.cache.windows.net:6380,password=xxx,ssl=True,abortConnect=False

# Python redis-py 需要的格式
rediss://:xxx@aimee-cache.redis.cache.windows.net:6380/0
```

gunicorn worker 启动时连接数据库失败 → worker 启动超时 → Azure 前端代理返回 504。

### 解决
写了两个解析函数自动转换格式：

```python
def _parse_pg_conn(raw):
    """Convert ADO.NET format to libpq format."""
    if not raw or raw.startswith("host="):
        return raw
    parts = dict(p.split("=", 1) for p in raw.split(";") if "=" in p)
    return f"host={parts['Server']} dbname={parts['Database']} user={parts['User Id']} password={parts['Password']}"

def _parse_redis_conn(raw):
    """Convert Azure Redis format to rediss:// URL."""
    if not raw or raw.startswith("redis"):
        return raw
    parts = dict(p.split("=", 1) for p in raw.split(",") if "=" in p)
    host_port = raw.split(",")[0]
    password = parts.get("password", "")
    use_ssl = parts.get("ssl", "False").lower() == "true"
    scheme = "rediss" if use_ssl else "redis"
    return f"{scheme}://:{password}@{host_port}/0"
```

### 教训
Azure 的 "Web App + Database" 模板面向 .NET 生态，生成的连接字符串格式对 Python 不友好。使用非 .NET 语言时，需要手动解析。

---

## Issue #3: Azure Connection String 环境变量前缀

### 问题
代码中用 `os.environ.get("AZURE_POSTGRESQL_CONNECTIONSTRING")` 读取连接字符串，但在 Azure 上读不到值。

### 原因
Azure App Service 的 **Connection Strings**（在 Environment variables → Connection strings 标签页配置的）在注入为环境变量时，会自动加前缀：

| Connection String Type | 环境变量前缀 |
|----------------------|------------|
| Custom | `CUSTOMCONNSTR_` |
| SQL Server | `SQLCONNSTR_` |
| SQL Azure | `SQLAZURECONNSTR_` |
| MySQL | `MYSQLCONNSTR_` |
| Redis Cache | `REDISCACHECONNSTR_` |
| PostgreSQL | `POSTGRESQLCONNSTR_` |

所以 `AZURE_POSTGRESQL_CONNECTIONSTRING`（Custom 类型）实际的环境变量名是：
```
CUSTOMCONNSTR_AZURE_POSTGRESQL_CONNECTIONSTRING
```

而 Redis 的名字 `azure_redis_cache`（Redis Cache 类型）实际是：
```
REDISCACHECONNSTR_azure_redis_cache
```

### 解决
代码中同时检查两种名字：
```python
_raw_pg = (
    os.environ.get("AZURE_POSTGRESQL_CONNECTIONSTRING")
    or os.environ.get("CUSTOMCONNSTR_AZURE_POSTGRESQL_CONNECTIONSTRING")
    or "host=localhost ..."
)
```

### 教训
Azure App Settings vs Connection Strings 是两个不同的东西：
- **App Settings** → 直接变成环境变量，名字不变
- **Connection Strings** → 变成环境变量时加前缀

建议用 App Settings 存，这样代码更简单。但 Azure "Web App + Database" 自动创建的放在 Connection Strings 里，所以我们都要兼容。

---

## Issue #4: Azure PostgreSQL 必须 SSL 连接

### 问题
即使连接字符串正确了，psycopg2 连接 Azure PostgreSQL 仍然失败。

### 原因
Azure PostgreSQL Flexible Server 默认强制 SSL 连接。本地 psycopg2 默认不使用 SSL。

### 解决
在连接时加上 `sslmode="require"`：
```python
def get_db():
    conn = psycopg2.connect(DATABASE_URL, sslmode="require")
    return conn
```

---

## Issue #5: GitHub Push Rejected — Remote Contains Work

### 问题
第一次 `git push` 被拒绝：
```
! [rejected] main -> main (fetch first)
error: failed to push some refs
```

### 原因
在 Azure 门户配置 GitHub Actions 部署时，Azure 自动向 GitHub repo 推送了：
- `.github/workflows/main_aimeelan.yml`（CI/CD workflow 文件）
- `LICENSE`

本地仓库是全新 `git init` 的，和远程没有共同 commit 历史。

### 解决
```bash
git pull origin main --allow-unrelated-histories
# 合并两个不相关的历史
git push
```

`--allow-unrelated-histories` 告诉 git "我知道这两个分支没有共同祖先，强制合并"。

---

## Issue #6: PowerShell 的 curl 不是真正的 curl

### 问题
用 `curl` 命令测试网站时报错：
```
Missing an argument for parameter 'SessionVariable'
```

### 原因
Windows PowerShell 里 `curl` 是 `Invoke-WebRequest` 的别名，不是 Linux 的 curl。参数格式完全不同。

### 解决
改用 PowerShell 原生语法：
```powershell
Invoke-WebRequest -Uri "https://aimeelan.azurewebsites.net" -UseBasicParsing
```

---

## Issue #7: Gmail App Password 配置

### 问题
需要通过 Gmail SMTP 发送邮件通知，但 Google 不允许直接用 Gmail 密码。

### 原因
Google 要求开启 2-Step Verification 后，使用 **App Password**（16 位专用密码）代替账号密码。

### 解决步骤
1. Google Account → Security → 开启 **2-Step Verification**
2. 访问 https://myaccount.google.com/apppasswords
3. 输入 App 名称 → Generate → 得到 16 位密码
4. 在 Azure App Settings 中设置 `SMTP_PASS` = 该 16 位密码

**注意事项：**
- App Password 只显示一次，需要立即保存
- 一定不要把 App Password 写在代码里，只放在 Azure 环境变量中
- 如果 2FA 没开启，App Passwords 页面不会出现

---

## 问题排查清单

如果网站再出问题，按以下顺序检查：

| Step | Check | Command / Location |
|------|-------|--------------------|
| 1 | GitHub Actions 是否成功 | https://github.com/hahAI111/aimeewebpage/actions |
| 2 | Azure App Service 是否 Running | Azure Portal → aimeelan → Overview |
| 3 | Startup Command 是否正确 | Configuration → General settings → `gunicorn --bind=0.0.0.0:8000 --timeout 600 app:app` |
| 4 | 环境变量是否完整 | Environment variables → App settings + Connection strings |
| 5 | 查看应用日志 | `az webapp log tail --name aimeelan --resource-group aimee-test-env` |
| 6 | PostgreSQL 是否可达 | VNet Integration + Private Endpoint 是否正常 |
| 7 | Redis 是否可达 | Azure Portal → aimee-cache → Activity log |
