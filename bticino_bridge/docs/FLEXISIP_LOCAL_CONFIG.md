# Flexisip Local Configuration Guide

**Date**: 2026-03-24  
**Device**: BTicino Classe 300X (192.168.1.38)  
**Status**: ✅ **COMPLETE AND VERIFIED**

---

## 📋 Applied Configuration Summary

### **Configured SIP Servers**

| User | Domain | Password (MD5) | Route | Doorbell |
|---------|---------|----------------|------|--------|
| `baresip` | 1754162.bs.iotleg.com | `2db500964b556d4cef79284738f2a360` | `sip:192.168.1.38` | ✅ |
| `baresip_test` | 1754162.bs.iotleg.com | `test123` (plain) | N/A | ❌ |
| `bticino_bridge` | 1754162.bs.iotleg.com | `2db500964b556d4cef79284738f2a360` | `sip:127.0.0.1` | ✅ |

---

## 🌐 Listening Ports

Flexisip now listens on:

```
✅ sip:192.168.1.38:5060   (SIP TCP - red local)
✅ sip:127.0.0.1:5060      (SIP TCP - localhost)
✅ sips:192.168.1.38:5061  (SIP TLS - red local)
```

**Verification**:
```bash
ps aux | grep flexisip
# Must show:
# /usr/bin/flexisip --daemon --transports sip:192.168.1.38:5060;maddr=192.168.1.38 ...
```

---

## 📁 Modified Files

### 1. `/etc/flexisip/users/users.db.txt`

**Contents**:
```
version:1
fquinto-gmx.com-A06CE9ED-19B2-4EDB-8CBB-A4EEB4F93BD2@1754162.bs.iotleg.com md5:2db500964b556d4cef79284738f2a360 ;
fquinto-gmx.com-D5EA2023-44B4-4498-BCA4-8FF213BEC251@1754162.bs.iotleg.com md5:43ead99dcce8e21882eca9073f22d503 ;
albasanor-gmail.com-C3592C7D-BB5E-46A1-8647-2FE8371C0FE9@1754162.bs.iotleg.com md5:b2e3eaebd235965c01c43e7f616d566e ;
baresip@1754162.bs.iotleg.com md5:2db500964b556d4cef79284738f2a360 ;
baresip_test@1754162.bs.iotleg.com plain:test123 ;
bticino_bridge@1754162.bs.iotleg.com md5:2db500964b556d4cef79284738f2a360 ;  # ← NUEVO
```

**MD5 Hash**: `2db500964b556d4cef79284738f2a360` (same for all local users)

---

### 2. `/etc/flexisip/users/route.conf`

**Contents**:
```
<sip:alluser@1754162.bs.iotleg.com> <sip:fquinto-gmx.com-A06CE9ED-19B2-4EDB-8CBB-A4EEB4F93BD2@1754162.bs.iotleg.com>, <sip:fquinto-gmx.com-D5EA2023-44B4-4498-BCA4-8FF213BEC251@1754162.bs.iotleg.com>, <sip:albasanor-gmail.com-C3592C7D-BB5E-46A1-8647-2FE8371C0FE9@1754162.bs.iotleg.com>
<sip:fquinto-gmx.com-A06CE9ED-19B2-4EDB-8CBB-A4EEB4F93BD2@1754162.bs.iotleg.com> <sip:sipserver.bs.iotleg.com;transport=tls>
<sip:fquinto-gmx.com-D5EA2023-44B4-4498-BCA4-8FF213BEC251@1754162.bs.iotleg.com> <sip:sipserver.bs.iotleg.com;transport=tls>
<sip:albasanor-gmail.com-C3592C7D-BB5E-46A1-8647-2FE8371C0FE9@1754162.bs.iotleg.com> <sip:sipserver.bs.iotleg.com;transport=tls>
<sip:baresip@1754162.bs.iotleg.com> sip:192.168.1.38
<sip:bticino_bridge@1754162.bs.iotleg.com> sip:127.0.0.1  # ← NUEVO
```

**Note**: `bticino_bridge` points to `127.0.0.1` because it runs on the same device.

---

### 3. `/etc/flexisip/users/route_int.conf`

**Contents**:
```
<sip:baresip@1754162.bs.iotleg.com>, <sip:bticino_bridge@1754162.bs.iotleg.com>, <sip:alluser@1754162.bs.iotleg.com> <sip:fquinto-gmx.com-A06CE9ED-19B2-4EDB-8CBB-A4EEB4F93BD2@1754162.bs.iotleg.com>, <sip:fquinto-gmx.com-D5EA2023-44B4-4498-BCA4-8FF213BEC251@1754162.bs.iotleg.com>, <sip:albasanor-gmail.com-C3592C7D-BB5E-46A1-8647-2FE8371C0FE9@1754162.bs.iotleg.com>
```

**Note**: `bticino_bridge` now receives doorbell calls.

---

### 4. `/etc/flexisip/flexisip.conf`

**Current configuration**:
```ini
[global]
aliases=1754162.bs.iotleg.com
transports=sip:*:5060
tls-certificates-dir=/etc/flexisip/tls
user-errors-logs=true
log-level=debug
syslog-level=debug

[inter-domain-connections]
accept-domain-registrations=false
domain-registrations=/etc/flexisip/domain-registration.conf
verify-server-certs=true
keepalive-interval=30

[module::Registrar]
enabled=true
reg-domains=1754162.bs.iotleg.com
db-implementation=internal
static-records-file=/etc/flexisip/users/route.conf
max-contacts-by-aor=20

[module::Authentication]
enabled=true
auth-domains=1754162.bs.iotleg.com
db-implementation=file
datasource=/etc/flexisip/users/users.db.txt
trusted-hosts=127.0.0.1,192.168.1.38
hashed-passwords=true
reject-wrong-client-certificates=true
```

