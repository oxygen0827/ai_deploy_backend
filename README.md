# 小智AI 后台管理系统

基于 ESP32 的 AI 硬件设备运营管控平台。已与 EspLink BLE 配网系统打通：用户通过微信小程序蓝牙配网后，设备自动注册上线并与账号绑定；管理员通过后台掌握所有设备、租户和用量数据。平台提供 API Key 管理、用量统计、多租户隔离等完整 SaaS 管控能力。

---

## 功能特性

- **BLE 蓝牙配网** — 微信小程序通过 EspLink BLE 协议为 ESP32 配网，配网成功后设备自动注册并绑定到微信账号
- **扫码配对** — 同时支持微信小程序扫描设备二维码完成配对（备用流程）
- **WebSocket 长连接** — 固件通过 `/ws/device` 与后端保持长连接，支持实时指令下发和 OTA 推送
- **多租户管理** — 支持按客户/团队隔离，每个租户独立的 API Key 池和用量配额
- **API Key 管控** — 生成、启停、限额、过期时间，支持 Redis 缓存加速验证（命中率 >95%）
- **设备管理** — 实时在线状态、心跳检测、强制下线、设备解绑、手动分配 Key
- **用量统计** — 今日/本月调用量、趋势图、模型占比、调用明细（支持导出 CSV）
- **用量告警** — 达到阈值时向租户配置的 Webhook 推送告警（支持钉钉/企微/飞书）
- **分布式限流** — 基于 Redis 令牌桶，多进程部署下共享限流计数
- **数据分层** — 明细日志按月分区，每小时预聚合，统计查询永远不扫全表

---

## 系统架构

```
微信小程序 (EspLink)               ESP32 设备 (EspLink 固件)
    │  BLE 配网 → WiFi 连接          │  POST /api/ota/check  → 注册+获取 token
    │  POST /api/auth/wechat         │  WS  /ws/device       → 长连接
    │  GET  /api/device/lookup       │  心跳 ping → last_seen 更新
    │  POST /api/device/bind         │
    └──────────────┬─────────────────┘
                   ▼
        ┌─────────────────────┐
        │   后端 API (8088)   │  Express 4.x + Prisma + Redis + ws
        │   /api/v1/...  管理 │
        │   /api/...  EspLink │
        │   /ws/device   WS   │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  管理后台 (5173)    │  React 18 + Ant Design 5
        │  仪表盘/租户/Key/   │
        │  设备/用量统计      │
        └─────────────────────┘
                   │
        ┌──────────▼──────────┐
        │  官方 xiaozhi DB    │  MySQL 8 + Redis（复用）
        │  + 扩展业务表       │
        └─────────────────────┘
```

---

## 技术栈

| 层 | 技术 | 版本 |
|---|---|---|
| 后端框架 | Express.js | ^4.22.1（锁定，禁止升级至 v5） |
| ORM | Prisma | ^5.x |
| 缓存 / 限流 | Redis (ioredis) | ^5.x |
| 数据库 | MySQL | 8.x |
| 前端框架 | React + Vite | 18.x / ^5.x |
| UI 组件库 | Ant Design | ^5.x |
| 图表 | Recharts | ^2.x |
| 状态管理 | Zustand | ^4.x |
| HTTP 客户端 | Axios | ^1.x |

---

## 数据库表结构

| 表名 | 说明 |
|---|---|
| `tenants` | 租户，含每日/月限额和告警 Webhook |
| `api_keys` | API Key，含用量计数和过期时间 |
| `devices` | 设备，以 MAC 地址为主键；含 `device_key`（WebSocket 认证）、`board_type`、`capabilities`、`wechat_user_id` |
| `wechat_users` | 微信用户，通过 EspLink 小程序登录自动创建，与设备关联 |
| `pair_records` | 二维码配对记录，存储 device_id 与用户 openid 的绑定过程 |
| `usage_logs` | 调用明细，按月分区，保留 7 天 |
| `usage_hourly` | 每小时预聚合，统计查询的主要数据源 |

---

## API 概览

### 管理接口（`/api/v1/` 前缀，需 Bearer 管理员 token）

