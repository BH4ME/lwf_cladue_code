# 服务器配置日志

**日期**: 2026-05-15  
**服务器**: 47.251.162.209 (阿里云 us-west-1)  
**操作系统**: Ubuntu 22.04.5 LTS

---

## 1. Cockpit 面板修复 & 配置

- 安装 `sscg` 修复 Cockpit 证书生成错误
- 通过 nginx 反向代理，从 `https://47.251.162.209:9090` → `https://ray.mecho.top`
- 签发 Let's Encrypt 正规 SSL 证书

**访问**: `https://ray.mecho.top`

---

## 2. 用户管理

| 用户名 | 密码 | sudo | SSH |
|--------|------|------|-----|
| root | 原有 | ✅ | 密钥 |
| admin | 原有 | ✅ | 密钥 |
| owen | 67215837 | ✅ | 密钥 |

---

## 3. SSH 安全加固

- **fail2ban**: 10分钟内失败3次封禁2小时
- **禁用密码登录**: 仅允许 SSH 密钥认证
- **root 限制**: `PermitRootLogin prohibit-password`

### SSH 密钥

**admin**:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDqs52IkIWCNOprzWhdALpxfi2TO/dCLEMkgFurgoAQz admin@ray.mecho.top
```

**owen**:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINTRhRVJuLtf1c7WHyr8pwOLLCWttn+b20OKm7YAcVyL owen@ray.mecho.top
```

连接方式:
```bash
ssh -i <私钥文件> admin@ray.mecho.top
ssh -i <私钥文件> owen@ray.mecho.top
```

---

## 4. EMQX MQTT Broker

- 版本: EMQX 6.0.0 (Docker)
- Dashboard: `https://mqtt.mecho.top`
- 账号: `admin` / `Aa797900`

### 连接地址

| 方式 | 地址 |
|------|------|
| MQTT TCP | `mqtt.mecho.top:1883` |
| MQTT WebSocket | `wss://mqtt.mecho.top/mqtt` |

### EMQX 端口

| 端口 | 协议 | 代理方式 |
|------|------|----------|
| 1883 | MQTT TCP | nginx stream → emqx |
| 443 | MQTT WebSocket | nginx HTTPS /mqtt → emqx:8083 |
| 443 | Dashboard | nginx HTTPS → emqx:18083 |

---

## 5. nginx 架构

```
外网
├─ :80   → HTTP (Let's Encrypt验证 + HTTPS跳转)
├─ :443  → ray.mecho.top  → Cockpit (172.19.0.1:9090)
│         → mqtt.mecho.top → EMQX Dashboard (emqx:18083)
│                           → MQTT WebSocket /mqtt (emqx:8083)
└─ :1883 → MQTT TCP stream → emqx:1883
```

---

## 6. Docker 容器

| 容器 | 镜像 | 端口 |
|------|------|------|
| nginx | nginx:alpine | 80, 443, 1883 |
| certbot | certbot/certbot:latest | - |
| emqx | emqx/emqx:latest | 内部网络 |

配置路径: `/srv/web/` (docker-compose)

---

## 7. 安全组需开放端口

| 端口 | 协议 | 用途 |
|------|------|------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 1883 | TCP | MQTT |

---

## 8. SSL 证书

| 域名 | 到期 | 自动续期 |
|------|------|----------|
| ray.mecho.top | 2026-08-13 | certbot ✅ |
| mqtt.mecho.top | 2026-08-13 | certbot ✅ |
