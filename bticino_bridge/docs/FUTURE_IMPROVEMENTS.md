# Future Improvements Plan - BTicino Classe 300X Bridge

Current status: **v0.15.5** deployed and working (2026-04-04).
Last updated: 2026-04-04 — End-to-end working RTSP video streaming.

---

## v0.15.5 STATUS SUMMARY

### COMPLETED Features

| # | Feature | Version | Status |
|---|---------|---------|--------|
| 1 | MQTT Home Assistant (39+ entities, 8 command topics) | v0.10.0 | COMPLETE |
| 2 | OpenWebNet bus monitor (ports 20000/30006) | v0.10.0 | COMPLETE |
| 3 | GPIO monitoring (13 pins, real time) | v0.10.0 | COMPLETE |
| 4 | LED status publishing | v0.14.8 | COMPLETE |
| 5 | Answering machine (4-step protocol) | v0.10.0 | COMPLETE |
| 6 | Svelte Web Dashboard (Dashboard, Settings, Controls, Logs, Messages) | v0.14.2 | COMPLETE |
| 7 | Swagger UI API docs (/api/docs/) | v0.14.5 | COMPLETE |
| 8 | Device config parser (conf.xml, settings.xml, aswm, tvcc) | v0.14.0 | COMPLETE |
| 9 | Device config MQTT publisher + file watcher | v0.14.8 | COMPLETE |
| 10 | SSE (Server-Sent Events) for real-time updates | v0.15.0 | COMPLETE |
| 11 | Memos API (voice + text) | v0.15.0 | COMPLETE |
| 12 | Input monitor (keypad, touchscreen, GPIO events) | v0.10.0 | COMPLETE |
| 13 | Multicast listener (239.255.76.67:7667, OPEN handler) | v0.10.0 | COMPLETE |
| 14 | Dual-role SIP client (webrtc + c300x via local Flexisip) | v0.15.5 | COMPLETE |
| 15 | RTSP server (:6554) with H.264+Speex SDP | v0.15.5 | COMPLETE |
| 16 | Direct GStreamer pipelines (imxvpuenc_h264 VPU + alsasrc) | v0.15.5 | COMPLETE |
| 17 | RTP relay (UDP fan-out to RTSP clients) | v0.15.5 | COMPLETE |
| 18 | End-to-end video streaming (RTSP->SIP->GStreamer->RTP->client) | v0.15.5 | COMPLETE |

### v0.15.5 Totals
- **49 Home Assistant entities** (locks, switches, buttons, sensors, binary_sensors)
- **8 command topics** listening
- **RTSP streams**: `/doorbell`, `/doorbell-video`, `/doorbell-recorder`
- **Ports**: 8082 (web), 6554 (RTSP), 5060 (local SIP), 10000 (audio RTP), 10002 (video RTP)

---

## PENDING IMPROVEMENTS (ordered by priority)

### HIGH Priority — Video streaming improvements

#### A.1 Reduce packet loss in RTP relay
- **Problem**: ~7.5 Mbps actual vs 1.5 Mbps configured. The VPU doesn't respect the bitrate well with `idr-interval=1`.
- **Cause**: an IDR every 7 frames generates large I-frames that saturate WiFi.
- **Proposed solution**: try `idr-interval=0` with `gop-size=15` to reduce I-frames, or lower resolution to 352x288.
- **Impact**: smoother video, fewer artifacts.

#### A.2 TCP interleaved RTSP mode
- **Problem**: `handleConnection` uses `ReadString('\n')`, which breaks on interleaved binary RTP data after PLAY.
- **Solution**: after sending the PLAY response, switch the reader loop to read binary frames `$channel+length+data`.
- **Impact**: enables TCP streaming (more reliable than UDP over WiFi).

#### A.3 VPU/IPU hardware state recovery
- **Problem**: after many GStreamer open/close cycles, the VPU can get stuck (`mxc_v4l_dqueue timeout enc_counter 0`). It only recovers with a power cycle.
- **Proposed solution**: limit cycles, or reset the VPU between sessions via `/dev/mxc_vpu`.
- **Impact**: long-term stability.

#### A.4 Integrate with go2rtc / Home Assistant camera entity
- **Status**: documentation and scripts created, but not tested end-to-end with go2rtc.
- **Solution**: configure go2rtc in HA pointing to `rtsp://192.168.1.38:6554/doorbell`.
- **Impact**: video on the Home Assistant dashboard via WebRTC/HLS.

### HIGH Priority — Web Dashboard