| 模块 | 路径 | 说明 |
|---|---|---|
| 认证 | `POST /api/v1/auth/login` | 管理员登录 |
| 租户 | `GET/POST/PATCH/DELETE /api/v1/tenants` | 租户 CRUD |
| API Key | `GET/POST/PATCH/DELETE /api/v1/keys` | Key 管理 |
| 设备 | `GET /api/v1/devices` | 设备列表 |
| 设备 | `POST /api/v1/devices/register` | 固件自注册（无需认证） |
| 设备 | `POST /api/v1/devices/:mac/kick` | 强制下线 |
| 设备 | `POST /api/v1/devices/:mac/unbind` | 解绑 |
| 配对 | `POST /api/v1/pair/verify` | 小程序发起配对（无需认证） |
| 配对 | `POST /api/v1/pair/confirm` | 小程序确认配对（无需认证） |
| 用量 | `GET /api/v1/usage/summary` | 汇总统计 |
| 用量 | `GET /api/v1/usage/daily` | 按天趋势 |
| 用量 | `GET /api/v1/usage/logs` | 调用明细（7天内） |
| 运营 | `GET /api/v1/operation/overview` | 运营大盘 |
| 健康 | `GET /api/v1/health/ready` | 就绪检查（含 DB + Redis） |

### EspLink 接口（`/api/` 前缀，固件和小程序使用）

| 模块 | 路径 | 认证 | 说明 |
|---|---|---|---|
| 微信登录 | `POST /api/auth/wechat` | 无 | 小程序 code 换 JWT token |
| 固件注册 | `POST /api/ota/check` | 无 | 设备上电注册，返回 device_key + ws 地址 |
| 设备列表 | `GET /api/device/list` | 微信 JWT | 当前用户的绑定设备 |
| 设备查找 | `GET /api/device/lookup?mac_suffix=AABBCC` | 微信 JWT | 按 MAC 后三字节查找刚上线设备 |
| 设备绑定 | `POST /api/device/bind` | 微信 JWT | 绑定设备到微信账号 |
| 下发指令 | `POST /api/device/:mac/command` | 微信 JWT | 通过 WebSocket 向设备推送指令 |
| WebSocket | `WS /ws/device` | device_key | 固件长连接（hello/ping/command） |

---

## 设备配网与配对流程

### EspLink BLE 配网流程（主流程）

```
1. 设备上电，无 WiFi 凭证 → 启动 BLE 广播 "Device-AABBCC"
2. 用户打开微信小程序 → BLE 扫描发现设备
3. 小程序通过 BluFi 协议发送 WiFi SSID/密码 → 设备连接 WiFi
4. 设备 WiFi 上线 → 调用 POST /api/ota/check
   → 后端自动注册设备（首次），返回 device_key + websocket_url
5. 固件建立 WebSocket 长连接 /ws/device，发送 hello 握手
6. 小程序轮询 GET /api/device/lookup?mac_suffix=AABBCC 等待设备上线
7. 发现设备 → 调用 POST /api/device/bind → 设备绑定到微信账号
```

### 二维码配对流程（备用流程）

```
1. 设备出厂时二维码内含唯一 device_id
2. 用户微信扫码 → 小程序获得 device_id
3. 小程序调用 POST /api/v1/pair/verify  →  返回 pair_token（5分钟有效）
4. 用户在小程序确认绑定
5. 小程序调用 POST /api/v1/pair/confirm  →  配对完成，openid 与设备绑定
6. 设备开机后调用 POST /api/v1/devices/register（携带 mac_address + device_id）
7. 后端自动关联配对记录，设备进入已配对状态
```

---

## 定时任务

| 任务 | 频率 | 说明 |
|---|---|---|
| 心跳检测 | 每分钟 | 超过 2 分钟无心跳的设备标记为离线 |
| 用量聚合 | 每小时第5分钟 | 将上一小时明细聚合到 `usage_hourly` |
| 明细清理 | 每天凌晨2点 | 删除 7 天前的 `usage_logs` 明细 |

---

## 项目结构

```
backend/
├── src/
│   ├── app.js              # 入口
│   ├── config/             # Prisma + Redis 客户端
│   ├── middleware/         # requestId / adminAuth / keyValidator / rateLimiter
│   ├── routes/             # auth / tenants / keys / devices / usage / pair / operation
│   ├── services/           # 业务逻辑层
│   └── jobs/               # 定时任务
├── prisma/
│   └── schema.prisma
├── admin-frontend/         # React 管理后台
│   └── src/
│       ├── pages/          # Dashboard / Tenants / ApiKeys / Devices / Usage / Login
│       ├── api/            # Axios 封装
│       └── store/          # Zustand 全局状态
└── .env.example
```

---

## 部署

- **服务器**：Spark2（`150.158.146.192`），通过 FRP 隧道访问
- **后端**：port 8088，PM2 守护
- **前端**：port 8080，`npm run build` 后由 nginx / serve 托管 `admin-frontend/dist/`

详见 [open.md](./open.md) 的生产部署章节。

## License

MIT
