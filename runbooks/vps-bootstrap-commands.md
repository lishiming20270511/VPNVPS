# VPS 初始化与 3x-ui 部署命令

> 用途：后台登记一台 VPS 后，按顺序初始化。真实 IP、密码、面板路径和 SOCKS5 凭据不要提交到明文文档。

执行前先确定后台里的 `VPS ID`，例如 `vps_us_lax_001`。下面命令里的 `<VPS-ID>` 都替换为该值。

## 1. 首次登录

```bash
ssh root@<VPS-IP>
passwd
```

## 2. 系统更新

```bash
apt update && apt upgrade -y
apt install -y curl wget vim ufw unzip ca-certificates gnupg lsb-release
```

## 3. 开启 BBR

```bash
grep -q '^net.core.default_qdisc=fq' /etc/sysctl.conf || echo 'net.core.default_qdisc=fq' >> /etc/sysctl.conf
grep -q '^net.ipv4.tcp_congestion_control=bbr' /etc/sysctl.conf || echo 'net.ipv4.tcp_congestion_control=bbr' >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.tcp_congestion_control
```

预期输出：

```text
net.ipv4.tcp_congestion_control = bbr
```

## 4. 安装 Docker

```bash
if ! command -v docker >/dev/null 2>&1; then
  curl -fsSL https://get.docker.com | bash
fi
systemctl enable --now docker
docker --version
docker compose version
```

## 5. 创建 3x-ui compose

```bash
mkdir -p /opt/3x-ui/db /opt/3x-ui/cert
cat >/opt/3x-ui/docker-compose.yml <<'YAML'
version: '3.8'
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    hostname: 3x-ui-<VPS-ID>
    volumes:
      - /opt/3x-ui/db/:/etc/x-ui/
      - /opt/3x-ui/cert/:/root/cert/
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      X_UI_ENABLE_FAIL2BAN: "true"
    tty: true
    network_mode: host
    restart: unless-stopped
YAML
cd /opt/3x-ui
docker compose up -d
docker ps
```

## 6. 查看初始面板信息

```bash
docker exec -it 3x-ui /app/x-ui setting -show true
```

记录：

- username
- password
- port
- webBasePath

随后在面板里改：

- Panel Listening Port：`54321`
- Panel Username：保存到密码管理器：`3xui-<VPS-ID>`
- Panel Password：保存到密码管理器：`3xui-<VPS-ID>`
- Panel URL Root Path：保留随机值或改成长随机路径

改完后重启：

```bash
cd /opt/3x-ui
docker compose restart 3x-ui
```

## 7. 配置 UFW

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 54321/tcp
ufw allow 8443/tcp
ufw allow 2096/tcp
ufw --force enable
ufw status
```

确认面板改端口后，如曾放行 `2053`：

```bash
ufw delete allow 2053/tcp
ufw status
```

## 8. SOCKS5 联通测试

```bash
curl --max-time 10 --socks5-hostname '<USERNAME>:<PASSWORD>@<ENDPOINT>:<PORT>' https://ipinfo.io
```

预期返回住宅 IP 的 JSON。

## 9. 备份命令

在本机 PowerShell 执行：

```powershell
scp root@<VPS-IP>:/opt/3x-ui/db/x-ui.db "C:/Users/Administrator/docs/superpowers/runbooks/backups/x-ui-<VPS-ID>-YYYY-MM-DD.db"
scp root@<VPS-IP>:/opt/3x-ui/docker-compose.yml "C:/Users/Administrator/docs/superpowers/runbooks/backups/compose-<VPS-ID>-YYYY-MM-DD.yml"
```
