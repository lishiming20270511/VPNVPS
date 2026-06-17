# VPN 服务 MVP 实施计划（首台 VPS → 首位客户上线）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第一台美区测试 VPS 的部署，配置 Reality + Trojan + SOCKS5 出口，跑通"中国客户终端 → VPS → 住宅 IP → TikTok"全链路，并固化为可重复的 SOP 与运维笔记。

**Architecture:** 单 VPS（RackNerd 美国） + 单住宅 IP（IPRoyal 美国静态 ISP），用 Docker 跑 3x-ui 面板，Xray 做内核，inbound 同时启 VLESS+Reality（主）和 Trojan+TLS（备），outbound 走 SOCKS5 到住宅 IP。

**Tech Stack:** Docker + 3x-ui + Xray-core，VLESS+Reality + Trojan，IPRoyal SOCKS5，Shadowrocket（iOS）/ NekoBox（Android）/ v2rayN（Win）/ ClashX（Mac）。

**参考设计文档：** `C:\Users\Administrator\docs\superpowers\specs\2026-05-30-vpn-service-design.md`

---

## 关键约定

- **每完成一个任务**，把关键参数（IP、端口、密码、UUID 等）记录到运维笔记：`C:\Users\Administrator\docs\superpowers\runbooks\vpn-service-runbook.md`
- **凡涉及付费**：先确认账户余额，再下单
- **凡涉及外部服务下单**：保留订单截图 / 邮件，作为对账凭据
- **本计划"提交"= 把成果写入运维笔记 + 备份配置文件**，不需要 git commit

---

## Task 1：注册 IPRoyal 并购买首个静态住宅 IP

**目标：** 拥有一个可用的美国静态住宅 SOCKS5 代理，准备接入 VPS。

**前置条件：** PayPal 账户已就绪。

**预算：** ~$5–7（首次单 IP 月费）。

- [ ] **Step 1：注册 IPRoyal 账号**

打开浏览器访问：`https://iproyal.com/`

操作：
1. 右上角 Sign Up
2. 邮箱 + 密码注册
3. 邮箱验证激活

预期：登录后能进入 Dashboard。

- [ ] **Step 2：进入 Royal Residential Static ISP 购买页**

在左侧菜单选择：**Residential Proxies → Static ISP / Royal Residential ISP**（不同时期 UI 略有差异，认准"Static" 关键字）。

⚠️ **重要：千万不要选 Residential（流量计费）或 Datacenter，必须是 "Static ISP" 或 "Royal Residential Static"**。

- [ ] **Step 3：选购参数**

配置：
- Country / Location：**United States**
- 数量：**1 IP**
- 时长：**1 month**
- 协议：勾选 **SOCKS5 + HTTPS**（系统通常默认两者都有）

预期价格：约 $5–7。

- [ ] **Step 4：用 PayPal 完成支付**

支付成功后回到 Dashboard，菜单 **Active Products** 应能看到刚买的 Static ISP。

- [ ] **Step 5：获取代理凭据**

在 Active Products 详情页找到：
- Endpoint（如：`x.xxx.iproyal.com`）
- Port（如：`12321` 这种 5 位数字）
- Username
- Password

⚠️ 这四个值是接下来 VPS 配置的核心，**立即记录到运维笔记**。

- [ ] **Step 6：记录到运维笔记（首次创建文件）**

创建 `C:\Users\Administrator\docs\superpowers\runbooks\vpn-service-runbook.md`，内容：

```markdown
# VPN 服务运维笔记

## 资源台账

### 住宅 IP

| 编号 | 供应商 | 地区 | Endpoint | Port | Username | Password | 月费 | 到期日 | 分配给 |
|---|---|---|---|---|---|---|---|---|---|
| RIP-001 | IPRoyal Static ISP | 美国 | x.xxx.iproyal.com | 12321 | xxxx | xxxx | $X | 2026-06-30 | 测试 |

### VPS

（待 Task 3 填）

### 客户档案

（待 Task 9 填）
```

