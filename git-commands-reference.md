# Git 常用命令速查手册

> 针对阿里云服务器 + Gitee/GitHub 双平台推送场景整理

---

## 1. 环境配置（一次性）

```bash
# 设置用户名和邮箱（会显示在提交记录中）
git config --global user.name  "goldf2"
git config --global user.email "your-email@example.com"

# 默认分支名统一为 main（与 GitHub/Gitee 默认一致）
git config --global init.defaultBranch main

# 简化推送行为：只推当前分支，避免误推所有分支
git config --global push.default simple

# 设置默认编辑器（如提交信息需要修改时）
git config --global core.editor "vim"

# 查看当前所有配置
git config --global --list
```

---

## 2. 仓库初始化与克隆

```bash
# 初始化本地仓库
git init

# 克隆远程仓库（自动添加 origin）
git clone git@gitee.com:goldf2/web2py-service-spec.git

# 克隆指定分支
git clone -b main git@gitee.com:goldf2/web2py-service-spec.git
```

---

## 3. 日常提交流程（最高频）

```bash
# 1. 查看状态 — 每次提交前必做
git status

# 2. 添加文件到暂存区
git add filename.py          # 添加单个文件
git add .                    # 添加所有修改（常用）
git add -p                   # 分块添加，适合精细控制

# 3. 查看修改内容（确认无误再提交）
git diff                     # 工作区 vs 暂存区
git diff --cached            # 暂存区 vs 最新提交

# 4. 提交
git commit -m "feat: 添加巡检异常闭环通知功能"

# 5. 修改最后一次提交（提交后发现漏了文件或描述写错）
git commit --amend

# 6. 推送（已设置 upstream 后可直接 git push）
git push
```

---

## 4. 多远程仓库配置（本场景核心）

### 当前推荐配置：Gitee 为主（fetch），GitHub + Gitee 双推送

```bash
# 1. 设置 Gitee 为 fetch 和默认 push
git remote add origin git@gitee.com:goldf2/web2py-service-spec.git

# 2. 追加 GitHub 为推送地址（不影响 fetch）
git remote set-url --add --push origin git@github.com:goldf2/web2py-service-spec.git

# 3. 再追加 Gitee 推送地址（确保 push 时也同步到 Gitee）
git remote set-url --add --push origin git@gitee.com:goldf2/web2py-service-spec.git
```

### 验证配置

```bash
git remote -v
```

**正确输出应为：**
```
origin	git@gitee.com:goldf2/web2py-service-spec.git (fetch)
origin	git@github.com:goldf2/web2py-service-spec.git (push)
origin	git@gitee.com:goldf2/web2py-service-spec.git (push)
```

> ⚠️ 注意：必须显示两个 `(push)` 地址，否则 Gitee 不会收到推送。如果只有一个 push 地址，请按下方"修复"步骤操作。

### 常用远程操作

```bash
git remote -v                          # 查看所有 remote 配置
git remote rm origin                   # 删除 remote（本地配置，不影响远程仓库）
git remote set-url origin <url>        # 修改 fetch 地址
git remote set-url --add --push origin <url>   # 追加 push 地址
git remote set-url --delete --push origin <url> # 删除某个 push 地址
git remote prune origin                # 清理已删除的远程分支引用
```

### 默认 URL 与显式 push URL 的区别（关键理解）

Git 的 remote URL 有两种机制：

| 类型 | 说明 |
|------|------|
| **默认 URL** | `git remote add origin <url>` 创建的地址，同时用于 `fetch` 和 `push` |
| **显式 push URL** | `git remote set-url --add --push origin <url>` 创建的地址，**仅用于 push** |

**关键陷阱**：一旦你给某个 remote 添加了**任何一个**显式 push URL，Git 就会**不再使用默认 URL 进行 push**。这意味着原来默认 URL 的 push 能力被"静默降级"为仅用于 fetch。

**示例**：

