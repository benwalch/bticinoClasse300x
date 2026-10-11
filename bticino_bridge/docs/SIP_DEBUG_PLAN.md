# SIP Registration Debug Plan

**Date**: 2026-03-25  
**Status**: ⏳ **PENDING**  
**Priority**: Medium (bridge works without SIP)

---

## 📋 Current Status

### **What We Know**:

1. ✅ **Flexisip configured** - Listening on 192.168.1.38:5060/5061
2. ✅ **User created** - `bticino_bridge@1754162.bs.iotleg.com`
3. ✅ **Route configured** - `sip:127.0.0.1`
4. ✅ **Doorbell configured** - In route_int.conf
5. ❌ **Registration fails** - `i/o timeout` after 10 seconds
6. ❌ **No flexisip logs** - Receives no REGISTER

---

## 🎯 Goal

Get `bticino_bridge` to register successfully with local `flexisip` to enable:
- RTSP streaming
- HKSV recording
- Outgoing/incoming calls

---

## 🔍 Hypotheses

### **Hypothesis 1: SIP client doesn't send REGISTER**
**Probability**: 30%  
**Test**: capture network traffic with tcpdump

### **Hypothesis 2: Flexisip doesn't answer REGISTER**
**Probability**: 40%  
**Test**: detailed flexisip logs (level 9)

### **Hypothesis 3: Wrong username format**
**Probability**: 20%  
**Test**: try with an existing user (baresip)

### **Hypothesis 4: Internal firewall blocks**
**Probability**: 10%  
**Test**: check iptables rules

---

## 🧪 Testing Plan

### **Phase 1: Traffic Capture** ⏳

**Goal**: verify whether REGISTER leaves the bridge

```bash
# On the device
ssh bticino

# Start capture
tcpdump -i lo -n -s 0 -w /tmp/sip_register.pcap port 5060

# In another terminal, restart bridge
pkill -9 bticino_bridge
cd /home/bticino/cfg/extra
./bticino_bridge -config config_streaming.yaml &

# Stop capture after 15 seconds
killall tcpdump

# Analizar captura
tcpdump -r /tmp/sip_register.pcap -X | head -100
```

**Expected Result**:
- ✅ Should show `REGISTER sip:1754162.bs.iotleg.com SIP/2.0`
- ❌ If it shows nothing, the bridge isn't sending

---

### **Phase 2: Detailed Flexisip Logs** ⏳

**Goal**: see what flexisip receives

```bash
# Editar flexisip.conf
ssh bticino
cat > /etc/flexisip/flexisip.conf << 'EOF'
[global]
aliases=1754162.bs.iotleg.com
transports=sip:*:5060
log-level=debug
syslog-level=debug

[module::SipAg]
log-level=9

[module::Authentication]
enabled=true
auth-domains=1754162.bs.iotleg.com
trusted-hosts=127.0.0.1,192.168.1.38
hashed-passwords=true
EOF

# Restart flexisip
/etc/init.d/flexisipsh restart

# Monitorear logs
tail -f /var/log/log_rotation.log | grep -i register
```

**Expected Result**:
- ✅ Should show REGISTER received
- ❌ If it shows nothing, flexisip isn't receiving

---

### **Phase 3: Try with Existing User** ⏳

**Goal**: rule out a configuration problem

```bash
# Editar config del bridge
ssh bticino
cat > /home/bticino/cfg/extra/config_test.yaml << 'EOF'
sip:
  enabled: true
  server_host: 127.0.0.1
  server_port: 5060
  transport: tcp
  domain: 1754162.bs.iotleg.com
  username: baresip@1754162.bs.iotleg.com
  password: 2db500964b556d4cef79284738f2a360
  use_ha1: true
EOF

# Restart bridge
pkill -9 bticino_bridge
cd /home/bticino/cfg/extra
./bticino_bridge -config config_test.yaml &

# Ver logs
tail -f /var/log/bticino_bridge.log | grep -i sip
```

**Expected Result**:
- ✅ If it works with baresip → the problem is bticino_bridge's configuration
- ❌ If it fails the same way → the problem is the bridge SIP client

---

### **Phase 4: Test with Sipsak** ⏳