**验证：** 笔记文件存在，凭据已记录。

---

## Task 2：测试住宅 IP 纯净度（三件套）

**目标：** 确保买到的 IP 是干净的住宅 IP，不是被滥用过的脏 IP。

**关键判断：** 任一项不达标 → 联系 IPRoyal 客服换 IP，换完重测。

- [ ] **Step 1：用 curl 通过 SOCKS5 验证 IP 可用**

在本机（Windows）打开 Git Bash 或 PowerShell：

```bash
curl --socks5-hostname USERNAME:PASSWORD@ENDPOINT:PORT https://ipinfo.io
```

把 USERNAME/PASSWORD/ENDPOINT/PORT 替换为 Task 1 拿到的值。

预期输出：JSON 格式，包含 `"ip"`、`"city"`、`"country": "US"`、`"org"` 字段。

⚠️ 如果连接失败：检查凭据是否正确、IPRoyal 后台是否绑定了你的本机 IP（部分套餐需要白名单）。

- [ ] **Step 2：检查 IP 类型必须是 residential**

复制 Step 1 输出中的 IP 地址，在浏览器打开：
```
https://ipinfo.io/<IP地址>
```

预期：
- `Type: ISP` 或 `residential`（✅ 合格）
- 含家庭运营商（Comcast / Spectrum / AT&T / Verizon Fios 等）

不合格：
- `Type: hosting` / `datacenter` ❌ 立刻换 IP
- `org` 是 AWS / Google / DigitalOcean / OVH 等机房 ❌ 立刻换

- [ ] **Step 3：whoer.net 综合匿名分**

浏览器配置 SOCKS5 代理后访问 `https://whoer.net`，或在 IPRoyal 后台用「Test Proxy」功能。

要求：分数 ≥ 80%。

- [ ] **Step 4：scamalytics 风险评分**

打开：`https://scamalytics.com/ip/<IP地址>`

要求：
- Risk: **Low Risk** ✅
- Fraud Score: **< 25** ✅

不合格立刻换 IP。

- [ ] **Step 5：把测试结果记录到运维笔记**

在笔记的 RIP-001 行加一列「纯净度」字段，填类似：
```
whoer 92% / scamalytics Low (15) / ipinfo residential Comcast
```

**验证：** 三件套全部合格，已记录。

---

## Task 3：在 RackNerd 开一台美国测试 VPS

**目标：** 拿到一台干净的美国 VPS，作为首台节点。

**预算：** ~$15/年（折合 $1.25/月）。

⚠️ **不要复用其他业务（如 API 中转）正在占用的 VPS**，必须新开一台，避免混合工作负载。

- [ ] **Step 1：登录 RackNerd 客户后台**

打开 `https://my.racknerd.com/clientarea.php`，用之前注册的账号登入。

- [ ] **Step 2：选购套餐**

进入 https://www.racknerd.com/specials.php，选择：
- 推荐："1 GB KVM VPS" 或类似 1C/1G/15G/1TB 流量套餐
- 机房：**Los Angeles, CA, USA - DC02**（对中国延迟较优）
- 系统：**Ubuntu 22.04 64-bit**
- 计费周期：**Annually** 年付（最划算）

PayPal 支付。

- [ ] **Step 3：等待开通邮件**

邮件主题类似 "RackNerd - New Service Information"，包含：
- IP Address（如 `192.3.x.x`）
- root Password
- SSH Port（默认 22）

⚠️ 立刻把 IP 和 root 密码记录到 runbook 的 VPS 表，并改密码（Step 4）。

- [ ] **Step 4：SSH 首次登录并改 root 密码**

打开 PowerShell 或 Git Bash：

```bash
ssh root@<新VPS的IP>
# 输入邮件里的初始密码
```

登录后立即改密：
```bash
passwd
# 输入两次新密码（用强密码生成器，至少 20 位）
```

- [ ] **Step 5：基础系统更新**

```bash
apt update && apt upgrade -y
apt install -y curl wget vim ufw unzip
```

预期：无报错完成。

