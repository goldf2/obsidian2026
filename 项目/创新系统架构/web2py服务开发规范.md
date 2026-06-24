# web2py 服务平台开发规范

> 更新日期: 2026-06-23  
> 适用范围: web2py 服务的新 app、新功能、既有 app 改造和上线部署  
> 部署原则: web 开发规范保持平台中立；PaaS、容器平台和传统 systemd 都是部署实现选项  
> 当前运行入口: `https://<domain>`

本文档用于把 web2py 服务平台后续 app 和功能开发统一到同一套登录、权限、菜单实现、目录、数据库和交付规范上。既有老应用如果暂时不改造，可以保留现状；新开发和被主动改造的功能必须按本文执行。部署治理、PaaS 选型和迁移阶段以 `数据存储与PaaS部署治理路线图.md` 为总控，本文只规定开发侧必须满足的落地约束。

## 1. 文档范围与架构引用

本文档只规定 web2py 服务平台的开发落地规则，包括目录、认证、权限、菜单实现、UI、模块拆分、发布验收和维护要求。

系统分层、应用边界、运行架构、导航层级和数据关系不在本文重复定义，统一以以下文档为准：

- `创新系统架构/系统架构与功能图.md`
- `创新系统架构/系统导航与页面总览.md`
- `菜单架构统一规范.md`

开发时按以下顺序使用文档：

1. 先读系统架构文档，确认平台 app、业务 app、存量 app 和运行数据边界。
2. 再读本文档，按规范实现模型、控制器、菜单、权限、模块和页面。
3. 涉及运行数据、数据库、上传目录和迁移时，以 `数据代码分离统一规范.md` 为准。
4. 涉及 PaaS 选型、Docker Compose、传统 systemd 兼容和生产切换时，以 `数据存储与PaaS部署治理路线图.md` 和 `持续集成与持续部署标准流程方案.md` 为准。

## 2. 基本开发约束

- 新 app 命名必须使用小写英文、数字和下划线，例如 `quality_trace`、`equipment_maintenance`。
- 新 app 不允许自建独立认证体系，默认接入平台 app 的统一认证。
- 平台级能力只放一处；业务能力按业务边界进入对应 app。
- app 代码目录与运行数据目录必须分离。
- 旧应用不作为新规范模板；主动改造时逐步接入统一认证、导航、权限和数据分离规范。

## 3. 目录和数据分离规范

本节只记录开发规范层面的原则。目录创建、配置项、DAL `folder=`、upload 字段、旧 app 迁移、回滚和验收细节统一以 `数据代码分离统一规范.md` 为准。

### 3.1 代码目录与数据目录边界

新 app 代码目录。传统宿主机部署通常位于 `/opt/<web2py_service>/applications/<app>`；容器内只要求位于 web2py 根目录的 `applications/<app>`，业务代码不得硬编码宿主机 `/opt` 路径:

```text
/opt/<web2py_service>/applications/<app>/
├── controllers/
├── models/
├── modules/
├── views/
├── static/
├── private/
├── cron/
├── languages/
├── README.md
└── INTEGRATION.md
```

运行数据目录。传统宿主机、容器和 PaaS 环境均推荐使用 `/data/<web2py_service>/<app>` 作为逻辑运行数据路径；具体平台通过持久化磁盘、volume、对象存储或托管数据库实现持久化:

```text
/data/<web2py_service>/<app>/
├── config/
├── databases/
├── uploads/
├── runtime/
│   ├── sessions/
│   ├── errors/
│   ├── cache/
│   └── logs/
└── backups/
```

app 目录只放代码、模板、静态资源、配置模板和 app 级说明文档。真实数据库、上传文件、生产配置、会话、错误、缓存、日志和备份统一放到 `/data/<web2py_service>/<app>`。

### 3.2 配置驱动的数据代码分离

新 app 和主动改造的 app 必须通过配置文件显式读取运行数据目录，不再把软链接作为标准方案。

最小要求:

- 配置模板提交到 `applications/<app>/private/appconfig.ini.example`。
- 真实配置放到 `/data/<web2py_service>/<app>/config/appconfig.ini`，不提交 Git。
- 数据库连接使用 DAL `folder=` 指向外部数据目录。
- 上传字段使用 `uploadfolder=` 指向外部上传目录。
- 上传真实目录通过 `applications/service_center/modules/platform_storage.py` 或 app 级配置模块统一解析。
- 业务代码不得重新拼接 `request.folder/uploads` 或依赖 app 内 `databases/`。

