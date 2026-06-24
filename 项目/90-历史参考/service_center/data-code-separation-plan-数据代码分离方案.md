# 数据与代码分离方案

> **版本**: 1.0.0
> **创建日期**: 2026-01-22
> **状态**: 待实施
> **适用范围**: service_center 应用

---

## 📋 目录

- [背景与目标](#背景与目标)
- [当前状态分析](#当前状态分析)
- [目标架构](#目标架构)
- [实施步骤](#实施步骤)
- [回滚方案](#回滚方案)
- [验证清单](#验证清单)

---

## 背景与目标

### 为什么需要数据代码分离？

| 问题 | 当前风险 | 分离后改善 |
|------|---------|-----------|
| **部署覆盖** | `git pull` 可能覆盖数据目录 | 数据在外部，代码可安全替换 |
| **备份复杂** | 需要排除代码目录中的数据 | 数据目录独立，备份简单 |
| **权限混乱** | 代码和数据权限相同 | 代码只读，数据可写 |
| **仓库膨胀** | 误提交数据文件导致仓库变大 | 数据完全隔离，无法误提交 |
| **容器化困难** | 数据卷挂载路径复杂 | 清晰的数据卷挂载点 |

### 目标

1. **代码目录**：只包含代码，可以完全通过 Git 管理
2. **数据目录**：独立存储，与代码生命周期分离
3. **配置管理**：敏感配置通过环境变量或外部文件管理

---

## 当前状态分析

### 当前目录结构

```
/opt/<web2py_service>/applications/service_center/
├── controllers/          # 代码 ✅
├── models/               # 代码 ✅
├── views/                # 代码 ✅
├── static/               # 代码 ✅
├── modules/              # 代码 ✅
├── languages/            # 代码 ✅
├── docs/                 # 代码 ✅
├── cron/                 # 代码 ✅
├── private/
│   ├── appconfig.ini     # 敏感配置 ⚠️ (包含 SMTP 密码)
│   └── appconfig.ini.example  # 模板 ✅
├── databases/            # 数据 ❌ (应外置)
│   ├── storage.sqlite    # SQLite 数据库
│   └── *.table           # 表结构文件
├── sessions/             # 数据 ❌ (应外置)
├── errors/               # 数据 ❌ (应外置)
├── uploads/              # 数据 ❌ (应外置)
└── cache/                # 数据 ❌ (应外置)
```

### 当前数据库配置

```python
# models/00_db.py (当前)
db = DAL('sqlite://storage.sqlite',
    folder=os.path.join(request.folder, 'databases'),
    migrate=True,
    pool_size=10
)
```

**问题**：数据库路径硬编码在应用目录内。

---

## 目标架构

### 目录结构

```
# 代码目录（Git 管理）
/opt/<web2py_service>/applications/service_center/
├── controllers/
├── models/
├── views/
├── static/
├── modules/
├── languages/
├── docs/
├── cron/
├── private/
│   └── appconfig.ini.example   # 只保留模板
├── databases/                   # 空目录，保留 .gitkeep
├── sessions/                    # 空目录，保留 .gitkeep
├── errors/                      # 空目录，保留 .gitkeep
├── uploads/                     # 空目录，保留 .gitkeep
└── cache/                       # 空目录，保留 .gitkeep

# 数据目录（独立管理）
/var/web2py-data/service_center/
├── databases/
│   ├── storage.sqlite
│   └── *.table
├── sessions/
├── errors/
├── uploads/
├── cache/
└── config/
    └── appconfig.ini           # 实际配置文件
```

### 配置方式

**方案 A：符号链接（推荐，改动最小）**

```bash
# 数据目录使用符号链接指向外部
databases -> /var/web2py-data/service_center/databases
sessions  -> /var/web2py-data/service_center/sessions
errors    -> /var/web2py-data/service_center/errors
uploads   -> /var/web2py-data/service_center/uploads
private/appconfig.ini -> /var/web2py-data/service_center/config/appconfig.ini
```

**优点**：
- 代码无需修改
- Web2py 完全兼容
- 迁移简单

**缺点**：
- 依赖文件系统符号链接
- Windows 兼容性较差

---

**方案 B：环境变量配置（更灵活）**

```python
# models/00_db.py (修改后)
import os

# 从环境变量读取数据目录，默认使用应用内目录
DATA_DIR = os.environ.get('WEB2PY_DATA_DIR',
                          '/var/web2py-data/service_center')

# 数据库配置
db_folder = os.path.join(DATA_DIR, 'databases')
db = DAL('sqlite://storage.sqlite',
    folder=db_folder,
    migrate=True,
    pool_size=10
)
```

**优点**：
- 更灵活，支持不同环境
- 容器化友好
- 跨平台兼容

**缺点**：
- 需要修改代码
- 需要设置环境变量

---

### 推荐方案：A + B 混合

1. **数据库**：使用环境变量配置路径（方案 B）
2. **其他目录**：使用符号链接（方案 A）
3. **配置文件**：使用符号链接（方案 A）

---

## 实施步骤

### 第一阶段：准备工作

#### 1.1 创建外部数据目录

```bash
# 创建数据根目录
sudo mkdir -p /var/web2py-data/service_center/{databases,sessions,errors,uploads,cache,config}

# 设置权限（假设 web2py 以 www-data 用户运行）
sudo chown -R www-data:www-data /var/web2py-data/service_center
sudo chmod -R 755 /var/web2py-data/service_center
```

#### 1.2 备份现有数据

```bash
# 备份数据库
cp -r /opt/<web2py_service>/applications/service_center/databases /tmp/service_center_backup_databases

# 备份配置
cp /opt/<web2py_service>/applications/service_center/private/appconfig.ini /tmp/service_center_backup_appconfig.ini

# 备份上传文件（如有）
cp -r /opt/<web2py_service>/applications/service_center/uploads /tmp/service_center_backup_uploads
```

### 第二阶段：迁移数据

#### 2.1 迁移数据库文件

```bash
# 复制数据库文件到外部目录
cp -r /opt/<web2py_service>/applications/service_center/databases/* /var/web2py-data/service_center/databases/

# 复制配置文件
cp /opt/<web2py_service>/applications/service_center/private/appconfig.ini /var/web2py-data/service_center/config/
```

#### 2.2 迁移其他数据

```bash
# 迁移会话（可选，会话可以重新生成）
cp -r /opt/<web2py_service>/applications/service_center/sessions/* /var/web2py-data/service_center/sessions/ 2>/dev/null || true

# 迁移错误日志
cp -r /opt/<web2py_service>/applications/service_center/errors/* /var/web2py-data/service_center/errors/ 2>/dev/null || true

# 迁移上传文件
cp -r /opt/<web2py_service>/applications/service_center/uploads/* /var/web2py-data/service_center/uploads/ 2>/dev/null || true
```

### 第三阶段：修改代码

#### 3.1 修改数据库配置

```python
# models/00_db.py

import os
from gluon.contrib.appconfig import AppConfig

# =========================================================================
# 数据目录配置
# =========================================================================
# 优先使用环境变量，否则使用默认外部路径，最后回退到应用内目录
DATA_DIR = os.environ.get('WEB2PY_DATA_DIR')

if DATA_DIR is None:
    # 检查外部数据目录是否存在
    external_data_dir = '/var/web2py-data/service_center'
    if os.path.exists(external_data_dir):
        DATA_DIR = external_data_dir
    else:
        # 回退到应用内目录（开发环境）
        DATA_DIR = request.folder

# 确保数据目录存在
db_folder = os.path.join(DATA_DIR, 'databases')
if not os.path.exists(db_folder):
    os.makedirs(db_folder)

# =========================================================================
# 配置文件加载
# =========================================================================
# 优先从外部配置目录加载
external_config = os.path.join(DATA_DIR, 'config', 'appconfig.ini')
internal_config = os.path.join(request.folder, 'private', 'appconfig.ini')

if os.path.exists(external_config):
    configuration = AppConfig(external_config, reload=True)
elif os.path.exists(internal_config):
    configuration = AppConfig(reload=True)  # 默认路径
else:
    raise HTTP(500, "配置文件不存在")

# =========================================================================
# 数据库初始化
# =========================================================================
db = DAL('sqlite://storage.sqlite',
    folder=db_folder,
    migrate=True,
    pool_size=10
)
```

#### 3.2 创建符号链接（可选）

如果不想修改太多代码，可以使用符号链接：

```bash
# 停止 Web2py 服务
sudo systemctl stop web2py

# 删除原目录（确保已备份！）
rm -rf /opt/<web2py_service>/applications/service_center/databases
rm -rf /opt/<web2py_service>/applications/service_center/sessions
rm -rf /opt/<web2py_service>/applications/service_center/errors
rm -rf /opt/<web2py_service>/applications/service_center/uploads
rm -rf /opt/<web2py_service>/applications/service_center/cache

# 创建符号链接
ln -s /var/web2py-data/service_center/databases /opt/<web2py_service>/applications/service_center/databases
ln -s /var/web2py-data/service_center/sessions /opt/<web2py_service>/applications/service_center/sessions
ln -s /var/web2py-data/service_center/errors /opt/<web2py_service>/applications/service_center/errors
ln -s /var/web2py-data/service_center/uploads /opt/<web2py_service>/applications/service_center/uploads
ln -s /var/web2py-data/service_center/cache /opt/<web2py_service>/applications/service_center/cache

# 配置文件符号链接
ln -sf /var/web2py-data/service_center/config/appconfig.ini /opt/<web2py_service>/applications/service_center/private/appconfig.ini

# 启动 Web2py 服务
sudo systemctl start web2py
```

### 第四阶段：更新 .gitignore

```gitignore
# ============================================
# 数据目录（符号链接或实际目录都排除）
# ============================================

# 数据库
databases/

# 会话
sessions/

# 错误日志
errors/

# 上传文件
uploads/

# 缓存
cache/

# ============================================
# 敏感配置
# ============================================
private/appconfig.ini
!private/appconfig.ini.example
```

### 第五阶段：验证与测试

```bash
# 1. 检查符号链接
ls -la /opt/<web2py_service>/applications/service_center/

# 2. 检查数据库连接
python -c "
import sys
sys.path.insert(0, '/opt/<web2py_service>')
from gluon.shell import exec_environment
env = exec_environment('/opt/<web2py_service>/applications/service_center/models/00_db.py')
print('数据库连接成功')
"

# 3. 访问应用测试
curl -I http://localhost:8000/service_center/

# 4. 检查日志
tail -f /var/web2py-data/service_center/errors/*.log
```

---

## 回滚方案

如果迁移出现问题，按以下步骤回滚：

### 快速回滚

```bash
# 1. 停止服务
sudo systemctl stop web2py

# 2. 删除符号链接
rm /opt/<web2py_service>/applications/service_center/databases
rm /opt/<web2py_service>/applications/service_center/sessions
rm /opt/<web2py_service>/applications/service_center/errors
rm /opt/<web2py_service>/applications/service_center/uploads

# 3. 恢复备份
cp -r /tmp/service_center_backup_databases /opt/<web2py_service>/applications/service_center/databases
mkdir -p /opt/<web2py_service>/applications/service_center/{sessions,errors,uploads}

# 4. 恢复配置
cp /tmp/service_center_backup_appconfig.ini /opt/<web2py_service>/applications/service_center/private/appconfig.ini

# 5. Git 回滚代码修改
cd /opt/<web2py_service>/applications/service_center
git checkout -- models/00_db.py

# 6. 启动服务
sudo systemctl start web2py
```

---

## 验证清单

### 迁移前检查

- [ ] 已备份 databases/ 目录
- [ ] 已备份 private/appconfig.ini
- [ ] 已备份 uploads/ 目录（如有数据）
- [ ] 已记录当前 Web2py 运行状态
- [ ] 已确认外部数据目录权限

### 迁移后检查

- [ ] 外部数据目录存在且权限正确
- [ ] 符号链接创建成功（如使用方案 A）
- [ ] 数据库文件已迁移
- [ ] 配置文件已迁移
- [ ] Web2py 服务正常启动
- [ ] 应用首页可访问
- [ ] 登录功能正常
- [ ] 数据库读写正常
- [ ] 文件上传功能正常（如有）
- [ ] Git 仓库状态干净（无意外文件）

### 长期监控

- [ ] 定期检查外部数据目录磁盘空间
- [ ] 配置外部数据目录备份策略
- [ ] 监控错误日志

---

## 附录

### 环境变量配置示例

```bash
# /etc/environment 或 ~/.bashrc
export WEB2PY_DATA_DIR=/var/web2py-data/service_center
```

### systemd 服务配置

```ini
# /etc/systemd/system/web2py.service
[Service]
Environment="WEB2PY_DATA_DIR=/var/web2py-data/service_center"
```

### Docker 配置示例

```yaml
# docker-compose.yml
services:
  web2py:
    image: web2py:latest
    volumes:
      - ./applications:/app/applications:ro  # 代码只读
      - web2py-data:/var/web2py-data         # 数据卷
    environment:
      - WEB2PY_DATA_DIR=/var/web2py-data/service_center

volumes:
  web2py-data:
```

---

*文档创建: 2026-01-22*
*作者: Claude AI*
