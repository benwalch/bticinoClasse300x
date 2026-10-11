# Real Deploy - Step-by-Step Instructions

> ⚠️ **SUPERSEDED (historical, v0.12.0).** The current deployment is
> `deploy-standard.sh` (repo root) / `make deploy` — see **[DEPLOY.md](DEPLOY.md)**.
> This document is based on `scripts/deploy_auto.sh` and manual startup, already
> replaced by `start-bridge.sh` (which also starts go2rtc). Kept
> for reference. Binary: `bticino_bridge` (underscore).

**Date**: 2026-03-23  
**Device**: BTicino Classe 300X (192.168.1.38)  
**Version**: v0.12.0 with WebRTC/RTSP

---

## ⚠️ PREREQUISITES

### 1. Accept the Device Host Key

The device may have a different host key. Run:

```bash
ssh-keygen -f '/home/fquinto/.ssh/known_hosts' -R '192.168.1.38'
```

Then connect manually to accept the new key:

```bash
ssh root@192.168.1.38
# When it asks "Are you sure you want to continue connecting (yes/no/[fingerprint])?"
# I typed: yes
```

If it asks for a password, you need to set it up. If you have the password, you can copy the key:

```bash
# If you have the device password:
ssh-copy-id root@192.168.1.38
# (te va a pedir password una vez)
```

---

## 📋 MANUAL DEPLOY STEPS

### Step 1: Verify SSH connection

```bash
ssh root@192.168.1.38 "echo 'SSH OK'"
```

**Expected**: `SSH OK`

---

### Step 2: Remount filesystem as read-write

```bash
ssh root@192.168.1.38 "mount -o remount,rw /"
```

---

### Step 3: Create recordings directory

```bash
ssh root@192.168.1.38 "mkdir -p /home/bticino/cfg/extra/recordings && chmod 755 /home/bticino/cfg/extra/recordings"
```

---

### Step 4: Transfer binary (BASE64 METHOD)

#### 4.1: Encode the binary on your development machine

```bash
cd /home/fquinto/DATOS/USB_Security/Seguridad/Dispositivos/Videoportero_classe300_interfono_bticino/bticino_bridge

# Codificar a base64
base64 bticino_bridge > /tmp/bticino_bridge.base64

# Check size
wc -l /tmp/bticino_bridge.base64
# Expected: ~256000 lines
```

#### 4.2: Transfer base64 to the device

**Option A: If scp works with the special options**

```bash
scp -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    /tmp/bticino_bridge.base64 \
    root@192.168.1.38:/home/bticino/cfg/extra/
```

**Option B: If scp does NOT work (most likely)**

Use this script, which transfers automatically:

```bash
cd /home/fquinto/DATOS/USB_Security/Seguridad/Dispositivos/Videoportero_classe300_interfono_bticino/bticino_bridge

# The script splits the file into 2MB chunks and transfers them one by one
./scripts/deploy_auto.sh
```

The script will:
1. Split the base64 into 10 2MB chunks
2. Transfer each chunk via SSH
3. Assemble and decode on the device
4. Verify the binary

**Estimated time**: 3-5 minutes

---

### Step 5: Verify transferred binary

```bash
ssh root@192.168.1.38 "ls -lh /home/bticino/cfg/extra/bticino_bridge"
```

**Expected**: `-rwxr-xr-x ... 14M ... bticino_bridge`

---

### Step 6: Check binary version

```bash
ssh root@192.168.1.38 "/home/bticino/cfg/extra/bticino_bridge -version"
```

**Expected**:
```
BTicino Classe 300X ENHANCED MEGA Bridge v0.12.0
🚀 ENHANCED UNIFIED BRIDGE - Real Device Integration:
  ✅ Enhanced OpenWebNet (50+ proven commands, real device analysis)
  🌐 Web Dashboard (Port 8082, Enhanced API endpoints)
  ...
```

---

### Step 7: Create configuration

```bash
ssh root@192.168.1.38
cd /home/bticino/cfg/extra

# Copy example
cp configs/config-streaming-example.yaml config.yaml

# Edit configuration (nano or vi if available)
nano config.yaml

# If nano is not available, use the echo method:
cat > config.yaml << 'EOF'
bridge:
  name: "BTicino Bridge Enhanced"
  log_level: "info"

openwebnet:
  host: "localhost"
  port: 30006

sip:
  enabled: true
  local_host: "192.168.1.38"
  server_host: "sipserver.bs.iotleg.com"
  server_port: 5061
  transport: "tls"
  domain: "bs.iotleg.com"
  username: "TU_USERNAME_AQUI"
  password: "TU_PASSWORD_AQUI"
  dev_addr: "20"
  use_ha1: false
  insecure_tls: true

mqtt:
  enabled: true
  host: "192.168.1.3"
  port: 1883
  username: "mqtt_user"
  password: "CHANGE_ME"
  topic_prefix: "bticino"

streaming:
  enabled: true
  rtsp_port: 6554
  recording_path: "/home/bticino/cfg/extra/recordings"
  max_duration: 60
  auto_stop_on_last_client: true
EOF
```

**IMPORTANT**: replace `TU_USERNAME_AQUI` and `TU_PASSWORD_AQUI` with your real SIP credentials.

---

### Step 8: Test the configuration

```bash
ssh root@192.168.1.38
cd /home/bticino/cfg/extra

# Modo test (no inicia servicios, solo verifica config)
./bticino_bridge -config config.yaml -test
```

**Expected**: configuration-loaded messages with no critical errors.

---

### Step 9: Start the service manually

