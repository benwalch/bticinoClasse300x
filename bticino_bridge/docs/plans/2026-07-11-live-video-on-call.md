# Live video in HA during a call — Implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** show the door's live video in Home Assistant when a call comes in to our unit, cooperatively (`*7*300`, no self-INVITE), without the official app.

**Architecture:** a `LiveVideoOnCall` component subscribes to the bus call events (`sip.call.incoming`/`.connected`/`.ended`). When the call starts, it marks the RTSP server as "call active" (so it won't fire a self-INVITE) and sends a video `*7*300` (type 0) to the relay input port (10002). The existing relay fans out to go2rtc, which serves the live feed to HA via WebRTC. On hangup, it disarms.

**Tech Stack:** Go 1.x (ARM cross-compile), logrus, GStreamer/ffmpeg on device, go2rtc, OpenWebNet.

## Global Constraints

- **SAFETY — never re-send `*7*300` in a tight loop.** Retrying it in a loop against native state tripped the watchdog (reboot) on 2026-07-07. The `openwebnet.Client.ActivateVideoStream` primitive does a **single-shot send with no retry**; respect it. The default keepalive is **0 = single shot**; if >0, **minimum 5s** cadence.
- **Non-interference:** monitor on :20000 only; commands via `nc` to localhost:30006 (existing pattern). Don't open sockets to 30006/30007/video port. ~310 ms spacing between OWN commands. `VideoPort`/`ConfigPort` = 0.
- **No self-INVITE in this flow.** The native session already exists during an incoming call.
- **Building privacy:** enable video **only** for calls to OUR unit (the `sip.call.incoming` event, which is specific to our c300x). Never try to grab the camera if another flat holds it; the "busy" NACK is normal → degrade quietly.
- **Language:** comments and docs in Spanish (matches the edited file).
- **Relay input ports:** video `10002`, audio `10000` (`sip.DefaultGStreamerConfig()`).
- **Video primitive signature:** `func (c *openwebnet.Client) ActivateVideoStream(targetIP string, rtpPort int, highRes bool) error` — `highRes=true` → `*7*300#...#<port>#0*##` (type 0, hi-res).
- **Build/deploy:** `make build-go` (GOOS=linux GOARCH=arm GOARM=7); `./deploy-standard.sh full`. Go tests: `go test ./pkg/sip/ -run X -v`.
- **Audio out of scope** (needs bridge-as-SIP-party; see `docs/BACKLOG.md`).

---

### Task 1: Phase 0 — live validation (safety and functionality gate)

**Manual. Requires the real device + a doorbell call to our unit + the user.** It has no code-test cycle; its output fixes the `live_keepalive_seconds` default and confirms the approach is safe before enabling it on the device.

**Goal:** confirm that **ONE** type-0 `*7*300` toward `127.0.0.1:10002` during an incoming call makes bt_av_media duplicate H.264 video, and **how long it sustains** with a single command (the whole call? just a burst?).

- [x] **Step 1: Prepare the validation script on the device**

Create `/tmp/video_spike.sh` on the device (via ssh) — sends ONE type-0 `*7*300` to 10002 and measures with minicap how many seconds of video RTP arrive:

```sh
#!/bin/sh
PORT=10002; DUR=40; PCAP=/tmp/video_spike.pcap
rm -f "$PCAP"
nohup /home/bticino/cfg/extra/minicap -o "$PCAP" -t $((DUR+5)) >/tmp/mc.log 2>&1 &
sleep 1
echo "[phase0] *7*300 type0 (video, ONCE) -> 127.0.0.1:$PORT"
echo "*7*300#127#0#0#1#$PORT#0*##" | nc 0 30006
sleep $((DUR+3))
echo "[fase0] === minicap ==="; tail -2 /tmp/mc.log
```

- [x] **Step 2: Coordinate the call and run**

Ask the user: press the panel doorbell (call to OUR unit) and say "on call". Then: `ssh -o ClearAllForwardings=yes bticino 'sh /tmp/video_spike.sh'`.

- [x] **Step 3: Fetch the pcap and measure sustainability**

```bash
DEST=_investigacion/device_backups/video_spike_2026-07-11.pcap
ssh -o ClearAllForwardings=yes bticino 'base64 /tmp/video_spike.pcap' | base64 -d > "$DEST"
# time window of the video flow to 10002 (first/last timestamp):
tcpdump -nn -r "$DEST" 'udp and dst port 10002' 2>/dev/null | sed -n '1p;$p' | awk '{print $1}'
# nº de paquetes:
tcpdump -nn -r "$DEST" 'udp and dst port 10002' 2>/dev/null | wc -l
```

Expected: RTP packets to 10002 during the call. **Decision:**
- If the flow lasts **the whole call** with a single command → `live_keepalive_seconds: 0` (Task 4). **Preferred.**
- If it only lasts a burst (e.g. <10s) → note the duration; `live_keepalive_seconds` ≈ (burst_duration − 1s) will be used, **minimum 5s**. If the burst were <5s, **stop and rethink** (re-sending faster isn't safe).

- [x] **Step 4: Document the result**

Add the result (sustained duration, keepalive decision) to `_investigacion/docs/SPIKE_LISTEN_HA_2026-07-11.md`.

---

**PHASE 0 RESULT (2026-08-18): GATE FAILED — rethink.** With a real call, our
own `*7*300` is ACKed but ignored: the native side holds both video slots
(type 0→5007, type 1→5002, local SIP legs) at ring time. 0 packets
to 10002. In exchange, video flows IN THE CLEAR (H.264 RTP PT=96) over loopback to
5007 for the whole call (~205 pps sustained 45 s). **Proposed pivot:** passive tap
on `lo` (pcap `udp and dst port 5007`) → re-injection to 127.0.0.1:10002; zero bus
commands. Full detail: `_investigacion/docs/SPIKE_LISTEN_HA_2026-07-11.md`
(2026-08-18 section) and pcap
`_investigacion/device_backups/video_spike_2026-08-18.pcap`. Tasks 2-6 (already
implemented) keep their structure (events, config gate, SetCallActive, relay), but
`LiveVideoOnCall`'s activation mechanism must be replaced with the passive tap
(pending user decision).

---

### Task 2: `LiveVideoOnCall` component (TDD)

**Files:**
- Create: `pkg/sip/live_video.go`
- Test: `pkg/sip/live_video_test.go`

**Interfaces:**
- Consumes: nothing from previous tasks.
- Produces:
  - `type VideoActivator interface { ActivateVideoStream(targetIP string, rtpPort int, highRes bool) error }`
  - `type CallActiveMarker interface { SetCallActive(active bool) }`
  - `func NewLiveVideoOnCall(activator VideoActivator, marker CallActiveMarker, videoPort int, keepalive time.Duration, logger *logrus.Logger) *LiveVideoOnCall`
  - `func (l *LiveVideoOnCall) Start(reason string)` (idempotent, non-fatal)
  - `func (l *LiveVideoOnCall) Stop()`
  - `func (l *LiveVideoOnCall) IsActive() bool`
  - const `LiveVideoLocalHost = "127.0.0.1"`, `LiveVideoMinKeepalive = 5 * time.Second`

- [ ] **Step 1: Write the failing tests**

Create `pkg/sip/live_video_test.go`:

```go
package sip

import (
	"errors"
	"sync"
	"testing"

	"github.com/sirupsen/logrus"
)

type mockActivator struct {
	mu    sync.Mutex
	calls []struct {
		ip   string
		port int
		hi   bool
	}
	err error
}

func (m *mockActivator) ActivateVideoStream(ip string, port int, hi bool) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.calls = append(m.calls, struct {
		ip   string
		port int
		hi   bool
	}{ip, port, hi})
	return m.err
}

func (m *mockActivator) count() int {
	m.mu.Lock()
	defer m.mu.Unlock()
	return len(m.calls)
}

type mockMarker struct {
	mu     sync.Mutex
	states []bool
}

func (m *mockMarker) SetCallActive(a bool) {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.states = append(m.states, a)
}

func (m *mockMarker) last() (bool, bool) {
	m.mu.Lock()
	defer m.mu.Unlock()
	if len(m.states) == 0 {
		return false, false
	}
	return m.states[len(m.states)-1], true
}

func newTestLive(act VideoActivator, mk CallActiveMarker) *LiveVideoOnCall {
	// keepalive=0 → single send, deterministic (no timer goroutine).
	return NewLiveVideoOnCall(act, mk, 10002, 0, logrus.New())
}

func TestStartActivatesOnceAndMarks(t *testing.T) {
	act := &mockActivator{}
	mk := &mockMarker{}
	l := newTestLive(act, mk)

	l.Start("test")

	if !l.IsActive() {
		t.Fatal("esperaba IsActive()=true tras Start")
	}
	if act.count() != 1 {
		t.Fatalf("expected 1 activation, got %d", act.count())
	}
	c := act.calls[0]
	if c.ip != "127.0.0.1" || c.port != 10002 || c.hi != true {
		t.Fatalf("activation with wrong args: %+v", c)
	}
	if v, ok := mk.last(); !ok || v != true {
		t.Fatalf("esperaba marker SetCallActive(true), got ok=%v v=%v", ok, v)
	}
}

func TestStartIdempotent(t *testing.T) {
	act := &mockActivator{}
	l := newTestLive(act, &mockMarker{})
	l.Start("a")
	l.Start("b")
	if act.count() != 1 {
		t.Fatalf("expected 1 activation (idempotent), got %d", act.count())
	}
}

func TestStopDisarmsAndAllowsRestart(t *testing.T) {
	act := &mockActivator{}
	mk := &mockMarker{}
	l := newTestLive(act, mk)
	l.Start("a")
	l.Stop()
	if l.IsActive() {
		t.Fatal("esperaba IsActive()=false tras Stop")
	}
	if v, ok := mk.last(); !ok || v != false {
		t.Fatalf("esperaba marker SetCallActive(false), got ok=%v v=%v", ok, v)
	}
	l.Start("b")
	if act.count() != 2 {
		t.Fatalf("esperaba 2 activaciones tras re-Start, hubo %d", act.count())
	}
}

func TestActivateErrorIsNonFatal(t *testing.T) {
	act := &mockActivator{err: errors.New("NACK ocupado")}
	l := newTestLive(act, &mockMarker{})
	l.Start("busy") // must not panic
	if !l.IsActive() {
		t.Fatal("esperaba IsActive()=true aunque el activador devuelva error")
	}
	if act.count() != 1 {
		t.Fatalf("esperaba 1 intento, hubo %d", act.count())
	}
}

func TestKeepaliveClampedToMinimum(t *testing.T) {
	// keepalive below the minimum must be raised to the safe minimum.
	l := NewLiveVideoOnCall(&mockActivator{}, &mockMarker{}, 10002, 1, logrus.New())
	if l.keepalive != LiveVideoMinKeepalive {
		t.Fatalf("esperaba keepalive clamp a %s, got %s", LiveVideoMinKeepalive, l.keepalive)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd bticino_bridge && go test ./pkg/sip/ -run 'TestStart|TestStop|TestActivate|TestKeepalive' -v`
Expected: FAIL (compile: `LiveVideoOnCall`, `NewLiveVideoOnCall`, etc. undefined).

- [ ] **Step 3: Implement `live_video.go`**

Create `pkg/sip/live_video.go`:

```go
package sip

import (
	"context"
	"sync"
	"time"

	"github.com/sirupsen/logrus"
)

// VideoActivator abstracts sending the *7*300 video command (cooperative).
// Lo implementa *openwebnet.Client (ActivateVideoStream).
type VideoActivator interface {
	ActivateVideoStream(targetIP string, rtpPort int, highRes bool) error
}

// CallActiveMarker flags that a native call is already in progress, so
// the RTSP server does NOT fire a self-INVITE. Implemented by *EnhancedRTSPServer.
type CallActiveMarker interface {
	SetCallActive(active bool)
}

const (
	// LiveVideoLocalHost is the duplication target (loopback → relay).
	LiveVideoLocalHost = "127.0.0.1"
	// LiveVideoMinKeepalive is the MINIMUM safe cadence for *7*300 re-sends.
	// Re-sending faster can trigger the device watchdog (2026-07-07).
	LiveVideoMinKeepalive = 5 * time.Second
)

// LiveVideoOnCall arms the door's live video during an incoming call
// a NUESTRA unidad, de forma COOPERATIVA (*7*300 tipo 0, sin self-INVITE).
//
// SAFETY (critical): re-sending *7*300 in a tight loop rebooted the
// device (watchdog) on 2026-07-07. That's why the default mode is SINGLE SEND
// (keepalive=0). If keepalive>0, re-sends run at a slow cadence (min. 5s) and ALWAYS
// stop on hangup. Each tick uses ActivateVideoStream (single send, no retry).
type LiveVideoOnCall struct {
	activator VideoActivator
	marker    CallActiveMarker // puede ser nil
	videoPort int
	keepalive time.Duration
	logger    *logrus.Logger

	mu     sync.Mutex
	active bool
	cancel context.CancelFunc
}

// NewLiveVideoOnCall creates the orchestrator. keepalive=0 → single send (recommended).
// A keepalive>0 but < LiveVideoMinKeepalive is raised to the safe minimum.
func NewLiveVideoOnCall(activator VideoActivator, marker CallActiveMarker, videoPort int, keepalive time.Duration, logger *logrus.Logger) *LiveVideoOnCall {
	if logger == nil {
		logger = logrus.New()
	}
	if keepalive > 0 && keepalive < LiveVideoMinKeepalive {
		keepalive = LiveVideoMinKeepalive
	}
	return &LiveVideoOnCall{
		activator: activator,
		marker:    marker,
		videoPort: videoPort,
		keepalive: keepalive,
		logger:    logger,
	}
}

// Start arms live video. Idempotent: if already active, does nothing.
// Returns no error: a device NACK (e.g. camera busy with another flat)
// degrades gracefully (no video, no noise).
func (l *LiveVideoOnCall) Start(reason string) {
	l.mu.Lock()
	if l.active {
		l.mu.Unlock()
		return
	}
	if l.marker != nil {
		l.marker.SetCallActive(true)
	}
	l.active = true
	if l.keepalive > 0 {
		ctx, cancel := context.WithCancel(context.Background())
		l.cancel = cancel
		go l.keepaliveLoop(ctx)
	}
	l.mu.Unlock()

	l.logger.Infof("live-video: armando (motivo=%s, puerto=%d, keepalive=%s)", reason, l.videoPort, l.keepalive)
	l.activateOnce()
}

// activateOnce sends ONE *7*300 video frame (hi-res, type 0). Non-fatal errors.
func (l *LiveVideoOnCall) activateOnce() {
	if l.activator == nil {
		return
	}
	if err := l.activator.ActivateVideoStream(LiveVideoLocalHost, l.videoPort, true); err != nil {
		l.logger.WithError(err).Debug("live-video: *7*300 without ACK (camera busy?) — degrading without video")
	}
}

func (l *LiveVideoOnCall) keepaliveLoop(ctx context.Context) {
	ticker := time.NewTicker(l.keepalive)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			l.mu.Lock()
			active := l.active
			l.mu.Unlock()
			if !active {
				return
			}
			l.activateOnce()
		}
	}
}

// Stop disarms (stops the keepalive and releases the marker). Idempotent.
func (l *LiveVideoOnCall) Stop() {
	l.mu.Lock()
	if !l.active {
		l.mu.Unlock()
		return
	}
	l.active = false
	if l.cancel != nil {
		l.cancel()
		l.cancel = nil
	}
	l.mu.Unlock()

	if l.marker != nil {
		l.marker.SetCallActive(false)
	}
	l.logger.Info("live-video: desarmado")
}

// IsActive reports whether live video is armed.
func (l *LiveVideoOnCall) IsActive() bool {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.active
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd bticino_bridge && go test ./pkg/sip/ -run 'TestStart|TestStop|TestActivate|TestKeepalive' -v`
Expected: PASS (5 tests).

- [ ] **Step 5: `go vet` and ARM build**

Run: `cd bticino_bridge && go vet ./pkg/sip/ && make build-go`
Expected: no errors; ARM binary generated.

- [ ] **Step 6: Commit**

```bash
git add bticino_bridge/pkg/sip/live_video.go bticino_bridge/pkg/sip/live_video_test.go
git commit -m "feat(sip): LiveVideoOnCall — cooperative live video during call

Componente que arma *7*300 tipo 0 al puerto del relay durante una llamada
incoming call, no self-INVITE. Single send by default (keepalive=0) for safety
(loop re-sends triggered the watchdog on 2026-07-07). TDD with mocks.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: `EnhancedRTSPServer.SetCallActive` + reorder `ensureSIPCallActive`

**Files:**
- Modify: `pkg/sip/rtsp_server_enhanced.go` (add `SetCallActive` method; reorder guards in `ensureSIPCallActive`, ~lines 946-960)

**Interfaces:**
- Consumes: nothing.
- Produces: `func (r *EnhancedRTSPServer) SetCallActive(active bool)` — satisfies `sip.CallActiveMarker`.

- [ ] **Step 1: Add the `SetCallActive` method**

Add in `pkg/sip/rtsp_server_enhanced.go` (next to the other setters, e.g. after `SetVideoBackend`):

```go
// SetCallActive flags whether a native call is in progress. When true,
// ensureSIPCallActive() SKIPS the self-INVITE (the video session already exists).
// Used by LiveVideoOnCall during an incoming call.
func (r *EnhancedRTSPServer) SetCallActive(active bool) {
	r.mutex.Lock()
	r.callActive = active
	r.mutex.Unlock()
	if active {
		r.logger.Info("RTSP: callActive=true (self-INVITE skipped; native session in progress)")
	} else {
		r.logger.Info("RTSP: callActive=false")
	}
}
```

- [ ] **Step 2: Reorder `ensureSIPCallActive` (callActive BEFORE the videoActivation gate)**

Replace the start of `ensureSIPCallActive()` (the `if !r.videoActivation {...}` block followed by the `callActive` check) with:

```go
func (r *EnhancedRTSPServer) ensureSIPCallActive() error {
	// If a native call is already in progress (flagged by LiveVideoOnCall), NO
	// self-INVITE is needed: the video session already exists. Checked BEFORE
	// the videoActivation gate so the flow isn't blocked and logs stay clean when
	// go2rtc connects during a call.
	r.mutex.Lock()
	if r.callActive {
		r.mutex.Unlock()
		r.logger.Debug("SIP call already active (native call in progress), skipping self-INVITE")
		return nil
	}
	r.mutex.Unlock()

	// SAFETY: if on-demand video activation is disabled, we fire NOTHING
	// at the native device (neither self-INVITE nor *7*300).
	if !r.videoActivation {
		r.logger.Warn("Video on-demand disabled: ignoring RTSP/snapshot activation request")
		return fmt.Errorf("video on-demand activation is disabled")
	}

	r.logger.Info("=== ENSURESIP: Initiating SIP call for RTSP streaming ===")
	// ... (resto sin cambios: SIPTarget, MakeCall, activateMediaStreamsWhenConnected)
```

(Keep the rest of the original body from `target := r.sipClient.config.SIPTarget` on.)

- [ ] **Step 3: Verify build + vet**

Run: `cd bticino_bridge && go vet ./pkg/sip/ && go build ./...`
Expected: no errors. (No unit test: `ensureSIPCallActive` depends on a real `sipClient`; validated in the device integration, Task 7.)

- [ ] **Step 4: Verify `*EnhancedRTSPServer` satisfies `CallActiveMarker`**

Add a compile-time assertion at the end of `rtsp_server_enhanced.go`:

```go
// Compile-time assertion: the RTSP server satisfies CallActiveMarker.
var _ CallActiveMarker = (*EnhancedRTSPServer)(nil)
```

Run: `cd bticino_bridge && go build ./...`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add bticino_bridge/pkg/sip/rtsp_server_enhanced.go
git commit -m "feat(sip): SetCallActive en RTSP + skip self-INVITE si llamada nativa activa

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: Config — `live_on_call` and `live_keepalive_seconds`

**Files:**
- Modify: `pkg/config/config.go` (`StreamingConfig` struct ~line 160; defaults in `GetStreamingConfig` ~line 194)
- Modify: `configs/config.yaml` (`streaming:` section ~line 86)

**Interfaces:**
- Produces: `cfg.Streaming.LiveOnCall bool`, `cfg.Streaming.LiveKeepaliveSeconds int`.

- [ ] **Step 1: Add fields to the `StreamingConfig` struct**

After the `VideoOnDemand` field in `pkg/config/config.go`:

```go
	// LiveOnCall enables COOPERATIVE live video during a call
	// incoming call to our unit (*7*300 type 0, no self-INVITE). Safe: same
	// cooperative mechanism as the snapshot on ring. Independent of VideoOnDemand.
	LiveOnCall bool `yaml:"live_on_call"`
	// LiveKeepaliveSeconds: every how many seconds to re-arm the *7*300 video
	// during the call. 0 = SINGLE SEND (default, safest). If >0, the
	// effective minimum is 5s: re-sending in a tight loop rebooted the device on
	// 2026-07-07. Adjust only if Phase 0 shows a single send doesn't sustain
	// video for the whole call.
	LiveKeepaliveSeconds int `yaml:"live_keepalive_seconds"`
```

- [ ] **Step 2: Add defaults in `GetStreamingConfig`**

In `GetStreamingConfig()`'s `return StreamingConfig{...}`, add:

```go
		LiveOnCall:           true,
		LiveKeepaliveSeconds: 0,
```

(Default `LiveOnCall: true` because it's cooperative and safe, like snapshot-on-ring. Adjust `LiveKeepaliveSeconds` per Task 1.)

- [ ] **Step 3: Add to `configs/config.yaml`**

In the `streaming:` section, after `video_backend: "avmedia"`:

```yaml
  # Cooperative live video during an incoming call to OUR unit
  # (*7*300 tipo 0, sin self-INVITE). Seguro; independiente de video_on_demand.
  live_on_call: true
  # Re-arming *7*300 during the call. 0 = single send (recommended and
  # default). If >0, effective minimum 5s (tight-loop re-sends rebooted the
  # device on 2026-07-07). Adjust only if Phase 0 shows a single send
  # doesn't sustain video for the whole call.
  live_keepalive_seconds: 0
```

- [ ] **Step 4: Verify build**

Run: `cd bticino_bridge && go build ./... && go run ./cmd/main.go -test -config configs/config.yaml 2>&1 | head -5`
Expected: compiles; the `-test`-mode binary starts reading the config with no parse error.

- [ ] **Step 5: Commit**

```bash
git add bticino_bridge/pkg/config/config.go bticino_bridge/configs/config.yaml
git commit -m "feat(config): streaming.live_on_call + live_keepalive_seconds

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: Wiring in `main.go` (call-event subscription)

**Files:**
- Modify: `cmd/main.go` (after the `rtspServer` setup with `owClient`/`eventBus` available, ~line 500)

**Interfaces:**
- Consumes: `sip.NewLiveVideoOnCall`, `(*EnhancedRTSPServer).SetCallActive`, `sip.DefaultGStreamerConfig()`, `cfg.Streaming.LiveOnCall/LiveKeepaliveSeconds`, `sip.call.incoming/.connected/.ended` events.
- Produces: nothing (startup effect).

- [ ] **Step 1: Add the wiring**

After `rtspServer.SetVideoActivation(cfg.Streaming.VideoOnDemand)` in `cmd/main.go`:

```go
		// COOPERATIVE live video during an incoming call to our unit
		// (*7*300 type 0, no self-INVITE). Independent of video_on_demand.
		// `sip.call.incoming` is specific to our c300x (published by the SIP
		// client on receiving an INVITE addressed to us), not other flats' activity.
		if cfg.Streaming.LiveOnCall && owClient != nil {
			keepalive := time.Duration(cfg.Streaming.LiveKeepaliveSeconds) * time.Second
			liveVideo := sip.NewLiveVideoOnCall(owClient, rtspServer,
				sip.DefaultGStreamerConfig().VideoPort, keepalive, logger)

			eventBus.Subscribe("sip.call.incoming", func(_ *events.Event) {
				liveVideo.Start("sip.call.incoming")
			})
			eventBus.Subscribe("sip.call.connected", func(_ *events.Event) {
				liveVideo.Start("sip.call.connected")
			})
			eventBus.Subscribe("sip.call.ended", func(_ *events.Event) {
				liveVideo.Stop()
			})
			logger.Info("   Live video on call: habilitado (cooperativo, sin self-INVITE)")
		}
```

- [ ] **Step 2: Check imports**

Check that `cmd/main.go` imports `time` and `bticino_bridge/pkg/events` (already used in the file). If either is missing, add it.

Run: `cd bticino_bridge && go build ./...`
Expected: compiles with no errors.

- [ ] **Step 3: Verify startup in test mode**

Run: `cd bticino_bridge && go run ./cmd/main.go -test -config configs/config.yaml 2>&1 | grep -i "live video"`
Expected: `Live video on call: habilitado` line (if owClient initializes in test mode; otherwise at least it compiles and doesn't break startup).

- [ ] **Step 4: Commit**

```bash
git add bticino_bridge/cmd/main.go
git commit -m "feat(main): cablear LiveVideoOnCall a eventos sip.call.* (gateado por config)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 6: go2rtc + HA documentation

**Files:**
- Modify: `docs/HOME_ASSISTANT.md` (new section: live video during a call)
- Reference: `pkg/sip/go2rtc_manager.go` (`doorbell-video` config already exists: `rtsp://<ip>:6554/doorbell-video`)

**Interfaces:** none (config/ops/doc).

- [ ] **Step 1: Verify go2rtc points to `/doorbell-video`**

Confirm on the device that the go2rtc config includes a stream pulling
`rtsp://127.0.0.1:6554/doorbell-video` (video only, no audio — that's the scope).
If the current (hand-made/Companion) config lacks it or mixes in nonexistent audio,
add/adjust a `doorbell-video` stream:

```yaml
streams:
  doorbell-video: rtsp://127.0.0.1:6554/doorbell-video
```

Ownership decision (from spec §11.2): **don't overwrite** the existing Companion
config in this MVP; **add** the `doorbell-video` stream if missing, coexisting.
Document in the commit what was touched.

- [ ] **Step 2: Document in HA**

Add a "Live video during a call" section to `docs/HOME_ASSISTANT.md`:

```markdown
## Live video during a call (live_on_call)

When a call comes in to the unit, the bridge cooperatively activates the
door video and publishes it via go2rtc. To watch it live in HA:

- go2rtc exposes WebRTC at `http://<ip-device>:1984` (stream `doorbell-video`).
- In HA, use the `custom:webrtc-camera` card or the go2rtc integration pointing
  at the `doorbell-video` stream, or add a `generic`/`rtsp` camera at
  `rtsp://<ip-device>:6554/doorbell-video`.
- Outside a call there is no live video (that's normal): the `camera.doorbell_camera`
  entity keeps showing the last snapshot as a preview.

Only works for calls to **our** unit; if the door station is busy with a call
to another flat, there is no video (building privacy).
```

- [ ] **Step 3: Commit**

```bash
git add bticino_bridge/docs/HOME_ASSISTANT.md
git commit -m "docs(ha): live video during call (go2rtc doorbell-video)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 7: Device integration + no-regression + CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`, `VERSION`

**Interfaces:** none.

- [x] **Step 1: PRE health check**

```bash
ssh -o ClearAllForwardings=yes bticino 'ps | grep -E "bt_av_media|bt_vct|openserver" | grep -v grep'
```
Expected: the 3 native processes alive. Note PIDs.

- [x] **Step 2: Deploy**

Run: `./deploy-standard.sh full`
Expected: ARM build + deploy + health check OK; the bridge starts and logs `Live video on call: habilitado`.

- [x] **Step 3: Live test (with the user)** — real doorbell ring 17:04: armed via doorbell.pressed + 688×480 frame via go2rtc OK; caught a teardown panic (fixed with TDD). Disarm verified on the 2026-08-19 10:01 ring: full cycle arm (doorbell.pressed) → live frame via go2rtc (19.6 KB) → disarm on hangup (call.hangup, 27 s call, 2770 mirrored RTP packets, tap stopped cleanly), no panics and 2/2 natives. E2E CLOSED.

Ask the user for a doorbell call to our unit. Check the bridge logs:
```bash
ssh -o ClearAllForwardings=yes bticino 'tail -40 /tmp/bridge.log' | grep -iE "live-video|callActive|sip.call"
```
Expected: `live-video: armando`, `RTSP: callActive=true`. Open the camera in HA / go2rtc (`http://<ip>:1984`) during the call and confirm the live picture. On hangup: `live-video: desarmado`, `callActive=false`.

- [x] **Step 4: Native no-regression** — bt_vct/bt_av_media intact after 2 doorbell rings and 3 deploys (device uptime 39 days).

```bash
ssh -o ClearAllForwardings=yes bticino 'ps | grep -E "bt_av_media|bt_vct|openserver" | grep -v grep'
```
Expected: the 3 processes still alive (same or cleanly restarted, no watchdog). Confirm the unit did NOT reboot (stable uptime): `ssh ... 'uptime'`.

- [ ] **Step 5: Gate-off test (no-regression)**

Set `live_on_call: false` in the config, redeploy, and confirm behaviour returns to current (no call-triggered video activation). Restore to `true` afterwards.

- [x] **Step 6: CHANGELOG + VERSION + commit** — v0.18.0.

Add a `CHANGELOG.md` entry (new minor version) describing cooperative live video during calls, and bump `VERSION`.

```bash
git add bticino_bridge/CHANGELOG.md bticino_bridge/VERSION
git commit -m "chore: live video during call — CHANGELOG + version bump

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Execution notes

- **Recommended order:** Task 1 (Phase 0, safety gate) **first** — if a single shot doesn't sustain video and the safe keepalive (≥5s) isn't enough, rethink before writing more code. Tasks 2-6 are offline (code + unit + docs) and can advance in parallel while waiting for the call window for Task 1/7. Task 7 closes on device.
- **Out of scope (BACKLOG):** audio (bridge-as-SIP-party), talk-back, "call home", lock-based access classifier.

### Explicitly deferred from this MVP (with justification)
- **Safety-timeout auto-stop (spec §10):** if `sip.call.ended` never arrived, `active`/`callActive` would hang. Deferred because the `keepalive=0` default sends no repeated commands (the only effect of a missed hangup is that a future self-INVITE gets skipped), and `sip.call.ended` is published reliably from several points of the SIP client. If hung state is observed in integration (Task 7), add a guard `time.AfterFunc` in `Start` calling `Stop` after N minutes.
- **Genuinely-hung-session cleanup (spec §11.6):** rare case; not automated in the MVP. Documented manual commands: `*8*3#5#4*416##` + `*7*219*##`.
