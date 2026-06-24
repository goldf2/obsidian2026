# SSH 密钥对认证自动登录作业指导书

| 文档信息 | |
|---------|---|
| 文档编号 | SOP-SSH-001 |
| 版本 | V1.0 |
| 适用系统 | macOS / Linux / Windows (WSL/Git Bash) |
| 目标读者 | 系统管理员、开发工程师、运维人员 |
| 编写日期 | 2025-06-06 |

---

## 1. 目的

本作业指导书规范使用 **SSH 密钥对认证** 实现客户端免密自动登录远程服务器的操作流程，替代传统的密码认证方式，提升安全性与操作效率。

## 2. 适用范围

- 需要频繁登录远程服务器的日常运维场景
- CI/CD 自动化部署中的 SSH 免密登录
- 脚本批量管理多台服务器的场景
- 需要消除密码输入交互的自动化任务

## 3. 原理简述

SSH 密钥对由**私钥**（保存在本地）和**公钥**（部署到远程服务器）组成。远程服务器通过验证客户端持有的私钥是否与已授权的公钥匹配，来完成身份认证，无需传输密码。

> **安全等级：高**（基于非对称加密，不存在密码泄露或暴力破解风险）

## 4. 前提条件

| 检查项 | 要求 |
|--------|------|
| 本地系统 | macOS、Linux 或 Windows（需安装 WSL/Git Bash/OpenSSH） |
| 远程服务器 | 已开启 SSH 服务，且允许密钥认证（默认允许） |
| 网络 | 本地可正常通过 SSH 连接到远程服务器 |
| 权限 | 首次配置需知道远程服务器的用户名和密码（用于部署公钥） |

## 5. 操作步骤

### 步骤 1：检查本地是否已有 SSH 密钥

在本地终端执行：

```bash
ls ~/.ssh/
```

查看是否存在以下文件：
- `id_ed25519`（私钥）
- `id_ed25519.pub`（公钥）
- 或 `id_rsa` / `id_rsa.pub`（旧版 RSA 密钥）

**判断分支：**
- **已存在**：跳至步骤 3
- **不存在**：继续执行步骤 2

---

### 步骤 2：生成新的 SSH 密钥对

在本地终端执行：

```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)"
```

系统交互提示及处理方式：

| 提示 | 推荐操作 | 说明 |
|------|---------|------|
| `Enter file in which to save the key` | 直接按 **回车** | 使用默认路径 `~/.ssh/id_ed25519` |
| `Enter passphrase` | 直接按 **回车**（留空） | 如需完全免密自动化登录，不设置密码；如需额外安全层，可设置密码 |
| `Enter same passphrase again` | 再次按 **回车** | 确认密码 |

**验证生成结果：**

```bash
ls -la ~/.ssh/
```

确认出现 `id_ed25519`（私钥）和 `id_ed25519.pub`（公钥）。

> ⚠️ **安全警告**：私钥文件 `id_ed25519` 相当于你的"身份证原件"，**绝对不要**复制、传输或泄露给任何人。

---

### 步骤 3：将公钥部署到远程服务器

#### 方式 A：使用 `ssh-copy-id`（推荐，最简单）

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@remote_ip
```

替换说明：
- `user`：远程服务器的用户名（如 `root`、`ubuntu`）
- `remote_ip`：远程服务器 IP 地址（如 `8.153.198.105`）

执行后输入远程用户密码，公钥将自动追加到远程服务器的 `~/.ssh/authorized_keys`。

#### 方式 B：手动复制（当 `ssh-copy-id` 不可用时）

```bash
# 1. 在本地读取公钥内容
cat ~/.ssh/id_ed25519.pub

# 2. 手动登录远程服务器
ssh user@remote_ip

# 3. 在远程服务器上执行
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "粘贴刚才的公钥内容" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

### 步骤 4：验证免密自动登录

在本地终端直接执行：

```bash
ssh user@remote_ip
```

**预期结果**：无需输入密码，直接登录远程服务器，出现远程服务器的命令提示符。

---

### 步骤 5：（可选）配置 SSH 客户端快捷登录

编辑本地文件 `~/.ssh/config`：

```bash
nano ~/.ssh/config
```

添加以下内容：

