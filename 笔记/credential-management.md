# Git 凭据管理最佳实践

> **版本**: 1.0.0
> **最后更新**: 2026-01-16
> **适用范围**: 所有使用 Git 的项目和环境

---

## 📋 目录

- [认证方式对比](#认证方式对比)
- [SSH 密钥管理](#ssh-密钥管理)
- [HTTPS 凭据管理](#https-凭据管理)
- [环境分级策略](#环境分级策略)
- [自动化场景](#自动化场景)
- [安全审计](#安全审计)

---

## 认证方式对比

### 三种主要方式

| 方式 | 安全性 | 便捷性 | 自动化 | 适用场景 |
|-----|--------|--------|--------|---------|
| **SSH 密钥** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 服务器、CI/CD、开发环境 |
| **个人访问令牌 (PAT)** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | HTTPS 环境、临时访问 |
| **用户名密码** | ⭐⭐ | ⭐⭐ | ⭐ | 不推荐 |

---

## SSH 密钥管理

### ✅ 最佳实践

#### 1. 密钥生成

```bash
# 推荐：RSA 4096 位
ssh-keygen -t rsa -b 4096 -C "user@example.com" -f ~/.ssh/id_rsa_company

# 或：Ed25519（现代推荐）
ssh-keygen -t ed25519 -C "user@example.com" -f ~/.ssh/id_rsa_ed25519
```

**命名规范**：
- `id_rsa`: 默认密钥
- `id_rsa_github`: GitHub 专用
- `id_rsa_company`: 公司项目专用
- `id_rsa_personal`: 个人项目专用

#### 2. 权限设置

```bash
# 私钥：仅所有者可读写
chmod 600 ~/.ssh/id_rsa

# 公钥：所有者可读写，其他人只读
chmod 644 ~/.ssh/id_rsa.pub

# .ssh 目录
chmod 700 ~/.ssh
```

#### 3. 配置文件

```bash
# ~/.ssh/config

# 工作项目
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_work
    IdentitiesOnly yes

# 个人项目
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_personal
    IdentitiesOnly yes

# 使用示例
# git clone git@github-work:company/repo.git
# git clone git@github-personal:username/repo.git
```

#### 4. 密钥备份

```bash
# 加密备份私钥
tar -czf ssh-keys-backup.tar.gz ~/.ssh/id_rsa*
gpg -c ssh-keys-backup.tar.gz  # 设置密码加密
rm ssh-keys-backup.tar.gz

# 存储到安全位置
mv ssh-keys-backup.tar.gz.gpg /secure/backup/location/
```

#### 5. 密钥轮换

```bash
# 每 1-2 年轮换密钥

# 1. 生成新密钥
ssh-keygen -t rsa -b 4096 -C "user@example.com" -f ~/.ssh/id_rsa_new

# 2. 添加新密钥到 Git 平台
cat ~/.ssh/id_rsa_new.pub

# 3. 测试新密钥
ssh -i ~/.ssh/id_rsa_new -T git@github.com

# 4. 替换旧密钥
mv ~/.ssh/id_rsa ~/.ssh/id_rsa_old
mv ~/.ssh/id_rsa_new ~/.ssh/id_rsa

# 5. 从平台移除旧公钥
# 访问 GitHub/Gitee 设置页面删除旧密钥

# 6. 安全删除旧私钥
shred -vfz -n 10 ~/.ssh/id_rsa_old
```

---

## HTTPS 凭据管理

### 方式 1: 凭据存储（Credential Helper）

#### Linux - 明文存储

```bash
# 启用凭据存储
git config --global credential.helper store

# 凭据保存位置
~/.git-credentials

# 格式
https://username:password@github.com
```

**⚠️ 安全风险**：
- 密码明文存储
- 不适合共享服务器
- 仅适合私人开发环境

#### macOS - Keychain

```bash
# 使用系统钥匙串（安全）
git config --global credential.helper osxkeychain
```

#### Windows - Credential Manager

```bash
# 使用 Windows 凭据管理器
git config --global credential.helper manager
```

#### 自定义缓存时间

```bash
# 缓存 1 小时
git config --global credential.helper 'cache --timeout=3600'

# 缓存 1 天
git config --global credential.helper 'cache --timeout=86400'
```

---

### 方式 2: 个人访问令牌 (PAT)

#### GitHub PAT

**生成步骤**：
1. 访问：https://github.com/settings/tokens
2. 点击 "Generate new token" → "Generate new token (classic)"
3. 设置：
   - Note: 描述用途（如 "AL03 Server"）
   - Expiration: 过期时间（建议 90 天）
   - Scopes: 勾选 `repo`（完整仓库访问）
4. 点击 "Generate token"
5. **立即复制**（只显示一次）

**使用方式**：

```bash
# 方法 1: 在 URL 中包含（不推荐，会显示在日志）
git clone https://TOKEN@github.com/user/repo.git

# 方法 2: 使用 credential helper
git config --global credential.helper store
git clone https://github.com/user/repo.git
# 输入用户名和令牌（作为密码）
```

#### Gitee PAT

**生成步骤**：
1. 访问：https://gitee.com/profile/personal_access_tokens
2. 点击 "生成新令牌"
3. 设置：
   - 描述：用途说明
   - 权限：勾选 `projects`
4. 点击 "提交"
5. **复制令牌**

---

### 方式 3: Git Credential Manager (GCM)

```bash
# 安装 GCM (Linux)
wget https://github.com/GitCredentialManager/git-credential-manager/releases/download/v2.0.0/gcm-linux_amd64.2.0.0.deb
sudo dpkg -i gcm-linux_amd64.2.0.0.deb

# 配置
git-credential-manager configure

# 使用
git config --global credential.credentialStore secretservice
```

**优势**：
- ✅ 跨平台
- ✅ 支持多因素认证
- ✅ 安全存储凭据
- ✅ 自动刷新令牌

---

## 环境分级策略

### Level 1: 生产环境（最严格）

```bash
# 仅使用 SSH 密钥
# 密钥必须设置密码短语
ssh-keygen -t rsa -b 4096 -C "prod@company.com"

# 使用 ssh-agent 缓存密码短语
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

# 禁用凭据存储
git config --global --unset credential.helper
```

**检查清单**：
- [ ] 使用 SSH 密钥（4096 位 RSA 或 Ed25519）
- [ ] 密钥有密码短语保护
- [ ] 私钥权限 600
- [ ] 定期轮换密钥（每年）
- [ ] 禁用 HTTPS 凭据存储

---

### Level 2: 开发/测试环境（平衡）

```bash
# 使用 SSH 密钥（可选密码短语）
ssh-keygen -t rsa -b 4096 -C "dev@company.com" -N ""

# 或使用 PAT + credential helper
git config --global credential.helper 'cache --timeout=86400'
```

**检查清单**：
- [ ] SSH 密钥或 PAT
- [ ] 凭据缓存时间 ≤ 24 小时
- [ ] 不在共享服务器明文存储密码
- [ ] 定期审查访问权限

---

### Level 3: 本地开发（便捷）

```bash
# 使用任意方式
# SSH 密钥（推荐）
ssh-keygen -t ed25519

# 或 credential.helper store（不推荐）
git config --global credential.helper store
```

**注意事项**：
- ⚠️ 仅在私人设备使用
- ⚠️ 不要在公共电脑保存凭据
- ⚠️ 定期清理 `~/.git-credentials`

---

## 自动化场景

### CI/CD 管道

#### GitHub Actions

```yaml
# .github/workflows/deploy.yml

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # 使用 SSH 密钥
      - uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      # 或使用 PAT
      - name: Clone private repo
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git clone https://x-access-token:${GH_TOKEN}@github.com/user/repo.git
```

#### GitLab CI

```yaml
# .gitlab-ci.yml

before_script:
  # 配置 SSH
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
  - chmod 600 ~/.ssh/id_rsa
  - ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```

---

### Cron 定时任务

```bash
# /root/.claude/sync-rules.sh

#!/bin/bash

# 确保 SSH 密钥可用
if [ ! -f ~/.ssh/id_rsa ]; then
    echo "❌ SSH 密钥不存在"
    exit 1
fi

# 检查密钥权限
PERMS=$(stat -c %a ~/.ssh/id_rsa)
if [ "$PERMS" != "600" ]; then
    echo "⚠️ 修正密钥权限"
    chmod 600 ~/.ssh/id_rsa
fi

# 同步操作
cd /root/claude_ruler || exit 1
git pull origin main
git push origin main
```

---

### Docker 容器

```dockerfile
# Dockerfile

FROM ubuntu:22.04

# 复制 SSH 配置
COPY --chown=root:root .ssh /root/.ssh
RUN chmod 700 /root/.ssh && \
    chmod 600 /root/.ssh/id_rsa

# 或使用构建参数传递 PAT
ARG GITHUB_TOKEN
RUN git config --global credential.helper store && \
    echo "https://x-access-token:${GITHUB_TOKEN}@github.com" > ~/.git-credentials
```

**安全建议**：
- 使用 Docker secrets
- 不要在镜像中硬编码凭据
- 使用多阶段构建清理敏感信息

---

## 安全审计

### 审计清单

#### 密钥审计

```bash
# 1. 检查所有 SSH 密钥
ls -la ~/.ssh/id_*

# 2. 检查密钥权限
find ~/.ssh -type f -name "id_*" ! -perm 600

# 3. 检查密钥年龄
stat -c '%y %n' ~/.ssh/id_rsa

# 4. 列出所有已添加的密钥
ssh-add -l
```

#### 凭据审计

```bash
# 1. 检查 credential helper 配置
git config --global --get credential.helper

# 2. 查看存储的凭据（如果使用 store）
cat ~/.git-credentials

# 3. 检查 PAT 过期时间
# 访问 GitHub/Gitee 设置页面查看

# 4. 审查仓库远程地址
find ~ -type d -name ".git" -exec sh -c 'cd {} && git remote -v' \;
```

---

### 泄露响应

#### 如果私钥泄露

```bash
# 1. 立即从 Git 平台删除公钥
# 访问 GitHub/Gitee 设置页面

# 2. 生成新密钥
ssh-keygen -t rsa -b 4096 -C "new-key@example.com"

# 3. 更新所有使用该密钥的系统

# 4. 安全删除旧私钥
shred -vfz -n 10 ~/.ssh/id_rsa_compromised

# 5. 审查访问日志
# 检查 Git 平台的访问历史
```

#### 如果密码/PAT 泄露

```bash
# 1. 立即撤销 PAT
# GitHub: https://github.com/settings/tokens
# Gitee: https://gitee.com/profile/personal_access_tokens

# 2. 如果是密码，立即更改密码

# 3. 清理存储的凭据
rm ~/.git-credentials
git config --global --unset credential.helper

# 4. 审查最近的推送记录
git reflog
```

---

## 常见错误

### ❌ 错误 1: 密码硬编码

```bash
# ❌ 错误
git clone https://username:password@github.com/user/repo.git

# ✅ 正确
git clone git@github.com:user/repo.git  # SSH
# 或
git clone https://github.com/user/repo.git  # 使用 credential helper
```

---

### ❌ 错误 2: 凭据提交到仓库

```bash
# ❌ 错误：包含密码的脚本提交到 Git
#!/bin/bash
git clone https://user:MyPassword123@github.com/repo.git

# ✅ 正确：使用环境变量
#!/bin/bash
git clone https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/repo.git
```

---

### ❌ 错误 3: 错误的权限

```bash
# ❌ 私钥权限过宽
-rw-r--r-- 1 root root 3243 Jan 16 10:00 id_rsa

# ✅ 正确权限
-rw------- 1 root root 3243 Jan 16 10:00 id_rsa

# 修正
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh
```

---

## 推荐配置

### 个人开发者

```bash
# 1. 使用 SSH 密钥
ssh-keygen -t ed25519 -C "your_email@example.com"

# 2. 添加到 ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 3. 上传公钥到 GitHub/Gitee
cat ~/.ssh/id_ed25519.pub
```

---

### 团队协作

```bash
# 1. 每个成员使用独立 SSH 密钥
# 2. 使用有意义的注释区分
ssh-keygen -t rsa -b 4096 -C "john@company.com-workstation"

# 3. 定期审查团队成员访问权限
# 4. 强制密钥轮换策略（每年）
```

---

### 服务器自动化

```bash
# 1. 使用无密码短语的 SSH 密钥
ssh-keygen -t rsa -b 4096 -C "server-automation" -N ""

# 2. 限制密钥使用范围（在 authorized_keys 中）
command="/usr/local/bin/deploy.sh" ssh-rsa AAAAB3...

# 3. 使用专用部署密钥
# GitHub: Settings → Deploy keys → Add deploy key
# 勾选 "Allow write access" 如需推送权限
```

---

## 参考资料

### 官方文档
- [GitHub 认证文档](https://docs.github.com/en/authentication)
- [Git Credential Storage](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage)
- [SSH.com Best Practices](https://www.ssh.com/academy/ssh/key-management)

### 相关文档
- [SSH 密钥配置指南](./ssh-key-setup-guide.md)
- [RULE-DEVOPS-004: Git 配置管理规则](../../claude_ruler/devops/git-configuration-management.md)
- [AL03 服务器 Git SSH 配置](../../AL03/configuration/git-ssh-setup.md)

---

## 快速参考

### SSH 密钥流程

```bash
# 生成 → 添加 → 测试 → 使用
ssh-keygen -t rsa -b 4096 -C "email@example.com"
cat ~/.ssh/id_rsa.pub  # 复制到 GitHub/Gitee
ssh -T git@github.com
git clone git@github.com:user/repo.git
```

### HTTPS PAT 流程

```bash
# 生成令牌 → 配置 → 使用
# 在 GitHub/Gitee 生成 PAT
git config --global credential.helper store
git clone https://github.com/user/repo.git
# 输入用户名和 PAT
```

---

*版本: 1.0.0*
*最后更新: 2026-01-16*
*维护者: DevOps 团队*