```bash
git remote add origin git@gitee.com:goldf2/gitlearn.git     # 默认 URL（fetch + push）
git remote set-url --add --push origin git@github.com:...   # 添加显式 push URL
```

此时 `git remote -v` 显示：
```
origin	git@gitee.com:goldf2/gitlearn.git (fetch)
origin	git@github.com:goldf2/gitlearn.git (push)
```

**Gitee 的 push 地址消失了！** 不是被覆盖，而是被降级为 fetch-only。如果需要双平台推送，必须**显式把 Gitee 也加回 push URL 列表**。

### 修复：恢复双平台推送

如果 `git remote -v` 只有一个 push 地址，执行以下步骤：

```bash
# 1. 彻底删除 origin（清除所有 URL 配置）
git remote rm origin

# 2. 重新设置 Gitee 为默认 URL（fetch + push）
git remote add origin git@gitee.com:goldf2/gitlearn.git

# 3. 追加 GitHub 为显式 push 地址
git remote set-url --add --push origin git@github.com:goldf2/gitlearn.git

# 4. 再次追加 Gitee 为显式 push 地址（关键！）
git remote set-url --add --push origin git@gitee.com:goldf2/gitlearn.git

# 5. 验证
git remote -v
```

**预期输出**：
```
origin	git@gitee.com:goldf2/gitlearn.git (fetch)
origin	git@github.com:goldf2/gitlearn.git (push)
origin	git@gitee.com:goldf2/gitlearn.git (push)
```

### 删除某个 push 地址

```bash
# 删除 GitHub 的 push 地址（保留 Gitee 的 push）
git remote set-url --delete --push origin git@github.com:goldf2/gitlearn.git

# 删除后如果没有任何显式 push URL，Git 会自动回退使用默认 URL（Gitee）来 push
git remote -v
```

> 注意：`--delete` 后面必须跟**完整的 URL**，不能省略。

### 命令区分速查表

| 命令 | 作用 | 是否保留现有 push URL |
|------|------|----------------------|
| `git remote add origin <url>` | 设置默认 URL（fetch + push） | 初始化 |
| `git remote set-url --add --push origin <url>` | **追加**一个显式 push URL | ✅ 保留 |
| `git remote set-url --push origin <url>` | **替换**所有 push URL 为这一个 | ❌ 清空旧的 |
| `git remote set-url --delete --push origin <url>` | 删除指定的显式 push URL | ✅ 保留其他的 |
| `git remote set-url --add origin <url>` | 添加**默认 URL**（不用于 push） | ❌ 对 push 无影响 |

```bash
# 首次推送本地分支并设置上游（之后可直接 git push）
git push -u origin main

# 查看本地分支跟踪的上游
git branch -vv

# 修改当前分支的上游为其他 remote
# git branch --set-upstream-to=gitee/main main
```

---

## 5. 拉取与同步

```bash
# 从 Gitee 拉取最新代码并合并（日常更新）
git pull

# 仅下载远程更新，不合并（适合先看看情况）
git fetch

# 下载所有 remote 的更新
# git fetch --all

# 拉取指定分支
# git pull origin main
```

> 提示：`git pull` 在阿里云服务器上优先从 Gitee 拉取，速度更快更稳定。

---

## 6. 分支操作

```bash
# 查看分支
git branch                # 本地分支
git branch -a             # 所有分支（含远程）
git branch -r             # 仅远程分支

# 创建与切换
git switch 分支名          # 切换分支（推荐）
git switch -c 新分支名      # 创建并切换
git checkout -b 新分支名    # 旧版写法，同上

# 合并分支
git merge 分支名            # 合并指定分支到当前分支

# 删除分支
git branch -d 分支名       # 删除已合并分支
git branch -D 分支名       # 强制删除未合并分支

# 推送本地分支到远程
# git push origin 分支名
# git push -u origin 分支名
```

---

## 7. 撤销与回退（谨慎操作）

