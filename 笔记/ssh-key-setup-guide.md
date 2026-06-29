# Git SSH 密钥配置通用指南

> **版本**: 1.0.0
> **最后更新**: 2026-01-16
> **适用范围**: Linux 服务器、开发环境

---

## 📋 目录

- [什么是 SSH 密钥](#什么是-ssh-密钥)
- [为什么使用 SSH](#为什么使用-ssh)
- [生成 SSH 密钥](#生成-ssh-密钥)
- [配置 Git 平台](#配置-git-平台)
- [测试连接](#测试连接)
- [高级配置](#高级配置)
- [故障排查](#故障排查)

---

## 什么是 SSH 密钥

SSH 密钥是一种**公钥加密**认证方式，包含两个文件：

- **私钥** (`id_rsa`): 保存在本地，绝不分享
- **公钥** (`id_rsa.pub`): 上传到 Git 平台（GitHub、Gitee 等）

**工作原理**：
```
本地仓库 (私钥) ←→ Git 平台 (公钥)
         ↓
    加密验证，无需密码
```

---

## 为什么使用 SSH

### ✅ SSH 的优势

| 特性 | SSH | HTTPS |
|-----|-----|-------|
| **安全性** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **便捷性** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **自动化** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **配置难度** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**SSH 优势**：
- ✅ 一次配置，永久使用
- ✅ 无需每次输入密码
- ✅ 更安全（密钥长度 4096 位）
- ✅ 适合自动化脚本（CI/CD、定时任务）
- ✅ 不会在命令行暴露密码

**HTTPS 劣势**：
- ❌ 密码明文存储在 `~/.git-credentials`
- ❌ 每次推送都要输入密码（除非配置凭据存储）
- ❌ 不适合自动化场景
- ❌ 共享服务器上不安全

---

## 生成 SSH 密钥

### 方法 1: 标准生成（推荐）

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

**交互式提示**：
```
Enter file in which to save the key (/root/.ssh/id_rsa):
# 按回车使用默认路径

Enter passphrase (empty for no passphrase):
# 输入密码短语（可选，自动化场景建议留空）

Enter same passphrase again:
# 再次输入密码短语
```

### 方法 2: 非交互式生成（自动化）

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" \
  -f ~/.ssh/id_rsa -N ""
```

**参数说明**：
- `-t rsa`: 密钥类型（RSA 算法）
- `-b 4096`: 密钥长度（4096 位，更安全）
- `-C`: 注释（通常使用邮箱，方便识别）
- `-f`: 指定密钥文件路径
- `-N ""`: 无密码短语（适合自动化）

### 方法 3: 使用 Ed25519（现代推荐）

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**Ed25519 优势**：
- ✅ 更短的密钥（256 位）
- ✅ 更快的加密速度
- ✅ 相同安全级别（等效 RSA 3072 位）
- ⚠️ 需要较新的系统支持（2014+ 年）

---

## 配置 Git 平台

### GitHub

#### 1. 查看公钥
```bash
cat ~/.ssh/id_rsa.pub
```

#### 2. 添加到 GitHub
1. 访问：https://github.com/settings/keys
2. 点击 "New SSH key"
3. 标题：填写识别名称（如 "AL03 Server"）
4. 密钥：粘贴公钥内容
5. 点击 "Add SSH key"

#### 3. 测试连接
```bash
ssh -T git@github.com
```

**成功输出**：
```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

---

### Gitee（码云）

#### 1. 添加到 Gitee
1. 访问：https://gitee.com/profile/sshkeys
2. 点击 "添加公钥"
3. 标题：填写识别名称
4. 公钥：粘贴 `~/.ssh/id_rsa.pub` 的内容
5. 点击 "确定"

#### 2. 测试连接
```bash
ssh -T git@gitee.com
```

**成功输出**：
```
Hi username! You've successfully authenticated, but GITEE.COM does not provide shell access.
```

---

### GitLab

#### 1. 添加到 GitLab
1. 访问：https://gitlab.com/-/profile/keys
2. 粘贴公钥内容
3. 标题：自动生成或手动填写
4. 点击 "Add key"

#### 2. 测试连接
```bash
ssh -T git@gitlab.com
```

---

## 修改仓库使用 SSH

### 查看当前远程地址

```bash
git remote -v
```

**HTTPS 格式**：
```
origin  https://github.com/user/repo.git (fetch)
origin  https://github.com/user/repo.git (push)
```

### 修改为 SSH 格式

#### GitHub
```bash
git remote set-url origin git@github.com:user/repo.git
```

#### Gitee
```bash
git remote set-url origin git@gitee.com:user/repo.git
```

#### GitLab
```bash
git remote set-url origin git@gitlab.com:user/repo.git
```

### 验证修改

```bash
git remote -v
```

**SSH 格式**：
```
origin  git@github.com:user/repo.git (fetch)
origin  git@github.com:user/repo.git (push)
```

---

## 高级配置

### SSH 配置文件

创建 `~/.ssh/config` 文件可以简化 SSH 连接：

```bash
cat > ~/.ssh/config << 'EOF'
# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Gitee
Host gitee.com
    HostName gitee.com
    User git
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3

# GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF

# 设置权限
chmod 600 ~/.ssh/config
```

**配置说明**：
- `IdentityFile`: 指定使用的私钥
- `ServerAliveInterval`: 每 60 秒发送心跳包
- `ServerAliveCountMax`: 最多 3 次无响应后断开

---

### 多密钥管理

如果需要为不同平台使用不同密钥：

```bash
# 生成 GitHub 专用密钥
ssh-keygen -t rsa -b 4096 -C "github@example.com" -f ~/.ssh/id_rsa_github

# 生成 Gitee 专用密钥
ssh-keygen -t rsa -b 4096 -C "gitee@example.com" -f ~/.ssh/id_rsa_gitee
```

**配置文件**：
```bash
# ~/.ssh/config

Host github.com
    IdentityFile ~/.ssh/id_rsa_github

Host gitee.com
    IdentityFile ~/.ssh/id_rsa_gitee
```

---

### 密钥密码短语管理

如果设置了密码短语，可以使用 `ssh-agent` 缓存：

```bash
# 启动 ssh-agent
eval "$(ssh-agent -s)"

# 添加密钥到 agent
ssh-add ~/.ssh/id_rsa

# 查看已添加的密钥
ssh-add -l

# 永久生效（添加到 ~/.bashrc）
cat >> ~/.bashrc << 'EOF'
if [ -z "$SSH_AUTH_SOCK" ]; then
   eval $(ssh-agent -s)
   ssh-add ~/.ssh/id_rsa 2>/dev/null
fi
EOF
```

---

## 安全最佳实践

### ✅ 推荐做法

1. **使用强密钥**
   - RSA 至少 4096 位
   - 或使用 Ed25519

2. **保护私钥**
   ```bash
   chmod 600 ~/.ssh/id_rsa
   chmod 644 ~/.ssh/id_rsa.pub
   ```

3. **设置密码短语**（非自动化场景）
   - 增加额外安全层
   - 使用 ssh-agent 缓存

4. **定期轮换密钥**
   - 建议每 1-2 年更换
   - 旧密钥从平台移除

5. **备份私钥**
   - 加密存储到安全位置
   - 使用密码管理器

### ❌ 避免做法

1. **不要分享私钥**
   - 绝不通过邮件、聊天发送
   - 不要提交到 Git 仓库

2. **不要使用弱密钥**
   - 不要使用低于 2048 位的 RSA
   - 不要使用 DSA（已过时）

3. **不要忽略权限**
   - 私钥权限必须是 600
   - .ssh 目录权限必须是 700

---

## 故障排查

### 问题 1: Permission denied (publickey)

**原因**：
- 公钥未添加到 Git 平台
- 使用了错误的私钥
- 私钥权限不正确

**解决**：
```bash
# 检查公钥是否正确
cat ~/.ssh/id_rsa.pub

# 检查权限
ls -la ~/.ssh/id_rsa
# 应该显示: -rw------- (600)

# 修正权限
chmod 600 ~/.ssh/id_rsa

# 测试连接（调试模式）
ssh -vT git@github.com
```

---

### 问题 2: Could not resolve hostname

**原因**：网络问题或 DNS 解析失败

**解决**：
```bash
# 测试网络连接
ping github.com

# 测试 SSH 端口
nc -zv github.com 22

# 检查 /etc/hosts
cat /etc/hosts | grep github
```

---

### 问题 3: Connection timed out

**原因**：
- 防火墙阻止 SSH 端口（22）
- 网络限制
- Git 平台服务问题

**解决**：
```bash
# 使用 HTTPS 端口（443）作为 SSH
cat >> ~/.ssh/config << 'EOF'
Host github.com
    HostName ssh.github.com
    Port 443
    User git
EOF

# 测试
ssh -T git@github.com
```

---

### 问题 4: Host key verification failed

**原因**：
- 第一次连接到主机
- 主机密钥已更改（中间人攻击警告）

**解决**：
```bash
# 清除旧的主机密钥
ssh-keygen -R github.com

# 重新连接（自动添加新密钥）
ssh -T git@github.com
```

---

## 自动化场景

### CI/CD 配置

```yaml
# .gitlab-ci.yml 示例

before_script:
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
  - chmod 600 ~/.ssh/id_rsa
  - ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
  - git config --global user.email "ci@example.com"
  - git config --global user.name "CI Bot"
```

---

### 定时任务配置

```bash
# /root/.claude/sync-rules.sh

#!/bin/bash

# 确保 SSH 密钥可用
if [ ! -f ~/.ssh/id_rsa ]; then
    echo "SSH 密钥不存在"
    exit 1
fi

# 拉取更新
git pull origin main

# 推送本地提交
git push origin main
```

---

## 检查清单

### 初次配置
- [ ] SSH 密钥已生成
- [ ] 私钥权限设置为 600
- [ ] 公钥已添加到 Git 平台
- [ ] SSH 连接测试通过
- [ ] 仓库远程地址已修改为 SSH

### 安全检查
- [ ] 私钥未暴露或分享
- [ ] 使用了 4096 位 RSA 或 Ed25519
- [ ] 密码短语已设置（非自动化场景）
- [ ] .ssh 目录权限为 700
- [ ] 私钥有备份（加密存储）

### 功能验证
- [ ] 可以无密码 git pull
- [ ] 可以无密码 git push
- [ ] 自动化脚本正常工作
- [ ] 多台服务器配置正确

---

## 参考资料

### 官方文档
- [GitHub SSH 文档](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [GitLab SSH 文档](https://docs.gitlab.com/ee/ssh/)
- [Gitee SSH 文档](https://gitee.com/help/articles/4181)

### 相关文档
- [Git 凭据管理最佳实践](./credential-management.md)
- [RULE-DEVOPS-004: Git 配置管理规则](../../claude_ruler/devops/git-configuration-management.md)

---

*版本: 1.0.0*
*最后更新: 2026-01-16*
*维护者: DevOps 团队*