- [ ] **Step 6：开启 BBR 加速（提升对中国传输速度）**

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

验证：
```bash
sysctl net.ipv4.tcp_congestion_control
```
预期输出：`net.ipv4.tcp_congestion_control = bbr`

- [ ] **Step 7：配置防火墙（先全开，部署完成后再收紧）**

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 2053/tcp
ufw --force enable
ufw status
```

预期：22、80、443、2053 都是 ALLOW。

> 注：2053 是 3x-ui 默认面板端口，后续会改。

- [ ] **Step 8：记录到运维笔记**

在 runbook 的 VPS 表填入：

```markdown
### VPS

| 编号 | 供应商 | 地区 | IP | SSH 端口 | root 密码（保存到密码管理器） | 月费 | 到期日 | 分配给 |
|---|---|---|---|---|---|---|---|---|
| VPS-001 | RackNerd | 洛杉矶 DC02 | 192.3.x.x | 22 | （密码管理器中：vps-001） | $1.25 | 2027-05-30 | 测试 |
```

⚠️ root 密码不要写到笔记里，写到密码管理器（如 1Password / Bitwarden / 浏览器密码）。

**验证：** SSH 可登录，BBR 已启用，VPS-001 已记录。

---

## Task 4：在 VPS 上用 Docker 部署 3x-ui

**目标：** 跑起 3x-ui 面板，可通过浏览器访问。

**前置：** Task 3 已完成。

- [ ] **Step 1：检查 Docker 是否已装（RackNerd 默认未装）**

```bash
docker --version
```
如果未装：

```bash
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker
docker --version
```
预期：显示 `Docker version 24.x.x` 或更新。

- [ ] **Step 2：创建 3x-ui 工作目录**

```bash
mkdir -p /opt/3x-ui/db /opt/3x-ui/cert
cd /opt/3x-ui
```

- [ ] **Step 3：写 docker-compose.yml**

```bash
vim /opt/3x-ui/docker-compose.yml
```

内容：

```yaml
version: '3.8'
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    hostname: 3x-ui-vps001
    volumes:
      - /opt/3x-ui/db/:/etc/x-ui/
      - /opt/3x-ui/cert/:/root/cert/
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      X_UI_ENABLE_FAIL2BAN: "true"
    tty: true
    network_mode: host
    restart: unless-stopped
```

> 用 `network_mode: host` 是为了让 Xray inbound 直接监听 VPS 端口，无需 Docker 端口映射，性能最优。

- [ ] **Step 4：启动容器**

```bash
cd /opt/3x-ui
docker compose up -d
docker ps
```

预期：`3x-ui` 容器状态 `Up`。

- [ ] **Step 5：查看默认登录凭据**

```bash
docker exec -it 3x-ui /app/x-ui setting -show true
```

预期输出包含：
```
username: admin
password: admin
port: 2053
webBasePath: /xxxxxx (随机字符串)
```

记录这三个值（admin、admin、随机 path）。

- [ ] **Step 6：浏览器登录测试**

浏览器打开：
```
http://<VPS-IP>:2053/<webBasePath>/
```

预期：看到 3x-ui 登录页，用 admin/admin 登入。

- [ ] **Step 7：登录后立即改默认账号密码与端口**

进入 Panel Settings：
- Panel Listening Port：改为 **54321**（或其他 5 位非常用数字）
- Panel URL Root Path：保留随机 path 或改为更长的随机字符串
- Panel Username：改为 `vpnadmin` 或类似
- Panel Password：改为强密码

保存 → 重启面板：
```bash
docker compose restart 3x-ui
```

- [ ] **Step 8：更新防火墙**

```bash
ufw allow 54321/tcp
ufw delete allow 2053/tcp
ufw status
```

- [ ] **Step 9：记录到运维笔记**

runbook 增补章节：

```markdown
## VPS-001 部署详情

