# Git 更新规则

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **状态**: 强制执行
> **适用范围**: service_center 应用版本控制

---

## 📋 目录

- [核心规则](#核心规则)
- [版本号规范](#版本号规范)
- [CHANGELOG 规范](#changelog-规范)
- [提交流程](#提交流程)
- [Git Hooks 自动检查](#git-hooks-自动检查)
- [常见场景示例](#常见场景示例)

---

## 核心规则

### 🔴 强制规则

每次向远程仓库推送代码时，**必须**同时完成以下操作：

| 规则 | 说明 | 强制性 |
|------|------|--------|
| **更新版本号** | 在 `VERSION` 文件中更新版本号 | ✅ 必须 |
| **更新 CHANGELOG** | 在 `CHANGELOG.md` 中记录变更 | ✅ 必须 |
| **创建 Git Tag** | 为新版本创建标签 | ⚠️ 发布版本时必须 |

### 规则目的

1. **可追溯性**: 每个版本都有明确的变更记录
2. **团队协作**: 团队成员能快速了解项目进展
3. **回滚支持**: 出问题时能快速定位和回滚到特定版本
4. **发布管理**: 便于管理正式发布版本

---

## 版本号规范

### 语义化版本 (Semantic Versioning)

版本号格式: `MAJOR.MINOR.PATCH`

```
示例: 1.2.3
      │ │ │
      │ │ └── PATCH: 向后兼容的问题修复
      │ └──── MINOR: 向后兼容的功能新增
      └────── MAJOR: 不兼容的 API 变更
```

### 版本号递增规则

| 变更类型 | 版本号变化 | 示例 |
|---------|-----------|------|
| Bug 修复 | PATCH +1 | 1.0.0 → 1.0.1 |
| 新功能（向后兼容） | MINOR +1, PATCH 归零 | 1.0.1 → 1.1.0 |
| 重大变更（不兼容） | MAJOR +1, MINOR/PATCH 归零 | 1.1.0 → 2.0.0 |
| 文档更新 | PATCH +1 | 1.0.0 → 1.0.1 |
| 配置变更 | PATCH +1 | 1.0.0 → 1.0.1 |

### VERSION 文件

项目根目录必须包含 `VERSION` 文件：

```bash
# VERSION 文件内容（仅包含版本号）
1.0.0
```

### 更新版本号命令

```bash
# 查看当前版本
cat VERSION

# 更新版本号（手动编辑）
echo "1.0.1" > VERSION

# 或使用编辑器
vim VERSION
```

---

## CHANGELOG 规范

### 文件位置

`CHANGELOG.md` 位于项目根目录。

### 格式标准

基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/) 规范：

```markdown
## [版本号] - 日期

### 新增 (Added)
- 新功能描述

### 变更 (Changed)
- 功能变更描述

### 废弃 (Deprecated)
- 即将废弃的功能

### 移除 (Removed)
- 已移除的功能

### 修复 (Fixed)
- Bug 修复描述

### 安全 (Security)
- 安全相关修复
```

### 变更类型说明

| 类型 | 英文 | 使用场景 |
|------|------|---------|
| 新增 | Added | 新功能、新文件、新配置 |
| 变更 | Changed | 现有功能的修改 |
| 废弃 | Deprecated | 即将在未来版本移除的功能 |
| 移除 | Removed | 本版本移除的功能 |
| 修复 | Fixed | Bug 修复 |
| 安全 | Security | 安全漏洞修复 |
| 文档 | Documentation | 文档更新（自定义类型） |
| 基础设施 | Infrastructure | 构建、部署相关（自定义类型） |

### CHANGELOG 示例

```markdown
## [1.0.1] - 2026-01-24

### 修复 (Fixed)
- 修复用户登录时的会话过期问题
- 修复导出功能的编码错误

### 文档 (Documentation)
- 更新 API 文档

---

## [1.0.0] - 2026-01-23

### 新增 (Added)
- 初始化项目
- 实现用户管理模块
```

---

## 提交流程

### 标准提交流程（每次推送必须执行）

```bash
# 1. 确认当前状态
git status

# 2. 更新版本号
echo "1.0.1" > VERSION

# 3. 更新 CHANGELOG.md
# 在文件顶部添加新版本的变更记录

# 4. 添加所有变更到暂存区
git add .

# 5. 提交（包含版本号）
git commit -m "release: v1.0.1 - 简要描述变更内容"

# 6. 创建标签（发布版本时）
git tag -a v1.0.1 -m "Release v1.0.1: 简要描述"

# 7. 推送代码和标签
git push origin master
git push origin v1.0.1
# 或一次性推送所有标签
git push origin master --tags
```

### 提交信息规范

```
<type>: v<version> - <description>

类型 (type):
- release: 正式发布版本
- feat: 新功能
- fix: Bug 修复
- docs: 文档更新
- refactor: 代码重构
- chore: 构建/工具变更
```

**示例**：
```bash
git commit -m "release: v1.0.1 - 修复登录问题和更新文档"
git commit -m "feat: v1.1.0 - 添加数据导出功能"
git commit -m "fix: v1.0.2 - 修复编码问题"
```

---

## Git Hooks 自动检查

### 安装 pre-push Hook

创建 `.git/hooks/pre-push` 文件，在推送前自动检查版本号和 CHANGELOG：

```bash
#!/bin/bash
# .git/hooks/pre-push
# 推送前检查版本号和 CHANGELOG 是否已更新

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}[Git Hook] 检查版本更新规则...${NC}"

# 检查 VERSION 文件是否存在
if [ ! -f "VERSION" ]; then
    echo -e "${RED}错误: VERSION 文件不存在${NC}"
    echo "请创建 VERSION 文件并填写版本号"
    exit 1
fi

# 检查 CHANGELOG.md 是否存在
if [ ! -f "CHANGELOG.md" ]; then
    echo -e "${RED}错误: CHANGELOG.md 文件不存在${NC}"
    echo "请创建 CHANGELOG.md 文件并记录变更"
    exit 1
fi

# 获取当前版本号
CURRENT_VERSION=$(cat VERSION)

# 检查版本号格式
if ! [[ $CURRENT_VERSION =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo -e "${RED}错误: 版本号格式不正确${NC}"
    echo "当前版本: $CURRENT_VERSION"
    echo "正确格式: X.Y.Z (例如: 1.0.0)"
    exit 1
fi

# 检查 CHANGELOG 是否包含当前版本
if ! grep -q "\[$CURRENT_VERSION\]" CHANGELOG.md; then
    echo -e "${RED}错误: CHANGELOG.md 中未找到版本 $CURRENT_VERSION 的记录${NC}"
    echo "请在 CHANGELOG.md 中添加版本 $CURRENT_VERSION 的变更记录"
    exit 1
fi

# 检查是否有对应的 Git 标签（警告，不阻止推送）
if ! git tag | grep -q "v$CURRENT_VERSION"; then
    echo -e "${YELLOW}警告: 未找到标签 v$CURRENT_VERSION${NC}"
    echo "建议执行: git tag -a v$CURRENT_VERSION -m 'Release v$CURRENT_VERSION'"
fi

echo -e "${GREEN}[Git Hook] 检查通过！${NC}"
echo "版本号: $CURRENT_VERSION"
exit 0
```

### 安装 Hook

```bash
# 创建 hook 文件
cat > .git/hooks/pre-push << 'EOF'
#!/bin/bash
# 上面的脚本内容
EOF

# 添加执行权限
chmod +x .git/hooks/pre-push
```

### 跳过 Hook（紧急情况）

```bash
# 仅在紧急情况下使用，需要说明原因
git push --no-verify
```

---

## 常见场景示例

### 场景 1: Bug 修复

```bash
# 1. 修复代码
vim controllers/default.py

# 2. 更新版本号 (PATCH +1)
echo "1.0.1" > VERSION

# 3. 更新 CHANGELOG
cat >> CHANGELOG.md << 'EOF'

## [1.0.1] - 2026-01-24

### 修复 (Fixed)
- 修复用户登录验证失败的问题
EOF

# 4. 提交并推送
git add .
git commit -m "fix: v1.0.1 - 修复登录验证问题"
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin master --tags
```

### 场景 2: 新功能开发

```bash
# 1. 开发新功能
vim controllers/export.py
vim views/export/index.html

# 2. 更新版本号 (MINOR +1)
echo "1.1.0" > VERSION

# 3. 更新 CHANGELOG
# 编辑 CHANGELOG.md，在顶部添加：
## [1.1.0] - 2026-01-25

### 新增 (Added)
- 添加数据导出功能
- 支持 CSV 和 Excel 格式导出

# 4. 提交并推送
git add .
git commit -m "feat: v1.1.0 - 添加数据导出功能"
git tag -a v1.1.0 -m "Release v1.1.0: 数据导出功能"
git push origin master --tags
```

### 场景 3: 文档更新

```bash
# 1. 更新文档
vim docs/api-guide.md

# 2. 更新版本号 (PATCH +1)
echo "1.1.1" > VERSION

# 3. 更新 CHANGELOG
## [1.1.1] - 2026-01-26

### 文档 (Documentation)
- 更新 API 使用指南
- 添加常见问题解答

# 4. 提交并推送
git add .
git commit -m "docs: v1.1.1 - 更新 API 文档"
git push origin master
# 文档更新可以不创建标签
```

### 场景 4: 重大版本升级

```bash
# 1. 进行重大变更
# ... 代码修改 ...

# 2. 更新版本号 (MAJOR +1)
echo "2.0.0" > VERSION

# 3. 更新 CHANGELOG
## [2.0.0] - 2026-02-01

### ⚠️ 重大变更 (Breaking Changes)
- API 接口路径变更
- 数据库结构调整，需要执行迁移脚本

### 新增 (Added)
- 全新的用户界面
- 支持多语言

### 移除 (Removed)
- 移除旧版 API（v1 接口）

### 迁移指南
1. 备份数据库
2. 执行迁移脚本: `python migrate.py`
3. 更新配置文件

# 4. 提交并推送
git add .
git commit -m "release: v2.0.0 - 重大版本升级"
git tag -a v2.0.0 -m "Release v2.0.0: 重大版本升级"
git push origin master --tags
```

---

## 检查清单

### 推送前检查

- [ ] VERSION 文件已更新
- [ ] CHANGELOG.md 已更新，包含当前版本记录
- [ ] 版本号格式正确 (X.Y.Z)
- [ ] CHANGELOG 中的版本号与 VERSION 文件一致
- [ ] 提交信息包含版本号
- [ ] （发布版本）已创建 Git 标签

### 版本号选择

- [ ] Bug 修复 → PATCH +1
- [ ] 新功能 → MINOR +1
- [ ] 重大变更 → MAJOR +1

---

## 附录

### 快速命令参考

```bash
# 查看当前版本
cat VERSION

# 查看所有标签
git tag

# 查看最新标签
git describe --tags --abbrev=0

# 查看版本历史
git log --oneline --decorate

# 比较两个版本
git diff v1.0.0..v1.1.0

# 查看某个版本的变更
git show v1.0.0
```

### 相关文档

- [Git 操作标准文档](./git-operations-Git操作标准文档.md)
- [语义化版本规范](https://semver.org/lang/zh-CN/)
- [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)

---

*文档创建: 2026-01-23*
*作者: Claude AI*