### 3.3 Git 和历史软链接边界

运行数据、真实配置、SQLite 文件、日志和软链接不得进入 Git。历史 app 可以临时保留运行目录软链接以兼容旧代码，但软链接只由部署/维护脚本创建和审计。

审计命令:

```bash
scripts/runtime_links.sh audit --strict
scripts/ci_check.sh
```

如果本总览文档和数据分离统一规范存在差异，以 `数据代码分离统一规范.md` 为准。

## 4. 新 app 初始化规范

### 4.1 命名

- app 目录: 小写英文、数字、下划线，例如 `equipment`、`quality_trace`。
- Python 变量/函数: 小写下划线。
- 数据表: `<app>_<业务名>`，例如 `inspection_record`、`mes_product`。
- 权限角色: `<app>_<范围>_<操作>` 或 `<app>_<role>`。
- 菜单 seed_key: `top.<app>`、`sidebar.<domain>.<function>`。

### 4.2 必备文档

每个新 app 至少包含:

| 文件 | 用途 |
| --- | --- |
| `README.md` | app 目标、入口、核心页面、数据表、部署注意事项 |
| `INTEGRATION.md` | 与统一认证、导航、权限、其他 app 的关系 |
| `private/appconfig.ini.example` | 如果 app 有配置项，必须给示例，不提交真实密钥 |

## 5. 统一认证规范

### 5.1 基本原则

`service_center` 是唯一认证中心。共享模块位于:

```text
/opt/<web2py_service>/applications/service_center/modules/service_center_auth.py
```

它会初始化:

- `db`: 指向 `service_center/databases/storage.sqlite` 的认证数据库。
- `auth`: 统一 web2py Auth 对象。
- `crud`: 基于认证库的 Crud。
- `session.connect(..., masterapp='service_center')`: 共享 session。
- `auth_user.phone`: 统一手机号字段。

### 5.2 新 app 的 `models/00_db.py` 模板

如果 app 没有独立业务库，直接使用共享认证库:

```python
# -*- coding: utf-8 -*-

import os
import sys

shared_modules = os.path.join(request.folder, '..', 'service_center', 'modules')
if shared_modules not in sys.path:
    sys.path.append(shared_modules)

from service_center_auth import apply_service_center_auth
apply_service_center_auth(globals())

auth_db = db
```

如果 app 有独立业务库，必须先加载统一认证，再创建业务库:

```python
# -*- coding: utf-8 -*-

import os
import sys
from gluon.contrib.appconfig import AppConfig

shared_modules = os.path.join(request.folder, '..', 'service_center', 'modules')
if shared_modules not in sys.path:
    sys.path.append(shared_modules)

from service_center_auth import apply_service_center_auth
apply_service_center_auth(globals())

auth_db = db

configuration = AppConfig(reload=True)
runtime_root = configuration.get('storage.runtime_root') or os.path.join(
    '/data/<web2py_service>',
    request.application,
)
db_folder = os.path.join(runtime_root, 'databases')
if not os.path.exists(db_folder):
    os.makedirs(db_folder)

db_app = DAL(
    configuration.get('db.uri') or 'sqlite://%s.sqlite' % request.application,
    migrate=configuration.get('db.migrate'),
    pool_size=configuration.get('db.pool_size') or 10,
    folder=db_folder,
)
```

### 5.3 数据库变量命名

- `auth_db`: 统一认证数据库别名。
- `db`: 在 `apply_service_center_auth()` 后默认指向认证库。
- `db_<app>` 或 `db_app`: 独立业务数据库。

如果控制器中业务操作很多，建议显式设置:

```python
_db = db_app
```

不要在同一个文件里混用 `db.inspection_*` 和 `db_app.inspection_*`。

### 5.4 登录入口

所有 app 的登录、登出、个人资料、找回密码都应跳转到:

```text
/service_center/default/user/<action>
```

app 内的 `user()` 函数只做重定向:

```python
def user():
    target = URL('service_center', 'default', 'user', args=request.args, vars=request.vars)
    redirect(target)
```

