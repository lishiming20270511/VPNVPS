# VPS 资源池后台设计

## 目标

VPS 必须通过后台动态添加、编辑、停用和分配。业务逻辑只引用 `vps_id`，不能把 VPS IP、面板端口、SSH 端口、供应商等写死在代码里。

## 核心流程

1. 管理员在后台新增 VPS。
2. 系统保存 VPS 元数据和凭据引用。
3. 运维对该 VPS 执行初始化，安装 3x-ui。
4. 初始化完成后回填面板地址、面板路径、面板端口。
5. 将状态从 `provisioning` 改为 `ready`。
6. 新客户开通时，后台按地区筛选 `ready` VPS。
7. 客户绑定成功后，VPS 状态改为 `assigned`。
8. VPS 故障或到期时，状态改为 `maintenance` 或 `disabled`。

## 数据模型

### vps_nodes

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| id | string | 是 | VPS ID，例如 `vps_us_lax_001` |
| provider | string | 是 | 供应商 |
| country | string | 是 | 国家 |
| region | string | 是 | 城市或机房 |
| ip | string | 是 | VPS 公网 IP |
| ssh_port | number | 是 | 默认 22 |
| os | string | 否 | 例如 Ubuntu 22.04 |
| panel_url | string | 否 | 3x-ui 面板地址 |
| panel_port | number | 否 | 建议 54321 |
| panel_path | string | 否 | 3x-ui webBasePath |
| credential_ref | string | 是 | 密码管理器条目名 |
| status | enum | 是 | `available` / `provisioning` / `ready` / `assigned` / `maintenance` / `disabled` |
| monthly_cost | number | 否 | 月成本 |
| expires_at | date | 否 | 到期日 |
| assigned_customer_id | string | 否 | 当前绑定客户 |
| notes | text | 否 | 备注 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### residential_ips

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| id | string | 是 | 住宅 IP ID，例如 `rip_us_001` |
| provider | string | 是 | IPRoyal、922S5、NodeMaven 等 |
| country | string | 是 | 国家 |
| endpoint | string | 是 | SOCKS5 Endpoint |
| port | number | 是 | SOCKS5 Port |
| username_ref | string | 是 | 用户名保存位置或引用 |
| password_ref | string | 是 | 密码保存位置或引用 |
| clean_score | string | 否 | whoer / scamalytics / ipinfo 结果 |
| status | enum | 是 | `available` / `assigned` / `dirty` / `disabled` |
| assigned_vps_id | string | 否 | 绑定 VPS |
| expires_at | date | 否 | 到期日 |

### customers

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| id | string | 是 | 客户 ID，例如 `cust_001` |
| name | string | 是 | 客户名称或备注 |
| contact | string | 否 | 联系方式 |
| plan | string | 是 | 套餐 |
| target_country | string | 是 | 业务地区 |
| device_limit | number | 是 | 终端数 |
| vps_id | string | 否 | 绑定 VPS |
| residential_ip_id | string | 否 | 绑定住宅 IP |
| subscription_url | string | 否 | 订阅链接 |
| starts_at | date | 是 | 开通日 |
| expires_at | date | 是 | 到期日 |
| status | enum | 是 | `pending` / `active` / `expired` / `suspended` |

## 后台页面

### VPS 管理

功能：

- 新增 VPS
- 编辑 VPS
- 修改状态
- 记录初始化结果
- 查看绑定客户
- 标记维护或停用

列表字段：

- VPS ID
- 供应商
- 地区
- IP
- 状态
- 面板地址
- 到期日
- 绑定客户

### 客户开通

功能：

- 创建客户
- 选择目标地区
- 从 `ready` VPS 中选择一台
- 绑定住宅 IP
- 填写终端数和到期日
- 保存订阅链接

### 资源分配规则

默认规则：

1. 目标国家一致。
2. VPS 状态必须是 `ready`。
3. 住宅 IP 状态必须是 `available` 且纯净度合格。
4. 一个客户默认绑定一台 VPS 和一个住宅 IP。

## API 草案

```http
GET /admin/vps
POST /admin/vps
GET /admin/vps/:id
PATCH /admin/vps/:id
POST /admin/vps/:id/mark-ready
POST /admin/vps/:id/disable

GET /admin/residential-ips
POST /admin/residential-ips
PATCH /admin/residential-ips/:id

POST /admin/customers
POST /admin/customers/:id/assign-resources
PATCH /admin/customers/:id/subscription
```

## 安全要求

- 后台数据库不保存 root 密码、SOCKS5 密码、3x-ui 面板密码明文。
- 只保存密码管理器引用，例如 `1password:vps_us_lax_001/root`。
- 后台操作需要管理员登录。
- 所有资源状态变更记录操作时间和操作人。
- 删除 VPS 默认只软删除或改 `disabled`，避免误删历史客户关联。

## 禁止事项

- 禁止在代码里写死 `VPS-001`、固定 IP、固定面板 URL。
- 禁止在前端代码里暴露 SSH 密码、面板密码、SOCKS5 密码。
- 禁止客户自动随机切换 VPS，除非后续明确做多节点产品。

