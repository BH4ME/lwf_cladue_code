# 服务器接口文档

## 服务器地址

| 项目 | 值 |
|------|-----|
| 公网 IP | `47.251.162.209` |
| Tailscale IP | `100.89.61.8` |
| 域名 | `mecho.top` |

---

## 一、MQTT 接口（推荐）

### 连接参数

| 参数 | 值 |
|------|-----|
| 地址 | `mqtt.mecho.top` |
| 端口 | `1883` (TCP) |
| WebSocket | `wss://mqtt.mecho.top/mqtt` |
| 用户名 | `coap-server` |
| 密码 | `coap123456` |
| Client ID | 任意，建议用设备标识 |

### 发布（设备 → 服务器）

**主题**: `sensor/{mac后4位}/data`

**Payload** (JSON):
```json
{
    "mac_last4": "a1b2",
    "temperature": 30.1,
    "humidity": 55,
    "source": "mqtt"
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `mac_last4` | 建议 | mac 后 4 位，设备唯一标识 |
| `source` | 建议填 `"mqtt"` | 标记协议来源 |

### 订阅（前端实时接收）

**主题**: `sensor/+/data`（通配符，接收所有设备）

---

## 二、CoAP 接口（受限设备）

| 参数 | 值 |
|------|-----|
| 地址 | `coap://coap.mecho.top` |
| 端口 | `5683` (UDP) |
| 路径 | `/sensor` |
| 认证 | 无需 |

### 发送 (POST)

UDP 包 payload (JSON):
```json
{
    "mac_last4": "a1b2",
    "temperature": 30.1,
    "humidity": 55
}
```

服务器自动：
- 提取 `mac_last4` 后 4 位作为 device_id
- 注入 `"source": "coap"`
- 转发到 MQTT 主题 `sensor/a1b2/data`

---

## 三、HTTP API（Web 前端用）

### 获取所有节点最新数据

```
GET https://coap.mecho.top/sensor
```

返回:
```json
{
    "a1b2": {"mac_last4": "a1b2", "temperature": 30.1, "source": "coap"},
    "c3d4": {"mac": "cc:dd:ee:ff:c3:d4", "temperature": 22.7, "source": "mqtt"}
}
```

key 是 device_id（mac 后 4 位），value 是设备最新一条数据。

---

## 四、PostgreSQL（历史数据）

| 参数 | 值 |
|------|-----|
| 地址 | `47.251.162.209` 或 Tailscale `100.89.61.8` |
| 端口 | `5432`（需在 nginx stream 加转发，或走 Tailscale） |
| 库 | `mqtt` |
| 用户 | `mqtt` |
| 密码 | `mqttpass` |
| 表 | `mqtt_messages` |

查某设备最近 100 条:
```sql
SELECT topic, payload, timestamp
FROM mqtt_messages
WHERE topic = 'sensor/a1b2/data'
ORDER BY timestamp DESC
LIMIT 100;
```

---

## 五、EMQX 控制台

```
https://mqtt.mecho.top
admin / admin1234
```