未登录访问业务页面时，应跳转到统一登录，并带 `_next`:

```python
redirect(URL('service_center', 'default', 'user', args=['login'], vars={'_next': URL()}))
```

## 6. 权限规范

### 6.1 角色层级

平台级角色:

| 角色 | 说明 |
| --- | --- |
| `admin` | 系统管理员，默认拥有平台和各 app 管理权限 |

app 级角色:

| 命名 | 示例 | 说明 |
| --- | --- | --- |
| `<app>_super_admin` | `inspection_super_admin` | app 最高权限 |
| `<app>_admin` | `inspection_admin` | app 管理员 |
| `<app>_全部_<操作>` | `inspection_全部_查看` | 全局业务权限 |
| `<app>_<部门>_<操作>` | `inspection_质量部_录入` | 部门/范围权限 |

操作建议统一为:

| 英文动作 | 中文角色动作 | 说明 |
| --- | --- | --- |
| `view` | `查看` | 查看页面和列表 |
| `create`/`inspect` | `录入` | 新增采集、提交记录 |
| `review` | `审核` | 审核、确认、关闭异常 |
| `manage` | `管理` | 配置、删除、角色内业务管理 |
| `export` | `导出` | 数据导出 |

### 6.2 权限函数

每个 app 应提供一个集中权限模块，例如:

```text
models/04_permissions.py
```

规则:

- 权限判断集中在函数里，不要散落在控制器和视图里。
- `admin` 必须默认放行。
- 未登录跳统一登录。
- 无权限跳 app 首页或服务中心首页，不能跳回当前页造成重定向循环。
- 初始化角色可以自动创建，但不能在生产环境自动创建公开默认密码账号。

### 6.3 默认账号要求

开发环境可以有默认账号说明；生产环境禁止新增长期固定密码的默认管理员。确实需要初始化账号时，应满足:

- 首次部署后立即改密。
- 文档中标注用途和移除方式。
- 后续改为通过 `service_center` 用户管理或批量导入创建。

## 7. 导航与菜单实现规范

### 7.1 实现边界

导航层级、区域归属、页面清单和 `response.menu`、`response.sub_menu`、`response.page_menu` 的系统级契约由 `菜单架构统一规范.md` 定义。本文只保留开发落地摘要；具体菜单来源、禁止事项、存量治理顺序和验收清单以 `菜单架构统一规范.md` 为准。

开发实现必须遵守以下边界:

- 业务 app 不重新定义导航变量含义。
- layout 只渲染系统架构定义的导航变量，不硬编码另一套平行导航。
- 页面动作，例如新增、编辑、导入、导出、审核、刷新，放在当前页面工具栏，不放入菜单。

### 7.2 顶部导航

平台顶部导航的代码默认来源:

```text
/opt/<web2py_service>/applications/service_center/modules/platform_nav.py
```

各 app 通过加载:

```text
/opt/<web2py_service>/applications/service_center/models/xx_menu.py
```

获得:

```python
response.menu
```

新 app 必须在 layout 中渲染 `response.menu`，不得自己硬编码一个完全不同的顶部导航。

### 7.3 顶部菜单新增规范

新增平台入口时，优先在 `platform_nav.py` 添加:

```python
dict(
    seed_key='top.<app>',
    title='<中文名称>',
    url=URL('<app>', 'default', 'index'),
    icon='fas fa-...',
    sort_order=...,
    match_app='<app>',
    match_controller='default',
    match_functions='index,dashboard,...',
)
```

字段规范:

| 字段 | 说明 |
| --- | --- |
| `seed_key` | 稳定键，不随显示名称变化 |
| `title` | 菜单显示名 |
| `url` | 入口 URL |
| `icon` | Font Awesome 图标 |
| `sort_order` | 排序 |
| `match_app` | 当前 app 匹配 |
| `match_controller` | 当前 controller 匹配 |
| `match_functions` | 当前 function 匹配，逗号分隔 |

如果线上已经启用数据库菜单表，代码默认值和数据库 seed 要一起检查，避免只改代码但运行时仍读取旧数据库记录。

### 7.4 平台左侧菜单接入

平台左侧菜单的职责由系统架构文档定义。开发实现只做接入:

