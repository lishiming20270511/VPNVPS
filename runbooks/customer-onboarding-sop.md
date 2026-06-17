# 客户接入 SOP

> 每接 1 个新客户从头到尾走完，预计 30-45 分钟。涉及付款和外部账号登录的动作，先确认余额、订单金额和客户需求。

## 步骤总览

| # | 操作 | 耗时 |
|---|---|---|
| 1 | 接单确认：业务地区、终端数、订阅时长、收款金额 | 5 分钟 |
| 2 | 收款到账确认 | 视客户 |
| 3 | 在后台资源池选择可用 VPS，或登记新 VPS | 5 分钟 |
| 4 | 购买对应地区静态住宅 ISP IP | 10 分钟 |
| 5 | 三件套测纯净度：ipinfo、whoer、scamalytics | 5 分钟 |
| 6 | 在 VPS 的 3x-ui 配 SOCKS5 outbound 和客户 inbound client | 10 分钟 |
| 7 | 复制订阅链接 | 1 分钟 |
| 8 | TeamViewer / AnyDesk 远程帮客户装客户端、导入订阅、实测 | 15-20 分钟 |
| 9 | 客户档案登记到运维笔记 | 5 分钟 |

## 步骤 1：接单确认

确认以下信息：

- 客户姓名或内部备注
- 联系方式
- 目标业务地区：美国、日本、新加坡、香港等
- 终端数，默认不超过 10
- 订阅时长：月付、季付、年付
- 套餐：标准版或尊享版
- 收款金额和方式

## 步骤 2：收款确认

收款到账后再采购外部资源。保留订单截图、支付记录或邮件，方便后续对账。

## 步骤 3：选择或登记 VPS

优先从后台资源池选择状态为 `ready` 或 `available`、地区匹配的 VPS。不要在代码、配置模板或客户档案里直接写死某台 VPS。

后台登记字段：

- VPS ID：系统自动生成，例如 `vps_us_lax_001`
- 供应商：RackNerd、Vultr、BandwagonHost 等
- 地区：国家、城市、机房
- IP
- SSH 端口
- 系统版本
- 到期日
- 成本
- 状态：`available` / `provisioning` / `ready` / `assigned` / `maintenance` / `disabled`
- 凭据引用：密码管理器条目名，不保存明文密码

如果需要新开标准美国节点，可参考：

- 供应商：RackNerd
- 机房：Los Angeles, CA, USA - DC02
- 系统：Ubuntu 22.04 64-bit
- 配置：1C / 1G / 15G / 1TB 或相近配置
- 周期：年付优先

开通或发现已有 VPS 后，立刻登记到后台资源池：

- VPS ID
- IP
- SSH 端口
- 到期日
- 订单金额

root 初始密码必须尽快修改，并保存到密码管理器。不要写入明文运维笔记。

## 步骤 4：购买住宅 IP

优先购买静态住宅 ISP，不买流量型住宅代理，不买 datacenter 代理。

IPRoyal 参数：

- 产品：Static ISP / Royal Residential Static
- Country / Location：与 VPS 同地区
- Quantity：1 IP
- Duration：1 month
- Protocol：SOCKS5，HTTPS 可同时保留

采购后记录：

- Endpoint
- Port
- Username
- Password
- 到期日
- 月费

真实密码保存到密码管理器。

## 步骤 5：纯净度测试

先用 SOCKS5 验证联通：

```bash
curl --socks5-hostname USERNAME:PASSWORD@ENDPOINT:PORT https://ipinfo.io
```

检查项：

- ipinfo：类型应是 ISP / residential，不能是 hosting / datacenter
- whoer：综合匿名分 >= 80%
- scamalytics：Low Risk，Fraud Score < 25

任一项不达标，联系供应商换 IP，换完重新测试。

## 步骤 6：3x-ui 配置

后台已选中的 VPS：

- 如果状态是 `available`：先执行初始化并安装 3x-ui，完成后改为 `ready`
- 如果状态是 `ready`：直接配置住宅 IP outbound 与客户 client
- 如果状态是 `assigned`：除非是同一客户续费/维护，否则不要复用

- Xray Configs -> Outbounds：添加 `residential-socks`
- Xray Configs -> Routing：将所有 inbound 流量转到 `residential-socks`
- Inbounds -> Reality：添加客户 client
- Inbounds -> Trojan：添加客户 client

客户 client 标准配置：

- Email / Remark：`Customer-XXX`
- Limit IP：客户终端数，默认 `10`
- Total GB：`0`，表示不限
- Expire Date：按订阅周期填写
- Subscription：启用，ID 填 `customerXXX`

## 步骤 7：复制订阅链接

在 3x-ui 对应 client 行点击 Sub，复制订阅链接。

记录到客户档案：

- 订阅 URL
- 开通日
- 到期日
- VPS ID
- 住宅 IP 编号

## 步骤 8：远程装机话术

```text
您好 X 总，我们开始装机：
1. 请打开 TeamViewer 或 AnyDesk，发我 ID 与密码。
2. 如果是 iPhone，请准备一个海外 Apple ID；没有的话按我发您的教程操作。
3. 装机过程大约 15 分钟。
4. 装好后我会现场测试 TikTok、X、WhatsApp、Facebook。
```

客户端建议：

- iOS：Shadowrocket
- Android：NekoBox 或 v2rayNG
- Windows：v2rayN
- macOS：ClashX Pro / Stash / NekoRay

## 步骤 9：客户档案模板

| 编号 | 姓名 | 联系方式 | 套餐 | 业务地区 | 终端数 | VPS 编号 | IP 编号 | 开通日 | 到期日 | 收款 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| C001 | XXX | wechat:xxx | 月付标准 | 美国 | 10 | 后台选择的 VPS ID | RIP-001 | 2026-05-30 | 2026-08-30 | ¥180/支付宝 |  |

## 客户提醒话术

- 续费前 7 天主动联系客户。
- 出现连不上时，先让客户更新订阅，再切换协议：Reality -> Trojan。
- 严禁用于违法违规用途，否则保留切断服务的权利。