#### B.1 Video Streaming page on the web
- **New**: dedicated page to watch live video from the web dashboard.
- **Implementation**: embed an HLS/MSE player consuming RTSP via proxy, or periodic JPEG snapshots.
- **Info to show**: SIP status (registered/connected/idle), GStreamer (running/stopped), RTP stats (packets, bytes, consumers), active RTSP sessions.
- **Controls**: Start/Stop stream button, quality selection.

#### B.2 Improved system status page
- **Show**: correct version (v0.15.5), uptime, memory, CPU, temperature.
- **RTP relay stats**: packets received/sent, active consumers, errors.
- **SIP status**: registration state, active call, duration.
- **GStreamer**: active pipeline, PID, duration, stderr errors.

#### B.3 Streaming configuration from the web
- **Editable parameters**: bitrate, GOP size, IDR interval, resolution.
- **Saving**: to config.yaml, reload without restarting the bridge.
- **Presets**: "High quality" (1500kbps, gop=7), "Low latency" (800kbps, gop=3), "Low bandwidth" (500kbps, gop=15).

### MEDIUM Priority — MQTT / Home Assistant

#### C.1 Real door state sensor
- **Command**: `*#1013**1##`
- **Already exists**: `QueryDoorStatus(doorID)` in client.go
- **Improvement**: the current lock is optimistic. A real query confirms the state.
- **IMPORTANT**: use `ExecuteCommand()` (netcat 30006), NOT `owClient.SendCommand()` (port 20000).

#### C.2 Camera entity MQTT discovery
- **New**: publish MQTT discovery for the HA camera entity.
- **Topic**: `homeassistant/camera/bticino_doorbell/config`
- **Payload**: Snapshot URL + RTSP stream URL.
- **Impact**: shows up automatically in HA as a camera.

#### C.3 Answering-machine bug: dual source of truth
- **Problem**: MQTTBridge uses `b.voicemailEnabled` (memory), but the web API uses `messageParser.GetAnsweringMachineStatus()` (filesystem). They go out of sync.
- **Recommended solution**: query the real device state after each command.
- **Check**: whether the same problem applies to `doorbell_sound` and `display`.

#### C.4 Secondary lock (door 2)
- **Commands**: `*8*19*11##` / `*8*20*11##`
- **Prerequisite**: verify whether a secondary door is physically connected.

### MEDIUM Priority — Technical quality

#### D.1 Unit tests
- **Only exists**: `pkg/events/bus_test.go` (3 tests).
- **Priority**: tests for `pkg/mqtt/bridge.go` (discovery configs), `pkg/messageparser/`, command handlers, `pkg/sip/rtp_relay.go`.

#### D.2 HomeKit
- **Status**: port 8081, PIN 12345678 (insecure), no real SRTP.
- **Options**: (a) fix with valid PIN and SRTP, (b) remove HomeKit code and use MQTT->HA->HomeKit bridge.
- **Recommendation**: option (b) — less maintenance.

#### D.3 Robust OWN reconnection
- **Problem**: the OWN client reconnects but may lose the `OnMessage` callback.
- **Solution**: watchdog validating events every N minutes.

### LOW Priority — Research

| Topic | Description |
|---|---|
| Serial port `/dev/ttymxc2` | Unknown purpose |
| Audio channels 0-25 | What each channel represents |
| Gateway WHO=13 | SCS gateway network queries |
| GPIO 47,52,54,56,176 | Always ON, identify what they represent |
| Button-to-Command mapping | Send OWN when physical buttons are pressed |

---

## CURRENT ARCHITECTURE v0.15.5

### Video streaming flow (WORKING)

```
Cliente RTSP (ffplay/VLC/go2rtc)
    │
    │ RTSP (TCP :6554)
    │ OPTIONS -> DESCRIBE -> SETUP (video+audio) -> PLAY
    ▼
bticino_bridge (RTSP server)
    │
    │ SIP INVITE (webrtc -> c300x via Flexisip local :5060)
    │ Auto-answer 200 OK -> ACK -> Connected
    ▼
GStreamer pipelines (lanzados por bridge):
    Video: imxv4l2videosrc ! imxvpuenc_h264 gop-size=7 idr-interval=1
           ! rtph264pay config-interval=1 ! udpsink 127.0.0.1:10002
    Audio: alsasrc hw:0 ! speexenc ! rtpspeexpay ! udpsink 127.0.0.1:10000
    │
    │ RTP (UDP localhost)
    ▼
RTP Relay (fan-out)
    │
    │ UDP unicast a cada cliente RTSP
    ▼
Cliente recibe H.264 720x576 ~7fps + Speex 8kHz mono
```

### Key technical findings

