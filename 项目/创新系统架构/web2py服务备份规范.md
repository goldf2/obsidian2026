# <web2py_service> 备份规范

本文档规定 `<web2py_service>` 服务的代码、运行数据、容器持久化卷和 OSS 备份规则。当前执行口径如下：

- 代码目录：Git 仓库管理；传统部署实例路径通常为 `/opt/<web2py_service>`。
- 业务运行数据目录：逻辑路径为 `/data/<web2py_service>`。
- PaaS / 容器化部署：`/data/<web2py_service>` 必须映射为持久化存储，或把数据库、上传文件迁移到托管数据库/对象存储。
- 传统部署：服务名称通常为 `<web2py-service>.service`，本机监听端口按项目文档记录。

## 核心原则

代码资产与 `/data/<web2py_service>` 运行数据必须分开备份。传统部署中通常表现为 `/opt/<web2py_service>` 与 `/data/<web2py_service>` 分开备份；容器化部署中表现为镜像/仓库与持久化 volume 分开备份。

运行环境中可能存在大量指向 `/data/<web2py_service>` 的软链接，例如各 app 的 `databases`、`uploads`、`sessions`、`errors`、`cache`。这些软链接用于历史兼容，不属于代码资产，不进入 Git。备份代码目录时如果遇到这些软链接，只能保存软链接本身，不能跟随软链接把数据目录内容一起打包。

原因：

- 避免代码包重复包含运行数据。
- 避免 OSS 存储体积异常膨胀。
- 避免恢复时把数据恢复到错误位置。
- 保持代码发布、数据恢复、数据库回滚三者边界清晰。

## 备份分类

### 1. 代码备份

备份范围：

- `/opt/<web2py_service>`

包含：

- web2py 框架代码。
- `applications` 下的业务 app 代码。
- 静态资源。
- 视图、控制器、模型、模块。
- 配置模板和项目文档。
- 运行环境中已有的、指向 `/data/<web2py_service>` 的软链接本身。

不包含：

- 软链接目标内容。
- `/data/<web2py_service>` 里的数据库、上传附件、会话和错误票据。
- 临时压缩包、`.bak` 文件、历史手工备份目录。

推荐命令：

```bash
tar -czf <web2py_service>-code-$(date +%Y%m%d-%H%M%S).tar.gz -C /opt <web2py_service>
```

`tar` 默认不跟随软链接。不要添加 `-h` 或 `--dereference`。

### 2. 数据备份

备份范围：

- `/data/<web2py_service>`

包含：

- 各业务 app 的 `databases`。
- `uploads` 上传附件。

默认不包含：

- `sessions` 会话文件。
- `cache` 运行缓存。
- 可重新生成的临时预览缓存。

可选归档：

- `errors` 错误票据。
- app 自定义运行日志。

说明: `sessions`、`errors`、`cache` 不进入 Git，也不参与代码部署同步。`sessions` 和 `cache` 通常不需要常规备份；`errors` 可以按排障需要单独归档。

推荐命令：

```bash
tar -czf <web2py_service>-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /data <web2py_service>
```

如果只备份必须保留的业务数据，建议排除会话和缓存:

```bash
tar \
  --exclude='<web2py_service>/*/sessions' \
  --exclude='<web2py_service>/*/runtime/sessions' \
  --exclude='<web2py_service>/*/cache' \
  --exclude='<web2py_service>/*/runtime/cache' \
  -czf <web2py_service>-data-core-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C /data <web2py_service>
```

数据包可以按 app 拆分，例如：

```bash
tar -czf <web2py_service>-<business_app>-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /data/<web2py_service> <business_app>
tar -czf <web2py_service>-service_center-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /data/<web2py_service> service_center
```

## 禁止写法

代码备份禁止使用会跟随软链接的参数或工具模式：

```bash
tar -h ...
tar --dereference ...
rsync -L ...
rsync --copy-links ...
```

不建议直接使用普通 `zip -r` 备份代码目录，因为 zip 对软链接处理容易因版本和参数不同产生差异。

如果必须使用 zip，需要显式保存软链接：

```bash
zip -yr <web2py_service>-code.zip /opt/<web2py_service>
```

## OSS 上传结构