- 3x-ui 面板地址：http://192.3.x.x:54321/<webBasePath>/
- 用户名：vpnadmin
- 密码：（密码管理器：3xui-vps001）
- 部署时间：2026-05-30
```

**验证：** 浏览器能用新账号密码登入面板。

---

## Task 5：配置 SOCKS5 outbound 到住宅 IP

**目标：** 让 VPS 上所有出去的代理流量都走住宅 IP。

- [ ] **Step 1：3x-ui 面板 → Xray Configs → Outbounds**

左侧菜单 → Xray Configs → Outbounds（出站）。

- [ ] **Step 2：编辑 outbounds 数组（手工 JSON）**

3x-ui 的 outbounds 默认配置类似：
```json
[
  {"tag": "direct", "protocol": "freedom"},
  {"tag": "blocked", "protocol": "blackhole"}
]
```

把它改为：

```json
[
  {
    "tag": "residential-socks",
    "protocol": "socks",
    "settings": {
      "servers": [
        {
          "address": "x.xxx.iproyal.com",
          "port": 12321,
          "users": [
            {
              "user": "你的IPRoyal_Username",
              "pass": "你的IPRoyal_Password"
            }
          ]
        }
      ]
    },
    "streamSettings": { "network": "tcp" }
  },
  { "tag": "direct", "protocol": "freedom" },
  { "tag": "blocked", "protocol": "blackhole" }
]
```

⚠️ 用 Task 1 拿到的 endpoint/port/username/password 替换。

- [ ] **Step 3：配置路由 Routing → 让代理流量走 residential-socks**

3x-ui 面板 → Xray Configs → Routing。

routing 配置改为：

```json
{
  "domainStrategy": "AsIs",
  "rules": [
    {
      "type": "field",
      "inboundTag": ["api"],
      "outboundTag": "api"
    },
    {
      "type": "field",
      "outboundTag": "blocked",
      "protocol": ["bittorrent"]
    },
    {
      "type": "field",
      "outboundTag": "residential-socks",
      "network": "tcp,udp"
    }
  ]
}
```

> 含义：BT 流量直接屏蔽，其余所有 TCP/UDP 流量统一走住宅 IP。

- [ ] **Step 4：保存并重启 Xray**

面板顶部点击「Save Settings」→「Restart Xray Core」。

预期：状态变绿，无报错。

- [ ] **Step 5：在 VPS 上验证 SOCKS5 联通（旁路测）**

在 VPS 终端直接用 curl 测 SOCKS5（不经 Xray）：

```bash
curl --max-time 10 --socks5-hostname USERNAME:PASSWORD@x.xxx.iproyal.com:12321 https://ipinfo.io
```

把 USERNAME/PASSWORD/endpoint/port 替换成 Task 1 的值。

预期：返回 JSON，IP 与 Task 1 测的住宅 IP 一致。

> 注：此步只测「VPS → 住宅 SOCKS5」联通性。Xray 配置是否生效要到 Task 8 端到端验证才能确认。

- [ ] **Step 6：记录到运维笔记**

```markdown
## VPS-001 → 住宅 IP 关联

- 关联住宅 IP：RIP-001（IPRoyal x.xxx.iproyal.com:12321）
- outbound tag：residential-socks
- routing：所有 inbound 流量 → residential-socks
- 配置时间：2026-05-30
```

**验证：** outbound 配置已保存，Xray 重启无报错，VPS 内验证可经 SOCKS5 出网。

---

## Task 6：在 3x-ui 创建 VLESS+Reality inbound（主协议）

**目标：** 客户端用 Reality 协议接入 VPS。

- [ ] **Step 1：3x-ui → Inbounds → Add Inbound**

填入：
- Remark：`vps001-reality`
- Protocol：**VLESS**
- Listening IP：留空（监听全部）
- Port：**443**（直接占用 443 让流量像 HTTPS）

> 如果 Caddy / 其他服务占用 443，可改 8443，但 Reality 配 443 抗封性最好。

- [ ] **Step 2：Transmission 配置选 Reality**

- Transmission：**TCP**
- Security：**Reality**

Reality 参数（3x-ui 通常自动生成，没有的字段手动补）：
- uTLS：**chrome**
- Dest：**www.microsoft.com:443**
- Server Names：**www.microsoft.com,microsoft.com**
- Short Ids：点 ⟳ 按钮自动生成 1 个 8 字符 hex
- Private Key + Public Key：点 ⟳ 按钮自动生成
- SpiderX：留空

> Reality 把流量伪装成访问 microsoft.com 的 TLS 握手，GFW 难以识别。

- [ ] **Step 3：Clients 添加首个测试客户端**

Clients 区点 + 加一个：
- Email / Remark：`test-self`（自己测试用）
- ID：点 ⟳ 自动生成 UUID
- Flow：**xtls-rprx-vision**（Reality 标配）
- Limit IP：**1**（测试时用 1，正式客户填 10）
- Total GB：**0**（不限）
- Expire Date：**留空**（不限）
- Subscription：留空

保存。

- [ ] **Step 4：导出客户端配置链接**

刚创建的 inbound 行点 **Operate → QR Code**，复制链接（形如 `vless://uuid@ip:443?...&type=tcp&security=reality&...#vps001-reality-test-self`）。

