# 多服务器部署方案

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **状态**: 参考文档
> **适用范围**: service_center 应用多服务器/集群部署

---

## 📋 目录

- [方案概述](#方案概述)
- [架构设计](#架构设计)
- [配置管理策略](#配置管理策略)
- [代码修改](#代码修改)
- [部署配置](#部署配置)
- [运维管理](#运维管理)

---

## 方案概述

### 适用场景

| 场景 | 说明 |
|------|------|
| **多环境部署** | 开发、测试、预发布、生产环境 |
| **多服务器集群** | 负载均衡下的多实例部署 |
| **混合云部署** | 本地 + 云端混合架构 |
| **灾备部署** | 主从热备或异地多活 |

### 核心思路

采用**配置文件 + 环境变量**的分层配置策略：

```
┌─────────────────────────────────────────┐
│  环境变量 (最高优先级)                    │  ← 敏感信息、运行时覆盖
├─────────────────────────────────────────┤
│  环境配置文件 (appconfig.{env}.ini)      │  ← 环境特定配置
├─────────────────────────────────────────┤
│  基础配置文件 (appconfig.base.ini)       │  ← 通用默认配置
└─────────────────────────────────────────┘
```

### 优势

| 优势 | 说明 |
|------|------|
| **环境隔离** | 不同环境使用不同配置，互不影响 |
| **敏感信息保护** | 密码等通过环境变量注入，不进入代码仓库 |
| **配置版本化** | 配置文件可纳入版本控制 |
| **灵活覆盖** | 环境变量可动态覆盖配置 |
| **集中管理** | 支持配置中心集成 |

---

## 架构设计

### 部署架构图

```
                    ┌─────────────────┐
                    │   负载均衡器     │
                    │  (Nginx/HAProxy) │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Web Server 1  │ │   Web Server 2  │ │   Web Server 3  │
│  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │
│  │ Web2py    │  │ │  │ Web2py    │  │ │  │ Web2py    │  │
│  │ App Code  │  │ │  │ App Code  │  │ │  │ App Code  │  │
│  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │
│       │         │ │       │         │ │       │         │
│  环境变量配置    │ │  环境变量配置    │ │  环境变量配置    │
│  APP_ENV=prod   │ │  APP_ENV=prod   │ │  APP_ENV=prod   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │      共享存储 / 数据库     │
              │  ┌──────┐  ┌──────────┐  │
              │  │ NFS  │  │PostgreSQL│  │
              │  │共享卷 │  │  数据库   │  │
              │  └──────┘  └──────────┘  │
              └──────────────────────────┘
```

### 配置文件结构

```
config/
├── appconfig.base.ini          # 基础配置（通用默认值）
├── appconfig.development.ini   # 开发环境配置
├── appconfig.testing.ini       # 测试环境配置
├── appconfig.staging.ini       # 预发布环境配置
├── appconfig.production.ini    # 生产环境配置
└── secrets/                    # 敏感配置（不进入 Git）
    ├── .gitignore
    ├── development.env
    ├── testing.env
    ├── staging.env
    └── production.env
```

---

## 配置管理策略

### 配置分层

#### 第一层：基础配置 (appconfig.base.ini)

```ini
; config/appconfig.base.ini
; 通用默认配置，所有环境共享

[app]
name = Service Center
author = Your Team
description = 服务中心应用
keywords = web2py, python, framework
generator = Web2py Web Framework

[db]
; 默认使用 SQLite（开发环境）
uri = sqlite://storage.sqlite
migrate = true
pool_size = 10

[smtp]
tls = true
ssl = false

[scheduler]
enabled = false
heartbeat = 5

[session]
; 会话存储方式: file, db, redis
storage = file
expire = 3600
```

#### 第二层：环境配置 (appconfig.{env}.ini)

```ini
; config/appconfig.development.ini
; 开发环境特定配置

[app]
production = false
toolbar = true

[host]
names = localhost:*, 127.0.0.1:*, *:8000

[db]
uri = sqlite://storage.sqlite
migrate = true

[smtp]
server = logging
```

```ini
; config/appconfig.production.ini
; 生产环境特定配置

[app]
production = true
toolbar = false

[host]
names = service.example.com, *.example.com

[db]
; 生产环境使用 PostgreSQL
; 实际密码通过环境变量注入
uri = ${DB_URI}
migrate = false
pool_size = 20

[smtp]
server = ${SMTP_SERVER}
sender = ${SMTP_SENDER}
login = ${SMTP_LOGIN}

[session]
storage = redis
redis_host = ${REDIS_HOST}
redis_port = 6379

[cache]
storage = redis
redis_host = ${REDIS_HOST}
```

#### 第三层：环境变量 (secrets/{env}.env)

```bash
# config/secrets/production.env
# 敏感配置，不进入版本控制

# 数据库
DB_URI=postgres://web2py:SecurePassword123@db.example.com:5432/service_center

# 邮件
SMTP_SERVER=smtp.example.com:587
SMTP_SENDER=noreply@example.com
SMTP_LOGIN=apikey:SG.xxxxxxxxxxxxx

# Redis
REDIS_HOST=redis.example.com

# 应用密钥
APP_SECRET_KEY=your-very-long-secret-key-here

# 第三方 API
ALIYUN_OSS_KEY=xxxxxxxxxxxx
ALIYUN_OSS_SECRET=xxxxxxxxxxxx
```

### 配置优先级

```
环境变量 > 环境配置文件 > 基础配置文件 > 代码默认值
```

---

## 代码修改

### modules/config_loader.py

```python
# -*- coding: utf-8 -*-
"""
配置加载器 - 多环境配置管理
支持配置文件 + 环境变量的分层配置策略
"""

import os
import re
import logging
from configparser import ConfigParser, ExtendedInterpolation

logger = logging.getLogger(__name__)


class ConfigLoader:
    """
    多环境配置加载器

    优先级：环境变量 > 环境配置文件 > 基础配置文件 > 默认值
    """

    def __init__(self, app_folder, env=None):
        """
        初始化配置加载器

        Args:
            app_folder: 应用根目录
            env: 环境名称（development, testing, staging, production）
        """
        self.app_folder = app_folder
        self.env = env or os.environ.get('APP_ENV', 'development')
        self.config = ConfigParser(interpolation=ExtendedInterpolation())
        self._load_configs()

    def _load_configs(self):
        """加载配置文件"""
        config_dir = os.path.join(self.app_folder, 'config')

        # 如果 config 目录不存在，使用 private 目录
        if not os.path.exists(config_dir):
            config_dir = os.path.join(self.app_folder, 'private')

        # 1. 加载基础配置
        base_config = os.path.join(config_dir, 'appconfig.base.ini')
        if os.path.exists(base_config):
            self.config.read(base_config, encoding='utf-8')
            logger.info(f"加载基础配置: {base_config}")

        # 2. 加载环境配置
        env_config = os.path.join(config_dir, f'appconfig.{self.env}.ini')
        if os.path.exists(env_config):
            self.config.read(env_config, encoding='utf-8')
            logger.info(f"加载环境配置: {env_config}")
        else:
            # 回退到默认配置文件
            default_config = os.path.join(config_dir, 'appconfig.ini')
            if os.path.exists(default_config):
                self.config.read(default_config, encoding='utf-8')
                logger.info(f"加载默认配置: {default_config}")

    def get(self, key, default=None, cast=str):
        """
        获取配置值

        支持格式：
        - 'section.key' -> config[section][key]
        - 环境变量替换 ${VAR_NAME}

        Args:
            key: 配置键，格式为 'section.key'
            default: 默认值
            cast: 类型转换函数（str, int, float, bool）

        Returns:
            配置值
        """
        # 解析 section.key 格式
        if '.' in key:
            section, option = key.split('.', 1)
        else:
            section, option = 'app', key

        # 1. 尝试从环境变量获取（最高优先级）
        env_key = f"{section.upper()}_{option.upper()}"
        env_value = os.environ.get(env_key)
        if env_value is not None:
            return self._cast(env_value, cast)

        # 2. 从配置文件获取
        try:
            value = self.config.get(section, option)
            # 处理环境变量替换 ${VAR_NAME}
            value = self._expand_env_vars(value)
            return self._cast(value, cast)
        except:
            pass

        # 3. 返回默认值
        return default

    def _expand_env_vars(self, value):
        """展开配置值中的环境变量引用 ${VAR_NAME}"""
        if not value or '${' not in value:
            return value

        pattern = r'\$\{([^}]+)\}'

        def replacer(match):
            var_name = match.group(1)
            return os.environ.get(var_name, match.group(0))

        return re.sub(pattern, replacer, value)

    def _cast(self, value, cast):
        """类型转换"""
        if value is None:
            return None

        if cast == bool:
            return str(value).lower() in ('true', '1', 'yes', 'on')
        elif cast == int:
            return int(value)
        elif cast == float:
            return float(value)
        else:
            return str(value)

    def get_section(self, section):
        """获取整个配置段"""
        if self.config.has_section(section):
            return dict(self.config.items(section))
        return {}

    @property
    def environment(self):
        """当前环境名称"""
        return self.env

    @property
    def is_production(self):
        """是否生产环境"""
        return self.env == 'production'

    @property
    def is_development(self):
        """是否开发环境"""
        return self.env == 'development'


# 单例实例（在 models 中初始化）
_config_instance = None


def get_config(app_folder=None, env=None):
    """获取配置实例（单例）"""
    global _config_instance
    if _config_instance is None and app_folder:
        _config_instance = ConfigLoader(app_folder, env)
    return _config_instance


def init_config(app_folder, env=None):
    """初始化配置"""
    global _config_instance
    _config_instance = ConfigLoader(app_folder, env)
    return _config_instance
```

### models/00_db.py

```python
# -*- coding: utf-8 -*-
"""
数据库配置 - 多服务器部署版本
支持配置文件 + 环境变量的分层配置策略
"""

import os
import sys
import logging

# =========================================================================
# 配置加载
# =========================================================================

# 添加 modules 目录到路径
modules_path = os.path.join(request.folder, 'modules')
if modules_path not in sys.path:
    sys.path.insert(0, modules_path)

from config_loader import init_config, get_config

# 初始化配置加载器
# 环境通过 APP_ENV 环境变量指定：development, testing, staging, production
config = init_config(request.folder, os.environ.get('APP_ENV'))

logging.info(f"当前环境: {config.environment}")
logging.info(f"生产模式: {config.is_production}")


# =========================================================================
# 数据目录配置
# =========================================================================

# 数据目录：优先环境变量，其次配置文件，最后默认值
DATA_DIR = os.environ.get('WEB2PY_DATA_DIR')

if DATA_DIR is None:
    DATA_DIR = config.get('app.data_dir')

if DATA_DIR is None:
    # 检查常见外部目录
    for candidate in ['/data', '/var/web2py-data/service_center']:
        if os.path.exists(candidate):
            DATA_DIR = candidate
            break
    else:
        DATA_DIR = request.folder

# 确保数据目录存在
for subdir in ['databases', 'sessions', 'errors', 'uploads', 'cache']:
    path = os.path.join(DATA_DIR, subdir)
    if not os.path.exists(path):
        try:
            os.makedirs(path)
        except OSError:
            pass

logging.info(f"数据目录: {DATA_DIR}")


# =========================================================================
# 数据库初始化
# =========================================================================

# 数据库配置
db_uri = config.get('db.uri', 'sqlite://storage.sqlite')
db_migrate = config.get('db.migrate', True, cast=bool)
db_pool_size = config.get('db.pool_size', 10, cast=int)
db_folder = os.path.join(DATA_DIR, 'databases')

# 创建 DAL 实例
db = DAL(
    db_uri,
    folder=db_folder,
    migrate=db_migrate,
    pool_size=db_pool_size,
    check_reserved=['all']
)

logging.info(f"数据库: {db_uri}")


# =========================================================================
# 会话配置（多服务器需要共享会话）
# =========================================================================

session_storage = config.get('session.storage', 'file')

if session_storage == 'db':
    # 使用数据库存储会话
    session.connect(request, response, db=db)

elif session_storage == 'redis':
    # 使用 Redis 存储会话（推荐用于集群）
    try:
        import redis
        from gluon.contrib.redis_utils import RConn

        redis_host = config.get('session.redis_host', 'localhost')
        redis_port = config.get('session.redis_port', 6379, cast=int)
        redis_db = config.get('session.redis_db', 0, cast=int)

        redis_conn = redis.Redis(host=redis_host, port=redis_port, db=redis_db)
        session.connect(request, response, rconn=RConn(redis_conn))
        logging.info(f"会话存储: Redis ({redis_host}:{redis_port})")
    except ImportError:
        logging.warning("Redis 模块未安装，回退到文件会话")
        session_storage = 'file'

if session_storage == 'file':
    # 默认文件存储
    session_folder = os.path.join(DATA_DIR, 'sessions')
    # Web2py 默认使用文件会话，无需额外配置
    logging.info(f"会话存储: 文件 ({session_folder})")
```

### models/01_db.py

```python
# -*- coding: utf-8 -*-
"""
认证和应用配置 - 多服务器部署版本
"""

from gluon.tools import Auth, Crud
from config_loader import get_config

config = get_config()

# =========================================================================
# 响应配置
# =========================================================================

response.generic_patterns = []
if not config.is_production:
    response.generic_patterns.append('*')

response.formstyle = 'bootstrap4_inline'
response.form_label_separator = ''

# =========================================================================
# 认证系统
# =========================================================================

host_names = config.get('host.names', 'localhost:*,127.0.0.1:*')
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

if config.is_development:
    mail.settings.server = 'logging'
else:
    mail.settings.server = config.get('smtp.server', 'logging')

mail.settings.sender = config.get('smtp.sender', 'noreply@localhost')
mail.settings.login = config.get('smtp.login')
mail.settings.tls = config.get('smtp.tls', False, cast=bool)
mail.settings.ssl = config.get('smtp.ssl', False, cast=bool)

# =========================================================================
# 认证策略
# =========================================================================

auth.settings.registration_requires_verification = config.get(
    'auth.require_verification', False, cast=bool)
auth.settings.registration_requires_approval = config.get(
    'auth.require_approval', False, cast=bool)
auth.settings.reset_password_requires_verification = True

# =========================================================================
# 页面元数据
# =========================================================================

response.meta.author = config.get('app.author', 'Web2py')
response.meta.description = config.get('app.description', '服务中心应用')
response.meta.keywords = config.get('app.keywords', 'web2py, python')
response.meta.generator = config.get('app.generator', 'Web2py')
response.show_toolbar = config.get('app.toolbar', True, cast=bool)

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

## 部署配置

### Nginx 负载均衡配置

```nginx
# /etc/nginx/conf.d/service-center.conf

upstream service_center {
    # 负载均衡策略
    least_conn;  # 最少连接
    # ip_hash;   # IP 哈希（会话保持）

    server 192.168.1.101:8000 weight=3;
    server 192.168.1.102:8000 weight=3;
    server 192.168.1.103:8000 weight=2 backup;  # 备用服务器

    # 健康检查
    keepalive 32;
}

server {
    listen 80;
    server_name service.example.com;

    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name service.example.com;

    # SSL 配置
    ssl_certificate /etc/nginx/ssl/service.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/service.example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    # 静态文件
    location /service_center/static/ {
        alias /var/www/service_center/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 反向代理
    location / {
        proxy_pass http://service_center;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### systemd 服务配置

```ini
# /etc/systemd/system/web2py-service-center.service

[Unit]
Description=Web2py Service Center
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/opt/web2py

# 环境变量
Environment="APP_ENV=production"
Environment="WEB2PY_DATA_DIR=/var/web2py-data/service_center"
EnvironmentFile=/etc/web2py/service-center.env

# 启动命令
ExecStart=/usr/bin/python3 web2py.py \
    -a '<password>' \
    -i 0.0.0.0 \
    -p 8000 \
    --no_banner \
    --nogui

# 重启策略
Restart=always
RestartSec=5

# 资源限制
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Ansible 部署剧本

```yaml
# deploy/ansible/playbook.yml

---
- name: 部署 Service Center
  hosts: web_servers
  become: yes
  vars:
    app_name: service_center
    app_env: production
    web2py_root: /opt/web2py
    data_dir: /var/web2py-data/service_center

  tasks:
    - name: 创建数据目录
      file:
        path: "{{ data_dir }}/{{ item }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
      loop:
        - databases
        - sessions
        - errors
        - uploads
        - cache
        - config

    - name: 同步应用代码
      synchronize:
        src: ../applications/service_center/
        dest: "{{ web2py_root }}/applications/service_center/"
        delete: yes
        rsync_opts:
          - "--exclude=databases"
          - "--exclude=sessions"
          - "--exclude=errors"
          - "--exclude=uploads"
          - "--exclude=*.pyc"
          - "--exclude=__pycache__"
      notify: restart web2py

    - name: 部署环境配置
      template:
        src: appconfig.{{ app_env }}.ini.j2
        dest: "{{ data_dir }}/config/appconfig.ini"
        owner: www-data
        group: www-data
        mode: '0640'
      notify: restart web2py

    - name: 部署环境变量
      template:
        src: service-center.env.j2
        dest: /etc/web2py/service-center.env
        owner: root
        group: www-data
        mode: '0640'
      notify: restart web2py

    - name: 部署 systemd 服务
      copy:
        src: web2py-service-center.service
        dest: /etc/systemd/system/
      notify:
        - reload systemd
        - restart web2py

    - name: 启用并启动服务
      systemd:
        name: web2py-service-center
        enabled: yes
        state: started

  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes

    - name: restart web2py
      systemd:
        name: web2py-service-center
        state: restarted
```

---

## 运维管理

### 环境切换

```bash
# 切换到开发环境
export APP_ENV=development

# 切换到测试环境
export APP_ENV=testing

# 切换到生产环境
export APP_ENV=production
```

### 配置验证

```python
# scripts/validate_config.py
"""验证配置完整性"""

import os
import sys

sys.path.insert(0, '/opt/<web2py_service>/applications/service_center/modules')
from config_loader import ConfigLoader

def validate():
    env = os.environ.get('APP_ENV', 'development')
    config = ConfigLoader('/opt/<web2py_service>/applications/service_center', env)

    required_keys = [
        ('db.uri', '数据库连接'),
        ('host.names', '主机名'),
        ('smtp.server', '邮件服务器'),
    ]

    errors = []
    for key, name in required_keys:
        value = config.get(key)
        if not value or value.startswith('${'):
            errors.append(f"缺少配置: {name} ({key})")

    if errors:
        print("配置验证失败:")
        for e in errors:
            print(f"  - {e}")
        sys.exit(1)
    else:
        print(f"配置验证通过 (环境: {env})")

if __name__ == '__main__':
    validate()
```

### 日志聚合

```yaml
# Filebeat 配置
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/web2py-data/service_center/errors/*.log
    fields:
      app: service-center
      env: production
    multiline.pattern: '^Traceback'
    multiline.negate: true
    multiline.match: after

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "web2py-logs-%{+yyyy.MM.dd}"
```

### 监控指标

```python
# modules/metrics.py
"""Prometheus 监控指标"""

from prometheus_client import Counter, Histogram, Gauge

# 请求计数
REQUEST_COUNT = Counter(
    'web2py_requests_total',
    'Total requests',
    ['method', 'endpoint', 'status']
)

# 请求延迟
REQUEST_LATENCY = Histogram(
    'web2py_request_latency_seconds',
    'Request latency',
    ['method', 'endpoint']
)

# 数据库连接数
DB_CONNECTIONS = Gauge(
    'web2py_db_connections',
    'Active database connections'
)

# 活跃会话数
ACTIVE_SESSIONS = Gauge(
    'web2py_active_sessions',
    'Active user sessions'
)
```

---

## 附录

### 配置检查清单

#### 部署前检查

- [ ] 所有服务器已安装 Python 和依赖
- [ ] 数据库服务已配置并可连接
- [ ] Redis 服务已配置（如使用）
- [ ] 共享存储已挂载（如使用 NFS）
- [ ] 环境变量文件已正确配置
- [ ] SSL 证书已部署
- [ ] 防火墙规则已配置

#### 部署后验证

- [ ] 所有服务器应用可访问
- [ ] 会话可在服务器间共享
- [ ] 数据库读写正常
- [ ] 文件上传功能正常
- [ ] 邮件发送功能正常
- [ ] 监控指标正常上报

### 常见问题

**Q: 多服务器部署时会话不共享怎么办？**

A: 使用以下方案之一：
1. Redis 存储会话（推荐）
2. 数据库存储会话
3. 负载均衡器启用会话保持（ip_hash）

**Q: 如何实现零停机部署？**

A: 使用滚动更新：
```bash
# 逐个服务器更新
for server in server1 server2 server3; do
    # 1. 从负载均衡中移除
    # 2. 部署新代码
    # 3. 重启服务
    # 4. 健康检查
    # 5. 加回负载均衡
done
```

**Q: 如何处理数据库迁移？**

A:
1. 生产环境设置 `db.migrate = false`
2. 单独运行迁移脚本：
```bash
python web2py.py -S service_center -M -R scripts/migrate.py
```

---

*文档创建: 2026-01-23*
*作者: Claude AI*