- layout 渲染平台传入的 `response.sub_menu`。
- 业务 app 不写入、不覆盖 `response.sub_menu`。
- 不复制一套平行的侧栏 DOM/CSS 来绕开统一菜单。

### 7.5 app 页内菜单实现

app 页内菜单的职责由系统架构文档定义。开发实现只负责生成当前 app 的 `response.page_menu`。

推荐实现:

- 简单 app: 在 `<app>/models/*_menu.py` 中直接设置 `response.page_menu`。
- 复杂 app: 建独立业务菜单表，例如 `<app>_menu_item`，再把当前用户可访问的菜单写入 `response.page_menu`。
- 业务 app 不写入、不覆盖 `response.menu` 或 `response.sub_menu`。

### 7.6 菜单变更后的处理

修改 `service_center/modules/platform_nav.py` 后，如果线上没有立即生效，应重启:

```bash
systemctl restart <web2py-service>.service
```

如果复杂 app 将页内菜单 seed 到业务菜单表，还要确认数据库中存在对应 `seed_key`，必要时补数据。

## 8. 页面与 UI 规范

### 8.1 布局

- 顶部导航: 使用统一 `response.menu`。
- 平台侧边栏: 使用统一 `response.sub_menu`，展示平台和业务域入口。
- 页内菜单: 使用 `response.page_menu`，仅展示当前 app 内部页面入口。
- 登录页: 统一走 `service_center/default/user/login`。
- 管理页: 优先接入 `service_center/mgr` 或 app 自己的管理页，但认证权限仍共用。
- 业务 app 不复制平台侧边栏 DOM/CSS，不覆盖 `response.sub_menu`。

### 8.2 前端资源

- 复用 `service_center/static` 中已有 Bootstrap/AdminLTE/Font Awesome 资源。
- 不重复拷贝整套前端库到每个 app。
- 新增 app 样式放在本 app 的 `static/css/app.css` 或按功能拆分。
- CSS 类名建议加 app 前缀，避免影响其他 app，例如 `.<quality_app>-shell`、`.<business_app>-sidebar`。

### 8.3 交互一致性

- 列表页统一提供搜索、筛选、分页、空状态。
- 表单页统一提供保存、取消、返回入口。
- 删除/关闭/作废类操作必须二次确认。
- 成功/失败使用 `session.flash` 或统一 alert 区域。
- 业务页面不要直接显示 Python 异常，生产环境关闭 generic patterns。

## 9. Controller 规范

### 9.1 控制器职责

控制器负责:

- 登录和权限装饰。
- 读取 request 参数并校验。
- 调用本 app 的 model/module 函数。
- 返回视图所需数据。
- 处理 redirect、flash、HTTP 状态。

控制器不应该:

- 大量拼接 SQL。
- 重复定义权限规则。
- 写复杂业务算法。
- 写跨 app 的隐式副作用。

### 9.2 函数命名

- 对外页面函数: `index`、`dashboard`、`records`、`detail`、`create`、`edit`。
- 私有 helper: 以 `_` 开头，例如 `_parse_date_bounds()`。
- 下载/预览: `download`、`preview`、`inline_file`，必须校验权限和文件归属。

### 9.3 请求处理

- 修改数据优先使用 POST。
- POST 成功后 redirect，避免刷新重复提交。
- 参数必须做类型转换和默认值处理。
- 文件上传必须校验扩展名、大小、归属对象和下载权限。

### 9.4 Python 3.10 上传对象兼容

web2py 上传字段在 Python 3.10 环境下可能返回 `cgi.FieldStorage` 对象。该对象不能安全地参与布尔判断，下面写法会触发 `TypeError: Cannot be converted to bool.`:

```python
if not upload:
    return None

if filename and upload.file:
    ...
```

必须改为显式判断 `None`、文件名和文件对象:

```python
if upload is None:
    return None

filename = getattr(upload, "filename", "") or ""
fileobj = getattr(upload, "file", None)
if not filename or fileobj is None:
    return None
```

多文件上传列表收集也不能对单个上传对象做 `if not value`:

```python
def upload_list(*values):
    uploads = []

    def collect(value):
        if value is None:
            return
        if isinstance(value, (list, tuple)):
            for item in value:
                collect(item)
            return
        filename = getattr(value, "filename", "") or ""
        fileobj = getattr(value, "file", None)
        if filename and fileobj is not None:
            uploads.append(value)

    for value in values:
        collect(value)
    return uploads
```