- [ ] **Step 5：本机用客户端测试**

选一个本机客户端（推荐 v2rayN Windows 或 NekoBox 跨平台）：
1. 复制 vless:// 链接到客户端剪贴板导入
2. 设为活动节点
3. 启用系统代理
4. 浏览器访问 `https://ipinfo.io`

⚠️ 本机要先能访问境外站点（即你本机已有梯子），不然测不了。如果你本机此刻在境外网络（如使用旧节点），Reality 节点理论上能通。

预期：
- `ip` 是 Task 1 的住宅 IP
- `org` 是住宅 ISP（Comcast 等）
- 浏览器能访问 youtube.com / tiktok.com

- [ ] **Step 6：记录**

```markdown
## VPS-001 inbounds

| 协议 | 端口 | UUID/密码 | 备注 |
|---|---|---|---|
| VLESS+Reality | 443 | （记录到密码管理器） | 主协议 |
```

**验证：** 本机经 Reality 节点访问 ipinfo 能看到住宅 IP，TikTok 可访问。

---

## Task 7：在 3x-ui 创建 Trojan inbound（备协议）

**目标：** 多一条备路。Reality 万一被识别时客户端可切 Trojan。

- [ ] **Step 1：Inbounds → Add Inbound**

- Remark：`vps001-trojan`
- Protocol：**Trojan**
- Port：**8443**
- Security：**TLS**

- [ ] **Step 2：TLS 配置（自签证书简化版）**

- Domain：留空（用 IP）
- Certificate File / Key File：点 **Get New Cert** 自签

> 自签证书客户端会提示不可信，需在客户端设置 `allowInsecure: true`。生产场景建议买便宜域名 + Caddy 申 Let's Encrypt，但 MVP 先自签即可。

- [ ] **Step 3：Clients 添加测试客户端**

- Email：`test-self-trojan`
- Password：点 ⟳ 自动生成
- Limit IP：**1**

保存。

- [ ] **Step 4：导出 trojan:// 链接，本机测试**

同 Task 6 Step 5。客户端导入时记得勾 `allowInsecure / 跳过证书验证`。

预期：能连，出口 IP 是住宅 IP。

- [ ] **Step 5：UFW 放行 8443**

VPS 上：
```bash
ufw allow 8443/tcp
ufw status
```

- [ ] **Step 6：记录**

```markdown
| Trojan+TLS（自签） | 8443 | （密码管理器） | 备协议 |
```

**验证：** Trojan 节点本机测试可连可访问境外。

---

## Task 8：端到端验证（模拟首位真实客户）

**目标：** 在一个干净环境（最好真用一台设备），用订阅链接的方式跑通全流程，相当于客户实际体验。

- [ ] **Step 1：在 3x-ui 为「首位测试客户」创建独立账号**

⚠️ 先把 Task 6/7 中的 `test-self` / `test-self-trojan` 删除（避免 IP 互踩）。

