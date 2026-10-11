# Design spec — Live video in HA during an incoming call

**Date:** 2026-07-11
**Status:** SUPERSEDED in the activation mechanism (2026-08-18). The Phase 0 test
with a real doorbell ring showed that our own `*7*300` is ACKed but ignored during
a call (the native side holds both video slots with its SIP legs: type 0→5007,
type 1→5002). The current mechanism is a **passive mirror** (AF_PACKET tap on
`lo:5007`, cleartext H.264 RTP, → relay 10002) with ZERO bus commands — see
`pkg/sip/rtp_tap.go` and the Phase 0 result in the plan
(`docs/plans/2026-07-11-live-video-on-call.md`). The rest of the design (events,
privacy gating, relay→RTSP→go2rtc→HA) still stands.
**Author:** brainstorming session (BTicino bridge)
**Partially supersedes:** `docs/ROADMAP_VIDEO_AUDIO.md` (video phase)

## 1. Goal

When the **entrance panel** doorbell rings, show the door's **live video** in Home
Assistant (not just the existing snapshot), **without the official app** and
**without interfering** with native processes. The user keeps
answering/talking via panel or app; HA is for **seeing who's calling live**.

**Audio** is **out of scope** for this spec (see §12): the 2026-07-11 spike showed
the door only emits audio as RTP when a SIP party is in the call, which requires a
separate phase ("the bridge as SIP party").

## 2. Current state (existing infra being reused)

- **`EnhancedRTSPServer`** (`pkg/sip/rtsp_server_enhanced.go`, port 6554) with
  streams `/doorbell` (v+a), `/doorbell-video` (video only), `/doorbell-recorder`.
- **`RTPRelayPair`** (`pkg/sip/rtp_relay.go`): listens for RTP on **10002 (video)**
  and **10000 (audio)** and fans out to RTSP clients (go2rtc). Already works.
- **`Go2RTCManager`** (`pkg/sip/go2rtc_manager.go`): manages go2rtc, which pulls
  `rtsp://<ip>:6554/doorbell` and publishes it via WebRTC/HLS to HA.
- **Cooperative capture** (`pkg/sip/avmedia_capture.go`): proven pattern of sending
  **one** `*7*300` so bt_av_media duplicates its RTP to a local port. The
  snapshot-on-ring already uses it.
- **Call events on the bus**: `sip.call.incoming`, `sip.call.connected`,
  `sip.call.ended` (the snapshot already subscribes to `sip.call.incoming`).
- **Safety gate**: `EnhancedRTSPServer.videoActivation` (`streaming.video_on_demand`
  config, **disabled by default**). When false, the RTSP server listens but
  **triggers nothing** on the firmware.

### Why the current on-demand path is disabled
`ensureSIPCallActive()` does a **self-INVITE** (`MakeCall("c300x")`) to artificially
create a video session and then feeds it with `*7*300`/GStreamer. That self-INVITE
is what caused problems on the real device (v0.16.1), which is why
`video_on_demand` is false by default.

## 3. Spike findings (2026-07-11) behind the design

(Full detail in `_investigacion/docs/SPIKE_LISTEN_HA_2026-07-11.md`.)

- During an **incoming call**, bt_av_media **already produces continuous H.264 RTP
  video** (seen for 45 s straight). → **No self-INVITE needed.**
- Video is emitted **even while the call is ringing unanswered** (in the capture,
  RTP went to 5007/5002 with no consumer → 12,526 ICMP "port unreachable"). → Video
  for "seeing who's calling" can be obtained **before pickup**, which fits the use
  case. Only a copy needs redirecting to our port with type-1 `*7*300`.
- **Building-level privacy lock (key):** when someone rings a flat, the door panel
  stays **busy for the WHOLE building** until that flat hangs up or the timeout
  expires (a neighbour can't watch another flat's call on the camera). That's
  **by design**, not a fault. → The bridge **only** gets video when the call is
  **to our unit**; for calls to other flats the camera is busy (expected NACK) and
  **must not try to grab it** (privacy + building-level non-interference). In the
  spike capture, that "busy" state explains the RTP to dead ports. (Different from
  a genuinely hung session, see §11.5.)
- Type-1 **`*7*300`** duplicates video; **type 2** duplicates audio (Speex PT=110).
- **Audio** doesn't exist as RTP in panel-only answered calls (it goes through the
  internal DSP). → Video is cooperative and reliable; audio isn't. That's why this
  spec is **video-only**.

## 4. Scope

**Included:**
- Live panel video in HA for the duration of an incoming call.
- Automatic trigger on call event; stop on hangup.
- **Cooperative** path (type-1 `*7*300`), **no self-INVITE**.
- New independent, safe config gate.

