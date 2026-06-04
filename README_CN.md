# Hermes Tools

HTTP CONNECT 隧道转发工具，解决 ISP 深度包检测拦截非标准端口 TLS 流量的问题。

> 虽然名叫 Hermes Tools，但这个工具不依赖任何特定 AI 工具，在 Claude Code、Hermes Agent 或独立环境下均可使用。本质就是一个通过 HTTP CONNECT 代理建立 TCP 隧道的 Python 脚本。

---

## 安装

将脚本复制到 `~/.local/bin/` 目录下，并确保该目录在 `PATH` 中：

```bash
cp hermes-email-tunnel ~/.local/bin/
chmod +x ~/.local/bin/hermes-email-tunnel
```

### 依赖

- Python 3.6+
- 本地 HTTP 代理（如 mihomo / Clash，默认监听 `127.0.0.1:7890`）

> 如需修改代理地址，编辑脚本中的 `PROXY` 变量。

---

## 使用场景

某些 ISP 会对非标准端口的 TLS 流量进行深度包检测（DPI）并拦截。例如，QQ 邮箱的 IMAP/SMTP 服务使用 993/465 端口，在部分网络环境下 TLS 握手会被 ISP 阻断，导致邮件客户端无法连接。

`hermes-email-tunnel` 通过 HTTP CONNECT 隧道将本地流量经由代理转发，绕过 ISP 的 DPI 拦截：

```
本地应用 → localhost:本地端口 → HTTP CONNECT 隧道 → 代理 → 目标服务器
```

代理与目标服务器之间走的是标准 HTTP 代理协议，ISP 看到的只是普通的 HTTP CONNECT 请求，不会触发对非标准端口 TLS 的检测。

---

## 用法示例

### 启动隧道

```bash
# 将本地 11443 端口转发到 QQ 邮箱 IMAP 服务器 (imap.qq.com:993)
hermes-email-tunnel 11443 imap.qq.com 993

# 将本地 10465 端口转发到 QQ 邮箱 SMTP 服务器 (smtp.qq.com:465)
hermes-email-tunnel 10465 smtp.qq.com 465
```

启动后终端会显示：

```
隧道: 127.0.0.1:11443 → imap.qq.com:993
隧道: 127.0.0.1:10465 → smtp.qq.com:465
```

### 配置应用指向 localhost

在邮件客户端或其他应用中将服务器地址改为 `127.0.0.1`，端口改为本地转发端口：

| 服务 | 原始地址 | 隧道端口 |
|------|----------|----------|
| IMAP | `imap.qq.com:993` | `127.0.0.1:11443` |
| SMTP | `smtp.qq.com:465` | `127.0.0.1:10465` |

注意：TLS/SSL 仍然启用，因为隧道只是转发加密流量，证书验证不受影响。

---

## systemd 服务化

创建 systemd 服务文件以实现开机自启和后台运行。

### 1. 创建服务文件

`/etc/systemd/system/hermes-email-tunnel@.service`：

```ini
[Unit]
Description=Hermes Email Tunnel for %i
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/YOUR_USER/.local/bin/hermes-email-tunnel %i
Restart=always
RestartSec=10
User=YOUR_USER

[Install]
WantedBy=multi-user.target
```

> 将 `YOUR_USER` 替换为实际用户名。

### 2. 创建实例配置文件

`/etc/hermes-tunnel/` 目录下存放各隧道实例的配置：

```bash
sudo mkdir -p /etc/hermes-tunnel
```

`/etc/hermes-tunnel/imap@.conf`：

```
11443 imap.qq.com 993
```

`/etc/hermes-tunnel/smtp@.conf`：

```
10465 smtp.qq.com 465
```

### 3. 使用包装脚本

创建 `/usr/local/bin/hermes-tunnel-wrapper`：

```bash
#!/bin/bash
CONF="/etc/hermes-tunnel/${1}.conf"
if [ -f "$CONF" ]; then
    exec /home/YOUR_USER/.local/bin/hermes-email-tunnel $(cat "$CONF")
else
    echo "Config not found: $CONF"
    exit 1
fi
```

修改服务文件中的 `ExecStart` 为：

```
ExecStart=/usr/local/bin/hermes-tunnel-wrapper %i
```

### 4. 启用服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes-email-tunnel@imap
sudo systemctl enable hermes-email-tunnel@smtp
sudo systemctl start hermes-email-tunnel@imap
sudo systemctl start hermes-email-tunnel@smtp
```

### 5. 查看状态

```bash
sudo systemctl status hermes-email-tunnel@imap
sudo systemctl status hermes-email-tunnel@smtp
```

---

## 相关项目

- [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) — Claude Code on WSL2 踩坑指南
- [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) — WSL2 开发环境一键部署
- [hermes-setup-guide](https://github.com/2281516753/hermes-setup-guide) — Claude Code 安装配置指南

---

## 作者

王炯 (Wang Jiong) — 网络工程专业。

[GitHub](https://github.com/2281516753)

---

## License

MIT