```bash
# 取消暂存（保留工作区修改）
git restore --staged 文件名

# 丢弃工作区修改（未暂存的内容会丢失，慎用）
git restore 文件名

# 撤销最后一次提交，但保留修改到暂存区
git reset --soft HEAD~1

# 撤销最后一次提交，保留修改到工作区（未暂存）
git reset --mixed HEAD~1

# ⚠️ 彻底回退到某次提交，之后的内容全部删除（危险）
git reset --hard HEAD~1

# 更安全的方式：反向提交，不删历史
git revert <commit-id>

# 查看操作历史（可找回 reset --hard 丢失的提交）
git reflog
```

---

## 8. 查看历史与排查

```bash
# 图形化查看历史（推荐，最常用）
git log --oneline --graph --all -20

# 查看文件逐行修改记录（追查 bug 时很有用）
git blame 文件名

# 查看某次提交详情
# git show <commit-id>

# 查看工作区与某次提交的差异
# git diff <commit-id>

# 查看分支合并图
# git log --graph --decorate --all
```

---

## 9. 临时保存（切换分支时未提交的修改）

```bash
# 临时保存当前未提交的修改（干净地切去其他分支）
git stash

# 保存并加备注
git stash push -m "WIP: 巡检报表优化中"

# 查看 stash 列表
git stash list

# 恢复最近一次 stash 并删除
git stash pop

# 恢复但不删除
git stash apply

# 清空所有 stash
git stash clear
```

---

## 10. 标签管理

```bash
# 查看标签
git tag

# 打注释标签
git tag -a v1.0 -m "发布巡检系统 v1.0"

# 推送标签到远程
# git push origin --tags
```

---

## 11. 实用别名（减少重复输入）

```bash
# 设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all -20"
git config --global alias.last "log -1 HEAD"

# 使用别名
# git st       → git status
# git lg       → 图形化历史
# git last     → 查看最后一次提交
```

---

## 12. 针对本项目的日常操作速查

基于当前配置（阿里云服务器 + Gitee fetch + GitHub/Gitee 双 push）：

```bash
# 更新代码（从 Gitee 拉取，国内速度快）
git pull

# 提交修改
git add .
git commit -m "type: 描述"

# 推送到 GitHub + Gitee（同时）
git push

# 查看当前仓库状态
git status

# 查看修改历史
git log --oneline -10
```

---

## 13. 注意事项

1. **fetch 只能有一个地址**：Git 设计如此，本地 `origin/main` 只能指向一个来源。你的主 fetch 已设为 Gitee。
2. **push 可以多个地址**：当前配置会依次推送到 GitHub 和 Gitee。如果其中一个失败，另一个不受影响，Git 会在最后汇总报错。
3. **commit 描述规范**：建议采用 `type: description` 格式，如 `feat: 添加新功能`、`fix: 修复巡检异常`、`docs: 更新文档`。
4. **定期 `git pull`**：在服务器上开发前，先 pull 确保代码最新，避免 push 时冲突。
5. **SSH 密钥权限**：如果推送失败，检查 `~/.ssh/id_ed25519.pub` 是否已添加到 GitHub 和 Gitee 的账户设置中。

---

## 14. 快速参考表

| 需求 | 命令 |
|------|------|
| 查看状态 | `git status` |
| 添加所有修改 | `git add .` |
| 提交 | `git commit -m "xxx"` |
| 推送到双平台 | `git push` |
| 拉取最新代码 | `git pull` |
| 查看历史 | `git log --oneline -10` |
| 创建新分支 | `git switch -c 分支名` |
| 切换分支 | `git switch 分支名` |
| 合并分支 | `git merge 分支名` |
| 删除已合并分支 | `git branch -d 分支名` |
| 撤销暂存 | `git restore --staged 文件` |
| 丢弃修改 | `git restore 文件` |
| 临时保存 | `git stash` / `git stash pop` |
| 查看 remote | `git remote -v` |
| 找回丢失提交 | `git reflog` |

---

*文档生成时间：2024年*