建议 OSS 按日期和类别分层：

```text
oss://<bucket>/<web2py_service>/
  code/YYYY/MM/DD/<web2py_service>-code-YYYYMMDD-HHMMSS.tar.gz
  data/YYYY/MM/DD/<web2py_service>-data-YYYYMMDD-HHMMSS.tar.gz
  app-data/YYYY/MM/DD/<app>-data-YYYYMMDD-HHMMSS.tar.gz
```

代码包和数据包必须分开上传、分开保留周期。

建议保留策略：

- 代码备份：保留 30 天，重大版本长期保留。
- 全量数据备份：保留 30 天。
- 关键 app 数据备份：按业务要求额外保留 90 天或更长。
- 手工迁移前备份：确认稳定后再清理。

## 恢复原则

恢复代码。传统部署示例:

```bash
systemctl stop <web2py-service>.service
tar -xzf <web2py_service>-code-YYYYMMDD-HHMMSS.tar.gz -C /opt
systemctl start <web2py-service>.service
```

恢复数据。传统部署示例:

```bash
systemctl stop <web2py-service>.service
tar -xzf <web2py_service>-data-YYYYMMDD-HHMMSS.tar.gz -C /data
chown -R nobody:nogroup /data/<web2py_service>
systemctl start <web2py-service>.service
```

恢复到 PaaS 或容器化平台时，应先恢复持久化存储、托管数据库或对象存储，再按目标 tag / 镜像重新部署，并验证健康检查。

恢复后必须验证。传统部署示例:

```bash
systemctl is-active <web2py-service>.service
curl -sS -I http://127.0.0.1:8014/service_center/default/index
curl -sS -I http://127.0.0.1:8014/<business_app>/default/index
curl -sS -I http://127.0.0.1:8014/admin/default/index
```

PaaS 或容器化部署按项目实际域名验证:

```bash
curl -sS -I https://<domain>/service_center/default/index
curl -sS -I https://<domain>/<business_app>/default/index
```

## 当前目录规则

业务 app 的运行数据目录应放在 `/data/<web2py_service>/<app>/`。历史 app 如果仍依赖 web2py 默认目录，可以在运行环境中保留软链接；软链接由部署/维护脚本创建和审计，不提交到 Git。

例：

```text
/opt/<web2py_service>/applications/<business_app>/databases -> /data/<web2py_service>/<business_app>/databases
/opt/<web2py_service>/applications/<business_app>/uploads   -> /data/<web2py_service>/<business_app>/uploads
/opt/<web2py_service>/applications/<business_app>/sessions  -> /data/<web2py_service>/<business_app>/sessions
/opt/<web2py_service>/applications/<business_app>/errors    -> /data/<web2py_service>/<business_app>/errors
```

软链接检查和补建:

```bash
cd /opt/<web2py_service>
scripts/runtime_links.sh audit --strict
scripts/runtime_links.sh ensure --app <business_app> --execute
```

`ensure` 只创建缺失软链接，不迁移、不删除已有真实目录。

例外：

- `admin` 是 web2py 管理后台 app，保持原生目录结构，不迁移到 `/data/<web2py_service>/admin`。
- 历史归档目录不纳入常规运行数据分离策略。

## 备份前检查

执行 OSS 备份前建议检查：

```bash
find /opt/<web2py_service>/applications -maxdepth 2 -type l -printf '%p -> %l\n' | sort
find /opt/<web2py_service>/applications -maxdepth 3 -xtype l -printf 'broken: %p -> %l\n'
cd /opt/<web2py_service> && scripts/runtime_links.sh audit --strict
du -sh /opt/<web2py_service> /data/<web2py_service>
```

如果代码包体积突然接近或超过 `/opt/<web2py_service> + /data/<web2py_service>` 的合计，通常说明备份命令跟随了软链接，应立即停止上传并检查脚本。

## Git 配合

以后代码变更以 Git 为准，不再依赖 `.bak` 文件。

要求：

- 不在代码目录长期保留 `.bak`、`.bak_*`、`*bak*` 文件。
- 重要修改先提交 Git。
- 发布前确认 `git status`。
- OSS 代码备份用于灾难恢复，不替代 Git 历史。