审查规则:

- 禁止对上传对象写 `if upload`、`if not upload`、`bool(upload)`。
- 禁止用 `if filename and fileobj` 判断文件对象是否存在。
- 可以对普通列表使用 `if not uploads`，因为 `uploads` 是本地构造的 list，不是 `FieldStorage`。
- 修改上传逻辑后必须用模拟 `__bool__` 抛 `TypeError` 的对象测试一次。

### 9.5 模块分层与复用边界

模块归属必须先判断“复用范围”，再决定放置位置。基本原则:

- 大概率跨 app 公用的基础功能，放入 `service_center/modules/`。
- 只在当前 app 内复用的业务能力，放入当前 app 的 `modules/`。
- 只服务单个页面的组装逻辑，放入当前 app 的 `modules/pages/` 或保持在轻量页面服务中。
- 控制器只保留 URL 入口、权限装饰、模板选择和重定向，不承载复杂业务规则。

#### 9.5.1 放入 `service_center/modules/` 的条件

满足以下任意条件，优先放入 `service_center/modules/`:

- 两个及以上 app 已经需要，或大概率会被两个及以上 app 使用。
- 与业务域无关，例如搜索、筛选、分页、导出、表单值解析、日期解析、上传校验、主题模式、导航上下文。
- 与统一平台能力相关，例如认证、权限基础工具、顶部导航、系统设置、统一 UI 框架。
- 修改后应该让所有 app 行为保持一致。

推荐命名:

```text
service_center/modules/platform_filters.py
service_center/modules/platform_forms.py
service_center/modules/platform_uploads.py
service_center/modules/platform_export.py
service_center/modules/platform_ui.py
service_center/modules/platform_nav.py
```

`service_center/modules/` 中禁止放入只属于某个业务域的规则，例如:

- 质量检查公差判定。
- 业务系统 工单状态流转。
- 仓储出入库业务规则。
- 某个 app 专属表结构的写入流程。

#### 9.5.2 放入 `<app>/modules/` 的条件

满足以下条件，放入当前 app 的 `modules/`:

- 逻辑只理解当前 app 的业务表和业务规则。
- 逻辑会被当前 app 多个控制器或页面复用。
- 逻辑涉及当前 app 的数据写入、状态流转、领域术语和校验规则。

推荐目录:

```text
<app>/modules/domain/      # 领域规则，例如质量判定、状态机、标签生成
<app>/modules/services/    # 业务服务，例如记录写入、检查点维护、报表查询
<app>/modules/pages/       # 页面数据组装，例如 dashboard_page.py、records_page.py
```

以 `<quality_app>` 为例:

```text
<quality_app>/modules/domain/quality_rules.py       # 公差、合格/预警/不合格判定
<quality_app>/modules/domain/process_labels.py      # 质量作业指导书显示名称
<quality_app>/modules/services/record_service.py    # 检测记录查询、写入、详情
<quality_app>/modules/services/checkpoint_service.py # 检查点维护
<quality_app>/modules/services/photo_service.py     # 现场照片
<quality_app>/modules/pages/records_page.py         # 记录页面组装
```

#### 9.5.3 判断口径

新增函数、类或模块前，必须先回答:

1. 这个能力是否和具体业务表、业务状态、业务术语强相关？  
   是则放 app 内部模块。

2. 这个能力换一个 app 是否仍然可以直接使用？  
   是则优先放 `service_center/modules/`。

3. 这个能力是否只是页面数据组装？  
   是则放 `modules/pages/`，不要放平台公共模块。

4. 这个能力是否需要统一全平台行为？  
   是则放 `service_center/modules/`，并写清楚接口。

5. 当前只有一个 app 使用，但明显是平台基础能力吗？  
   是则也可以先放 `service_center/modules/`，但接口必须保持业务无关。

#### 9.5.4 复用方式

跨 app 公共模块引用建议使用明确导入:

```python
from applications.service_center.modules.platform_filters import SearchFilterToolkit
```

app 内模块引用建议使用:

```python
from applications.<quality_app>.modules.services.record_service import InspectionRecordService
```

