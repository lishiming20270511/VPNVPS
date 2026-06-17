# VPN 服务运维笔记

## 文档索引

- 客户接入 SOP：customer-onboarding-sop.md
- Shadowrocket 安装教程：shadowrocket-install-guide.md
- 设计文档：../specs/2026-05-30-vpn-service-design.md
- 实施计划：../plans/2026-05-30-vpn-service-mvp.md

## 执行状态

当前状态：已有部分 VPS，可先进入“后台资源池登记 + 节点初始化”阶段；IPRoyal 和 RackNerd 暂不采购。

已完成：

- [x] 运维笔记模板已创建
- [x] 客户接入 SOP 已固化
- [x] iOS Shadowrocket 教程已固化
- [x] 备份目录已创建

待完成：

- [ ] 在后台动态登记已有 VPS 资源
- [ ] 为已登记 VPS 执行初始化、Docker 部署 3x-ui
- [ ] 采购 IPRoyal 美国静态住宅 ISP IP（暂缓）
- [ ] 完成住宅 IP 三件套纯净度测试
- [ ] 配置 SOCKS5 outbound、Reality inbound、Trojan inbound
- [ ] 端到端测试订阅链接与目标平台访问
- [ ] 拉取 3x-ui 数据库和 compose 文件备份

## 资源台账

### 住宅 IP

| 编号 | 供应商 | 地区 | Endpoint | Port | Username | Password | 月费 | 到期日 | 纯净度 | 分配给 |
|---|---|---|---|---|---|---|---|---|---|---|
| RIP-001 | IPRoyal Static ISP | 美国 | 待采购 | 待采购 | 保存到密码管理器 | 保存到密码管理器 | 待填写 | 待填写 | 待测试 | 测试 |

说明：不要把真实密码长期明文保存在本文档。真实凭据建议保存到密码管理器，本文档只记录条目名称。

### VPS 资源池

| VPS ID | 供应商 | 地区 | IP | SSH 端口 | 面板地址 | 状态 | 凭据保存位置 | 到期日 | 分配给 |
|---|---|---|---|---|---|---|---|---|---|
| 待后台生成 | 待填写 | 待填写 | 待填写 | 22 | 待部署 | available | 密码管理器 | 待填写 | 未分配 |

状态枚举：

- `available`：已登记，未分配客户
- `provisioning`：正在初始化或安装 3x-ui
- `ready`：已部署面板，可分配客户
- `assigned`：已绑定客户
- `maintenance`：维护中，不参与自动分配
- `disabled`：停用或不可用

原则：业务代码和客户档案只引用 `VPS ID`，不硬编码某台 VPS 的 IP、端口、面板路径或 SSH 凭据。

### 客户档案

| 编号 | 姓名 | 联系方式 | 套餐 | 业务地区 | 终端数 | VPS 编号 | IP 编号 | 开通日 | 到期日 | 收款 | 订阅 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| C001 | 测试客户 | 内部测试 | 标准测试 | 美国 | 10 | 后台选择 | RIP-001 | 待填写 | 待填写 | 无 | 待生成 | 首位模拟客户 |

## 单台 VPS 部署详情模板

- VPS ID：后台生成
- 3x-ui 面板地址：待部署后填写
- 面板用户名：密码管理器：3xui-<VPS ID>
- 面板密码：密码管理器：3xui-<VPS ID>
- 面板端口：建议 `54321`
- 面板路径：待部署后填写
- 部署时间：待填写

## VPS 到住宅 IP 关联模板

- VPS ID：后台选择
- 关联住宅 IP：RIP-001
- outbound tag：`residential-socks`
- routing：所有 inbound 流量转发到 `residential-socks`，BT 协议转发到 `blocked`
- 配置时间：待填写

## VPS inbounds 模板

| 协议 | 端口 | 凭据保存位置 | 备注 |
|---|---|---|---|
| VLESS+Reality | 443 | 密码管理器：<VPS ID>-reality | 主协议 |
| Trojan+TLS（自签） | 8443 | 密码管理器：<VPS ID>-trojan | 备协议，客户端需允许不安全证书 |

## 端到端验证结果

执行日期：待填写

- Reality 节点：待测试
- Trojan 节点：待测试
- 出口 IP：待测试
- TikTok / X / WhatsApp / Facebook：待测试
- 设备数限制（10）：待测试
- 订阅链接更新：待测试

## 备份计划

- 频率：每周日晚备份所有 VPS 的 `x-ui.db`
- 位置：`runbooks/backups/x-ui-VPS编号-YYYY-MM-DD.db`
- 保留：最近 4 次（4 周）

执行命令模板：

```powershell
scp root@<VPS-IP>:/opt/3x-ui/db/x-ui.db "C:/Users/Administrator/docs/superpowers/runbooks/backups/x-ui-<VPS-ID>-YYYY-MM-DD.db"
scp root@<VPS-IP>:/opt/3x-ui/docker-compose.yml "C:/Users/Administrator/docs/superpowers/runbooks/backups/compose-<VPS-ID>-YYYY-MM-DD.yml"
```

## MVP 验收

- [ ] 至少一台后台登记 VPS 已部署 3x-ui
- [ ] 第一个住宅 IP（RIP-001）已采购、纯净度合格
- [ ] outbound 走住宅 IP，verified
- [ ] Reality + Trojan 双协议 inbound 已建
- [ ] 端到端测试：TikTok / X / WhatsApp / Facebook 全部正常
- [x] 客户接入 SOP 已固化
- [x] iOS Shadowrocket 教程已固化
- [x] 备份与恢复流程已定
- [ ] 3x-ui 数据库已备份到本机
