# S5 Docker 化规范

## 1. 阶段定位

S5 建立统一的容器化交付边界，是走向云服务商 PaaS、Coolify 或自建 Compose 的共同前提。

## 2. 阶段目标

- 形成可重建镜像。
- 通过环境变量和挂载连接外部数据。
- 建立 healthcheck 和基础运行约定。

## 3. 必做事项

- 设计 Dockerfile。
- 设计 Compose 样例。
- 明确 env、volume、healthcheck。
- 明确镜像内外职责边界。

## 4. 交付物

- Dockerfile 模板。
- docker-compose 样例。
- 环境变量清单。
- 健康检查样例。

## 5. 验收标准

- 镜像只包含代码和依赖。
- 容器重建后数据可恢复。
- 同一交付边界可用于 PaaS 和 Compose。

## 6. 关联文档

- `数据存储与PaaS部署治理路线图.md`
- `持续集成与持续部署标准流程方案.md`