不要在多个 app 中复制同一份工具代码。确实需要临时复制时，必须在注释或文档中标注后续统一归并位置。

#### 9.5.5 迁移和重构顺序

从大控制器或大模块瘦身时，按以下顺序拆:

1. 先拆跨 app 基础能力到 `service_center/modules/`，例如搜索、筛选、导出、上传、表单解析。
2. 再拆 app 内业务服务到 `<app>/modules/services/`。
3. 再拆领域规则到 `<app>/modules/domain/`。
4. 最后拆页面数据组装到 `<app>/modules/pages/`。

这样可以先消除重复代码，再降低单个 app 的业务复杂度。

## 10. Model 和数据库规范

### 10.1 表命名

表名必须带 app 或业务域前缀:

```text
inspection_product
inspection_record
mes_product
mes_work_instruction
```

禁止新建过于泛化的表名:

```text
record
file
product
settings
```

### 10.2 字段规范

通用字段建议:

```python
Field('created_by', 'reference auth_user', writable=False, readable=False)
Field('created_on', 'datetime', default=request.now, writable=False, readable=False)
Field('modified_by', 'reference auth_user', writable=False, readable=False)
Field('modified_on', 'datetime', default=request.now, update=request.now, writable=False, readable=False)
Field('is_active', 'boolean', default=True)
```

业务状态字段建议使用稳定英文值，展示时映射中文:

```python
STATUS_OPTIONS = [
    ('draft', '草稿'),
    ('active', '启用'),
    ('closed', '关闭'),
]
```

### 10.3 外键规范

- 引用统一用户: 使用认证库 `auth_user`。
- 引用本 app 业务表: 使用本 app 业务库。
- 跨 app 引用: 优先使用稳定业务键，例如产品编码、工单号、批次号；确实需要跨库查表时，封装到明确的 service/helper 函数。

### 10.4 migrate 规范

- 开发期可以 `migrate=True`。
- 生产期表结构变更必须先备份数据库。
- 高风险变更不依赖自动 migrate，应该写清楚迁移步骤。
- SQLite 表结构变更上线前必须在测试库验证。

## 11. 跨 app 集成规范

### 11.1 链接集成

跨 app 链接使用 web2py `URL()`:

```python
URL('<quality_app>', 'default', 'dashboard')
URL('<business_app>', 'master_data', 'products')
```

不要硬编码完整域名，除非是生成外部通知链接。

### 11.2 数据集成

跨 app 数据集成应遵循:

- 先定义业务主键: 产品编码、图号、工单号、批次号、供应商编码等。
- 再定义读取边界: 谁拥有源数据，谁只读引用。
- 最后定义同步方式: 实时查询、定时同步、导入导出或接口。

不要在一个 app 的控制器中直接修改另一个 app 的多张业务表。确实需要时，封装到模块函数并写入 `INTEGRATION.md`。

### 11.3 业务系统 与 <quality_app> 的边界

业务系统 内置质量管理可以保留“生产过程质量”的入口，例如工序检验、抽检结果摘要、质量状态回写。

<quality_app> 负责:

- 质量检查标准和检查点。
- 现场采集。
- 检测记录。
- 异常闭环。
- 质量统计报表。

两者通过产品、工单、批次、工序和质量状态关联。

## 12. 配置和密钥规范

- 真实密码、Token、SMTP 密码、短信密钥不得提交到仓库。
- 生产配置优先放在 `/data/<web2py_service>/<app>/config/appconfig.ini`，容器化部署可通过环境变量覆盖或指定 `WEB2PY_CONFIG_FILE`。
- app 内 `private/appconfig.ini` 只作为开发环境或历史兼容回退。
- 示例配置提交为 `private/appconfig.ini.example`。
- 文档中可以写配置项名称，不写生产密钥。
- 默认管理员密码不应写入长期生产文档；如果历史文档已存在，应尽快替换为初始化流程。

## 13. 日志、错误和审计

- PaaS 或容器化部署优先把 web2py 访问日志、错误日志输出到 stdout/stderr，便于平台采集。
- 传统宿主机部署可继续使用 `/var/log/<web2py_service>/httpserver.log` 和 Nginx 站点日志。
- app 错误、缓存、会话目录应由配置或运行时目录设置指向 `/data/<web2py_service>/<app>/runtime`；历史软链接只能作为兼容方式。
- 关键业务操作建议记录操作人、操作时间、原状态、新状态。
- 删除数据优先软删除或状态作废，除非业务明确要求物理删除。

