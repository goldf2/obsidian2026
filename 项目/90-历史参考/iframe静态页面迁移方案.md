# Iframe 迁移方案

## 目标
使用 iframe 嵌入完整 HTML 页面，避免被 web2py 的 layout 模板影响原始布局。

## 范围
- 纯静态、独立的 HTML 页面（不包含 web2py 模板标签）。
- 动态页面（数据库、模板循环、权限控制等）仍保留为普通视图。

## 文件结构
- 原始 HTML：static/html/<name>.html
- 视图包装页：views/default/<name>.html

## 迁移策略
1. 识别候选视图：
   - 包含完整 HTML 文档（"<!DOCTYPE html>"）。
   - 除首行 "{{extend 'layout.html'}}" 外不包含任何 "{{...}}" 模板标签。
2. 将原始 HTML 移到 static/html/，文件名保持一致。
3. 用最小 iframe 包装页替换原视图，并继续 extend layout：
   - 只渲染一个 iframe，指向 static/html/<name>.html。
4. 在 views/layout.html 中加入统一的 iframe 尺寸样式。
5. 修复静态 HTML 内部的相对链接，改为控制器路由或应用内绝对路径。
6. 确保 CSS/JS/图片链接为正确路径（"/app/static/..." 或静态目录可解析的相对路径）。

## 权限与安全注意事项
- 静态文件绕过控制器鉴权。如果页面需要登录权限，应通过 iframe 包装页访问，而不是直接暴露 static/html/。
- 若启用 CSP，需允许 frame-src 'self'。

## 验收清单
- 页面渲染效果与原始 HTML 一致。
- 内层页面不受 layout CSS 干扰。
- 页面内链接可正常跳转。
- 静态资源加载无 404。
- 需要权限的页面仍被保护。

## 回滚方案
- 恢复原视图内容，删除 static/html 中的对应文件。
- 如无 iframe 页面需求，可移除 views/layout.html 中的 iframe 样式。

## 本应用已迁移页面
视图与静态 HTML 同名：
- ERP_Matrix
- ERP_YS
- Gantt_chart
- PGT_CHK_1
- PGT_COST_1
- PGT_QUO_1
- SC2E_PGT
- ZCY_PGT_1
- board
- pfb20250620
- table_11
- table_11_B
- table_13
- table_14
- table_16
- table_17
- table_18
- table_19
- table_20
- table_21
- table_22
- table_23
- table_24
- table_25
- table_26
- table_27
- table_28
- table_29
- table_30
- table_31
- table_32

## 已修复的静态链接
- PGT_CHK_1（链接更新为 default/PGT_QUO_1、default/ZCY_PGT_1、default/PGT_COST_1）
- table_13（链接更新为 default/PGT_COST_1）

## 不在本次范围（示例）
- views/default/index2.html（动态分类循环）
- views/default/example.html（动态分类循环）
- views/default/BOM.html（数据库驱动表格）
