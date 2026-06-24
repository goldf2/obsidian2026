# Web2py 应用数据备份策略

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **适用范围**: /data/<web2py_service>/ 下所有应用数据
> **状态**: 🟢 Active

---

## 📋 目录

- [概述](#概述)
- [备份范围](#备份范围)
- [备份策略](#备份策略)
- [备份脚本](#备份脚本)
- [恢复流程](#恢复流程)
- [监控与告警](#监控与告警)

---

## 概述

### 备份目标

确保 Web2py 应用数据的安全性和可恢复性：

- ✅ 防止数据丢失（硬件故障、误操作）
- ✅ 支持快速恢复（RTO < 1小时）
- ✅ 保留历史版本（RPO < 24小时）
- ✅ 支持灾难恢复

### 数据目录结构

```
/data/<web2py_service>/
├── service_center/          # 已分离
│   ├── databases/           # SQLite 数据库
│   ├── sessions/            # 会话文件
│   ├── errors/              # 错误日志
│   └── uploads/             # 用户上传
├── <legacy_app>/                    # 待分离
├── <legacy_app>/                    # 待分离
└── ...                      # 其他应用
```

---

## 备份范围

### 必须备份

| 目录 | 说明 | 优先级 | 备份频率 |
|------|------|--------|----------|
| databases/ | SQLite 数据库文件 | 🔴 最高 | 每日 |
| uploads/ | 用户上传文件 | 🔴 最高 | 每日 |

### 建议备份

| 目录 | 说明 | 优先级 | 备份频率 |
|------|------|--------|----------|
| errors/ | 错误日志 | 🟡 中 | 每周 |

### 无需备份

| 目录 | 说明 | 原因 |
|------|------|------|
| sessions/ | 会话文件 | 临时数据，可重新生成 |
| cache/ | 缓存文件 | 临时数据，可重新生成 |

---

## 备份策略

### 3-2-1 备份原则

```
3 份数据副本
├── 1 份生产数据（原始）
├── 1 份本地备份
└── 1 份异地备份

2 种存储介质
├── 本地磁盘
└── 远程存储（NAS/云存储）

1 份异地存储
└── 防止本地灾难
```

### 备份周期

| 类型 | 频率 | 保留时间 | 说明 |
|------|------|----------|------|
| 每日增量备份 | 每天 02:00 | 7 天 | 只备份变化的文件 |
| 每周全量备份 | 每周日 03:00 | 4 周 | 完整备份所有数据 |
| 每月归档备份 | 每月 1 日 04:00 | 12 个月 | 长期保存 |

### 备份存储位置

```
本地备份目录: /backup/web2py-data/
├── daily/                   # 每日备份
│   ├── 2026-01-23/
│   ├── 2026-01-22/
│   └── ...
├── weekly/                  # 每周备份
│   ├── 2026-W04/
│   ├── 2026-W03/
│   └── ...
└── monthly/                 # 每月备份
    ├── 2026-01/
    └── ...

异地备份: /mnt/nas/backup/web2py-data/
         或 云存储（阿里云 OSS / AWS S3）
```

---

## 备份脚本

### 每日备份脚本

```bash
#!/bin/bash
# 文件: /opt/<web2py_service>/scripts/backup-daily.sh
# 用途: 每日增量备份 Web2py 应用数据
# 创建: 2026-01-23

set -e

# ============================================
# 配置
# ============================================
SOURCE_DIR="/data/<web2py_service>"
BACKUP_BASE="/backup/web2py-data/daily"
DATE=$(date +%Y-%m-%d)
BACKUP_DIR="${BACKUP_BASE}/${DATE}"
LOG_FILE="/var/log/web2py-backup.log"
RETENTION_DAYS=7

# 需要备份的应用列表
APPS=(
    "service_center"
    "<legacy_app>"
    "<legacy_app>"
    "jsll001"
    "jsll002"
    "shipping_management"
    "testing_center"
    "<legacy_app>"
    "<legacy_app>"
)

# 需要备份的子目录
BACKUP_DIRS=("databases" "uploads")

# ============================================
# 函数
# ============================================
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

backup_app() {
    local app_name=$1
    local app_source="${SOURCE_DIR}/${app_name}"
    local app_backup="${BACKUP_DIR}/${app_name}"

    if [ ! -d "$app_source" ]; then
        log "警告: 应用目录不存在: ${app_source}"
        return 1
    fi

    mkdir -p "$app_backup"

    for subdir in "${BACKUP_DIRS[@]}"; do
        local src="${app_source}/${subdir}"
        local dst="${app_backup}/${subdir}"

        if [ -d "$src" ]; then
            log "备份: ${app_name}/${subdir}"
            rsync -av --delete "$src/" "$dst/" >> "$LOG_FILE" 2>&1
        fi
    done

    return 0
}

cleanup_old_backups() {
    log "清理 ${RETENTION_DAYS} 天前的备份..."
    find "$BACKUP_BASE" -maxdepth 1 -type d -mtime +${RETENTION_DAYS} -exec rm -rf {} \;
}

# ============================================
# 主程序
# ============================================
main() {
    log "========== 开始每日备份 =========="
    log "备份目录: ${BACKUP_DIR}"

    mkdir -p "$BACKUP_DIR"

    local success=0
    local failed=0

    for app in "${APPS[@]}"; do
        if backup_app "$app"; then
            ((success++))
        else
            ((failed++))
        fi
    done

    # 清理旧备份
    cleanup_old_backups

    # 生成备份清单
    find "$BACKUP_DIR" -type f > "${BACKUP_DIR}/manifest.txt"
    du -sh "$BACKUP_DIR" >> "${BACKUP_DIR}/manifest.txt"

    log "========== 备份完成 =========="
    log "成功: ${success}, 失败: ${failed}"
    log "备份大小: $(du -sh "$BACKUP_DIR" | cut -f1)"
}

main "$@"
```

### 每周全量备份脚本

```bash
#!/bin/bash
# 文件: /opt/<web2py_service>/scripts/backup-weekly.sh
# 用途: 每周全量备份 Web2py 应用数据
# 创建: 2026-01-23

set -e

# ============================================
# 配置
# ============================================
SOURCE_DIR="/data/<web2py_service>"
BACKUP_BASE="/backup/web2py-data/weekly"
WEEK=$(date +%Y-W%V)
BACKUP_DIR="${BACKUP_BASE}/${WEEK}"
LOG_FILE="/var/log/web2py-backup.log"
RETENTION_WEEKS=4

# ============================================
# 主程序
# ============================================
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

main() {
    log "========== 开始每周全量备份 =========="
    log "备份目录: ${BACKUP_DIR}"

    mkdir -p "$BACKUP_DIR"

    # 全量备份整个数据目录
    log "执行全量备份..."
    tar -czf "${BACKUP_DIR}/web2py-data-${WEEK}.tar.gz" \
        -C "$(dirname $SOURCE_DIR)" \
        "$(basename $SOURCE_DIR)" \
        --exclude="*/sessions/*" \
        --exclude="*/cache/*" \
        2>> "$LOG_FILE"

    # 生成校验和
    cd "$BACKUP_DIR"
    sha256sum *.tar.gz > checksums.sha256

    # 清理旧备份
    log "清理 ${RETENTION_WEEKS} 周前的备份..."
    find "$BACKUP_BASE" -maxdepth 1 -type d -mtime +$((RETENTION_WEEKS * 7)) -exec rm -rf {} \;

    log "========== 全量备份完成 =========="
    log "备份大小: $(du -sh "$BACKUP_DIR" | cut -f1)"
}

main "$@"
```

### 数据库专用备份脚本

```bash
#!/bin/bash
# 文件: /opt/<web2py_service>/scripts/backup-databases.sh
# 用途: SQLite 数据库热备份（使用 sqlite3 .backup 命令）
# 创建: 2026-01-23

set -e

SOURCE_DIR="/data/<web2py_service>"
BACKUP_DIR="/backup/web2py-data/databases/$(date +%Y-%m-%d)"
LOG_FILE="/var/log/web2py-backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

backup_sqlite() {
    local db_file=$1
    local backup_file=$2

    if [ -f "$db_file" ]; then
        log "备份数据库: $db_file"
        sqlite3 "$db_file" ".backup '$backup_file'"
        return 0
    fi
    return 1
}

main() {
    log "========== 开始数据库备份 =========="

    mkdir -p "$BACKUP_DIR"

    # 查找所有 SQLite 数据库文件
    find "$SOURCE_DIR" -name "*.sqlite" -o -name "*.db" | while read db_file; do
        # 构建备份路径
        relative_path="${db_file#$SOURCE_DIR/}"
        backup_path="${BACKUP_DIR}/${relative_path}"
        backup_dir=$(dirname "$backup_path")

        mkdir -p "$backup_dir"
        backup_sqlite "$db_file" "$backup_path"
    done

    log "========== 数据库备份完成 =========="
}

main "$@"
```

### 安装备份脚本

```bash
# 1. 创建脚本目录
mkdir -p /opt/<web2py_service>/scripts

# 2. 创建备份目录
mkdir -p /backup/web2py-data/{daily,weekly,monthly,databases}

# 3. 保存脚本文件
# 将上述脚本保存到对应位置

# 4. 添加执行权限
chmod +x /opt/<web2py_service>/scripts/backup-*.sh

# 5. 配置定时任务
crontab -e

# 添加以下内容：
# 每日备份 - 凌晨 2:00
0 2 * * * /opt/<web2py_service>/scripts/backup-daily.sh

# 每周全量备份 - 周日凌晨 3:00
0 3 * * 0 /opt/<web2py_service>/scripts/backup-weekly.sh

# 数据库热备份 - 每 6 小时
0 */6 * * * /opt/<web2py_service>/scripts/backup-databases.sh

# 6. 验证定时任务
crontab -l
```

---

## 恢复流程

### 恢复前检查

```bash
# 1. 确认备份文件完整性
cd /backup/web2py-data/weekly/2026-W04/
sha256sum -c checksums.sha256

# 2. 检查备份内容
tar -tzf web2py-data-2026-W04.tar.gz | head -20

# 3. 确认恢复目标
ls -la /data/<web2py_service>/
```

### 恢复单个应用

```bash
#!/bin/bash
# 恢复单个应用的数据

APP_NAME="service_center"
BACKUP_DATE="2026-01-23"
BACKUP_DIR="/backup/web2py-data/daily/${BACKUP_DATE}/${APP_NAME}"
TARGET_DIR="/data/<web2py_service>/${APP_NAME}"

# 1. 停止 Web2py 服务（可选，建议）
systemctl stop web2py

# 2. 备份当前数据（以防万一）
mv "$TARGET_DIR" "${TARGET_DIR}.before-restore.$(date +%Y%m%d%H%M%S)"

# 3. 创建目标目录
mkdir -p "$TARGET_DIR"

# 4. 恢复数据
cp -r "${BACKUP_DIR}/databases" "$TARGET_DIR/"
cp -r "${BACKUP_DIR}/uploads" "$TARGET_DIR/"

# 5. 恢复权限
chown -R www-data:www-data "$TARGET_DIR"
chmod -R 755 "$TARGET_DIR"

# 6. 重新创建符号链接（如果需要）
cd /opt/<web2py_service>/applications/${APP_NAME}
rm -f databases uploads sessions errors
ln -s /data/<web2py_service>/${APP_NAME}/databases databases
ln -s /data/<web2py_service>/${APP_NAME}/uploads uploads
ln -s /data/<web2py_service>/${APP_NAME}/sessions sessions
ln -s /data/<web2py_service>/${APP_NAME}/errors errors

# 7. 启动服务
systemctl start web2py

# 8. 验证
curl -I http://localhost:8000/${APP_NAME}/
```

### 恢复全部数据（灾难恢复）

```bash
#!/bin/bash
# 灾难恢复 - 恢复全部数据

BACKUP_FILE="/backup/web2py-data/weekly/2026-W04/web2py-data-2026-W04.tar.gz"
TARGET_DIR="/data"

# 1. 停止服务
systemctl stop web2py

# 2. 验证备份
sha256sum -c /backup/web2py-data/weekly/2026-W04/checksums.sha256

# 3. 备份当前数据
if [ -d "/data/<web2py_service>" ]; then
    mv /data/<web2py_service> /data/<web2py_service>.disaster-backup.$(date +%Y%m%d%H%M%S)
fi

# 4. 解压恢复
cd "$TARGET_DIR"
tar -xzf "$BACKUP_FILE"

# 5. 恢复权限
chown -R www-data:www-data /data/<web2py_service>

# 6. 验证符号链接
for app in /opt/<web2py_service>/applications/*/; do
    app_name=$(basename "$app")
    if [ -L "${app}/databases" ]; then
        echo "检查 ${app_name}: $(readlink ${app}/databases)"
    fi
done

# 7. 启动服务
systemctl start web2py

# 8. 验证服务
systemctl status web2py
curl -I http://localhost:8000/
```

### 恢复单个数据库

```bash
# 恢复单个 SQLite 数据库

DB_BACKUP="/backup/web2py-data/databases/2026-01-23/service_center/databases/storage.sqlite"
DB_TARGET="/data/<web2py_service>/service_center/databases/storage.sqlite"

# 1. 停止服务（推荐）
systemctl stop web2py

# 2. 备份当前数据库
cp "$DB_TARGET" "${DB_TARGET}.before-restore.$(date +%Y%m%d%H%M%S)"

# 3. 恢复数据库
cp "$DB_BACKUP" "$DB_TARGET"

# 4. 恢复权限
chown www-data:www-data "$DB_TARGET"

# 5. 启动服务
systemctl start web2py
```

---

## 监控与告警

### 备份监控脚本

```bash
#!/bin/bash
# 文件: /opt/<web2py_service>/scripts/check-backup.sh
# 用途: 检查备份状态并发送告警

BACKUP_DIR="/backup/web2py-data/daily"
MAX_AGE_HOURS=26  # 超过 26 小时未备份则告警
ALERT_EMAIL="admin@example.com"

# 检查最新备份时间
latest_backup=$(ls -td ${BACKUP_DIR}/*/ 2>/dev/null | head -1)

if [ -z "$latest_backup" ]; then
    echo "错误: 未找到任何备份" | mail -s "[告警] Web2py 备份异常" "$ALERT_EMAIL"
    exit 1
fi

# 检查备份年龄
backup_age=$(( ($(date +%s) - $(stat -c %Y "$latest_backup")) / 3600 ))

if [ $backup_age -gt $MAX_AGE_HOURS ]; then
    echo "警告: 最新备份已超过 ${backup_age} 小时" | mail -s "[告警] Web2py 备份过期" "$ALERT_EMAIL"
    exit 1
fi

# 检查备份大小
backup_size=$(du -s "$latest_backup" | cut -f1)
if [ $backup_size -lt 1000 ]; then  # 小于 1MB 可能有问题
    echo "警告: 备份大小异常 (${backup_size}KB)" | mail -s "[告警] Web2py 备份异常" "$ALERT_EMAIL"
    exit 1
fi

echo "备份状态正常: ${latest_backup} (${backup_age}小时前, ${backup_size}KB)"
exit 0
```

### 添加监控定时任务

```bash
# 每天检查备份状态
0 8 * * * /opt/<web2py_service>/scripts/check-backup.sh
```

### 备份日志查看

```bash
# 查看备份日志
tail -f /var/log/web2py-backup.log

# 查看最近的备份记录
grep "备份完成" /var/log/web2py-backup.log | tail -10

# 查看备份错误
grep -i "error\|错误\|失败" /var/log/web2py-backup.log
```

---

## 备份检查清单

### 每日检查

- [ ] 检查备份日志是否有错误
- [ ] 确认备份文件已生成
- [ ] 验证备份文件大小合理

### 每周检查

- [ ] 验证全量备份完整性（sha256sum）
- [ ] 测试恢复流程（在测试环境）
- [ ] 检查备份存储空间

### 每月检查

- [ ] 完整恢复测试
- [ ] 审查备份策略
- [ ] 清理过期备份
- [ ] 更新备份文档

---

## 相关文档

- [应用分离方案](./app-separation-plan-应用分离方案.md)
- [Git 更新规则](./git-update-rules-Git更新规则.md)
- [代码数据分离规则](~/.claude/rules/devops/code-data-separation.md)

---

*文档创建: 2026-01-23*
*维护者: Claude AI*
