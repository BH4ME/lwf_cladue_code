# 服务器配置日志

**服务器**: 47.251.162.209 (阿里云 us-west-1)  
**操作系统**: Ubuntu 22.04.5 LTS

---

## 2026-05-16 — IoT 多协议后端 & 多节点支持

### CoAP Server 重构

- 支持多节点上传，按 `mac_last4` / `mac` / `device_id` 后 4 位隔离存储
- POST `/sensor` → 自动转发到 MQTT，注入 `"source": "coap"`
- 启动时从 PostgreSQL `mqtt_messages` 表加载缓存，重启数据不丢
- systemd: `coap-server.service`
- 地址: `coap://coap.mecho.top:5683/sensor`

### CoAP Gateway

- HTTP → CoAP 桥接: `172.19.0.1:5684`
- nginx: `https://coap.mecho.top` → gateway
- GET `https://coap.mecho.top/sensor` 返回所有节点最新 JSON

### MQTT (EMQX)

- 新建 MQTT 用户 `coap-server` / `coap123456`，设备共用
- 统一主题: `sensor/{device_id}/data`（device_id = mac 后 4 位）
- EMQX 规则 `persist_all_mqtt_messages` 自动持久化所有消息到 PostgreSQL

### SSH 加固

- MaxStartups: 10:30:100 → 100:30:200（防暴力破解误杀合法连接）
- 新增 PerSourceMaxStartups 5（单 IP 限制）

### 数据流

```
设备 ──CoAP──→ coap-server:5683 ──MQTT──→ EMQX ──规则──→ PostgreSQL
设备 ──MQTT──→ EMQX:1883 ───────────────────────────→ PostgreSQL  
前端 ←──HTTPS── coap.mecho.top/sensor  ←── 内存缓存
前端 ←──WSS──── mqtt.mecho.top/mqtt    ←── 实时推送
```

### 接口速查

| 用途 | 协议 | 地址 | 认证 |
|------|------|------|------|
| 设备上报 | CoAP | `coap://coap.mecho.top:5683/sensor` | 无 |
| 设备上报 | MQTT | `mqtt.mecho.top:1883` | coap-server / coap123456 |
| 前端查询 | HTTP | `GET https://coap.mecho.top/sensor` | 无 |
| 前端实时 | MQTT WS | `wss://mqtt.mecho.top/mqtt` | coap-server / coap123456 |
| 历史数据 | PG | `100.89.61.8:5432` (Tailscale) | mqtt / mqttpass |
| 控制台 | Web | `https://mqtt.mecho.top` | admin / Aa797900 |

### MQTT 主题

| 主题 | 方向 | 说明 |
|------|------|------|
| `sensor/{id}/data` | 设备→服务器 | 传感器上报 |
| `sensor/+/data` | 前端订阅 | 通配符，实时接收所有设备 |

### 区分来源

| source 值 | 含义 |
|-----------|------|
| `"coap"` | CoAP 设备，服务器自动添加 |
| `"mqtt"` | MQTT 设备，设备端写入 |

### 修改文件

| 文件 | 改动 |
|------|------|
| `/home/admin/coap-server/server.py` | 多节点隔离 + MQTT 转发 + PG 持久化 + source 标记 |
| `/home/admin/coap-gateway/gateway.py` | 提示更新 |
| `/etc/ssh/sshd_config` | MaxStartups 100:30:200 + PerSourceMaxStartups 5 |
| `/home/admin/iot-api-docs.md` | 接口文档 |

### 设备标识提取

Payload JSON 优先取: `mac_last4` > `mac` > `device_id` > `device` > `node_id` > `id` → 最后 4 字符 → MD5 fallback

---

## 2026-05-15 — 初始搭建

### Cockpit 面板

- 安装 `sscg` 修复证书，nginx 反向代理
- `https://ray.mecho.top` → `172.19.0.1:9090`

### 用户

| 用户名 | sudo | SSH |
|--------|------|-----|
| root | ✅ | 密钥 |
| admin | ✅ | 密钥 |
| owen | ✅ | 密钥 (67215837) |

### SSH 安全

- fail2ban (10分/3次/2小时)，禁用密码，PermitRootLogin prohibit-password

### EMQX MQTT Broker

- EMQX 6.0.0 Docker
- Dashboard: `https://mqtt.mecho.top` — `admin` / `Aa797900`

### nginx

```
:80   → HTTP (Let's Encrypt + HTTPS)
:443  → ray.mecho.top  → Cockpit (9090)
:443  → mqtt.mecho.top → EMQX Dashboard (18083) + WS (8083)
:443  → coap.mecho.top → CoAP Gateway (5684)   ← new
:1883 → MQTT TCP → EMQX
```

### Docker

| 容器 | 用途 |
|------|------|
| nginx | 入口网关 |
| emqx | MQTT Broker |
| postgres | PostgreSQL 16 |
| certbot | SSL |

### SSL

| 域名 | 到期 |
|------|------|
| ray.mecho.top | 2026-08-13 |
| mqtt.mecho.top | 2026-08-13 |
| coap.mecho.top | 2026-08-13 |