**Excluded (future phases, §12):** audio (requires bridge-as-SIP-party),
bidirectional talk-back, "call home", on-demand video at rest (no call).

## 5. Proposed architecture

```
Timbre placa → monitor OWN (:20000) → evento sip.call.incoming / .connected
                                              │
                    ┌─────────────────────────┴───────────────────────┐
                    │ (ya existe) snapshot cooperativo                 │
                    │   → camera.doorbell_camera (MQTT base64)           │
                    │                                                  │
                    │ NUEVO: LiveVideoOnCall                           │
                    │   1. marca RTSP.callActive=true (evita self-INVITE)
                    │   2. *7*300 type1 (video) → 127.0.0.1:10002 [keepalive]
                    │   3. bt_av_media duplica su H.264 RTP → relay    │
                    └─────────────────────────┬───────────────────────┘
                                              │
                     RTPRelayPair (10002) ── fan-out ──> go2rtc (rtsp 6554/doorbell-video)
                                              │
                                         WebRTC/HLS → camera card in HA (LIVE)
                                              │
                    evento sip.call.ended → LiveVideoOnCall.Stop()
                                              (para keepalive; *7*0 opcional)
```

### Guiding principle
Reuse the **snapshot's cooperative mechanism**, but **sustained** for the call, and
**hooked to call events** instead of an RTSP client connection (which is what
triggers the self-INVITE today).

## 6. Components and responsibilities

### 6.1 `LiveVideoOnCall` (new, `pkg/sip/live_video.go`)
- **What it does:** orchestrates the live-video lifecycle during a call.
- **Interface:**
  - `Start(reason string) error` — marks the session active and starts the type-1
    `*7*300` keepalive toward the relay port (10002).
  - `Stop()` — stops the keepalive; the relay is left sourceless (go2rtc/HA
    consumers will see end of stream).
  - `IsActive() bool`.
- **Depends on:** `openwebnet.Client` (for `ActivateVideoStream`),
  `EnhancedRTSPServer` (to mark `callActive` and skip self-INVITE), a
  `logger`.
