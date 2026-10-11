# SIP Debugging Report - Session 2026-03-24

**Goal**: identify the root cause of the SIP registration failure with local flexisip  
**Status**: ⚠️ **UNRESOLVED** - More research required

---

## 📋 Debug Summary

**24 tests** were run to identify why the bridge doesn't register with local flexisip.

### Key Findings:

1. ✅ **flexisip is running** and listening on ports 5060 (TCP) and 5061 (TLS)
2. ✅ **Network connectivity** verified (open ports)
3. ✅ **Authentication configuration** correct (trusted-hosts, hashed-passwords)
4. ✅ **Users exist** in users.db.txt (4 users, including baresip)
5. ❌ **The bridge reports `i/o timeout`** when sending REGISTER
6. ❌ **flexisip doesn't answer** the bridge's REGISTER
7. ❌ **The bridge process crashes** after starting (possible panic)

---

## 🔍 Tests Performed

### Tests 1-2: Network Connectivity
```bash
# Puertos SIP escuchando (5060=0x13C4, 5061=0x13C5):
0100007F:13C4 (127.0.0.1:5060) - ✅ LISTENING
2601A8C0:13C4 (192.168.1.38:5060) - ✅ LISTENING
2601A8C0:13C5 (192.168.1.38:5061) - ✅ LISTENING
```

**Result**: ✅ ports open correctly

---

### Tests 3-4: Debug Logs Enabled
```ini
# flexisip.conf
log-level=debug
syslog-level=debug

# config_streaming.yaml
log_level: debug
```

**Result**: ✅ detailed logs enabled on both services

---

### Tests 5-10: Authentication Configuration
```ini
# /etc/flexisip/flexisip.conf
[module::Authentication]
enabled=true
auth-domains=1754162.bs.iotleg.com
db-implementation=file
datasource=/etc/flexisip/users/users.db.txt
trusted-hosts=127.0.0.1,192.168.1.38
hashed-passwords=true
reject-wrong-client-certificates=false
```

**Available users**:
```
baresip@1754162.bs.iotleg.com md5:2db500964b556d4cef79284738f2a360
baresip_test@1754162.bs.iotleg.com plain:test123 (agregado en prueba 12)
```

**Result**: ✅ correct configuration

---

### Tests 11-14: Plain-Password Test
A `baresip_test` user with plain password `test123` was added to rule out MD5 hash problems.

**Configuration used**:
```yaml
sip:
  username: baresip_test@1754162.bs.iotleg.com
  password: test123
  use_ha1: false
```

**Error obtained**:
```
failed to register: failed to read REGISTER response: 
read tcp 192.168.1.38:34174->192.168.1.38:5060: i/o timeout
```

**Result**: ❌ same error - not a hash problem

---

### Tests 15-17: Bridge Process Check
The old process (PID 17923, started Mar23) was found still running.

**Action**: the old process was killed and a new one started.

**Result**: ⚠️ the new process crashes after starting

---

### Tests 18-24: Bridge Crash Analysis
The bridge starts correctly but crashes silently after ~30 seconds.

**Last logs before crashing**:
```
INFO[2026-03-24T10:12:13+01:00] HomeKit bridge created successfully
INFO[2026-03-24T10:12:13+01:00] HomeKit bridge started successfully
```

**There are no logs of**:
- ❌ "Starting BTicino SIP client"
- ❌ fatal errors or panic
- ❌ shutdown message

**Result**: ❌ the process disappears without a trace

---

## 🐛 Identified Problems

### Problem 1: `i/o timeout` on SIP Registration

**Symptom**:
```
failed to register: failed to read REGISTER response: 
read tcp 192.168.1.38:34174->192.168.1.38:5060: i/o timeout
```

**Possible causes**:
1. flexisip isn't processing incoming REGISTERs
2. The REGISTER is sent to a wrong IP/port
3. Internal firewall blocks the connection
4. flexisip requires additional REGISTER parameters

**Evidence**:
- flexisip is listening on 5060
- No flexisip logs of REGISTER received
- The timeout is ~10 seconds (TCP default)

---

### Problem 2: Bridge Process Crashes Silently

**Symptom**:
- Process starts normally
- Logs stop abruptly
- No error or panic message
- Process disappears from `ps aux`

**Possible causes**:
1. Uncaught Go panic
2. Segfault in C code (if any CGO)
3. Out of memory (OOM killer)
4. External signal (kill)

**Evidence**:
- No "fatal" or "panic" logs
- No core dump generated
- Available memory: ~425 MB (enough)

---

## 📊 Current Status

| Component | Status | Notes |
|------------|--------|-------|
| **flexisip** | ✅ Running | PID 11415, 11416 |
| **SIP ports** | ✅ Listening | 5060, 5061 |
| **Configuration** | ✅ Correct | Auth, users, domains |
| **Bridge** | ❌ Crashed | Crashes after ~30s |
| **SIP registration** | ❌ Failed | i/o timeout |
| **flexisip logs** | ❌ Empty | Shows no REGISTERs |

---

## 🔧 Recommended Next Steps

### 1. Check whether flexisip receives REGISTERs

```bash
# On the device, with tcpdump if available:
tcpdump -i lo -n port 5060 -X

# Or use strace on flexisip:
strace -p <PID_flexisip> -e trace=network
```

### 2. Run the bridge in foreground to see the panic

```bash
cd /home/bticino/cfg/extra
./bticino_bridge -config config_streaming.yaml
# Watch stdout for panics
```

### 3. Check OOM killer

```bash
dmesg | grep -i 'out of memory\|killed process'
cat /proc/meminfo | grep -i 'memavailable'
```

### 4. Try an alternative SIP client

```bash
# Use sipsak or pjsua to test registration:
sipsak -s sip:baresip@1754162.bs.iotleg.com -u baresip -p test123 -P tcp
```

### 5. Enable more detailed flexisip logs

```ini
# Add to flexisip.conf
[module::SipAg]
log-level=9
```

---

## 📁 Current Configuration Files

### /etc/flexisip/flexisip.conf
```ini
[global]
aliases=1754162.bs.iotleg.com
transports=sip:*:5060
log-level=debug
syslog-level=debug

[module::Authentication]
enabled=true
auth-domains=1754162.bs.iotleg.com
datasource=/etc/flexisip/users/users.db.txt
trusted-hosts=127.0.0.1,192.168.1.38
hashed-passwords=true
```

### /home/bticino/cfg/extra/config_streaming.yaml
```yaml
sip:
  enabled: true
  server_host: 192.168.1.38
  server_port: 5060
  transport: tcp
  domain: 1754162.bs.iotleg.com
  username: baresip_test@1754162.bs.iotleg.com
  password: test123
  use_ha1: false
```

---

## 📝 Conclusions

1. **flexisip is configured correctly** but doesn't answer REGISTERs
2. **The bridge has a bug** causing a silent crash after starting
3. **Network-level debugging (tcpdump) is required** to see whether REGISTER leaves the bridge
4. **The bridge needs to run in foreground** to capture the panic

**Next session**: run bridge in foreground + tcpdump to capture SIP traffic

---

**Date**: 2026-03-24  
**Tests run**: 24  
**Status**: ⏸️ Pending further debugging  
**Priority**: Medium (streaming is an optional feature)