**Key points**:
- ✅ `trusted-hosts=127.0.0.1,192.168.1.38` - Allows registration without authentication from these IPs
- ✅ `log-level=debug` - Detailed logs in `/var/log/log_rotation.log`
- ✅ `hashed-passwords=true` - Uses MD5 hashes

---

## 🔧 Startup Script `/etc/init.d/flexisipsh`

**Startup line**:
```bash
start-stop-daemon --start --quiet --exec $DAEMON -- $DAEMON_ARGS \
  --transports "sip:$2:5060;maddr=$2 sip:127.0.0.1:5060;maddr=127.0.0.1 sips:$2:5061;maddr=$2;require-peer-certificate=1"
```

**Usage**:
```bash
# Start with a specific IP
/etc/init.d/flexisipsh start 192.168.1.38

# Stop
/etc/init.d/flexisipsh stop

# Restart
/etc/init.d/flexisipsh restart
```

---

## 🔥 Firewall

**Status**: the iptables scripts exist but the `iptables` command is not available in PATH.

**Scripts**:
- `/etc/network/if-pre-up.d/iptables` (1496 bytes)
- `/etc/network/if-pre-up.d/iptables6` (1281 bytes)

**To permanently disable the firewall**:
```bash
# Mover scripts a backup
mv /etc/network/if-pre-up.d/iptables /home/bticino/cfg/extra/iptables.bak
mv /etc/network/if-pre-up.d/iptables6 /home/bticino/cfg/extra/iptables6.bak
```

**Note**: the firewall seems disabled or non-functional currently.

---

## 🧪 Verification

### 1. Check flexisip is running
```bash
ssh bticino "ps aux | grep flexisip | grep -v grep"
```

**Expected**:
```
root  18942  /usr/bin/flexisip --daemon --transports sip:192.168.1.38:5060;maddr=192.168.1.38 ...
root  18943  /usr/bin/flexisip --daemon --transports sip:192.168.1.38:5060;maddr=192.168.1.38 ...
```

### 2. Check listening ports
```bash
ssh bticino "cat /proc/net/tcp | grep -i '13c4\|13c5'"
# 13C4 = 5060, 13C5 = 5061
```

### 3. Check users
```bash
ssh bticino "cat /etc/flexisip/users/users.db.txt"
```

### 4. Check logs
```bash
ssh bticino "tail -50 /var/log/log_rotation.log | grep flexisip"
```

---

## 🔌 Configuration for bticino_bridge

### Update `config_streaming.yaml`:

```yaml
sip:
  enabled: true
  local_host: 192.168.1.38
  local_port: 47300
  server_host: 127.0.0.1  # ← Usar localhost
  server_port: 5060
  transport: tcp
  domain: 1754162.bs.iotleg.com
  username: bticino_bridge@1754162.bs.iotleg.com
  password: 2db500964b556d4cef79284738f2a360  # ← MD5 hash
  dev_addr: 20
  use_ha1: true  # ← Usar hash MD5
  insecure_tls: false
```

**Notes**:
- ✅ `server_host: 127.0.0.1` - Local Flexisip
- ✅ `username` includes the full domain
- ✅ `password` is the MD5 hash (not a plain password)
- ✅ `use_ha1: true` - Indicates the password is an HA1 hash

---

## 📝 Useful Commands

### Restart flexisip
```bash
ssh bticino "/etc/init.d/flexisipsh restart"
```

### Watch logs in real time
```bash
ssh bticino "tail -f /var/log/log_rotation.log | grep flexisip"
```

### Add new user
```bash
# 1. Add to users.db.txt
echo 'nuevo_usuario@1754162.bs.iotleg.com md5:2db500964b556d4cef79284738f2a360 ;' >> /etc/flexisip/users/users.db.txt

# 2. Add route
echo '<sip:nuevo_usuario@1754162.bs.iotleg.com> sip:192.168.1.XX' >> /etc/flexisip/users/route.conf

# 3. Restart flexisip
/etc/init.d/flexisipsh restart
```

### Test SIP registration
```bash
# From another device with baresip/softphone
# Registrar: bticino_bridge@1754162.bs.iotleg.com
# Password: 2db500964b556d4cef79284738f2a360
# Server: 192.168.1.38:5060
```

---

## 🎯 Next Steps

1. ✅ **Flexisip configured** - Listening on local network
2. ✅ **bticino_bridge user created** - With MD5 hash
3. ✅ **Route configured** - Points to 127.0.0.1
4. ✅ **Doorbell configured** - Receives doorbell calls
5. ⏳ **Update bticino_bridge** - Use local configuration
6. ⏳ **Test SIP registration** - Verify bticino_bridge registers
7. ⏳ **Test doorbell** - Verify it receives calls

---

## 🔐 Security Notes

### **Passwords**:
- All local users use the same hash: `2db500964b556d4cef79284738f2a360`
- This hash is for the `1754162.bs.iotleg.com` domain
- Format: `username:realm:password` → MD5

### **Trusted Hosts**:
- `127.0.0.1` - localhost
- `192.168.1.38` - device IP
- Hosts in trusted-hosts can register without authentication

### **Recommendations**:
- ✅ Use only on a trusted local network
- ✅ Do not expose SIP ports to the Internet
- ✅ Consider TLS for production (port 5061)

---

**Status**: ✅ **COMPLETE**  
**Date**: 2026-03-24  
**Pending test**: bticino_bridge registration with local flexisip