在 Reality inbound 和 Trojan inbound 各 Add 一个 client：
- Email / Remark：**Customer-001**
- Limit IP：**10**（生产值）
- Expire Date：**3 个月后**（如 2026-08-30）
- Subscription：**勾选 Enable Subscription**，Subscription ID 用客户编号（如 `customer001`）

> 3x-ui 会自动维护一条订阅 URL：`http://VPS-IP:54321/sub/customer001`（路径由 panel 设置决定）。

- [ ] **Step 2：在 3x-ui Panel Settings → Subscription Settings 启用订阅服务**

- Enable Subscription：**ON**
- Listening Port：**2096**（或其他自由端口）
- Domain：留空（用 IP）
- Path：`/sub/`

保存重启。

- [ ] **Step 3：UFW 放行订阅端口**

```bash
ufw allow 2096/tcp
ufw status
```

- [ ] **Step 4：拿到订阅 URL**

3x-ui 面板对应 client 行点 **Sub** 按钮 → 复制链接。

形如：`http://192.3.x.x:2096/sub/customer001`。

- [ ] **Step 5：在一台设备上导入订阅**

推荐用一台**干净的 Windows 测试**：
1. 装 v2rayN（https://github.com/2dust/v2rayN/releases）
2. 服务器 → 添加[订阅]服务器 → 粘贴上面的 URL
3. 订阅 → 更新订阅
4. 应该看到 2 个节点：`vps001-reality-Customer-001` 和 `vps001-trojan-Customer-001`
5. 选 Reality 节点，启用系统代理

- [ ] **Step 6：验证基础访问**

浏览器打开 `https://ipinfo.io`：
- IP 应等于 Task 1 的住宅 IP
- org 应是 Comcast 等住宅 ISP

- [ ] **Step 7：验证目标平台访问**

浏览器依次打开：
- https://www.tiktok.com（视频流畅）
- https://x.com（Twitter）
- https://web.whatsapp.com
- https://www.facebook.com

每个站点要求：
- 页面加载不超过 3 秒
- 视频/图片正常加载
- 无 captcha 异常拦截

- [ ] **Step 8：切换到 Trojan 节点重测**

在 v2rayN 选 Trojan 节点，重复 Step 6-7。

- [ ] **Step 9：限速 / 设备数验证**

打开第 2、第 3、第 4... 一直到第 11 个客户端连接（用同一订阅在多台设备 / 多浏览器配置文件里）。

预期：第 11 台开始连不上（IP Limit = 10 生效）。

如果限速不生效：检查 3x-ui client 的 Limit IP 字段是否真的填了 10。

- [ ] **Step 10：到期验证（可选，跳过亦可）**

把 Customer-001 的 Expire Date 改为「1 分钟后」，观察客户端 1 分钟后是否断流。

测完改回正常到期日。

- [ ] **Step 11：把测试结果写到 runbook**

```markdown
## 端到端验证结果（2026-05-30）

- Reality 节点：✅ 可连，出口 IP 一致
- Trojan 节点：✅ 可连，出口 IP 一致
- TikTok / X / WhatsApp / FB：✅ 全部正常访问
- 设备数限制（10）：✅ 生效
- 订阅链接更新：✅ 正常
```

**验证：** 全部 ✅，否则回到对应 Task 排查。

---

## Task 9：固化客户接入 SOP（可重复操作清单）

**目标：** 把第一个客户的开通流程整理成一份"傻瓜式"清单，下次接客户照着抄。

- [ ] **Step 1：创建 SOP 文档**

新建：`C:\Users\Administrator\docs\superpowers\runbooks\customer-onboarding-sop.md`

- [ ] **Step 2：写入 SOP 内容**