```bash
ssh root@192.168.1.38
cd /home/bticino/cfg/extra

# Start in background
nohup ./bticino_bridge -config config.yaml > /var/log/bticino_bridge.log 2>&1 &

# Check that it is running
ps aux | grep bticino_bridge | grep -v grep
```

**Expected**: `./bticino_bridge -config config.yaml` process in the list.

---

### Step 10: Check logs

```bash
ssh root@192.168.1.38 "tail -50 /var/log/bticino_bridge.log"
```

**Expected** (look for these lines):
```
✅ SIP: Client started successfully
✅ RTSP: Enhanced server started on port 6554
   RTSP streams:
     - rtsp://192.168.1.38:6554/doorbell (Full stream)
     - rtsp://192.168.1.38:6554/doorbell-video (Video only)
     - rtsp://192.168.1.38:6554/doorbell-recorder (HKSV recording)
✅ Video: Stream manager started successfully
```

---

### Step 11: Test the Web API

```bash
# From your dev machine
curl http://192.168.1.38:8082/api/status
```

**Expected**: JSON with version, uptime, components.

---

### Step 12: Test RTSP streaming

```bash
# Con ffplay
ffplay -f rtsp -i rtsp://192.168.1.38:6554/doorbell

# Con VLC
vlc rtsp://192.168.1.38:6554/doorbell

# With go2rtc (if you have it configured)
# Check go2rtc logs
```

**Note**: the stream may take ~10-20 seconds to start because the device needs to establish the SIP call.

---

### Step 13: Test recording (HKSV)

```bash
# Start recording via API
curl -X POST http://192.168.1.38:8082/api/streaming/record \
  -H "Content-Type: application/json" \
  -d '{"duration": 10}'

# Esperar 10 segundos...

# Check recording file
ssh root@192.168.1.38 "ls -lh /home/bticino/cfg/extra/recordings/"
```

**Expected**: new timestamped `.ts` file.

---

## 🔧 TROUBLESHOOTING

### Problem: SSH keeps asking for password

**Solution**: set up SSH key-based auth

```bash
# On your dev machine
ssh-keygen -t rsa -b 4096 -f ~/.ssh/bticino_key

# Copy key to the device (if you have the password)
ssh-copy-id -o HostKeyAlgorithms=+ssh-rsa root@192.168.1.38

# O manualmente (si no funciona ssh-copy-id)
cat ~/.ssh/bticino_key.pub | ssh root@192.168.1.38 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

### Problem: Binary won't run

```bash
ssh root@192.168.1.38
cd /home/bticino/cfg/extra

# Check binary format
file bticino_bridge
# Expected: ELF 32-bit LSB executable, ARM

# Check permissions
ls -l bticino_bridge
# Expected: -rwxr-xr-x

# Si no es ejecutable:
chmod +x bticino_bridge

# Try running with debug
./bticino_bridge -log-level debug 2>&1 | head -20
```

---

### Problem: Port 6554 already in use

```bash
ssh root@192.168.1.38 "netstat -tlnp | grep 6554 || ss -tlnp | grep 6554"

# If another process is running, kill it
ssh root@192.168.1.38 "fuser -k 6554/tcp 2>/dev/null || kill \$(lsof -t -i:6554) 2>/dev/null || echo 'Manual kill required'"
```

---

### Problem: No video/audio on RTSP

1. **Check SIP is registered**:
   ```bash
   curl http://192.168.1.38:8082/api/status | grep -i sip
   ```

2. **Check SIP credentials in config.yaml**

3. **Try a manual SIP call**:
   ```bash
   ssh root@192.168.1.38
   cd /home/bticino/cfg/extra
   # Iniciar bridge en modo debug
   ./bticino_bridge -config config.yaml -log-level debug
   ```

---

### Problem: Recording doesn't work

```bash
# Check directory exists and is writable
ssh root@192.168.1.38 "ls -ld /home/bticino/cfg/extra/recordings"
ssh root@192.168.1.38 "touch /home/bticino/cfg/extra/recordings/test && rm /home/bticino/cfg/extra/recordings/test && echo 'Writable OK'"

# Check config
ssh root@192.168.1.38 "grep recording_path /home/bticino/cfg/extra/config.yaml"
```

---

## 📊 DEPLOY CHECKLIST

Mark each completed step:

- [ ] Host key accepted
- [ ] Passwordless SSH works
- [ ] Filesystem remounted as RW
- [ ] Recordings directory created
- [ ] Binary transferred (base64)
- [ ] Binary decoded and verified
- [ ] Version v0.12.0 confirmed
- [ ] Configuration created and edited
- [ ] Test mode passed
- [ ] Service started manually
- [ ] Logs checked (no critical errors)
- [ ] Web API responds
- [ ] RTSP stream works (ffplay/VLC)
- [ ] Recording works

---

## 🔄 ROLLBACK (if something goes wrong)

```bash
ssh root@192.168.1.38
cd /home/bticino/cfg/extra

# Stop current process
ps aux | grep bticino_bridge | grep -v grep | awk '{print $2}' | xargs kill -9 2>/dev/null || true

# Restaurar backup (si existe)
if ls bticino_bridge.backup.* 1>/dev/null 2>&1; then
    cp bticino_bridge.backup.* bticino_bridge
    echo "Backup restored"
else
    echo "No backup available"
fi

# Restaurar config (si existe)
if ls config.yaml.backup.* 1>/dev/null 2>&1; then
    cp config.yaml.backup.* config.yaml
    echo "Config restored"
fi

# Restart
./bticino_bridge -config config.yaml &
```

---

**Last updated**: 2026-03-23  
**Status**: lab-tested ✅  
**Next**: deploy on real device
