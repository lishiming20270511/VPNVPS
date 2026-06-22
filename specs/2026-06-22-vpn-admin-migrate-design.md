# vpn-admin-mvp 迁移设计：阿里云 → 海外

**日期**：2026-06-22
**目标**：把 `vpn-admin-mvp` 管理后台从阿里云（123.57.133.209）迁到海外（199.193.126.80），让阿里云专注跑副项目 `upwork-hunter`。

## 背景

### 现状（实际探查 2026-06-22）

- **阿里云 123.57.133.209**：跑 `vpn-admin-mvp`（Node.js :8787 + nginx :80）+ `/root/upwork-hunter/`（Python 接单监控）。Node 进程由 `vpn-admin.service` systemd 拉起，root 身份执行；数据在 `data/db.json`（无数据库）。
- **海外 199.193.126.80**：跑 `xray.service` :8443（给本机爬虫做出站代理）+ python3 爬虫 :8080 + redis :6379。**未装 Node、nginx**，iptables INPUT 全 ACCEPT，:80 / :8787 空闲。1 GiB 内存，已用 ~426 MiB。

### 动机

1. 让阿里云那台机器留给 `upwork-hunter` 独占。
2. 降低运维复杂度（VPN 业务集中到一台）。

### 已知顾虑（已接受）

- 海外 IP 从国内访问 ~162 ms（vs 阿里云 28 ms），admin UI 操作有可感卡顿。
- 海外那台再加 admin 后，"VPN 节点 + 爬虫 + admin" 三合一，单点故障域更大。
- 海外仅 1 GiB 内存，加 nginx + Node 后剩余 ~330 MiB。本次迁移后**禁止再往这台机器加新服务**。

## 总体方案

**镜像迁移、阿里云保留 30 天可回滚**：

- 海外 apt 装 `nodejs` + `npm` + `nginx`
- `/opt/vpn-admin-mvp/` 完整复制（含 `db.json` 和 `.env`）
- `systemd` unit + nginx vhost 沿用阿里云原文件
- 验证通过后阿里云 service 仅 `disable + stop`（不删除）
- 30 天后回头清理

**不改代码、不改密码、不改端口、不改文件路径** —— 纯位置搬家。这是回滚承诺。

## §1 — 服务拓扑（迁移后）

### 阿里云 123.57.133.209（瘦身后）

- 只跑：`upwork-hunter`（Python，无 systemd 依赖、无外部 web 服务）
- 保留但停用：`/opt/vpn-admin-mvp/`、`vpn-admin.service`、`/etc/nginx/sites-available/vpn-admin`
- 可选：`systemctl stop nginx && systemctl disable nginx`（upwork-hunter 用不到，省 ~30 MiB 内存）

### 海外 199.193.126.80（增加 admin）

- 原有：`xray.service` :8443 + python3 爬虫 :8080 + redis :6379 + sshd :22
- 新增：
  - `apt install -y nodejs npm nginx`
  - `useradd -r -m -d /home/vpnadmin -s /bin/bash vpnadmin`（仅文件归属用；service 仍以 root 身份跑，与阿里云原状一致）
  - `/opt/vpn-admin-mvp/`（rsync 自阿里云）
  - `/etc/systemd/system/vpn-admin.service`（沿用阿里云原 unit）
  - `/etc/nginx/sites-enabled/vpn-admin`（沿用阿里云原 vhost）
- 对外入口：`http://199.193.126.80/` → nginx :80 → Node :8787

## §2 — 数据迁移与切换时序

### 待迁移文件清单

| 路径 | 性质 | 处理 |
|---|---|---|
| `/opt/vpn-admin-mvp/server.js` | 代码 | 复制 |
| `/opt/vpn-admin-mvp/.env` | 配置（含 `ADMIN_PASSWORD`） | 复制 |
| `/opt/vpn-admin-mvp/data/db.json` | **唯一有状态文件** | 时机敏感：先复制 baseline，T+7 再 rsync 一次 |
| `/opt/vpn-admin-mvp/public/` | 静态前端 | 复制 |
| `/opt/vpn-admin-mvp/scripts/` `deploy/` `README.md` `CONTINUE.md` | 文档/工具 | 复制 |
| `/etc/systemd/system/vpn-admin.service` | unit | 复制（路径完全一致，开箱即用） |
| `/etc/nginx/sites-available/vpn-admin` + `sites-enabled` symlink | nginx vhost | 复制 |
| `server.out.log` `server.err.log` | 运行日志 | 不复制，海外重新生成 |
| `node_modules/` | 不存在（server.js 仅用 Node 内置模块） | 跳过 |