```markdown
# 客户接入 SOP

> 每接 1 个新客户从头到尾走完，预计 30–45 分钟。

## 步骤总览

| # | 操作 | 耗时 |
|---|---|---|
| 1 | 接单确认（业务地区/终端数/订阅时长/收款） | 5 分钟 |
| 2 | 收款到账 | 视客户 |
| 3 | 在对应地区开 VPS 或复用闲置 VPS | 5 分钟 |
| 4 | 在 IPRoyal 买对应地区静态 ISP IP | 10 分钟 |
| 5 | 三件套测纯净度 | 5 分钟 |
| 6 | 在 VPS 的 3x-ui 配 SOCKS5 outbound + 客户 inbound client | 10 分钟 |
| 7 | 复制订阅链接 | 1 分钟 |
| 8 | TeamViewer/AnyDesk 远程帮客户装客户端、导入订阅、实测 | 15-20 分钟 |
| 9 | 客户档案登记到运维笔记 | 5 分钟 |

## 详细步骤

### 步骤 3：开 VPS

...（沿用本计划 Task 3 全部步骤）...

### 步骤 4：买住宅 IP

...（沿用本计划 Task 1 + Task 2）...

### 步骤 6：3x-ui 配置

3x-ui 已部署的 VPS：
- Inbounds → 在已有 Reality inbound 加 client（不需要新建 inbound）
- Email / Remark：`Customer-XXX`
- Limit IP：客户的终端数
- Expire Date：（订阅时长 + 今日）
- Subscription：启用，ID 填 `customerXXX`

### 步骤 8：远程装机模板话术

> 您好 X 总，我们开始装机：
> 1. 请打开 TeamViewer / AnyDesk，发我 ID 与密码
> 2. iPhone 用户：请准备一个海外 Apple ID（教程：xxx）
> 3. 装机过程约 15 分钟
> ...

### 步骤 9：客户档案模板

| 编号 | 姓名 | 联系方式 | 套餐 | 业务地区 | 终端数 | VPS 编号 | IP 编号 | 开通日 | 到期日 | 收款 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| C001 | XXX | wechat:xxx | 月付标准 | 美国 | 10 | VPS-001 | RIP-001 | 2026-05-30 | 2026-08-30 | ¥180/支付宝 |  |

## 客户提醒话术

- 续费前 7 天主动联系
- 出现连不上时第一招：在 v2rayN 重新「更新订阅」+ 切换协议（Reality → Trojan）
- 严禁用于违规用途，否则保留切断权
```

- [ ] **Step 3：再写一份 iOS Shadowrocket 安装教程**

新建：`C:\Users\Administrator\docs\superpowers\runbooks\shadowrocket-install-guide.md`

```markdown
# Shadowrocket（小火箭）购买与安装教程

## 准备工作

- iPhone / iPad
- 一个**海外（美区/日区/港区）Apple ID**
  - 没有：参考 https://www.applechinarefund.com/ 或类似教程注册免信用卡的美区 ID
- App Store 礼品卡（如需，10 美元面值即可）

## 步骤 1：切换 App Store 到海外区

1. 设置 → 顶部 Apple ID → 媒体与购买项目 → 退出登录
2. App Store → 顶部头像 → 退出登录
3. App Store → 顶部头像 → 用海外 Apple ID 登录

## 步骤 2：购买 Shadowrocket

1. App Store 搜索 `Shadowrocket`
2. 点购买（约 $2.99）
3. 用礼品卡或绑定的海外支付方式付款

## 步骤 3：导入订阅

1. 打开 Shadowrocket
2. 右上角 + 号 → 类型选「Subscribe」
3. 粘贴订阅链接 → 完成
4. 回到首页应看到 2 个节点（Reality + Trojan）
5. 点节点右侧 ⓘ 测试延迟，选延迟低的
6. 顶部开关启用代理

## 步骤 4：日常使用

- 看到连不上时：右上角 ⟳ 重新订阅 → 换 Trojan 节点

## 常见问题

- Q：装不上 Shadowrocket
  - A：确认 App Store 已切到海外区（顶部头像 → 国家/地区）

- Q：订阅链接打不开
  - A：联系客服重新生成
```

- [ ] **Step 4：把 SOP 与教程的位置加到 runbook 索引**

在 `vpn-service-runbook.md` 顶部加：

