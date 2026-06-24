# 待整理摘录：来自 Git 文档合并

> 这些内容来自根目录原来的 `git 工具学习.txt` 和 `gittest.txt`，不属于 Git 命令手册正文，先单独保留，后续可以再归类。

## 开发规范仓库想法

应以一个 git 仓库来存放各种开发规范。

## 智厨

希望加上 yazio 的功能，并且可以和各种厨房料理机进行配套，比如美善品。

请你完善开发功能需求。

## Codex 启动权限摘录

可以，但这不能在当前对话里由我自己切换，需要你重新启动 Codex 会话时指定权限。

最直接的方式，在服务器终端里这样启动：

```bash
cd /opt/web2py_yf
codex --sandbox danger-full-access --ask-for-approval never
```

或者用官方提供的全跳过模式：

```bash
cd /opt/web2py_yf
codex --dangerously-bypass-approvals-and-sandbox
```

等价别名：

```bash
codex --yolo
```

如果你想永久默认这样运行，编辑 `~/.codex/config.toml`：

```toml
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

然后重新进入 `/opt/web2py_yf` 启动 Codex。

## 编号摘录

```text
051214847177
11130056
```

## SSH 公钥摘录

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKid2ygIcWt0tGIeuKB0rVdpM6NgoN1Gu9ABctJri/wz goldf2@163.com
```
