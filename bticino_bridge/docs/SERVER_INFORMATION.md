# BTicino Classe 300X - Server Information & Network Requirements

**Date**: 2026-03-24  
**Source**: official installation and configuration manual  
**Status**: ✅ Verified - Official information

---

## 📋 Server Summary

| Service | Server/Domain | Port | Protocol | Status | Use |
|----------|--------------|--------|-----------|--------|-----|
| **Main Cloud** | `nv2-bncx.netatmo.net` | 25050 | TCP | ✅ Official | Netatmo cloud communication |
| **NTP** | `pool.ntp.org` | 123 | NTP | ⚙️ Configurable | Time synchronization |
| **Logs** | `log.bs.iotleg.com` | 5001 | Syslog | ✅ Official | Remote log shipping |
| **Updates** | `n3tfw.blob.core.windows.net` | 443 | HTTPS | ✅ Official | Firmware/update downloads |
| **Email** | Depends on user | Variable | SMTP/IMAP | ⚙️ Configurable | Email notifications |

---

## 🔍 Server Analysis

### 1. **Cloud Service - nv2-bncx.netatmo.net:25050/tcp**

**Purpose**: main communication with the Netatmo cloud

**Implications for bticino_bridge**:
- ⚠️ **IMPORTANT**: the cloud server changed from `iotleg.com` to `netatmo.net`
- 🔄 The bridge must be able to reach this server if it wants to emulate the official device
- 🔒 TCP port 25050 must be open toward the internet (outbound)

**Suggested configuration for the bridge**:
```yaml
# Future configuration in web dashboard
cloud:
  enabled: false  # Por defecto deshabilitado (privacy)
  server: nv2-bncx.netatmo.net
  port: 25050
  protocol: tcp
```

**Note**: Netatmo acquired BTicino, hence the domain change.

---

### 2. **NTP Service - pool.ntp.org:123/ntp**

**Purpose**: system clock synchronization

**Status**: preset, user-modifiable

**Implications for bticino_bridge**:
- ✅ The bridge already uses the OS NTP
- 🕐 Important for logs, message timestamps, TLS certificates

**Current device configuration**:
```bash
# En el BTicino real
ntpd -p pool.ntp.org
```

**Recommendation**: ensure the system has NTP configured:
```bash
# En Linux host
timedatectl set-ntp true
```

---

### 3. **Logs Service - log.bs.iotleg.com:5001/syslog**

**Purpose**: shipping telemetry logs to BTicino

**Implications for bticino_bridge**:
- ⚠️ **IMPORTANT**: this domain is still active (`iotleg.com`)
- 📊 The device ships logs automatically
- 🔒 The bridge can intercept/block this traffic for privacy

**Suggested configuration**:
```yaml
# Block log uploads to BTicino (privacy)
privacy:
  block_telemetry: true
  block_log_server: true
  
# O redirigir a servidor local
logging:
  remote_enabled: true
  remote_host: localhost
  remote_port: 5001
  protocol: udp
```

**Note**: `iotleg.com` looks like the legacy domain coexisting with `netatmo.net`.

---

### 4. **Updates Service - n3tfw.blob.core.windows.net:443/https**

**Purpose**: firmware and update downloads

**Implications for bticino_bridge**:
- ⚠️ **IMPORTANT**: Azure Blob Storage (Microsoft)
- 🔄 The device checks for updates periodically
- 🔒 HTTPS (port 443) must be open outbound

**Suggested configuration**:
```yaml
# Disable automatic updates (the bridge handles updates)
updates:
  auto_check: false
  auto_download: false
  
# Or use local proxy
updates:
  proxy_enabled: true
  proxy_url: http://localhost:8080/updates
```

**Note**: using Azure Blob Storage suggests Netatmo runs on Microsoft infrastructure.

---

### 5. **Email Service - User configurable**

**Purpose**: email notifications (doorbell, alarms, etc.)

**Implications for bticino_bridge**:
- ✅ The bridge can implement its own notification service
- 📧 MQTT + Home Assistant already provides notifications
- 📱 Alternative: push notifications via API (Telegram, Pushover, etc.)

**Suggested configuration for the bridge**:
```yaml
notifications:
  # Option 1: Email (if required)
  email:
    enabled: false
    smtp_server: smtp.gmail.com
    smtp_port: 587
    username: user@gmail.com
    password: "***"
    
  # Option 2: MQTT (recommended)
  mqtt:
    enabled: true
    topic: bticino/notifications
    
  # Option 3: Push notifications
  pushover:
    enabled: false
    api_key: "***"
    user_key: "***"
```

---

## 🔐 Security Considerations

### Firewalls Required (Outbound)