**Goal**: test registration with an external tool

```bash
# From another machine (not the device)
# Install sipsak
sudo apt-get install sipsak

# Probar registro
sipsak -s sip:bticino_bridge@1754162.bs.iotleg.com \
       -u bticino_bridge \
       -p 2db500964b556d4cef79284738f2a360 \
       -P tcp \
       -a 192.168.1.38 \
       -v
```

**Expected Result**:
- ✅ `SIP/2.0 200 OK` → Flexisip works
- ❌ `SIP/2.0 403 Forbidden` → Auth problem
- ❌ Timeout → Flexisip isn't responding

---

### **Phase 5: SIP Code Debug** ⏳

**Goal**: add logs to the bridge SIP client

```bash
# In the bridge Go code
# pkg/sip/client.go

func (c *BTicinoSIPClient) register() error {
    c.logger.Debug("Starting SIP registration...")  # AGREGAR
    c.logger.Debugf("Server: %s:%d", c.config.ServerHost, c.config.ServerPort)  # AGREGAR
    c.logger.Debugf("Username: %s", c.config.Username)  # AGREGAR
    c.logger.Debugf("Domain: %s", c.config.Domain)  # AGREGAR
    
    // ... rest of the code
}
```

**Compile with logs**:
```bash
go build -tags debug -o bticino_bridge_debug ./cmd/main.go
```

---

## 🔧 Potential Solutions

### **Solution 1: Change username format**

```yaml
# Option A: No domain in username
sip:
  username: bticino_bridge
  domain: 1754162.bs.iotleg.com

# Option B: With full domain
sip:
  username: bticino_bridge@1754162.bs.iotleg.com
  domain: 1754162.bs.iotleg.com
```

### **Solution 2: Change transport**

```yaml
# Probar UDP en lugar de TCP
sip:
  transport: udp
  server_port: 5060
```

### **Solution 3: Disable hashed-passwords**

```ini
# En flexisip.conf
[module::Authentication]
hashed-passwords=false
```

### **Solution 4: Add user without hash**

```bash
# En users.db.txt
bticino_bridge@1754162.bs.iotleg.com plain:password123 ;
```

### **Solution 5: Use IP instead of localhost**

```yaml
sip:
  server_host: 192.168.1.38  # En lugar de 127.0.0.1
```

---

## 📊 Estimated Timeline

| Phase | Duration | Priority |
|------|----------|-----------|
| **Phase 1: Capture** | 30 min | 🔴 High |
| **Phase 2: Flexisip logs** | 30 min | 🔴 High |
| **Phase 3: Existing user** | 30 min | 🟡 Medium |
| **Phase 4: Sipsak** | 1 hour | 🟡 Medium |
| **Phase 5: Code debug** | 2 hours | 🟢 Low |

**Estimated total**: 4-5 hours

---

## 🎯 Success Criteria

### **Successful Registration**:
```
✅ Logs muestran: "SIP registration successful"
✅ API muestra: "sip_registered": true
✅ Flexisip logs muestran: REGISTER 200 OK
✅ No more timeout errors
```

### **Enabled Functionality**:
```
✅ RTSP streaming funciona
✅ HKSV recording available
✅ Llamadas entrantes/salientes
```

---

## 📁 Required Resources

### **Tools**:
- [x] tcpdump (or wireshark)
- [x] sipsak
- [x] SSH to the device
- [x] Compiled v0.13.0 binary

### **Access**:
- [x] SSH: `ssh bticino`
- [x] Web: `http://192.168.1.38:8082`
- [x] Logs: `/var/log/bticino_bridge.log`

---

## 🔄 Plan B

If local SIP registration still doesn't work after debugging:

### **Option A: Use external SIP server**
```yaml
sip:
  server_host: sipserver.bs.iotleg.com
  server_port: 5061
  transport: tls
```

### **Option B: Skip SIP for now**
- Working bridge without streaming
- Focus on other features
- Return to SIP later

### **Option C: Implement alternative SIP client**
- Use a different Go library
- Pion SIP or similar
- More work but more control

---

**Next Step**: start Phase 1 (traffic capture)  
**Owner**: TBD  
**Start Date**: Immediate
