# Hermes Tools

HTTP CONNECT tunnel forwarding tool — bypasses ISP deep packet inspection (DPI) for non-standard port TLS traffic.

> Despite the name, this tool works in **any** environment: Claude Code, Hermes Agent, or standalone use. It's just a Python script that creates TCP tunnels over HTTP CONNECT proxy.

---

## Installation

Copy the script to `~/.local/bin/` and ensure that directory is in your `PATH`:

```bash
cp hermes-email-tunnel ~/.local/bin/
chmod +x ~/.local/bin/hermes-email-tunnel
```

### Dependencies

- Python 3.6+
- A local HTTP proxy (e.g., mihomo / Clash, default listening on `127.0.0.1:7890`)

> To change the proxy address, edit the `PROXY` variable in the script.

---

## Use Case

Some ISPs perform deep packet inspection (DPI) on non-standard port TLS traffic and block it. For example, QQ Mail's IMAP/SMTP services use ports 993/465 — in some network environments the TLS handshake gets blocked by the ISP, preventing email clients from connecting.

`hermes-email-tunnel` forwards local traffic through an HTTP CONNECT tunnel via your proxy, bypassing ISP DPI:

```
Local App → localhost:port → HTTP CONNECT Tunnel → Proxy → Target Server
```

Between the proxy and target server, traffic looks like normal HTTP CONNECT requests. The ISP sees nothing unusual and doesn't trigger the non-standard-port TLS detection.

---

## Usage

### Start tunnels

```bash
# Forward local port 11443 → QQ Mail IMAP (imap.qq.com:993)
hermes-email-tunnel 11443 imap.qq.com 993

# Forward local port 10465 → QQ Mail SMTP (smtp.qq.com:465)
hermes-email-tunnel 10465 smtp.qq.com 465
```

Output:

```
Tunnel: 127.0.0.1:11443 → imap.qq.com:993
Tunnel: 127.0.0.1:10465 → smtp.qq.com:465
```

### Configure apps to use localhost

| Service | Original Address | Tunnel Port |
|---------|-----------------|-------------|
| IMAP | `imap.qq.com:993` | `127.0.0.1:11443` |
| SMTP | `smtp.qq.com:465` | `127.0.0.1:10465` |

TLS/SSL stays enabled — the tunnel only forwards encrypted traffic; certificate verification is unaffected.

---

## systemd Service

### 1. Create service file

`/etc/systemd/system/hermes-email-tunnel@.service`:

```ini
[Unit]
Description=Email Tunnel for %i
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

> Replace `YOUR_USER` with your actual username.

### 2. Create instance configs

```bash
sudo mkdir -p /etc/hermes-tunnel
```

`/etc/hermes-tunnel/imap@.conf`:
```
11443 imap.qq.com 993
```

`/etc/hermes-tunnel/smtp@.conf`:
```
10465 smtp.qq.com 465
```

### 3. Wrapper script

`/usr/local/bin/hermes-tunnel-wrapper`:

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

Update the service file's `ExecStart`:
```
ExecStart=/usr/local/bin/hermes-tunnel-wrapper %i
```

### 4. Enable services

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes-email-tunnel@imap
sudo systemctl enable hermes-email-tunnel@smtp
sudo systemctl start hermes-email-tunnel@imap
sudo systemctl start hermes-email-tunnel@smtp
```

### 5. Check status

```bash
sudo systemctl status hermes-email-tunnel@imap
sudo systemctl status hermes-email-tunnel@smtp
```

---

## Related Projects

- [claude-code-pitfalls](https://github.com/2281516753/hermes-pitfalls) — Claude Code on WSL2 pitfalls & solutions
- [wsl-dev-setup](https://github.com/2281516753/wsl-dev-setup) — WSL2 dev environment one-click setup
- [hermes-setup-guide](https://github.com/2281516753/hermes-setup-guide) — Claude Code installation guide

---

## Author

Wang Jiong (王炯) — Network Engineering student.

[GitHub](https://github.com/2281516753)

---

## License

MIT