## 14. 测试与验收规范

### 14.1 代码检查

每次修改 Python 文件后至少做语法检查:

```bash
python3 -m py_compile <file.py>
```

或批量 AST 检查。

### 14.2 登录和权限检查

新增 app 或新增受保护页面必须验证:

- 未登录访问业务 URL，会跳到 `/service_center/default/user/login`。
- 登录后能回到 `_next` 指定页面。
- 没有权限的普通用户不会看到受限功能。
- `admin` 用户可以进入平台管理和 app 管理功能。

### 14.3 页面冒烟检查

至少检查:

```text
/service_center/default/index
/<business_app>/default/index
/<app>/default/index 或 /<app>/default/dashboard
```

公网部署后检查:

```bash
curl -sS -I https://<domain>/<app>/default/index
```

### 14.4 数据库检查

涉及数据库写入时，检查:

- 表是否创建在正确数据库。
- 认证表没有被新 app 重复创建。
- 业务表名前缀正确。
- 菜单 seed_key 不重复。
- 角色已创建且用户 membership 正确。

### 14.5 上传兼容检查

涉及上传逻辑时，必须检查是否误用上传对象布尔判断:

```bash
rg -n 'if not .*upload|if .*upload|bool\(.*upload|filename and fileobj|if not value' applications/<app> -g '*.py'
```

命中后需要人工区分普通列表/字符串和上传对象。对 `cgi.FieldStorage`、web2py upload 字段值、`request.vars.<upload_field>` 等上传对象，只能用 `is None`、`filename`、`file is None` 判断。

## 15. 部署与变更流程

### 15.1 常规流程

1. 读现有代码和文档，确认 app 边界。
2. 修改代码和文档。
3. 做语法检查。
4. 如果涉及数据库，先备份相关 sqlite。
5. 如修改共享模块、`models/` 初始化逻辑或 appconfig 影响启动期逻辑，需要重启运行实例；PaaS/容器平台通过重新部署或重启实例，传统部署通过重启 `<web2py-service>.service`。
6. 验证健康检查入口。
7. 验证公网入口。
8. 记录变更点和后续风险。

### 15.2 数据库变更前备份

SQLite 数据库修改前，至少备份:

```text
/data/<web2py_service>/service_center/databases/storage.sqlite
/data/<web2py_service>/<business_app>/databases/<business_app>.sqlite
/data/<web2py_service>/<app>/databases/<app>.sqlite
```

备份文件名建议:

```text
<db-name>.sqlite.bak-YYYYMMDD-HHMMSS
```

### 15.3 服务重启条件

以下修改通常需要重启运行实例。PaaS/容器平台重启实例或重新部署；传统宿主机环境重启 `<web2py-service>.service`:

- `modules/` 下共享模块。
- `models/` 初始化逻辑。
- 菜单源 `platform_nav.py`。
- appconfig 影响启动期逻辑。

纯视图/CSS/静态资源通常不需要重启，但仍需刷新浏览器缓存。

## 16. 新 app 接入检查清单

新 app 立项/上线前逐项确认:

- [ ] app 名称符合小写英文/下划线规范。
- [ ] 已创建 `/data/<web2py_service>/<app>` 运行目录。
- [ ] 数据库、上传、会话、错误、缓存目录已由配置文件指向 `/data/<web2py_service>/<app>`。
- [ ] 如目标部署到 PaaS 或容器平台，运行数据目录、数据库和上传文件已有持久化方案。
- [ ] 未把软链接作为新 app 的正式数据访问方案；如临时使用软链接，已有迁移和回滚说明。
- [ ] `models/00_db.py` 使用 `apply_service_center_auth(globals())`。
- [ ] 没有重复定义独立 `auth_user`。
- [ ] 如有业务库，使用 `db_<app>` 或清晰别名。
- [ ] 顶部导航接入 `platform_nav.py`。
- [ ] layout 渲染统一 `response.menu`。
- [ ] layout 渲染统一 `response.sub_menu`，且业务 app 不覆盖它。
- [ ] app 页内菜单使用 `response.page_menu` 或由 app 菜单表生成。
- [ ] 权限角色使用 `<app>_...` 前缀。
- [ ] `admin` 角色默认可管理。
- [ ] 未登录跳统一登录且 `_next` 正确。
- [ ] README 和 INTEGRATION 文档已写。
- [ ] 已完成语法检查和页面冒烟检查。
- [ ] 大概率跨 app 公用的基础功能已放入 `service_center/modules/`。
- [ ] app 内复用业务能力已放入 `<app>/modules/`，没有塞进控制器。