```ssh-config
Host myserver
    HostName 8.153.198.105
    User root
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

**配置说明：**

| 参数 | 说明 |
|------|------|
| `Host` | 快捷名称，自定义 |
| `HostName` | 远程服务器 IP 或域名 |
| `User` | 登录用户名 |
| `Port` | SSH 端口，默认 22 |
| `IdentityFile` | 本地私钥路径 |
| `IdentitiesOnly` | 只使用指定的密钥，不尝试其他密钥 |

保存后，使用快捷名称登录：

```bash
ssh myserver
```

---

## 6. 安全加固（强烈推荐）

### 6.1 远程服务器关闭密码认证

登录远程服务器后，编辑 SSH 配置文件：

```bash
sudo nano /etc/ssh/sshd_config
```

修改以下参数：

```ini
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password
```

重启 SSH 服务：

```bash
# Ubuntu/Debian
sudo systemctl restart ssh

# CentOS/RHEL
sudo systemctl restart sshd
```

> ⚠️ **警告**：执行此操作前，请确保密钥认证已正常工作，否则可能导致无法登录。

### 6.2 本地私钥权限保护

```bash
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 700 ~/.ssh
```

---

## 7. 故障排除

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| 仍提示输入密码 | 公钥未正确部署到远程 `authorized_keys` | 重新执行 `ssh-copy-id`，或检查远程文件权限应为 600 |
| 提示 `Permission denied (publickey)` | 远程服务器禁用了密钥认证 | 检查远程 `/etc/ssh/sshd_config` 中 `PubkeyAuthentication yes` |
| 提示 `Bad permissions` | 本地 `.ssh` 目录或私钥权限过松 | 执行 `chmod 700 ~/.ssh` 和 `chmod 600 ~/.ssh/id_ed25519` |
| 多个密钥冲突 | SSH 客户端尝试了错误的密钥 | 在 `~/.ssh/config` 中配置 `IdentitiesOnly yes` |
| 连接被重置 | 防火墙或安全组阻止 SSH 端口 | 检查远程服务器防火墙和安全组规则，确认端口 22（或自定义端口）开放 |

**启用调试模式查看详细错误：**

```bash
ssh -vvv user@remote_ip
```

---

## 8. 多台服务器批量部署

当需要为多台服务器部署相同的公钥时，使用循环脚本：

```bash
#!/bin/bash
SERVERS=("192.168.1.10" "192.168.1.11" "192.168.1.12")
USER="root"
for ip in "${SERVERS[@]}"; do
    ssh-copy-id -i ~/.ssh/id_ed25519.pub "${USER}@${ip}"
done
```

---

## 9. 关键安全守则

1. **私钥不外传**：`id_ed25519` 私钥文件等同于你的身份凭证，切勿通过邮件、即时通讯、云盘等方式传输。
2. **丢失即吊销**：如私钥疑似泄露，立即从远程服务器 `~/.ssh/authorized_keys` 中删除对应公钥行。
3. **定期轮换**：建议每 1-2 年重新生成密钥对并替换。
4. **备份策略**：私钥本身不可备份到不安全位置；如需备份，使用加密的密码管理器或硬件安全模块（HSM）。
5. **人员离职/交接**：工作人员变动时，立即将其公钥从所有服务器中移除。

---

## 10. 附录

### 附录 A：常用命令速查表

| 命令 | 用途 |
|------|------|
| `ssh-keygen -t ed25519` | 生成 Ed25519 类型密钥对 |
| `ssh-keygen -t rsa -b 4096` | 生成 RSA 4096 位密钥对（兼容性场景） |
| `ssh-copy-id -i ~/.ssh/id.pub user@host` | 部署公钥到远程服务器 |
| `ssh -i ~/.ssh/id_ed25519 user@host` | 指定私钥文件登录 |
| `cat ~/.ssh/id_ed25519.pub` | 查看公钥内容 |

### 附录 B：生成作业记录模板

每次执行本作业时，填写以下记录：

| 项目 | 内容 |
|------|------|
| 执行日期 | |
| 执行人 | |
| 远程服务器 IP | |
| 登录用户名 | |
| 密钥类型 | Ed25519 / RSA |
| 是否设置密码短语 | 是 / 否 |
| 验证结果 | 成功 / 失败 |
| 备注 | |

---

> **文档结束** | 如有疑问，请联系系统管理员或查阅 `man ssh-keygen`、`man ssh-copy-id` 手册页。