- **bt_av_media MQTT is unusable** from external publishers (bug in libjel.so routing).
- **Direct GStreamer works**: full bt_av_media bypass.
- **Camera**: /dev/video0, 720x576 UYVY interlaced PAL, ~7fps actual.
- **VPU encoder**: H.264 Constrained Baseline Level 3.0, byte-stream.
- **Dual-role SIP**: bridge registers as `webrtc` AND `c300x` in Flexisip. Self-INVITE.
- **ARM 32-bit**: atomic uint64s must be 64-bit aligned (fields first in struct).
- **Read-only root FS**: use `mount -oremount,rw /` for changes.

### Current SIP configuration

```yaml
sip:
  enabled: true
  server_host: "127.0.0.1"
  server_port: 5060
  transport: "tcp"
  domain: "2617372.bs.iotleg.com"
  username: "webrtc"
  c300x_username: "c300x"
```

- **webrtc HA1**: `<CHANGE_ME>` (MD5 of `username:realm:password`, see your flexisip.conf)
- **c300x HA1**: `<CHANGE_ME>`
- **Trusted-hosts**: `127.0.0.1` (no authentication for localhost)

### System ports

| Port | Protocol | Service | Notes |
|--------|-----------|----------|-------|
| 5060 | TCP | Local Flexisip SIP | localhost only |
| 5061 | TLS | External Flexisip SIP | Certificates |
| 6554 | TCP | RTSP server (bridge) | Video streaming |
| 8082 | TCP | Web dashboard (bridge) | Admin UI |
| 8081 | TCP | HomeKit (bridge) | Insecure PIN |
| 10000 | UDP | Audio RTP relay | GStreamer -> relay |
| 10002 | UDP | Video RTP relay | GStreamer -> relay |
| 20000 | TCP | OpenWebNet events | Read-only |
| 30006 | TCP | OpenWebNet commands | Via netcat |

### iptables (persisted in `/etc/network/if-pre-up.d/iptables`)

```bash
# BTicino Bridge ports
for i in 6554 8081 8082; do
    iptables -A INPUT -p tcp -m tcp --dport $i -j ACCEPT
done
iptables -A INPUT -p udp --dport 10000:10010 -j ACCEPT
```

---

## KEY PROJECT FILES

### Source code (pkg/sip/)
| File | Description |
|---------|-------------|
| `client.go` | Dual-role SIP client (webrtc + c300x), INVITE/BYE/REGISTER |
| `rtsp_server_enhanced.go` | RTSP server with RTP relay, SETUP/PLAY/TEARDOWN |
| `rtsp.go` | RTSP types (RTSPSession, RTSPRequest) |
| `rtp_relay.go` | UDP fan-out RTP relay, ARM-safe atomics |
| `gstreamer.go` | GStreamer pipeline launcher, VPU H.264 encoding |
| `video.go` | VideoStreamManager |

### Configuration
| File | Description |
|---------|-------------|
| `configs/config.yaml` | Main config (SIP, MQTT, streaming, web) |
| `VERSION` | Current version (0.15.5) |

### Documentation (docs/)
| File | Description | Status |
|---------|-------------|--------|
| `DEPLOYMENT_GUIDE.md` | Deploy guide | Updated |
| `DEVICE_COMMANDS_REFERENCE.md` | OpenWebNet command reference | Updated |
| `VIDEO_STREAMING_RESEARCH.md` | Video research (52K) | Updated v0.15.5 |
| `GO2RTC_INTEGRATION.md` | go2rtc integration | Pending verification |
| `HOMEKIT_INTEGRATION.md` | HomeKit (may be removed) | Obsolete |
| `FLEXISIP_LOCAL_CONFIG.md` | Local Flexisip config | Updated |
| `ha_config/` | YAML for Home Assistant | Updated |

---

## COMPARISON WITH SLYOLDFOX/C300X-CONTROLLER

| Feature | slyoldfox (Node.js) | bticino_bridge v0.15.5 (Go) |
|---------|---------------------|---------------------------|
| Video RTSP | RTSP relay server | RTSP + SIP + direct GStreamer |
| MQTT | Simple events | 49 HA discovery entities |
| Web UI | Basic dashboard | Modern Svelte (5 pages) |
| REST API | Limited | 40+ endpoints + Swagger |
| HomeKit | Experimental bundle | Basic (insecure PIN) |
| Deploy | Manual | Automated script |
| Language | Node.js | Go (more efficient on ARM) |
| Config sync | No | File watcher + MQTT publish |
| Device config | No | Full parser (conf.xml, settings) |
| Input monitor | No | Keypad + touchscreen + GPIO |

---

*Document created: 2026-03-11, v0.9.0*
*Rewritten: 2026-04-04, v0.15.5 — Actual project status with prioritized improvements*