### 切换时序（停机窗口 ≈ 2 分钟）

```
 T+0   [海外]  apt install -y nodejs npm nginx
 T+1   [海外]  useradd -r -m -d /home/vpnadmin -s /bin/bash vpnadmin
 T+2   [阿里云→本地→海外] tar 包传代码（含旧 db.json 作 baseline）
 T+3   [海外]  解包到 /opt/vpn-admin-mvp/，写 systemd unit + nginx vhost
              nginx -t / node --check 验证语法但不启动
 T+4   [海外]  确认云防火墙 :80 放行（小厂 VPS 通常无 security group，但要实测）
 T+5   [海外]  手动 node server.js 起一次，curl localhost:8787 验证启动 + 读 db.json，然后 Ctrl+C
 ────── 以上 admin 仍跑在阿里云，业务数据可正常修改 ──────

 T+6   [阿里云] systemctl stop vpn-admin   ⏱ 停机窗口开始
 T+7   [阿里云→本地→海外] 拷一次 db.json 覆盖到海外（与 T+2 同条传输路径）
 T+8   [海外]  systemctl daemon-reload && systemctl enable --now vpn-admin
 T+9   [海外]  systemctl reload nginx
 T+10  [本地]  curl + 浏览器登录验证（清单见下）
 ────── 验证通过 ⏱ 停机窗口结束（约 2 分钟） ──────

 T+11  [阿里云] systemctl disable vpn-admin（不删文件）
 T+12  [阿里云] 摘 nginx vhost symlink，nginx -s reload；可选 stop+disable nginx
```

### 验证清单（T+10 必跑）

- [ ] `curl -u admin:<密码> http://199.193.126.80/` 返回 200 + 正常 HTML
- [ ] 浏览器登录后能看到 vpsNodes / residentialIps / customers 列表（db.json 读得通）
- [ ] 在 admin 里改一条客户备注，刷新后改动持久（写得通）
- [ ] `journalctl -u vpn-admin --since="5 min ago"` 无 ERROR

### 回滚

任一验证项失败：

```bash
ssh root@199.193.126.80 "systemctl stop vpn-admin"
ssh root@123.57.133.209 "systemctl start vpn-admin && systemctl restart nginx"
```

最坏丢失迁移窗口内的几秒改动；阿里云的 `db.json` 在 stop 那一刻是干净 baseline，回滚后状态一致。

## §3 — 安全与监控（最小改动）

**不引入新组件**（不上 fail2ban、不上 Tailscale），仅 2 项稳妥调整：

1. nginx vhost 加 `access_log /var/log/nginx/vpn-admin.access.log;`，方便日后查是不是被扫
2. 沿用现有 `ADMIN_PASSWORD`（24 字符随机串，已经够强）

**不改 `server.js`**：当前监听 `*:8787`，公网可绕 nginx 直连 :8787。改为 `127.0.0.1:8787` 是 5 字节加固，但破坏 "行为完全不变" 的回滚承诺，**不在本次迁移范围**，留作单独加固项。

## §4 — 阿里云侧善后

### T+12（迁移当天）

- `systemctl disable vpn-admin`（不删文件、不卸载 node）
- 摘 `sites-enabled/vpn-admin` symlink，`nginx -s reload`
- 可选：`systemctl stop nginx && systemctl disable nginx`

### T+30 天（确认无回滚需求后）

```bash
rm -rf /opt/vpn-admin-mvp
rm /etc/systemd/system/vpn-admin.service
rm /etc/nginx/sites-available/vpn-admin
apt purge -y nginx nodejs npm   # 可选，看你想多干净
userdel -r vpnadmin              # 如果存在
```

T+30 步**不进入本次 implementation plan 第一阶段**，避免 30 天后忘记，再翻本设计文档可找回。

## 风险与未决

- **海外云防火墙是否放行 :80**：T+4 必须实测；如果阻拦，需登录 VPS 商家面板放行
- **海外 IP 被国内运营商封**：用户自有 VPN 业务通道，可绕；不阻塞迁移
- **数据丢失**：仅限切换瞬间几秒；`db.json` 单一来源 + 最后一次 rsync 覆盖即可

## 参考

- 资源池后台原始设计：[../runbooks/vps-resource-pool-admin-design.md](../runbooks/vps-resource-pool-admin-design.md)
- VPN 服务总体设计：[2026-05-30-vpn-service-design.md](./2026-05-30-vpn-service-design.md)
- 服务器凭据：本地 `Desktop/PSD.txt`（不入库）