- **Keepalive:** re-send type-1 `*7*300` every N seconds (to validate, see §11).
  Sending stays **cooperative** (duplicate, don't compete) and respects BTicino's
  ~310 ms spacing.

### 6.2 Changes in `EnhancedRTSPServer`
- New `SetCallActive(bool)` method (or expose a flag) so `LiveVideoOnCall`
  marks that a native call is already in progress; then `ensureSIPCallActive()` sees
  `callActive==true` and **skips `MakeCall`** (existing line doing
  `if r.callActive { return nil }`).
- New `SetLiveOnCall(bool)` gate, independent of `videoActivation`.

### 6.3 Trigger (in `cmd/main.go` or wherever subscriptions are wired)
- Subscribe `LiveVideoOnCall.Start` to `sip.call.incoming` **and/or**
  `sip.call.connected` (idempotent: if already active, no-op).
- **Only for calls to OUR unit** (our c300x), not for generic bus activity
  or calls to other flats (see §11.5, building privacy).
- Subscribe `LiveVideoOnCall.Stop` to `sip.call.ended`.
- Gated by `cfg.Streaming.LiveOnCall`.

### 6.4 go2rtc
- The bridge must **take control** of the go2rtc config for the `doorbell` /
  `doorbell-video` streams (today there's a hand-made Companion-style `go2rtc.yaml`
  that `ensureConfigExists()` does **not** overwrite). Decision: the bridge
  generates/manages its config, or coexists explicitly. (See §11, pending decision.)

## 7. Configuration (`configs/config.yaml`)

```yaml
streaming:
  video_on_demand: false      # (existente) self-INVITE en reposo — sigue en false
  live_on_call: true          # NEW: live video during call (cooperative, safe)
  video_backend: avmedia      # (existente) *7*300 cooperativo
  live_keepalive_seconds: 5   # NEW: how often to re-arm *7*300 type1 (0 = once)
```

`live_on_call` can default to **true**: it uses exactly the same mechanism as
snapshot-on-ring (already on by default), no self-INVITE. Default decision to
confirm with the user.

## 8. Safety / non-interference (critical constraint)

- **Never** self-INVITE in this flow. Only `*7*300` (duplicate), like the snapshot.
- Monitor on :20000 only; commands via `nc` to localhost:30006 (existing pattern).
- Don't open sockets to 30006/30007/native video port.
- Respect ~310 ms spacing between OWN commands.
- With no active call, **no** `*7*300` is sent (idle video NACKs and must not be
  forced).
- **Building-level non-interference:** only enable video for calls to our
  unit. Never try to grab the camera if another flat holds it (§11.5) —
  privacy and not breaking the neighbour's call.

## 9. Home Assistant integration

- Live video is served via **go2rtc** (WebRTC/HLS), not MQTT. The
  `camera.doorbell_camera` entity keeps the **snapshot** (MQTT base64) as
  `still`/preview; the **live stream** is exposed via go2rtc.
- Document in `docs/HOME_ASSISTANT.md` how to point HA at the go2rtc stream
  (WebRTC at `http://<ip>:1984` or the `webrtc`/`go2rtc` card).
- Ideally: while the call lasts, the camera card plays the live feed;
  outside a call, it shows the last snapshot.

## 10. Error handling

- NACKed `*7*300` (no native session) → log + don't retry aggressively; live video
  simply doesn't appear (graceful degradation; the snapshot remains).
- go2rtc down → `Go2RTCManager` already monitors/restarts.
- Call ending without a `sip.call.ended` event → safety timeout in
  `LiveVideoOnCall` (auto-stop after N s, like `video.go`'s 60 s auto-stop).

## 11. Risks and pending validations (before/during implementation)

1. **Sustainability of type-1 `*7*300` toward OUR port (10002):** the spike
   confirmed the *native* side sustains video 45 s to *its own* ports, but not that
   a single `*7*300` of ours to 10002 lasts the whole call. **Phase-0 validation**
   (live test: send type-1 `*7*300` to 10002 during a call and measure how many
   seconds of RTP arrive). Defines whether keepalive is needed and its period.
2. **go2rtc.yaml ownership:** decide whether the bridge overwrites the current
   (hand-made/Companion) config or coexists. Risk of clobbering the backchannel
   config.
3. **Interaction with the snapshot:** snapshot and live video share the `*7*300`
   mechanism; verify they don't compete (sequence: snapshot first, then start the
   live feed, or share the same activation).
4. **`live_on_call` default:** true (convenient) vs false (conservative). To confirm.
5. **Camera busy with another call (building privacy) — expected:** if the
   panel is busy with a call to **another flat**, `*7*300` NACKs. That's
   **normal**, not an error. `LiveVideoOnCall` must: (a) arm **only** for calls
   to **our unit** (use our c300x's `sip.call.incoming` event, not
   generic bus activity), (b) treat the "busy" NACK as **graceful
   degradation** (no video, no log noise), and (c) **never** try to wrest the
   camera from another flat's call (building-level interference). The state
   frees itself when the other flat hangs up or on timeout.
6. **Genuinely hung session (rare):** dim 35 doesn't return to 0 after everyone hangs
   up. Different case from the above; optional safe cleanup (`*8*3#5#4*416##` +
   `*7*219*##`), manual or automatic with guards. Don't confuse with the privacy
   lock (which is normal and self-resolving).

## 12. Out of scope — future phases

- **Live audio (listening):** requires the bridge to be a SIP party in the
  call (auto-answer as audio endpoint) so the door emits Speex RTP;
  then transcode Speex→Opus with the device ffmpeg (`/usr/local/bin/ffmpeg`,
  has libspeex+libopus) → go2rtc. Reuses the standalone SIP account (200 OK
  verified) and existing INVITE auto-answer.
- **Talk-back (speaking):** upstream audio injection; `talkback.go` and the
  RE already exist, but it requires the bridge to own the call (SSRC collision
  if injected in parallel with the app).
- **"Call home":** the bridge originates a SIP call to the intercom (outgoing
  intercom). Its audio does NOT go through bt_av_media (confirmed in the spike).

## 13. Testing

- **Phase 0 (manual/live):** measure type-1 `*7*300` → 10002 sustainability during
  a real call (spike-style script, with minicap + packet/time counting).
- **Unit:** `LiveVideoOnCall` with a mock `openwebnet.Client` — verify
  Start/Stop arms/disarms the keepalive and Start is idempotent.
- **Integration (device):** real call → confirm in HA (go2rtc) that the live feed
  appears during the call and stops on hangup; confirm natives stay
  healthy (`ps` of bt_av_media/bt_vct/openserver before and after).
- **No-regression:** with `live_on_call:false`, behaviour identical to current.

## 14. Phase plan (summary)

- **Phase 0:** validate type-1 `*7*300` → 10002 sustainability (live test).
- **Phase 1:** `LiveVideoOnCall` + config gate + event subscription + `SetCallActive`.
- **Phase 2:** go2rtc ownership/management + HA docs.
- **Phase 3:** device tests + no-regression + CHANGELOG.