| Destination | Port | Protocol | Priority |
|---------|--------|-----------|-----------|
| `nv2-bncx.netatmo.net` | 25050 | TCP | ⚠️ Medium (if cloud is used) |
| `pool.ntp.org` | 123 | UDP/NTP | ✅ High (time sync) |
| `log.bs.iotleg.com` | 5001 | UDP/TCP | ⚠️ Low (telemetry, optional) |
| `n3tfw.blob.core.windows.net` | 443 | TCP/HTTPS | ⚠️ Medium (updates) |

### Recommendation: Minimal Firewall

```bash
# Permitir solo lo esencial
# NTP (tiempo)
iptables -A OUTPUT -p udp --dport 123 -d pool.ntp.org -j ACCEPT

# Block telemetry to BTicino (privacy)
iptables -A OUTPUT -p udp --dport 5001 -d log.bs.iotleg.com -j DROP

# Bloquear cloud de Netatmo (si no se usa)
iptables -A OUTPUT -p tcp --dport 25050 -d nv2-bncx.netatmo.net -j DROP

# Permitir actualizaciones (opcional)
iptables -A OUTPUT -p tcp --dport 443 -d n3tfw.blob.core.windows.net -j ACCEPT
```

---

## 📊 Impact on bticino_bridge

### **Current Configuration (v0.12.0)**

```yaml
# config_streaming.yaml actual
sip:
  server_host: 192.168.1.38  # ✅ Local - OK
  server_port: 5060
```

**Status**: ✅ unaffected - uses local server

---

### **Future Configuration (v0.13.0+)**

```yaml
# New servers section
servers:
  # Cloud Netatmo (opcional, privacy-first por defecto)
  cloud:
    enabled: false
    host: nv2-bncx.netatmo.net
    port: 25050
    protocol: tcp
    
  # NTP (use operating system)
  ntp:
    enabled: true
    host: pool.ntp.org
    port: 123
    protocol: ntp
    
  # Logs remotos (opcional, local por defecto)
  logging:
    remote_enabled: false
    host: localhost
    port: 5001
    protocol: udp
    
  # Actualizaciones (deshabilitado por defecto)
  updates:
    auto_check: false
    host: n3tfw.blob.core.windows.net
    port: 443
    protocol: https
    
  # Notificaciones (MQTT por defecto)
  notifications:
    method: mqtt  # mqtt, email, pushover, none
```

---

## 🎯 Implementation Recommendations

### **Priority 1: Documentation** ✅
- [x] Create this document
- [ ] Update README with server information
- [ ] Add network requirements section

### **Priority 2: Configuration** ⏳
- [ ] Add `servers` section to config.yaml
- [ ] Privacy-first defaults (everything disabled except NTP)
- [ ] Document each server in comments

### **Priority 3: Firewall** ⏳
- [ ] Optional script to configure iptables
- [ ] Document required ports
- [ ] Warn about telemetry to BTicino

### **Priority 4: Web UI** ⏳ (v0.13.0)
- [ ] Settings page → Network/Servers tab
- [ ] Toggle per server
- [ ] Show network traffic per server

---

## 📝 Important Notes

### **Domain Change: BTicino → Netatmo**

```
2015-2020: BTicino independiente
  - Dominios: *.bticino.it, *.iotleg.com
  
2020-Presente: Netatmo adquiere BTicino
  - Dominios: *.netatmo.net, *.legrand.com
  - Cloud migrado a: nv2-bncx.netatmo.net
  - Updates en Azure: n3tfw.blob.core.windows.net
```

**Implication**: the bridge must support both domains for compatibility with legacy devices.

---

### **Domain Coexistence**

| Domain | Status | Use |
|---------|--------|-----|
| `iotleg.com` | ✅ Active (legacy) | SIP, logs |
| `netatmo.net` | ✅ Active (new) | Main cloud |
| `bs.iotleg.com` | ✅ Active | SIP server |
| `windows.net` | ✅ Active | Azure blob storage |

---

## 🔍 Pending Research

- [ ] Does the `nv2-bncx.netatmo.net` cloud require authentication?
- [ ] What protocol does port 25050 use? (HTTP, WebSocket, custom?)
- [ ] Can the cloud traffic be intercepted/emulated?
- [ ] Does the device work WITHOUT internet? (it should)
- [ ] Are more servers involved?

---

## 📁 References

- **Source**: official BTicino installation and configuration manual
- **Section**: "Network Services and Ports"
- **Manual date**: 2024-2025 (estimated)
- **Device version**: C300X-00-03-50-a8-a7-52-1754162

---

**Last updated**: 2026-03-24  
**Next review**: after connectivity testing  
**Status**: ✅ verified official information
