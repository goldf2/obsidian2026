# Docker 容器化部署方案

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **状态**: 历史参考，已被根目录 `数据存储与PaaS部署治理路线图.md`、`持续集成与持续部署标准流程方案.md` 和 `web2py服务备份规范.md` 替代
> **适用范围**: service_center 应用容器化部署

---

> 注意: 本文保留用于追溯早期 service_center 容器化设想，不再作为当前执行依据。新的技术方案以 PaaS 平台中立、标准 Docker/Compose 可移植、`/data/<web2py_service>/<app>` 或等价持久化存储为准；Coolify 只是可选部署平台之一。

## 📋 目录

- [方案概述](#方案概述)
- [架构设计](#架构设计)
- [环境变量配置](#环境变量配置)
- [代码修改](#代码修改)
- [Docker 配置文件](#docker-配置文件)
- [部署步骤](#部署步骤)
- [运维管理](#运维管理)

---

## 方案概述

### 核心思路

使用**环境变量**配置数据目录路径，通过 **Docker 卷**持久化数据，实现代码与数据的完全分离。

### 优势

| 优势 | 说明 |
|------|------|
| **环境一致性** | 开发、测试、生产环境完全一致 |
| **快速部署** | 一条命令启动完整环境 |
| **弹性扩展** | 轻松实现水平扩展 |
| **版本回滚** | 镜像版本管理，秒级回滚 |
| **隔离性好** | 容器间相互隔离 |

### 架构图

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │           web2py-service-center                  │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │  /app/applications/service_center (只读) │    │   │
│  │  │  - controllers/                          │    │   │
│  │  │  - models/                               │    │   │
│  │  │  - views/                                │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  │                      │                           │   │
│  │                      ▼                           │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │  /data (读写，Docker 卷挂载)             │    │   │
│  │  │  - databases/                            │    │   │
│  │  │  - sessions/                             │    │   │
│  │  │  - uploads/                              │    │   │
│  │  │  - config/appconfig.ini                  │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │        Docker Volume: service-center-data        │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 架构设计

### 目录结构

```
项目根目录/
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   └── .env.example
├── applications/
│   └── service_center/
│       ├── controllers/
│       ├── models/
│       ├── views/
│       └── ...
└── config/
    └── appconfig.ini.example
```

### 数据卷设计

| 卷名称 | 容器路径 | 用途 |
|--------|---------|------|
| `service-center-db` | `/data/databases` | SQLite 数据库 |
| `service-center-sessions` | `/data/sessions` | 会话文件 |
| `service-center-uploads` | `/data/uploads` | 用户上传 |
| `service-center-errors` | `/data/errors` | 错误日志 |
| `service-center-config` | `/data/config` | 配置文件 |

---

## 环境变量配置

### 环境变量清单

| 变量名 | 说明 | 默认值 | 示例 |
|--------|------|--------|------|
| `WEB2PY_DATA_DIR` | 数据根目录 | `/data` | `/data` |
| `WEB2PY_CONFIG_FILE` | 配置文件路径 | 自动检测 | `/data/config/appconfig.ini` |
| `DB_URI` | 数据库连接字符串 | `sqlite://storage.sqlite` | `postgres://user:pass@db:5432/app` |
| `DB_MIGRATE` | 是否启用迁移 | `true` | `false` |
| `DB_POOL_SIZE` | 连接池大小 | `10` | `20` |
| `APP_PRODUCTION` | 生产模式 | `false` | `true` |
| `SMTP_SERVER` | 邮件服务器 | - | `smtp.example.com:587` |
| `SMTP_SENDER` | 发件人 | - | `noreply@example.com` |
| `SMTP_LOGIN` | 邮件登录凭证 | - | `user:password` |

### .env 文件示例

```bash
# .env.example

# ============================================
# 数据目录配置
# ============================================
WEB2PY_DATA_DIR=/data

# ============================================
# 数据库配置
# ============================================
# SQLite（开发环境）
DB_URI=sqlite://storage.sqlite

# PostgreSQL（生产环境）
# DB_URI=postgres://user:password@db:5432/service_center

DB_MIGRATE=true
DB_POOL_SIZE=10

# ============================================
# 应用配置
# ============================================
APP_NAME=Service Center
APP_PRODUCTION=false

# ============================================
# 邮件配置
# ============================================
SMTP_SERVER=smtp.example.com:587
SMTP_SENDER=noreply@example.com
SMTP_LOGIN=username:password
SMTP_TLS=true
SMTP_SSL=false

# ============================================
# 主机配置
# ============================================
HOST_NAMES=localhost:8000,127.0.0.1:8000
```

---

## 代码修改

### models/00_db.py

```python
# -*- coding: utf-8 -*-
"""
数据库配置 - Docker 容器化版本
支持环境变量配置，适配容器化部署
"""

import os
import logging

# =========================================================================
# 环境变量读取辅助函数
# =========================================================================

def get_env(key, default=None, cast=str):
    """获取环境变量，支持类型转换"""
    value = os.environ.get(key, default)
    if value is None:
        return None
    if cast == bool:
        return str(value).lower() in ('true', '1', 'yes', 'on')
    return cast(value)


# =========================================================================
# 数据目录配置
# =========================================================================

# 数据根目录：优先环境变量，其次检测外部目录，最后回退到应用目录
DATA_DIR = get_env('WEB2PY_DATA_DIR')

if DATA_DIR is None:
    # 检查常见的外部数据目录
    for candidate in ['/data', '/var/web2py-data/service_center']:
        if os.path.exists(candidate):
            DATA_DIR = candidate
            break
    else:
        # 回退到应用内目录（开发环境）
        DATA_DIR = request.folder

# 确保子目录存在
for subdir in ['databases', 'sessions', 'errors', 'uploads', 'cache', 'config']:
    path = os.path.join(DATA_DIR, subdir)
    if not os.path.exists(path):
        try:
            os.makedirs(path)
        except OSError:
            pass  # 可能是只读文件系统

logging.info(f"数据目录: {DATA_DIR}")


# =========================================================================
# 配置文件加载
# =========================================================================

from gluon.contrib.appconfig import AppConfig

# 配置文件搜索顺序
config_candidates = [
    get_env('WEB2PY_CONFIG_FILE'),                          # 1. 环境变量指定
    os.path.join(DATA_DIR, 'config', 'appconfig.ini'),      # 2. 数据目录
    os.path.join(request.folder, 'private', 'appconfig.ini') # 3. 应用目录
]

configuration = None
for config_path in config_candidates:
    if config_path and os.path.exists(config_path):
        configuration = AppConfig(config_path, reload=True)
        logging.info(f"加载配置文件: {config_path}")
        break

if configuration is None:
    # 无配置文件时使用环境变量
    logging.warning("未找到配置文件，使用环境变量配置")


# =========================================================================
# 配置获取辅助函数
# =========================================================================

def get_config(key, env_key=None, default=None, cast=str):
    """
    获取配置值，优先级：环境变量 > 配置文件 > 默认值
    """
    # 1. 尝试环境变量
    if env_key:
        env_value = get_env(env_key, cast=cast)
        if env_value is not None:
            return env_value

    # 2. 尝试配置文件
    if configuration:
        try:
            return configuration.get(key)
        except:
            pass

    # 3. 返回默认值
    return default


# =========================================================================
# 数据库初始化
# =========================================================================

# 数据库 URI
db_uri = get_config('db.uri', 'DB_URI', 'sqlite://storage.sqlite')

# 数据库目录
db_folder = os.path.join(DATA_DIR, 'databases')

# 迁移设置
db_migrate = get_config('db.migrate', 'DB_MIGRATE', True, cast=bool)

# 连接池
db_pool_size = get_config('db.pool_size', 'DB_POOL_SIZE', 10, cast=int)

# 创建 DAL 实例
db = DAL(
    db_uri,
    folder=db_folder,
    migrate=db_migrate,
    pool_size=db_pool_size,
    check_reserved=['all']
)

logging.info(f"数据库连接: {db_uri}, 目录: {db_folder}")
```

### models/01_db.py

```python
# -*- coding: utf-8 -*-
"""
认证和应用配置 - Docker 容器化版本
"""

from gluon.tools import Auth, Crud

# =========================================================================
# 响应配置
# =========================================================================

response.generic_patterns = []
if request.is_local and not get_config('app.production', 'APP_PRODUCTION', False, cast=bool):
    response.generic_patterns.append('*')

response.formstyle = 'bootstrap4_inline'
response.form_label_separator = ''

# =========================================================================
# 认证系统
# =========================================================================

host_names = get_config('host.names', 'HOST_NAMES', 'localhost:*,127.0.0.1:*')
auth = Auth(db, host_names=host_names)
crud = Crud(db)

# 扩展字段
auth.settings.extra_fields['auth_user'] = [
    Field('phone', 'string', length=20, label='手机号'),
]

auth.define_tables(username=True, signature=False)

# =========================================================================
# 邮件配置
# =========================================================================

mail = auth.settings.mailer

if request.is_local:
    mail.settings.server = 'logging'
else:
    mail.settings.server = get_config('smtp.server', 'SMTP_SERVER', 'logging')

mail.settings.sender = get_config('smtp.sender', 'SMTP_SENDER', 'noreply@localhost')
mail.settings.login = get_config('smtp.login', 'SMTP_LOGIN')
mail.settings.tls = get_config('smtp.tls', 'SMTP_TLS', False, cast=bool)
mail.settings.ssl = get_config('smtp.ssl', 'SMTP_SSL', False, cast=bool)

# =========================================================================
# 认证策略
# =========================================================================

auth.settings.registration_requires_verification = False
auth.settings.registration_requires_approval = False
auth.settings.reset_password_requires_verification = True

# =========================================================================
# 页面元数据
# =========================================================================

response.meta.author = get_config('app.author', 'APP_AUTHOR', 'Web2py')
response.meta.description = get_config('app.description', 'APP_DESCRIPTION', '服务中心应用')
response.meta.keywords = get_config('app.keywords', 'APP_KEYWORDS', 'web2py, python')
response.meta.generator = get_config('app.generator', 'APP_GENERATOR', 'Web2py')
response.show_toolbar = get_config('app.toolbar', 'APP_TOOLBAR', True, cast=bool)

# =========================================================================
# 自定义字段配置
# =========================================================================

db.auth_user.first_name.label = '姓名'
db.auth_user.last_name.readable = False
db.auth_user.last_name.writable = False
db.auth_user.last_name.default = ''
db.auth_user.last_name.requires = None

# =========================================================================
# 登录跳转
# =========================================================================

auth.settings.login_next = request.env.http_referer or URL('default', 'index')
auth.settings.on_failed_authorization = URL('default', 'index')

# =========================================================================
# 封存用户强制登出
# =========================================================================

if auth.user:
    current_user = db.auth_user(auth.user_id)
    if current_user and current_user.registration_key == 'blocked':
        auth.logout(next=URL('default', 'index'))
        session.flash = '您的账户已被封存，请联系管理员'
```

---

## Docker 配置文件

### Dockerfile

```dockerfile
# Dockerfile
FROM python:3.9-slim

# 元数据
LABEL maintainer="your-email@example.com"
LABEL description="Web2py Service Center Application"
LABEL version="1.0.0"

# 环境变量
ENV PYTHONUNBUFFERED=1
ENV WEB2PY_VERSION=2.24.1
ENV WEB2PY_DATA_DIR=/data
ENV WEB2PY_PASSWORD=admin

# 安装系统依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    wget \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# 下载并安装 Web2py
WORKDIR /app
RUN wget -q https://mdipierro.pythonanywhere.com/examples/static/web2py_src.zip \
    && unzip -q web2py_src.zip \
    && rm web2py_src.zip \
    && mv web2py/* . \
    && rm -rf web2py

# 安装 Python 依赖
COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt

# 创建数据目录
RUN mkdir -p /data/{databases,sessions,errors,uploads,cache,config}

# 复制应用代码
COPY applications/service_center /app/applications/service_center

# 设置权限
RUN chmod -R 755 /app \
    && chmod -R 777 /data

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8000/ || exit 1

# 启动命令
CMD ["python", "web2py.py", "-a", "${WEB2PY_PASSWORD}", "-i", "0.0.0.0", "-p", "8000"]
```

### docker-compose.yml (开发环境)

```yaml
# docker-compose.yml
version: '3.8'

services:
  web2py:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: service-center-dev
    ports:
      - "8000:8000"
    volumes:
      # 代码目录（开发时可写，方便调试）
      - ./applications/service_center:/app/applications/service_center
      # 数据卷
      - service-center-data:/data
    environment:
      - WEB2PY_DATA_DIR=/data
      - WEB2PY_PASSWORD=admin
      - DB_URI=sqlite://storage.sqlite
      - DB_MIGRATE=true
      - APP_PRODUCTION=false
    env_file:
      - .env
    restart: unless-stopped

volumes:
  service-center-data:
    driver: local
```

### docker-compose.prod.yml (生产环境)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  web2py:
    image: your-registry/service-center:${VERSION:-latest}
    container_name: service-center-prod
    ports:
      - "8000:8000"
    volumes:
      # 代码目录（生产环境只读）
      - ./applications/service_center:/app/applications/service_center:ro
      # 数据卷（持久化）
      - service-center-db:/data/databases
      - service-center-sessions:/data/sessions
      - service-center-uploads:/data/uploads
      - service-center-errors:/data/errors
      # 配置文件（只读）
      - ./config/appconfig.ini:/data/config/appconfig.ini:ro
    environment:
      - WEB2PY_DATA_DIR=/data
      - APP_PRODUCTION=true
    env_file:
      - .env.prod
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 可选：PostgreSQL 数据库
  db:
    image: postgres:14-alpine
    container_name: service-center-db
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=web2py
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=service_center
    restart: always

  # 可选：Nginx 反向代理
  nginx:
    image: nginx:alpine
    container_name: service-center-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - web2py
    restart: always

volumes:
  service-center-db:
  service-center-sessions:
  service-center-uploads:
  service-center-errors:
  postgres-data:
```

---

## 部署步骤

### 开发环境

```bash
# 1. 克隆代码
git clone git@gitee.com:<git-namespace>/<repo>.git
cd <repo>

# 2. 复制环境变量文件
cp .env.example .env

# 3. 编辑环境变量
vim .env

# 4. 启动容器
docker-compose up -d

# 5. 查看日志
docker-compose logs -f

# 6. 访问应用
open http://localhost:8000/service_center/
```

### 生产环境

```bash
# 1. 构建镜像
docker build -t your-registry/service-center:v1.0.0 -f docker/Dockerfile .

# 2. 推送镜像
docker push your-registry/service-center:v1.0.0

# 3. 在生产服务器上
export VERSION=v1.0.0
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d

# 4. 验证
docker-compose -f docker-compose.prod.yml ps
curl http://localhost:8000/service_center/
```

---

## 运维管理

### 常用命令

```bash
# 查看容器状态
docker-compose ps

# 查看日志
docker-compose logs -f web2py

# 进入容器
docker-compose exec web2py bash

# 重启服务
docker-compose restart web2py

# 停止服务
docker-compose down

# 停止并删除数据卷（危险！）
docker-compose down -v

# 备份数据卷
docker run --rm -v service-center-db:/data -v $(pwd):/backup \
    alpine tar czf /backup/db-backup-$(date +%Y%m%d).tar.gz -C /data .

# 恢复数据卷
docker run --rm -v service-center-db:/data -v $(pwd):/backup \
    alpine tar xzf /backup/db-backup-20260123.tar.gz -C /data
```

### 监控与告警

```yaml
# 添加到 docker-compose.prod.yml

  # Prometheus 监控
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  # Grafana 可视化
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
```

---

## 附录

### 常见问题

**Q: 如何迁移现有数据到 Docker 环境？**

```bash
# 1. 停止旧服务
# 2. 复制数据到本地目录
cp -r /opt/<web2py_service>/applications/service_center/databases ./data/

# 3. 启动 Docker
docker-compose up -d

# 4. 复制数据到卷
docker cp ./data/databases/. service-center-dev:/data/databases/
```

**Q: 如何升级应用版本？**

```bash
# 1. 构建新版本镜像
docker build -t your-registry/service-center:v1.1.0 .

# 2. 更新版本号
export VERSION=v1.1.0

# 3. 滚动更新
docker-compose -f docker-compose.prod.yml up -d --no-deps web2py
```

---

*文档创建: 2026-01-23*
*作者: Claude AI*