## 17. 新功能开发检查清单

新增功能上线前确认:

- [ ] 功能属于当前 app 边界，不应误放到别的 app。
- [ ] 已判断模块归属: 跨 app 公用放 `service_center/modules/`，app 内通用放 `<app>/modules/`。
- [ ] 数据表和字段命名带业务前缀。
- [ ] 页面入口已按架构归属加入顶部导航、平台侧栏或 app 页内菜单。
- [ ] 权限已集中定义，不在视图中散写复杂判断。
- [ ] 表单输入有校验。
- [ ] 文件上传/下载有权限和类型校验。
- [ ] 上传对象没有使用 `if upload`、`if not upload`、`bool(upload)` 或 `filename and fileobj` 这类 Python 3.10 不兼容写法。
- [ ] POST 成功后 redirect。
- [ ] 成功/失败提示清晰。
- [ ] 相关文档同步更新。
- [ ] 涉及数据库变更已备份并验证。

## 18. 当前技术债和整改建议

| 问题 | 当前状态 | 建议 |
| --- | --- | --- |
| 根 README 仍保留“示例项目”和旧路径 | 与当前 `/opt/<web2py_service>` 不一致 | 保留历史内容时必须在顶部加入当前规范入口；后续择机重写 |
| 某 app 数据目录仍在 app 内 | 业务库、上传目录仍是 app 内普通目录 | 安排维护窗口迁移到 `/data/<web2py_service>/<app>`，并通过配置文件读取真实路径 |
| 多数历史 app 依赖运行目录软链接 | 当前可运行，但部署、备份和排障边界不清晰 | 保留现有外置目标目录，后续改造时改成配置驱动访问 |
| 部分旧 app 可能有独立登录或旧 UI | 历史遗留 | 被改造时接入统一认证和顶部导航 |
| 顶部菜单同时存在代码默认值和部分 DB seed | 复杂 app 使用业务菜单表 | 菜单变更后同时检查代码和数据库 |
| 默认管理员账号历史存在 | 便于初始测试但生产风险高 | 改为系统后台创建/导入，并强制改密 |

## 19. 推荐参考文件

| 文件 | 用途 |
| --- | --- |
| `service_center/modules/service_center_auth.py` | 统一认证实现 |
| `service_center/modules/platform_nav.py` | 平台顶部导航代码默认来源 |
| `service_center/models/xx_menu.py` | 将顶部导航写入 `response.menu` |
| `service_center/models/menu_db.py` | 首页卡片和系统菜单管理 |
| `<business_app>/models/00_db.py` | 共享认证 + 独立业务库示例 |
| `<business_app>/models/10_menu.py` | 复杂 app 页内菜单表示例 |
| `<quality_app>/models/01_db.py` | 共享认证 + 质量检查业务库示例 |
| `<quality_app>/models/03_menu.py` | 统一顶部导航 + app 页内菜单示例 |
| `<quality_app>/models/04_permissions.py` | app 权限角色示例 |
| `<quality_app>/INTEGRATION.md` | 质量检查接入说明 |

## 20. 后续开发决策准则

遇到新需求时，先回答四个问题:

1. 这是平台能力还是业务能力？  
   平台能力放 `service_center`，业务能力放对应 app。

2. 这是哪个业务域的数据？  
   数据归属决定 app 边界，不由菜单位置决定。

3. 是否需要复用登录、权限、导航？  
   默认需要，且必须复用统一实现。

4. 是否会影响多个 app？  
   如果会，先定义接口/业务键/文档，再实现代码。

结论: 后续开发应以 `service_center` 为平台底座，以已接入统一平台的业务 app 作为样板，逐步把旧应用纳入同一套认证、导航、权限和数据分离规范。
