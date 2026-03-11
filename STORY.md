# 从零到上线：一个人搭建全栈 Azure 个人网站的完整故事

> **作者**：AimeeWang — Microsoft Technical Support Engineer (AI)  
> **项目地址**：[github.com/hahAI111/aimeewebpage](https://github.com/hahAI111/aimeewebpage)  
> **线上地址**：[aimeelan.azurewebsites.net](https://aimeelan.azurewebsites.net)

---

## 前言

这不是一份教程，是一个真实的项目故事。

从"我想要一个网站"这句话开始，到一个拥有访客验证、博客系统、GitHub 项目同步、
Admin 数据分析面板、Redis 缓存、PostgreSQL 数据库、CI/CD 自动部署的全栈应用——
一步一步，包括每一个踩过的坑和解决方案。

**这个项目展示的不仅是最终结果，更是解决问题的思路。**

---

## 目录

- [第一章：起点 — 一句话开始的项目](#第一章起点--一句话开始的项目)
- [第二章：从静态页面到 Flask 后端](#第二章从静态页面到-flask-后端)
- [第三章：上云 — 迁移到 Azure 全家桶](#第三章上云--迁移到-azure-全家桶)
- [第四章：第一个大坑 — 连接字符串之谜](#第四章第一个大坑--连接字符串之谜)
- [第五章：Git 与部署 — 让代码自动上线](#第五章git-与部署--让代码自动上线)
- [第六章：功能爆发 — 四大模块一次上线](#第六章功能爆发--四大模块一次上线)
- [第七章：细节打磨 — 一致性与品牌统一](#第七章细节打磨--一致性与品牌统一)
- [第八章：503 — 网站挂了](#第八章503--网站挂了)
- [第九章：自动化 — 让 GitHub 项目自己同步](#第九章自动化--让-github-项目自己同步)
- [第十章：409 部署冲突 — 流水线撞车](#第十章409-部署冲突--流水线撞车)
- [第十一章：文档即产品](#第十一章文档即产品)
- [技术栈总览](#技术栈总览)
- [最终项目结构](#最终项目结构)
- [技能展示总结](#技能展示总结)
- [写在最后](#写在最后)

---

## 第一章：起点 — 一句话开始的项目

**需求**："帮我建一个网站。"

就这一句话。没有设计稿，没有需求文档，没有技术选型会议。

**思路**：先做最小可用版本（MVP） — 一个简洁的个人介绍页面。

**方案**：
- 纯前端起步：HTML + CSS + JavaScript
- 响应式设计，适配手机和桌面
- 包含：自我介绍、技能展示、联系方式

```
第一个文件结构：
static/
├── index.html    ← 主页面
├── style.css     ← 样式
└── script.js     ← 交互
```

**技能点**：HTML5 语义化、CSS Flexbox/Grid 布局、响应式设计、DOM 操作

---

## 第二章：从静态页面到 Flask 后端

静态页面做完后，新需求来了：

> "要有访客验证、点击追踪、留言功能、防钓鱼邮件检测、社交链接、邮件通知..."

纯前端做不了这些。需要后端。

**技术选型**：
- **语言**：Python（我最熟悉的语言）
- **框架**：Flask（轻量、灵活、够用）
- **数据库**：先用 SQLite（开发方便，后面再换）

**设计思路**：
```
浏览器 → Flask API → SQLite
           │
           ├── /api/verify   → 访客注册验证
           ├── /api/track    → 点击行为追踪
           ├── /api/contact  → 留言 + 邮件通知
           └── /              → 静态文件服务
```

**安全设计**（这些不是事后想到的，是一开始就设计进去的）：
- 封杀一次性邮箱域名（tempmail.com、mailinator.com 等 15 个）
- 邮箱格式正则验证
- IP 地址 SHA-256 哈希存储（隐私保护，不存原始 IP）
- User-Agent 截取前 500 字符（防超长字符串攻击）

**技能点**：Python Flask、RESTful API 设计、SQLite、SMTP 邮件发送、安全防护思维

---

## 第三章：上云 — 迁移到 Azure 全家桶

本地跑得好好的，但一个个人网站不能永远在自己电脑上跑。决定上 Azure。

**为什么选 Azure？** 因为我在 Microsoft 工作，对 Azure 最熟悉，而且可以利用公司资源学习。

**架构升级**：

```
之前：Flask → SQLite（单文件数据库）
之后：Flask → Azure PostgreSQL（云数据库）
                → Azure Redis（缓存层）
                → Azure App Service（托管运行）
```

**创建的 Azure 资源**：

| 资源 | 名称 | 规格 | 用途 |
|---|---|---|---|
| App Service | aimeelan | Linux, Python 3.14 | 运行 Flask 应用 |
| PostgreSQL Flexible Server | aimeelan-server | | 持久化数据存储 |
| Azure Cache for Redis | aimee-cache | Basic SKU | 缓存加速 |

**代码改动**：
- `sqlite3` → `psycopg2`（PostgreSQL 驱动）
- `?` 占位符 → `%s`（PostgreSQL 语法）
- `lastrowid` → `RETURNING id`（PostgreSQL 特性）
- 新增 `redis` 缓存层
- 新增 `sslmode="require"`（Azure 强制 SSL）

**技能点**：Azure 云服务、SQLite→PostgreSQL 迁移、数据库方言差异、SSL 安全连接、Redis 缓存架构

---

## 第四章：第一个大坑 — 连接字符串之谜

### 问题

部署到 Azure 后，网站打不开。504 Gateway Timeout。

### 排查过程

1. **检查代码**：本地跑得通 ✓
2. **检查 Azure 配置**：App Service 运行正常 ✓
3. **检查数据库**：Azure PostgreSQL 在线 ✓
4. **查看日志**：`psycopg2.OperationalError: connection failed`

问题出在 **连接字符串格式**。

### 根因

Azure 门户提供的连接字符串是 **ADO.NET 格式**（给 C# 用的）：
```
Server=aimeelan-server.postgres.database.azure.com;Database=aimeelan-database;Port=5432;User Id=xxx;Password=xxx;
```

但 Python 的 psycopg2 需要 **libpq 格式**：
```
host=aimeelan-server.postgres.database.azure.com dbname=aimeelan-database user=xxx password=xxx
```

Redis 也是同样的问题——Azure 给的格式和 Python redis 库需要的格式不一样。

### 解决方案

写了两个解析函数，自动转换格式：

```python
def _parse_pg_conn(raw):
    """ADO.NET 格式 → libpq 格式"""
    if not raw or raw.startswith("host="):
        return raw  # 已经是正确格式
    parts = dict(p.split("=", 1) for p in raw.split(";") if "=" in p)
    return f"host={parts['Server']} dbname={parts['Database']} user={parts['User Id']} password={parts['Password']}"

def _parse_redis_conn(raw):
    """Azure Redis 格式 → redis:// URI 格式"""
    # aimee-cache.redis.cache.windows.net:6380,password=xxx,ssl=True
    # → rediss://:xxx@aimee-cache.redis.cache.windows.net:6380/0
```

### 教训

> **Azure 提供的连接字符串不一定能直接用。** 不同语言、不同驱动需要不同格式。
> 遇到连接失败，第一件事：**打印连接字符串的格式，对比驱动文档要求的格式**。

**技能点**：问题诊断、日志分析、连接字符串解析、Azure Service Connector 机制理解

---

## 第五章：Git 与部署 — 让代码自动上线

### 初始化 Git

```bash
git init
git remote add origin https://github.com/hahAI111/aimeewebpage.git
```

第一次 push 就遇到了问题——Azure 自动创建的 repo 里已有 workflow 文件，
导致 `git push` 被拒绝（双方都有独立的提交历史）。

**解决**：
```bash
git pull origin main --allow-unrelated-histories  # 合并两个不相关的历史
git push origin main
```

### CI/CD 流水线

Azure 在创建 App Service 时自动生成了 GitHub Actions 配置文件
（`.github/workflows/main_aimeelan.yml`）。

从此，每次 `git push` 到 main 分支：
1. GitHub Actions 自动拉取代码
2. 在 Ubuntu 虚拟机上安装 Python + 依赖
3. 打包代码上传到 Azure App Service
4. Azure 自动重启应用

**从写代码到上线，只需要 `git push` 一条命令。**

**技能点**：Git 版本控制、GitHub Actions CI/CD、OIDC 认证、自动化部署

---

## 第六章：功能爆发 — 四大模块一次上线

网站基础版上线后，一次性规划了四个大功能：

### 1. Admin 数据分析面板

```
GET /api/admin/stats → 8 条 SQL 查询 → Chart.js 可视化

包含：
├── KPI 卡片（访客数/PV 数/点击数/留言数）
├── 30 天访客趋势折线图
├── 30 天 PV 趋势折线图
├── 热门点击元素 Top 15 柱状图
├── 热门页面 Top 10（含平均停留时间）
├── 邮箱域名分布（了解访客公司）
├── 设备类型分布饼图（Mobile/Tablet/Desktop）
├── 最新留言列表
├── 热门文章排行
└── 留存分析（Day 0/1/7/30 回访率）
```

**SQL 亮点**：留存分析用了 CTE（WITH 子句）+ 条件计数 + 日期运算，
是整个项目中最复杂的一条查询。

### 2. 博客 CMS 系统

设计了完整的内容管理模型：

```
posts (文章) ←→ post_tags (关联表) ←→ tags (标签)

特性：
├── Markdown 渲染（marked.js + highlight.js 代码高亮）
├── 标签筛选
├── 分页查询
├── 浏览量统计
└── 5 篇种子文章（Azure AI、Python 工具、PostgreSQL 优化、网络架构、成长故事）
```

### 3. GitHub 项目展示

```
GitHub API → projects 表 → /projects 页面

特性：
├── 自动拉取 GitHub 仓库信息
├── 显示语言、Star、Fork、最后更新
├── Upsert 同步（ON CONFLICT DO UPDATE）
└── 精选项目置顶
```

### 4. 前端页面追踪

```javascript
// script.js — 每次页面加载自动发送
fetch('/api/pageview', {
  method: 'POST',
  body: JSON.stringify({ page, referrer, screen_width, duration_sec })
});
```

### Redis 缓存策略

不是所有 API 都缓存——只缓存 **读多写少** 的接口：

| 接口 | 缓存 | TTL | 原因 |
|---|---|---|---|
| 文章列表 | ✓ | 120s | 频繁访问，数据变化慢 |
| 文章详情 | ✓ | 300s | 内容不常变 |
| 标签列表 | ✓ | 300s | 几乎不变 |
| 项目列表 | ✓ | 300s | 每 6 小时才变 |
| Admin 统计 | ✓ | 60s | 需要接近实时 |
| 留存分析 | ✓ | 300s | 重型查询，结果变化慢 |
| 访客注册 | ✗ | — | 写操作 |
| 页面追踪 | ✗ | — | 写操作 |

**缓存失效**：新消息来 → 清除 `stats:*`；项目同步 → 清除 `projects:*`。

### 数据库设计

一次性创建了 9 张表 + 6 个索引：

```
visitors ──→ click_logs, messages, page_views, visitor_sessions
posts ──→ post_tags ←── tags
projects（独立表）
```

所有建表语句用 `CREATE TABLE IF NOT EXISTS`，种子数据用 `ON CONFLICT DO NOTHING`，
保证重复启动不会出问题。

**技能点**：复杂 SQL（CTE、Window Functions、聚合）、数据库 Schema 设计、
多对多关系、Redis 缓存策略、Chart.js 数据可视化、Markdown 渲染、RESTful API 设计

---

## 第七章：细节打磨 — 一致性与品牌统一

功能做完后，发现了两个 UI 问题：

### 问题 1：导航栏不一致

| 页面 | 导航链接 |
|---|---|
| index.html | Home, About, Skills, Contact, Blog, Projects ✓ |
| blog.html | Home, Blog, Projects ✗ |
| projects.html | Home, Blog, Projects ✗ |
| post.html | Home, Blog, Projects ✗ |
| admin.html | Home, Blog, Projects ✗ |

Blog/Projects 等页面只有 3 个导航链接，而首页有 6 个。

### 问题 2：名字不统一

页面上有的地方写 "Jing Wang"，有的写 "JW"，需要统一为 "AimeeWang"。

### 解决

一次性修改 5 个 HTML 文件，统一所有导航栏和名称。

**这种看似小的一致性问题，反映的是产品意识** — 用户不会只看一个页面。

**技能点**：前端一致性、品牌意识、批量文件修改

---

## 第八章：503 — 网站挂了

### 问题

导航栏改完 push 后，打开网站发现：

```
:( Application Error
If you are the application administrator, you can access the diagnostic resources.
```

HTTP 503 — 网站完全挂了。

### 排查

1. 首先测试各个 URL：
   - `/` → 200 ✓（verify.html，这是静态文件）
   - `/blog` → 503 ✗
   - `/projects` → 503 ✗
   - `/api/posts` → 200 ✓（Flask API 正常）
   - `/blog.html` → 200 ✓（静态文件正常）

2. 发现规律：
   - 静态文件服务 → 正常
   - Flask API 路由 → 正常
   - Flask 返回静态文件的路由（`/blog`, `/projects`） → 503

3. 继续测试发现：等了一会，所有路由都恢复了。

### 根因

**不是代码问题。** 503 是 Azure App Service 在部署过程中的临时状态。
因为我们短时间内连续 push 了多次（功能代码、导航修复、auto-sync），
App Service 在频繁重启中出现了短暂的服务不可用。

### 教训

> **503 不一定是代码 bug。** 部署期间网站会短暂不可用，这是正常的。
> 多次快速 push 会导致多次重部署，加剧这个问题。
> **最佳实践：把多个改动合并到一个 commit 再 push。**

**技能点**：生产环境排查、HTTP 状态码分析、部署过程理解、冷静分析而不是盲目改代码

---

## 第九章：自动化 — 让 GitHub 项目自己同步

### 问题

GitHub 上新建仓库后，网站的 Projects 页面不会自动更新。
之前的同步只在首次启动（projects 表为空时）触发一次。

### 方案选择

| 方案 | 优点 | 缺点 |
|---|---|---|
| A：定时后台同步 | 简单、无需外部配置 | 有延迟（最多 6 小时） |
| B：GitHub Webhook | 实时、精准 | 需要配置 Webhook URL、处理签名验证 |

选择了 **方案 A**：用 Python `threading` 创建守护线程，每 6 小时自动拉取 GitHub API。

```python
def _github_sync_loop():
    while True:
        time.sleep(GITHUB_SYNC_INTERVAL)  # 默认 21600 秒 = 6 小时
        _seed_github_projects()            # 复用已有的同步函数
        cache_delete("projects:*")         # 清除缓存

_sync_thread = threading.Thread(target=_github_sync_loop, daemon=True)
_sync_thread.start()
```

`daemon=True` 确保主进程退出时线程自动结束，不会阻止 App Service 重启。
同步间隔可通过环境变量 `GITHUB_SYNC_INTERVAL` 调整。

**技能点**：多线程编程、后台任务调度、守护线程、API 集成、可配置化设计

---

## 第十章：409 部署冲突 — 流水线撞车

### 问题

push 后 GitHub Actions 显示红色：

```
Error: Failed to deploy web package using OneDeploy to App Service.
Conflict (CODE: 409)
```

### 分析

409 Conflict = Azure App Service 上有一个部署正在进行，不接受新的部署请求。

原因：之前连续 push 了 REDIS.md、POSTGRESQL.md、auto-sync 代码，
每次 push 都触发一次部署，部署还没完成就来了新的——**撞车了**。

```
部署 #9 (REDIS.md)        ──→ 成功 ✓
部署 #10 (POSTGRESQL.md)   ──→ 成功 ✓（前一个刚完成）
部署 #11 (auto-sync 代码)  ──→ 409 ✗（#10 的 Azure 构建还在跑）
部署 #12 (重试)            ──→ 成功 ✓（#10 终于完成了）
部署 #13 (重试)            ──→ 409 ✗（仍有残留锁）
部署 #14 (重启后重试)      ──→ 成功 ✓
```

### 解决

```bash
# 1. 重启 App Service，清除部署锁
az webapp restart --name aimeelan --resource-group aimee-test-env

# 2. 等待重启完成
Start-Sleep -Seconds 10

# 3. 空 commit 重新触发部署
git commit --allow-empty -m "Retry deploy after restart"
git push origin main
```

### 教训

> **CI/CD 流水线也会有"交通事故"。** 和写代码一样，部署也需要考虑并发。
> 频繁 push 小改动会触发很多次部署，不如攒一批一起推。

**技能点**：CI/CD 故障排查、Azure CLI 操作、部署流水线理解、409 状态码含义

---

## 第十一章：文档即产品

代码写完不算完。一个好的项目需要好的文档。

为什么写文档？
1. **给自己** — 三个月后自己看还能懂
2. **给别人** — 其他人能快速理解项目
3. **展示能力** — 证明你不只是会写代码，还会系统性思考

最终项目包含 7 份文档：

| 文档 | 内容 | 目标读者 |
|---|---|---|
| README.md | 项目总览、快速开始 | 所有人 |
| ARCHITECTURE.md | 系统架构、组件关系 | 技术人员 |
| POSTGRESQL.md | 数据库 Schema、SQL 详解、监控 | 后端开发 |
| REDIS.md | 缓存策略、监控、测试方法 | 后端开发 |
| DEPLOYMENT.md | CI/CD 流水线全流程 | DevOps |
| TROUBLESHOOTING.md | 故障排查指南 | 运维 |
| STORY.md | 你正在读的这篇 | 所有人 |

**技能点**：技术写作、系统性思维、文档结构设计

---

## 技术栈总览

```
前端：HTML5 / CSS3 / JavaScript / Chart.js / marked.js / highlight.js
后端：Python / Flask / gunicorn
数据库：Azure PostgreSQL Flexible Server / psycopg2
缓存：Azure Cache for Redis
云平台：Azure App Service (Linux)
CI/CD：GitHub Actions / Azure OneDeploy
版本控制：Git / GitHub
安全：SSL/TLS / 邮箱验证 / IP 哈希 / bcrypt 密码哈希 / 一次性邮箱封锁
监控：Azure Metrics / Redis MONITOR / PostgreSQL pg_stat
```

---

## 最终项目结构

```
my-website/
├── .github/
│   └── workflows/
│       └── main_aimeelan.yml     ← CI/CD 流水线配置
├── static/
│   ├── index.html                ← 主页（自我介绍、技能、联系方式）
│   ├── verify.html               ← 访客验证入口
│   ├── blog.html                 ← 博客列表页
│   ├── post.html                 ← 博客文章详情页
│   ├── projects.html             ← GitHub 项目展示页
│   ├── admin.html                ← Admin 数据分析面板
│   ├── style.css                 ← 全局样式
│   └── script.js                 ← 页面追踪 + 交互逻辑
├── app.py                        ← Flask 后端（所有 API + 数据库）
├── requirements.txt              ← Python 依赖
├── README.md                     ← 项目说明
├── ARCHITECTURE.md               ← 系统架构
├── POSTGRESQL.md                 ← 数据库文档
├── REDIS.md                      ← 缓存文档
├── DEPLOYMENT.md                 ← 部署文档
├── TROUBLESHOOTING.md            ← 故障排查
└── STORY.md                      ← 项目故事（本文）
```

---

## 技能展示总结

### 后端开发
- Python Flask 框架，RESTful API 设计
- PostgreSQL 复杂查询（CTE、Window Functions、聚合分析、Upsert）
- Redis 缓存策略设计（Read-Through、TTL、主动失效）
- 多线程后台任务调度
- SMTP 邮件通知集成

### 前端开发
- 响应式 HTML/CSS 布局
- JavaScript 异步编程（fetch API）
- Chart.js 数据可视化
- Markdown 实时渲染

### 云计算与 DevOps
- Azure App Service / PostgreSQL / Redis 部署与配置
- GitHub Actions CI/CD 自动化流水线
- 连接字符串格式转换（ADO.NET → libpq / Redis URI）
- Azure Metrics 监控与告警

### 安全工程
- SSL/TLS 加密通信
- bcrypt 密码哈希
- IP 隐私保护（SHA-256 哈希存储）
- 一次性邮箱域名封锁
- SQL 注入防护（参数化查询 + 白名单）
- Session 管理与 Admin 认证

### 问题解决
- 504 超时排查 → 连接字符串格式不匹配
- 503 应用错误 → 部署过程中的临时状态
- 409 部署冲突 → App Service 部署锁
- Git 合并冲突 → `--allow-unrelated-histories`
- UI 不一致 → 系统性批量修复

### 系统设计
- 9 张表的关系型数据库 Schema
- 6 个索引的性能优化
- 读写分离的缓存策略
- 容错降级设计（Redis 挂了不影响核心功能）
- 可配置化（所有敏感配置通过环境变量注入）

### 技术写作
- 7 份系统性文档
- 流程图、表格、代码示例
- 面向不同读者的分层文档

---

## 写在最后

这个项目从第一行 HTML 到最终版本，经历了：

- **16 次 Git 提交**
- **9 张数据库表**
- **1200+ 行 Python 后端代码**
- **7 个 HTML 页面**
- **7 份技术文档**
- **4 次部署失败与修复**
- **无数次"为什么不工作"到"原来如此"的顿悟**

每一个问题都是真实遇到的，每一个解决方案都是当时想出来的。
没有完美的代码，只有不断改进的过程。

> **项目最有价值的不是最终结果，而是解决问题的每一步。**
