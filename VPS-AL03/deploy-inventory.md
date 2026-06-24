# 服务器部署清单

生成时间：2026-06-09 22:26:54

| 域名 | 协议 | 后端 | 服务 | 状态 | 项目目录 | 证书剩余 | 健康检查 |
|---|---|---|---|---|---|---|---|
| admin.xiangshu.me | HTTPS | `http://127.0.0.1:8015` | `web2py-opt.service` | active | `/opt/web2py_opt` | 89 | 401 |
| admin.xiangshu.me | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 401 |
| _ | HTTP | `/var/www/html` | `` |  | `` |  |  |
| heqing.art, www.heqing.art | HTTP | `301 https://heqing.art$request_uri` | `` |  | `` |  | 200 OK |
| www.heqing.art | HTTPS | `301 https://heqing.art$request_uri` | `` |  | `` | 56 | 200 OK |
| heqing.art | HTTPS | `http://127.0.0.1:8011` | `heqing-backend.service` | active | `/var/www/heqing/current/prototype-lab` | 56 | 200 OK |
| image.heqing.art | HTTPS | `http://127.0.0.1:8013` | `web2py-my.service` | active | `/opt/web2py_my` | 88 | 200 OK |
| image.heqing.art | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 200 OK |
| my.xiangshu.me | HTTPS | `http://127.0.0.1:8013` | `web2py-my.service` | active | `/opt/web2py_my` | 59 | 200 OK |
| my.xiangshu.me | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 200 OK |
| oaktechz.com, www.oaktechz.com | HTTPS | `http://127.0.0.1:8012` | `web2py-oak.service` | active | `/opt/web2py_oak` | 59 | 500 |
| oaktechz.com, www.oaktechz.com | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 500 |
| odoo19.xiangshu.me | HTTPS | `http://127.0.0.1:8068` | `odoo19.service` | active | `/opt/odoo19` | 52 | 200 OK |
| odoo19.xiangshu.me | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 200 OK |
| xiangshu.me, www.xiangshu.me, szxiangshu.com, www.szxiangshu.com | HTTP | `http://127.0.0.1:8012` | `web2py-oak.service` | active | `/opt/web2py_oak` |  | 500 |
| yf.xiangshu.me | HTTPS | `http://127.0.0.1:8014` | `web2py-yf.service` | active | `/opt/web2py_yf` | 59 | 200 OK |
| yf.xiangshu.me | HTTP | `301 https://$host$request_uri` | `` |  | `` |  | 200 OK |