```markdown
## 文档索引

- 客户接入 SOP：customer-onboarding-sop.md
- Shadowrocket 安装教程：shadowrocket-install-guide.md
- 设计文档：../specs/2026-05-30-vpn-service-design.md
- 实施计划：../plans/2026-05-30-vpn-service-mvp.md
```

**验证：** 三个文档可被相互找到，内容齐全。

---

## Task 10：备份与收尾

**目标：** 保证 VPS 挂掉时能 30 分钟内恢复。

- [ ] **Step 1：备份 3x-ui 数据库到本机**

VPS 上：
```bash
docker exec 3x-ui /app/x-ui setting -reset 0  # 不要执行，仅查看支持的命令
ls -la /opt/3x-ui/db/
```

预期：能看到 `x-ui.db` 文件。

在本机执行（用 scp 拉回）：
```bash
scp root@<VPS-IP>:/opt/3x-ui/db/x-ui.db "C:/Users/Administrator/docs/superpowers/runbooks/backups/x-ui-vps001-2026-05-30.db"
```

- [ ] **Step 2：备份 docker-compose.yml**

```bash
scp root@<VPS-IP>:/opt/3x-ui/docker-compose.yml "C:/Users/Administrator/docs/superpowers/runbooks/backups/compose-vps001-2026-05-30.yml"
```

- [ ] **Step 3：定下定期备份计划**

在 runbook 加：

```markdown
## 备份计划

- 频率：每周日晚备份所有 VPS 的 x-ui.db
- 位置：runbooks/backups/x-ui-VPS编号-日期.db
- 保留：最近 4 次（4 周）

执行命令模板：
scp root@<VPS-IP>:/opt/3x-ui/db/x-ui.db "C:/Users/Administrator/docs/superpowers/runbooks/backups/x-ui-VPS编号-YYYY-MM-DD.db"
```

- [ ] **Step 4：故障演练（可选，建议做）**

在测试环境执行一次「VPS 挂了怎么恢复」：
1. 新开一台 VPS（重复 Task 3 / 4）
2. 把 `x-ui.db` 通过 scp 上传到新 VPS 的 `/opt/3x-ui/db/`
3. `docker compose up -d`
4. 登录看是否客户配置都还在

预期：是。这意味着即使 VPS 被墙也能 30 分钟内恢复。

- [ ] **Step 5：最终验收清单**

在 runbook 末尾加：

```markdown
## MVP 验收（2026-05-30）

- [x] 第一台 VPS（VPS-001）已部署 3x-ui
- [x] 第一个住宅 IP（RIP-001）已采购、纯净度合格
- [x] outbound 走住宅 IP，verified
- [x] Reality + Trojan 双协议 inbound 已建
- [x] 端到端测试：TikTok / X / WhatsApp / FB 全部正常
- [x] 客户接入 SOP 已固化
- [x] iOS Shadowrocket 教程已固化
- [x] 备份与恢复流程已定
```

**验证：** 上述全部打勾，可以接首个真实客户。

---

## 实施完成判定

执行完 Task 1–10，下列条件全部满足才算 MVP 完成：

1. ✅ 一台已配置好的美国 VPS，3x-ui 可正常登入并管理
2. ✅ 一个干净的美国住宅 IP 已绑入 outbound
3. ✅ 一条订阅链接可在 v2rayN / Shadowrocket 客户端导入
4. ✅ 通过订阅访问 TikTok / X / WhatsApp / FB 全部正常
5. ✅ 出口 IP 验证为住宅 IP
6. ✅ 设备数限制（10 台）已实测生效
7. ✅ 运维笔记 + 客户接入 SOP + iOS 教程三个文档齐全
8. ✅ 数据库备份到本机一份

---

## 下一步规划（不属于本计划，仅作参考）

- 接到首位付费客户，按 SOP 走通
- 客户达到 5 个 → 写自动化部署脚本（一键开 VPS + 装 3x-ui）
- 客户达到 15 个 → 评估迁移到 Xboard 总控
- 香港 / 日本 / 新加坡节点按订单需要逐个补
