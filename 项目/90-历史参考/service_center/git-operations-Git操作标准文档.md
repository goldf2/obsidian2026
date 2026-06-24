# Git 操作标准文档

> **版本**: 1.0.0
> **创建日期**: 2026-01-23
> **状态**: 参考文档
> **适用范围**: service_center 应用版本控制

---

## 📋 目录

- [仓库信息](#仓库信息)
- [分支策略](#分支策略)
- [提交规范](#提交规范)
- [常用操作](#常用操作)
- [工作流程](#工作流程)
- [冲突解决](#冲突解决)
- [最佳实践](#最佳实践)

---

## 仓库信息

### 基本信息

| 项目 | 值 |
|------|-----|
| **仓库名称** | <repo> |
| **远程地址** | `git@gitee.com:<git-namespace>/<repo>.git` |
| **本地路径** | `/opt/<web2py_service>/applications/service_center` |
| **主分支** | `master` |
| **框架** | Web2py |

### 目录结构

```
service_center/
├── controllers/          # 控制器（版本控制）
├── models/               # 数据模型（版本控制）
├── views/                # 视图模板（版本控制）
├── static/               # 静态资源（版本控制）
├── modules/              # 自定义模块（版本控制）
├── private/              # 私有配置（部分版本控制）
│   ├── appconfig.ini         # ❌ 不提交（含敏感信息）
│   └── appconfig.ini.example # ✅ 提交（配置模板）
├── docs/                 # 文档（版本控制）
├── databases → 符号链接   # ❌ 不提交
├── sessions → 符号链接    # ❌ 不提交
├── errors → 符号链接      # ❌ 不提交
├── uploads → 符号链接     # ❌ 不提交
└── cache/                # ❌ 不提交
```

### 数据分离说明

本项目采用**代码与数据分离**架构：

```
代码目录（Git 管理）:
/opt/<web2py_service>/applications/service_center/

数据目录（独立存储）:
/data/<web2py_service>/service_center/
├── databases/    # SQLite 数据库
├── sessions/     # 会话文件
├── errors/       # 错误日志
└── uploads/      # 用户上传
```

通过符号链接连接：
```bash
databases → /data/<web2py_service>/service_center/databases
sessions  → /data/<web2py_service>/service_center/sessions
errors    → /data/<web2py_service>/service_center/errors
uploads   → /data/<web2py_service>/service_center/uploads
```

---

## 分支策略

### 分支类型

| 分支类型 | 命名规范 | 用途 | 生命周期 |
|---------|---------|------|---------|
| **master** | `master` | 主分支，生产代码 | 永久 |
| **develop** | `develop` | 开发分支（可选） | 永久 |
| **feature** | `feature/功能名` | 新功能开发 | 临时 |
| **bugfix** | `bugfix/问题描述` | Bug 修复 | 临时 |
| **hotfix** | `hotfix/紧急修复` | 生产环境紧急修复 | 临时 |
| **release** | `release/版本号` | 发布准备 | 临时 |

### 分支流程图

```
master ─────●─────────────●─────────────●───────→
            │             ↑             ↑
            │             │             │
develop ────●───●───●─────●───●───●─────●───────→
                │   ↑         │   ↑
                │   │         │   │
feature/xxx ────●───●         │   │
                              │   │
bugfix/yyy ───────────────────●───●
```

### 简化策略（小型项目）

对于小型项目或单人开发，可采用简化策略：

```
master ─────●─────●─────●─────●─────→
            │     ↑     │     ↑
            │     │     │     │
feature ────●─────●     │     │
                        │     │
bugfix ─────────────────●─────●
```

---

## 提交规范

### Commit Message 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type 类型

| 类型 | 说明 | 示例 |
|-----|------|------|
| `feat` | 新功能 | `feat(user): 添加用户注册功能` |
| `fix` | Bug 修复 | `fix(auth): 修复登录验证失败问题` |
| `docs` | 文档更新 | `docs: 更新部署文档` |
| `style` | 代码格式 | `style: 格式化代码缩进` |
| `refactor` | 重构 | `refactor(api): 重构接口响应格式` |
| `perf` | 性能优化 | `perf(query): 优化数据库查询` |
| `test` | 测试相关 | `test: 添加用户模块单元测试` |
| `chore` | 构建/工具 | `chore: 更新 .gitignore` |
| `revert` | 回滚 | `revert: 回滚 xxx 提交` |

### Scope 范围（可选）

| 范围 | 说明 |
|-----|------|
| `user` | 用户模块 |
| `auth` | 认证授权 |
| `api` | API 接口 |
| `db` | 数据库 |
| `config` | 配置相关 |
| `deploy` | 部署相关 |

### 提交示例

```bash
# 新功能
git commit -m "feat(user): 添加用户头像上传功能

- 支持 jpg/png 格式
- 限制文件大小 2MB
- 自动生成缩略图"

# Bug 修复
git commit -m "fix(auth): 修复 session 过期后无法重新登录的问题

问题原因：session 清理不完整
解决方案：登出时清除所有相关 cookie

Closes #123"

# 文档更新
git commit -m "docs: 添加 Git 操作标准文档"

# 简单修改
git commit -m "chore: 更新 .gitignore 忽略规则"
```

---

## 常用操作

### 初始化与克隆

```bash
# 克隆仓库
git clone git@gitee.com:<git-namespace>/<repo>.git

# 克隆到指定目录
git clone git@gitee.com:<git-namespace>/<repo>.git /path/to/directory

# 初始化新仓库
git init
git remote add origin git@gitee.com:<git-namespace>/<repo>.git
```

### 日常操作

```bash
# 查看状态
git status

# 查看变更
git diff                    # 工作区 vs 暂存区
git diff --staged           # 暂存区 vs 最新提交
git diff HEAD~1             # 最新提交 vs 上一次提交

# 添加文件
git add <file>              # 添加指定文件
git add .                   # 添加所有变更
git add -p                  # 交互式添加

# 提交
git commit -m "message"     # 提交
git commit -am "message"    # 添加并提交（仅已跟踪文件）
git commit --amend          # 修改最后一次提交

# 推送
git push                    # 推送到远程
git push -u origin master   # 首次推送并设置上游
git push --force            # 强制推送（谨慎使用）

# 拉取
git pull                    # 拉取并合并
git fetch                   # 仅拉取不合并
git pull --rebase           # 拉取并变基
```

### 分支操作

```bash
# 查看分支
git branch                  # 本地分支
git branch -r               # 远程分支
git branch -a               # 所有分支

# 创建分支
git branch <branch-name>              # 创建分支
git checkout -b <branch-name>         # 创建并切换
git checkout -b <branch> origin/<branch>  # 从远程创建

# 切换分支
git checkout <branch-name>
git switch <branch-name>    # Git 2.23+

# 合并分支
git merge <branch-name>     # 合并到当前分支
git merge --no-ff <branch>  # 非快进合并（保留分支历史）

# 删除分支
git branch -d <branch-name>           # 删除本地分支
git branch -D <branch-name>           # 强制删除
git push origin --delete <branch>     # 删除远程分支
```

### 撤销操作

```bash
# 撤销工作区修改
git checkout -- <file>      # 撤销指定文件
git checkout -- .           # 撤销所有修改
git restore <file>          # Git 2.23+

# 撤销暂存
git reset HEAD <file>       # 取消暂存
git restore --staged <file> # Git 2.23+

# 撤销提交
git reset --soft HEAD~1     # 撤销提交，保留修改在暂存区
git reset --mixed HEAD~1    # 撤销提交，保留修改在工作区
git reset --hard HEAD~1     # 撤销提交，丢弃所有修改

# 回滚到指定提交
git revert <commit-hash>    # 创建新提交来撤销
```

### 查看历史

```bash
# 查看提交历史
git log                     # 完整历史
git log --oneline           # 简洁模式
git log --graph             # 图形化显示
git log -n 10               # 最近 10 条
git log --author="name"     # 按作者筛选
git log --since="2026-01-01"  # 按日期筛选
git log -- <file>           # 指定文件历史

# 查看某次提交
git show <commit-hash>

# 查看文件变更历史
git blame <file>            # 逐行显示最后修改
```

### 标签操作

```bash
# 查看标签
git tag
git tag -l "v1.*"           # 筛选标签

# 创建标签
git tag v1.0.0                          # 轻量标签
git tag -a v1.0.0 -m "Release v1.0.0"   # 附注标签
git tag -a v1.0.0 <commit-hash>         # 给历史提交打标签

# 推送标签
git push origin v1.0.0      # 推送指定标签
git push origin --tags      # 推送所有标签

# 删除标签
git tag -d v1.0.0                       # 删除本地标签
git push origin --delete v1.0.0         # 删除远程标签
```

### 暂存操作

```bash
# 暂存当前修改
git stash                   # 暂存
git stash save "message"    # 带说明暂存
git stash -u                # 包含未跟踪文件

# 查看暂存列表
git stash list

# 恢复暂存
git stash pop               # 恢复并删除
git stash apply             # 恢复但保留
git stash apply stash@{0}   # 恢复指定暂存

# 删除暂存
git stash drop stash@{0}    # 删除指定
git stash clear             # 清空所有
```

---

## 工作流程

### 日常开发流程

```bash
# 1. 开始工作前，同步最新代码
git pull origin master

# 2. 创建功能分支（可选）
git checkout -b feature/new-feature

# 3. 开发并提交
git add .
git commit -m "feat: 实现新功能"

# 4. 推送到远程
git push origin feature/new-feature

# 5. 合并到主分支
git checkout master
git merge feature/new-feature

# 6. 推送主分支
git push origin master

# 7. 删除功能分支
git branch -d feature/new-feature
git push origin --delete feature/new-feature
```

### 紧急修复流程

```bash
# 1. 从 master 创建 hotfix 分支
git checkout master
git pull origin master
git checkout -b hotfix/critical-bug

# 2. 修复并提交
git add .
git commit -m "fix: 修复紧急问题"

# 3. 合并到 master
git checkout master
git merge hotfix/critical-bug

# 4. 打标签（可选）
git tag -a v1.0.1 -m "Hotfix release"

# 5. 推送
git push origin master --tags

# 6. 清理
git branch -d hotfix/critical-bug
```

### 代码审查流程（Pull Request）

```bash
# 1. 创建功能分支
git checkout -b feature/my-feature

# 2. 开发并提交
git add .
git commit -m "feat: 新功能"

# 3. 推送到远程
git push -u origin feature/my-feature

# 4. 在 Gitee 上创建 Pull Request

# 5. 代码审查通过后合并

# 6. 本地同步
git checkout master
git pull origin master

# 7. 清理本地分支
git branch -d feature/my-feature
```

---

## 冲突解决

### 冲突产生场景

1. **合并冲突**：两个分支修改了同一文件的同一位置
2. **拉取冲突**：本地和远程都有新提交
3. **变基冲突**：变基过程中的冲突

### 解决步骤

```bash
# 1. 查看冲突文件
git status

# 2. 打开冲突文件，手动解决
# 冲突标记：
# <<<<<<< HEAD
# 当前分支的内容
# =======
# 合并分支的内容
# >>>>>>> branch-name

# 3. 编辑文件，删除冲突标记，保留正确内容

# 4. 标记为已解决
git add <conflicted-file>

# 5. 继续操作
git commit                  # 合并冲突
git rebase --continue       # 变基冲突
```

### 冲突预防

1. **频繁同步**：经常拉取最新代码
2. **小步提交**：避免大范围修改
3. **沟通协调**：避免多人同时修改同一文件
4. **分支隔离**：不同功能使用不同分支

---

## 最佳实践

### 提交规范

| ✅ 推荐 | ❌ 避免 |
|--------|--------|
| 小步提交，每次只做一件事 | 大量修改一次提交 |
| 有意义的提交信息 | "fix"、"update" 等模糊信息 |
| 提交前检查 `git diff` | 盲目 `git add .` |
| 使用 `.gitignore` 排除无关文件 | 提交临时文件、日志等 |

### 分支管理

| ✅ 推荐 | ❌ 避免 |
|--------|--------|
| 功能开发使用独立分支 | 直接在 master 上开发 |
| 及时删除已合并分支 | 保留大量废弃分支 |
| 分支命名有意义 | 使用 test1、test2 等命名 |
| 定期同步主分支 | 长期不同步导致大量冲突 |

### 安全规范

| ✅ 推荐 | ❌ 避免 |
|--------|--------|
| 敏感信息放配置文件 | 密码硬编码在代码中 |
| 使用 `.gitignore` 排除敏感文件 | 提交 `appconfig.ini` 等 |
| 使用 SSH 密钥认证 | 使用 HTTPS + 密码 |
| 定期检查提交历史 | 忽略安全审计 |

### 本项目特殊注意

1. **符号链接处理**
   ```bash
   # 符号链接已在 .gitignore 中排除
   # 不要尝试提交 databases、sessions、errors、uploads
   ```

2. **配置文件处理**
   ```bash
   # 只提交示例配置
   git add private/appconfig.ini.example

   # 不要提交实际配置
   # private/appconfig.ini 已在 .gitignore 中
   ```

3. **数据目录**
   ```bash
   # 数据存储在 /data/<web2py_service>/service_center/
   # 不属于 Git 管理范围
   # 需要单独备份
   ```

---

## 常见问题

### Q1: 如何撤销已推送的提交？

```bash
# 方法 1: revert（推荐，保留历史）
git revert <commit-hash>
git push

# 方法 2: reset + force push（谨慎使用）
git reset --hard HEAD~1
git push --force
```

### Q2: 如何修改已推送的提交信息？

```bash
# 修改最后一次提交
git commit --amend -m "新的提交信息"
git push --force

# 注意：会改变提交历史，团队协作时需谨慎
```

### Q3: 如何找回删除的分支？

```bash
# 查看引用日志
git reflog

# 找到分支最后的提交
git checkout -b <branch-name> <commit-hash>
```

### Q4: 如何清理大文件历史？

```bash
# 使用 git filter-branch（谨慎操作）
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch <file-path>' \
  --prune-empty --tag-name-filter cat -- --all

# 或使用 BFG Repo-Cleaner（推荐）
java -jar bfg.jar --delete-files <filename>
```

### Q5: 如何查看某个文件的修改历史？

```bash
# 查看文件的提交历史
git log --follow -- <file-path>

# 查看文件每行的最后修改
git blame <file-path>

# 查看文件在某次提交时的内容
git show <commit-hash>:<file-path>
```

---

## 附录

### Git 配置

```bash
# 查看配置
git config --list

# 设置用户信息
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 设置默认编辑器
git config --global core.editor vim

# 设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"

# 设置换行符处理
git config --global core.autocrlf input  # Linux/Mac
git config --global core.autocrlf true   # Windows
```

### SSH 密钥配置

```bash
# 生成 SSH 密钥
ssh-keygen -t rsa -b 4096 -C "your@email.com"

# 查看公钥
cat ~/.ssh/id_rsa.pub

# 测试连接
ssh -T git@gitee.com
```

### 有用的命令别名

```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git pull'
alias gd='git diff'
alias gb='git branch'
alias gco='git checkout'
alias glog='git log --oneline --graph --all'
```

---

## 参考资料

- [Git 官方文档](https://git-scm.com/doc)
- [Pro Git 中文版](https://git-scm.com/book/zh/v2)
- [Gitee 帮助文档](https://gitee.com/help)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Git Flow 工作流](https://nvie.com/posts/a-successful-git-branching-model/)

---

*文档创建: 2026-01-23*
*作者: Claude AI*
