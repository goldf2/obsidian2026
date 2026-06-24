# 项目文档索引

本目录用于保存每个 web2py 服务项目的项目级文档。通用规范、模板和开发模式保留在仓库根目录分类中；项目目录只记录具体项目自己的事实、差异、应用清单和规范执行状态。

## 当前项目

| 项目 | 当前部署模式 | 代码目录/仓库 | 运行服务 / PaaS 应用 | 项目文档 |
| --- | --- | --- | --- | --- |
| `web2py_heqing` | 传统 systemd，待治理 | `/opt/web2py_heqing` | `web2py-heqing.service` | `projects/web2py_heqing/` |
| `web2py_cxmj` | 待确认 | 待确认 | 待确认 | `projects/web2py_cxmj/` |
| `web2py_my` | 传统 systemd，待治理 | `/opt/web2py_my` | `web2py-my.service` | `projects/web2py_my/` |
| `web2py_oak` | 传统 systemd，待治理 | `/opt/web2py_oak` | `web2py-oak.service` | `projects/web2py_oak/` |
| `web2py_opt` | 传统 systemd，待治理 | `/opt/web2py_opt` | `web2py-opt.service` | `projects/web2py_opt/` |
| `web2py_yf` | 传统 systemd，待治理 | `/opt/web2py_yf` | `web2py-yf.service` | `projects/web2py_yf/` |

## 维护规则

- 每个实际 web2py 服务项目必须有一个同名项目文件夹。
- 项目文件夹至少包含 `README.md`、`应用清单.md`、`运行数据与配置.md`、`规范对照.md`。
- 项目文档只记录项目事实和项目差异，不复制通用规范正文。
- 新项目创建后，先复制 `80-模板/项目文档模板.md` 的结构，再补充项目实际信息。
