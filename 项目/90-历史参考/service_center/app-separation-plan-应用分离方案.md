# Web2py 应用数据分离方案

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **适用范围**: /opt/<web2py_service>/applications/ 下所有应用
> **数据目录**: /data/<web2py_service>/

---

## 📋 目录

- [概述](#概述)
- [应用清单与状态](#应用清单与状态)
- [分离方案详情](#分离方案详情)
- [批量执行脚本](#批量执行脚本)
- [验证与回滚](#验证与回滚)

---

## 概述

### 分离目标

将 web2py 应用的运行时数据（databases、sessions、errors、uploads）从代码目录分离到独立的数据目录，实现：

- ✅ 代码可以安全地进行版本控制
- ✅ 数据独立备份，不受代码更新影响
- ✅ 支持 CI/CD 自动化部署
- ✅ 提高系统安全性

### 目录结构

```
代码目录: /opt/<web2py_service>/applications/<app_name>/
数据目录: /data/<web2py_service>/<app_name>/
```

### 分离的目录

| 目录 | 说明 | 分离必要性 |
|------|------|-----------|
| databases/ | SQLite 数据库文件 | 🔴 必须 |
| sessions/ | 用户会话文件 | 🔴 必须 |
| errors/ | 错误日志文件 | 🟡 建议 |
| uploads/ | 用户上传文件 | 🔴 必须 |

---

## 应用清单与状态

### 应用总览

| 序号 | 应用名称 | 分离状态 | 优先级 | 备注 |
|------|---------|---------|--------|------|
| 1 | service_center | ✅ 已完成 | - | 模板应用 |
| 2 | <legacy_app> | ⏳ 待分离 | 🔴 高 | 汽车审计系统 |
| 3 | <legacy_app> | ⏳ 待分离 | 🔴 高 | 焊接生产系统 |
| 4 | jsll001 | ⏳ 待分离 | 🟠 中 | 题库系统 |
| 5 | jsll002 | ⏳ 待分离 | 🟠 中 | 题库系统 |
| 6 | shipping_management | ⏳ 待分离 | 🟠 中 | 物流管理 |
| 7 | testing_center | ⏳ 待分离 | 🟠 中 | 测试中心 |
| 8 | <legacy_app> | ⏳ 待分离 | 🟠 中 | 业务应用 |
| 9 | <legacy_app> | ⏳ 待分离 | 🟡 低 | 业务应用 |
| 10 | <legacy_app> | ⏳ 待分离 | 🟡 低 | 业务应用 |
| 11 | <legacy_app> | ⏳ 待分离 | 🟡 低 | 首页应用 |
| 12 | tools | ⏳ 待分离 | 🟡 低 | 工具应用 |
| 13 | active_record | ⏳ 待分离 | 🟢 可选 | 示例应用 |
| 14 | <example_app> | ⏳ 待分离 | 🟢 可选 | 示例应用 |
| 15 | <legacy_app> | ⏳ 待分离 | 🟢 可选 | 默认应用 |
| 16 | admin | ⚠️ 特殊 | - | 系统管理应用 |
| 17 | 1/ | ⚠️ 检查 | - | 需确认用途 |

### 状态说明

- ✅ **已完成**: 已创建符号链接，数据已迁移
- ⏳ **待分离**: 需要执行分离操作
- ⚠️ **特殊**: 需要特殊处理或评估

---

## 分离方案详情

### 模板：service_center（已完成）

当前符号链接状态：
```
databases -> /data/<web2py_service>/service_center/databases
errors -> /data/<web2py_service>/service_center/errors
sessions -> /data/<web2py_service>/service_center/sessions
uploads -> /data/<web2py_service>/service_center/uploads
```

---

### 应用 1: <legacy_app>

**应用路径**: `/opt/<web2py_service>/applications/<legacy_app>/`
**数据路径**: `/data/<web2py_service>/<legacy_app>/`
**优先级**: 🔴 高

**执行步骤**:

```bash
# 1. 创建数据目录
mkdir -p /data/<web2py_service>/<legacy_app>/{databases,sessions,errors,uploads}

# 2. 备份现有数据
cd /opt/<web2py_service>/applications/<legacy_app>
cp -r databases /data/<web2py_service>/<legacy_app>/databases.bak.$(date +%Y%m%d)
cp -r sessions /data/<web2py_service>/<legacy_app>/sessions.bak.$(date +%Y%m%d)
cp -r errors /data/<web2py_service>/<legacy_app>/errors.bak.$(date +%Y%m%d)
cp -r uploads /data/<web2py_service>/<legacy_app>/uploads.bak.$(date +%Y%m%d) 2>/dev/null || true

# 3. 迁移数据
mv databases/* /data/<web2py_service>/<legacy_app>/databases/ 2>/dev/null || true
mv sessions/* /data/<web2py_service>/<legacy_app>/sessions/ 2>/dev/null || true
mv errors/* /data/<web2py_service>/<legacy_app>/errors/ 2>/dev/null || true
mv uploads/* /data/<web2py_service>/<legacy_app>/uploads/ 2>/dev/null || true

# 4. 删除原目录
rm -rf databases sessions errors uploads

# 5. 创建符号链接
ln -s /data/<web2py_service>/<legacy_app>/databases databases
ln -s /data/<web2py_service>/<legacy_app>/sessions sessions
ln -s /data/<web2py_service>/<legacy_app>/errors errors
ln -s /data/<web2py_service>/<legacy_app>/uploads uploads

# 6. 设置权限
chown -R www-data:www-data /data/<web2py_service>/<legacy_app>/
chmod -R 755 /data/<web2py_service>/<legacy_app>/
```

---

### 应用 2: <legacy_app>

**应用路径**: `/opt/<web2py_service>/applications/<legacy_app>/`
**数据路径**: `/data/<web2py_service>/<legacy_app>/`
**优先级**: 🔴 高

**执行步骤**:

```bash
# 1. 创建数据目录
mkdir -p /data/<web2py_service>/<legacy_app>/{databases,sessions,errors,uploads}

# 2. 备份现有数据
cd /opt/<web2py_service>/applications/<legacy_app>
cp -r databases /data/<web2py_service>/<legacy_app>/databases.bak.$(date +%Y%m%d)
cp -r sessions /data/<web2py_service>/<legacy_app>/sessions.bak.$(date +%Y%m%d)
cp -r errors /data/<web2py_service>/<legacy_app>/errors.bak.$(date +%Y%m%d)
cp -r uploads /data/<web2py_service>/<legacy_app>/uploads.bak.$(date +%Y%m%d) 2>/dev/null || true

# 3. 迁移数据
mv databases/* /data/<web2py_service>/<legacy_app>/databases/ 2>/dev/null || true
mv sessions/* /data/<web2py_service>/<legacy_app>/sessions/ 2>/dev/null || true
mv errors/* /data/<web2py_service>/<legacy_app>/errors/ 2>/dev/null || true
mv uploads/* /data/<web2py_service>/<legacy_app>/uploads/ 2>/dev/null || true

# 4. 删除原目录
rm -rf databases sessions errors uploads

# 5. 创建符号链接
ln -s /data/<web2py_service>/<legacy_app>/databases databases
ln -s /data/<web2py_service>/<legacy_app>/sessions sessions
ln -s /data/<web2py_service>/<legacy_app>/errors errors
ln -s /data/<web2py_service>/<legacy_app>/uploads uploads

# 6. 设置权限
chown -R www-data:www-data /data/<web2py_service>/<legacy_app>/
chmod -R 755 /data/<web2py_service>/<legacy_app>/
```

---

### 应用 3-8: 中优先级应用

以下应用使用相同的分离模式：

| 应用 | 应用路径 | 数据路径 |
|------|---------|---------|
| jsll001 | /opt/<web2py_service>/applications/jsll001/ | /data/<web2py_service>/jsll001/ |
| jsll002 | /opt/<web2py_service>/applications/jsll002/ | /data/<web2py_service>/jsll002/ |
| shipping_management | /opt/<web2py_service>/applications/shipping_management/ | /data/<web2py_service>/shipping_management/ |
| testing_center | /opt/<web2py_service>/applications/testing_center/ | /data/<web2py_service>/testing_center/ |
| <legacy_app> | /opt/<web2py_service>/applications/<legacy_app>/ | /data/<web2py_service>/<legacy_app>/ |
| <legacy_app> | /opt/<web2py_service>/applications/<legacy_app>/ | /data/<web2py_service>/<legacy_app>/ |

**通用执行模板**（替换 `<APP_NAME>`）:

```bash
APP_NAME="<APP_NAME>"

# 1. 创建数据目录
mkdir -p /data/<web2py_service>/${APP_NAME}/{databases,sessions,errors,uploads}

# 2. 备份现有数据
cd /opt/<web2py_service>/applications/${APP_NAME}
for dir in databases sessions errors uploads; do
    [ -d "$dir" ] && cp -r $dir /data/<web2py_service>/${APP_NAME}/${dir}.bak.$(date +%Y%m%d)
done

# 3. 迁移数据
for dir in databases sessions errors uploads; do
    [ -d "$dir" ] && mv ${dir}/* /data/<web2py_service>/${APP_NAME}/${dir}/ 2>/dev/null || true
done

# 4. 删除原目录并创建符号链接
for dir in databases sessions errors uploads; do
    rm -rf $dir
    ln -s /data/<web2py_service>/${APP_NAME}/${dir} ${dir}
done

# 5. 设置权限
chown -R www-data:www-data /data/<web2py_service>/${APP_NAME}/
chmod -R 755 /data/<web2py_service>/${APP_NAME}/
```

---

### 应用 9-12: 低优先级应用

| 应用 | 应用路径 | 数据路径 |
|------|---------|---------|
| <legacy_app> | /opt/<web2py_service>/applications/<legacy_app>/ | /data/<web2py_service>/<legacy_app>/ |
| <legacy_app> | /opt/<web2py_service>/applications/<legacy_app>/ | /data/<web2py_service>/<legacy_app>/ |
| tools | /opt/<web2py_service>/applications/tools/ | /data/<web2py_service>/tools/ |

使用上述通用执行模板。

---

### 应用 13-15: 可选分离应用

| 应用 | 说明 | 建议 |
|------|------|------|
| active_record | 示例应用 | 可选分离 |
| <example_app> | 示例应用 | 可选分离 |
| <legacy_app> | 默认欢迎页 | 可选分离 |

这些应用通常不包含重要业务数据，可根据实际需要决定是否分离。

---

### 特殊应用处理

#### admin 应用

**说明**: Web2py 系统管理应用，用于管理其他应用。

**建议**:
- ⚠️ 谨慎处理，建议保持原状
- 如需分离，先在测试环境验证

#### 1/ 目录

**说明**: 需要确认此目录的用途。

**建议**:
- 检查目录内容和用途
- 如果是无效目录，考虑删除
- 如果是有效应用，按标准流程分离

---

## 批量执行脚本

### 完整批量分离脚本

```bash
#!/bin/bash
# 文件: /opt/<web2py_service>/scripts/batch-separate-apps.sh
# 用途: 批量执行应用数据分离
# 创建: 2026-01-23

set -e

# 配置
APP_BASE="/opt/<web2py_service>/applications"
DATA_BASE="/data/<web2py_service>"
LOG_FILE="/var/log/web2py-separation.log"

# 需要分离的应用列表（排除已完成和特殊应用）
APPS=(
    "<legacy_app>"
    "<legacy_app>"
    "jsll001"
    "jsll002"
    "shipping_management"
    "testing_center"
    "<legacy_app>"
    "<legacy_app>"
    "<legacy_app>"
    "<legacy_app>"
    "tools"
)

# 日志函数
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# 分离单个应用
separate_app() {
    local app_name=$1
    local app_path="${APP_BASE}/${app_name}"
    local data_path="${DATA_BASE}/${app_name}"

    log "========== 开始分离应用: ${app_name} =========="

    # 检查应用是否存在
    if [ ! -d "$app_path" ]; then
        log "警告: 应用目录不存在: ${app_path}"
        return 1
    fi

    # 检查是否已经是符号链接
    if [ -L "${app_path}/databases" ]; then
        log "跳过: ${app_name} 已经完成分离"
        return 0
    fi

    # 1. 创建数据目录
    log "创建数据目录: ${data_path}"
    mkdir -p "${data_path}"/{databases,sessions,errors,uploads}

    # 2. 备份和迁移数据
    cd "$app_path"
    for dir in databases sessions errors uploads; do
        if [ -d "$dir" ]; then
            log "备份目录: ${dir}"
            cp -r "$dir" "${data_path}/${dir}.bak.$(date +%Y%m%d)" 2>/dev/null || true

            log "迁移数据: ${dir}"
            mv ${dir}/* "${data_path}/${dir}/" 2>/dev/null || true

            log "删除原目录: ${dir}"
            rm -rf "$dir"
        fi

        log "创建符号链接: ${dir}"
        ln -s "${data_path}/${dir}" "${dir}"
    done

    # 3. 设置权限
    log "设置权限: ${data_path}"
    chown -R www-data:www-data "${data_path}/"
    chmod -R 755 "${data_path}/"

    log "完成: ${app_name} 分离成功"
    return 0
}

# 主函数
main() {
    log "========== 批量分离开始 =========="
    log "应用数量: ${#APPS[@]}"

    local success=0
    local failed=0

    for app in "${APPS[@]}"; do
        if separate_app "$app"; then
            ((success++))
        else
            ((failed++))
        fi
    done

    log "========== 批量分离完成 =========="
    log "成功: ${success}, 失败: ${failed}"
}

# 执行
main "$@"
```

### 使用方法

```bash
# 1. 创建脚本目录
mkdir -p /opt/<web2py_service>/scripts

# 2. 保存脚本
# 将上述内容保存到 /opt/<web2py_service>/scripts/batch-separate-apps.sh

# 3. 添加执行权限
chmod +x /opt/<web2py_service>/scripts/batch-separate-apps.sh

# 4. 执行脚本（建议先在测试环境验证）
/opt/<web2py_service>/scripts/batch-separate-apps.sh

# 5. 查看日志
tail -f /var/log/web2py-separation.log
```

---

## 验证与回滚

### 验证步骤

#### 1. 检查符号链接

```bash
# 检查所有应用的符号链接状态
for app in /opt/<web2py_service>/applications/*/; do
    app_name=$(basename "$app")
    echo "=== $app_name ==="
    ls -la "$app" | grep -E "^l" | grep -E "databases|sessions|errors|uploads"
done
```

#### 2. 检查数据目录

```bash
# 检查数据目录是否存在
ls -la /data/<web2py_service>/

# 检查各应用数据目录
for dir in /data/<web2py_service>/*/; do
    echo "=== $(basename $dir) ==="
    du -sh "$dir"
done
```

#### 3. 功能验证

```bash
# 重启 web2py 服务
systemctl restart web2py

# 检查服务状态
systemctl status web2py

# 访问各应用验证功能
# http://server:8000/<app_name>
```

### 回滚方案

如果分离后出现问题，可以回滚：

```bash
APP_NAME="<APP_NAME>"
APP_PATH="/opt/<web2py_service>/applications/${APP_NAME}"
DATA_PATH="/data/<web2py_service>/${APP_NAME}"

# 1. 删除符号链接
cd "$APP_PATH"
rm -f databases sessions errors uploads

# 2. 从备份恢复
for dir in databases sessions errors uploads; do
    backup=$(ls -d ${DATA_PATH}/${dir}.bak.* 2>/dev/null | head -1)
    if [ -n "$backup" ]; then
        cp -r "$backup" "${APP_PATH}/${dir}"
    else
        mkdir -p "${APP_PATH}/${dir}"
    fi
done

# 3. 设置权限
chown -R www-data:www-data "${APP_PATH}"/{databases,sessions,errors,uploads}
```

---

## .gitignore 模板

每个应用的 `.gitignore` 应包含：

```gitignore
# ============================================
# 运行时数据（符号链接，不提交）
# ============================================
databases
sessions
errors
uploads

# ============================================
# 敏感配置
# ============================================
private/appconfig.ini
private/*.bak
!private/appconfig.ini.example

# ============================================
# Python 编译文件
# ============================================
*.pyc
__pycache__/
*.py[cod]

# ============================================
# 临时文件
# ============================================
*.log
*.tmp
*.bak
*.swp
*~

# ============================================
# IDE 配置
# ============================================
.vscode/
.idea/
```

---

## 执行计划

### 阶段 1: 高优先级应用（本周）

- [ ] <legacy_app>
- [ ] <legacy_app>

### 阶段 2: 中优先级应用（下周）

- [ ] jsll001
- [ ] jsll002
- [ ] shipping_management
- [ ] testing_center
- [ ] <legacy_app>
- [ ] <legacy_app>

### 阶段 3: 低优先级应用（按需）

- [ ] <legacy_app>
- [ ] <legacy_app>
- [ ] tools

### 阶段 4: 可选应用（评估后决定）

- [ ] active_record
- [ ] <example_app>
- [ ] <legacy_app>

---

## 相关文档

- [代码数据分离规则](~/.claude/rules/devops/code-data-separation.md)
- [Git 更新规则](./git-update-rules-Git更新规则.md)
- [数据代码分离方案](./data-code-separation-plan-数据代码分离方案.md)

---

*文档创建: 2026-01-23*
*维护者: Claude AI*
