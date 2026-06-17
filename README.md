# VPNVPS — VPN 订阅服务运维资料库

私有运维仓库。面向"1 客户 = 1 VPS + 1 住宅 IP"业务模式的全套设计、计划、SOP 与部署模板。

## 仓库结构

```
.
├── specs/        业务与技术设计
│   └── 2026-05-30-vpn-service-design.md
├── plans/        分阶段实施计划（带 checkbox）
│   └── 2026-05-30-vpn-service-mvp.md
└── runbooks/     一线运维 SOP 与部署模板
    ├── vpn-service-runbook.md            主运维笔记 + 资源台账模板
    ├── customer-onboarding-sop.md        客户接入标准流程
    ├── shadowrocket-install-guide.md     iOS 客户端安装教程
    ├── vps-bootstrap-commands.md         VPS 初始化 + 3x-ui 部署命令
    ├── vps-resource-pool-admin-design.md VPS 资源池后台数据模型
    └── backups/                          x-ui.db 等真实备份（已 .gitignore 排除）
```

## 在新 VPS 上快速部署

```bash
# 1. 在本机 / 跳板机克隆
git clone git@github.com:lishiming20270511/VPNVPS.git
cd VPNVPS

# 2. 按 runbooks/vps-bootstrap-commands.md 顺序操作
#    需要把命令里的 <VPS-ID> 替换成真实 VPS 编号
cat runbooks/vps-bootstrap-commands.md
```

完整流程对照 `plans/2026-05-30-vpn-service-mvp.md` 的 Task 3 → Task 10 逐步执行。

## 安全基线

- ❌ 真实凭据（IPRoyal 用户名/密码、面板密码、客户 UUID、SOCKS5 凭据）**永远**不提交到仓库
- ❌ `runbooks/backups/` 下的 `*.db` `*.yml` 真实备份**永远**不提交（已在 `.gitignore` 排除）
- ✅ 所有敏感凭据保存到密码管理器，文档里只写条目名称（如 `3xui-vps_us_lax_001`）
- ✅ 提交前自检：`git diff --staged | grep -Ei 'password|secret|uuid|[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'`

## 关联文档（不在本仓库）

- 设计参考：`C:\Users\Administrator\docs\superpowers\specs\` 已并入本仓库
- 客户档案 / 真实凭据：本机密码管理器，不入库

## License

私有仓库，未授权情况下不得对外分发。
