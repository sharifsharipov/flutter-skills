# Native C++ Call Engine — Implementation Guide

Migration reference for moving Sarbon's call subsystem from `flutter_webrtc` to a native C++/NDK core running in a separate Android process (`:callengine`).

> **Version note:** libwebrtc has no stable public API. Every snippet below targets roughly **m120–m132**. Pin an exact commit in `fetch_libwebrtc.sh` and expect signature drift if you move branches. When something doesn't compile, read the header — there is no reliable online API doc for libwebrtc.

---

## 1. Architecture

```
┌───────────────────────────────────────────────┐
│ UI process (Flutter / Dart)                   │
│   voice_call page  →  CallEngineChannel(Dart) │
└──────────────────┬────────────────────────────┘
                   │ MethodChannel
┌──────────────────▼────────────────────────────┐
│ Kotlin (UI process)  — thin AIDL client       │
└──────────────────┬────────────────────────────┘
                   │ AIDL / Binder  (process boundary)
┌──────────────────▼────────────────────────────┐
│ :callengine process                           │
│  CallEngineService (Kotlin, Foreground)       │
│              │ JNI                            │
│  ┌───────────▼─────────────────────────────┐  │
│  │ C++ core                                │  │
│  │  CallEngine (state machine)             │  │
│  │  PeerConnectionFactory                  │  │
│  │  AudioDeviceModule (AAudio)             │  │
│  │  CameraCapturer (Camera2 NDK)           │  │
│  │  SignalingClient (WebSocket)            │  │
│  │  QualityController (adaptive bitrate)   │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
```

**Rule:** media never crosses into Dart. Frames go camera → C++ → encoder → network, and network → decoder → C++ → `ANativeWindow` (Flutter `Texture`). Dart only sees state enums.

---

## 2. Threading model

libwebrtc requires three threads. Create them once, keep them alive for the process lifetime.

```cpp
// call_engine.h
#pragma once
#include <memory>
#include <api/peer_connection_interface.h>
#include <rtc_base/thread.h>

class CallEngine {
 public:
  static CallEngine& Instance();
  bool Initialize();

 private:
  std::unique_ptr<rtc::Thread> network_thread_;
  std::unique_ptr<rtc::Thread> worker_thread_;
  std::unique_ptr<rtc::Thread> signaling_thread_;
  rtc::scoped_refptr<webrtc::PeerConnectionFactoryInterface> factory_;
  rtc::scoped_refptr<webrtc::PeerConnectionInterface> pc_;
};
```

```cpp
// call_engine.cc
bool CallEngine::Initialize() {
  network_thread_ = rtc::Thread::CreateWithSocketServer();
  network_thread_->SetName("ce_network", nullptr);
  network_thread_->Start();

  worker_thread_ = rtc::Thread::Create();
  worker_thread_->SetName("ce_worker", nullptr);
  worker_thread_->Start();

  signaling_thread_ = rtc::Thread::Create();
  signaling_thread_->SetName("ce_signaling", nullptr);
  signaling_thread_->Start();

  // Raise audio-critical thread priority. See §9 — this can silently fail.
  RaiseThreadPriority(worker_thread_.get());
  return true;
}
```

**Thread rules that will bite you:**
- All `PeerConnectionInterface` calls must happen on the **signaling thread**. From JNI you are on a Binder/JVM thread → always hop:
  ```cpp
  signaling_thread_->BlockingCall([this] { pc_->Close(); });
  // or non-blocking:
  signaling_thread_->PostTask([this] { /* ... */ });
  ```
- Never block the signaling thread on network I/O. Ever.
- Observer callbacks arrive on the signaling thread. Don't do heavy work there — post to your own queue.

---

## 3. PeerConnectionFactory

```cpp
#include <api/create_peerconnection_factory.h>
#include <api/audio_codecs/builtin_audio_decoder_factory.h>
#include <api/audio_codecs/builtin_audio_encoder_factory.h>
#include <api/video_codecs/builtin_video_decoder_factory.h>
#include <api/video_codecs/builtin_video_encoder_factory.h>
#include <api/task_queue/default_task_queue_factory.h>
#include <modules/audio_device/include/audio_device.h>
#include <modules/audio_processing/include/audio_processing.h>

bool CallEngine::CreateFactory() {
  auto task_queue_factory = webrtc::CreateDefaultTaskQueueFactory();

  // AAudio path — low latency, NDK-native. Falls back to OpenSL ES internally
  // on devices where AAudio is unavailable (API < 26).
  rtc::scoped_refptr<webrtc::AudioDeviceModule> adm =
      worker_thread_->BlockingCall([&] {
        return webrtc::AudioDeviceModule::Create(
            webrtc::AudioDeviceModule::kAndroidAAudioAudio,
            task_queue_factory.get());
      });

  // Audio processing: AEC / NS / AGC. Tune these — defaults are for conference
  // calls, not for a driver in a truck cabin.
  webrtc::AudioProcessing::Config apm_config;
  apm_config.echo_canceller.enabled = true;
  apm_config.echo_canceller.mobile_mode = true;   // cheaper AEC for mobile
  apm_config.noise_suppression.enabled = true;
  apm_config.noise_suppression.level =
      webrtc::AudioProcessing::Config::NoiseSuppression::kHigh;  // road noise
  apm_config.gain_controller1.enabled = true;
  apm_config.high_pass_filter.enabled = true;

  rtc::scoped_refptr<webrtc::AudioProcessing> apm =
      webrtc::AudioProcessingBuilder().Create();
  apm->ApplyConfig(apm_config);

  factory_ = webrtc::CreatePeerConnectionFactory(
      network_thread_.get(),
      worker_thread_.get(),
      signaling_thread_.get(),
      adm,
      webrtc::CreateBuiltinAudioEncoderFactory(),
      webrtc::CreateBuiltinAudioDecoderFactory(),
      webrtc::CreateBuiltinVideoEncoderFactory(),   // see §6 for HW encoders
      webrtc::CreateBuiltinVideoDecoderFactory(),
      nullptr /* audio_mixer */,
      apm);

  return factory_ != nullptr;
}
```

---

## 4. PeerConnection + transceivers (the fix for your 0×0 texture bug)

Your existing bug — *"audio-to-video mid-call upgrade, renderer texture stuck at 0×0"* — happens because the video transceiver doesn't exist when the call starts as audio-only, so upgrading requires renegotiation, and the renderer never gets a size.

**Fix: always create both transceivers up front**, even for an audio-only call. Start the video track disabled. Upgrading to video then requires **no SDP renegotiation** — you just enable the track.

```cpp
#include <api/peer_connection_interface.h>

bool CallEngine::CreatePeerConnection() {
  webrtc::PeerConnectionInterface::RTCConfiguration config;
  config.sdp_semantics = webrtc::SdpSemantics::kUnifiedPlan;
  config.bundle_policy = webrtc::PeerConnectionInterface::kBundlePolicyMaxBundle;
  config.rtcp_mux_policy = webrtc::PeerConnectionInterface::kRtcpMuxPolicyRequire;
  config.continual_gathering_policy =
      webrtc::PeerConnectionInterface::GATHER_CONTINUALLY;  // survives net switch

  webrtc::PeerConnectionInterface::IceServer stun;
  stun.uris = {"stun:stun.l.google.com:19302"};
  config.servers.push_back(stun);

  webrtc::PeerConnectionInterface::IceServer turn;
  turn.uris = {"turn:turn.sarbon.me:3478?transport=udp"};
  turn.username = turn_user_;
  turn.password = turn_pass_;
  config.servers.push_back(turn);

  webrtc::PeerConnectionDependencies deps(this /* PeerConnectionObserver */);
  auto result = factory_->CreatePeerConnectionOrError(config, std::move(deps));
  if (!result.ok()) {
    LOGE("CreatePeerConnection failed: %s", result.error().message());
    return false;
  }
  pc_ = result.MoveValue();

  CreateTracksAndTransceivers();
  CreateMediaStateDataChannel();
  return true;
}
```

```cpp
void CallEngine::CreateTracksAndTransceivers() {
  // ---- AUDIO ----
  cricket::AudioOptions audio_opts;
  audio_opts.echo_cancellation = true;
  audio_opts.noise_suppression = true;
  audio_opts.auto_gain_control = true;
  auto audio_source = factory_->CreateAudioSource(audio_opts);
  audio_track_ = factory_->CreateAudioTrack("ce_audio", audio_source.get());

  webrtc::RtpTransceiverInit audio_init;
  audio_init.direction = webrtc::RtpTransceiverDirection::kSendRecv;
  audio_init.stream_ids = {"ce_stream"};
  auto audio_tr = pc_->AddTransceiver(audio_track_, audio_init);
  audio_sender_ = audio_tr.value()->sender();

  // ---- VIDEO (always created, even for an audio call) ----
  video_source_ = rtc::make_ref_counted<CameraVideoSource>();  // §5
  video_track_ = factory_->CreateVideoTrack("ce_video", video_source_.get());
  video_track_->set_enabled(false);          // audio-only start: muted, not absent

  webrtc::RtpTransceiverInit video_init;
  video_init.direction = webrtc::RtpTransceiverDirection::kSendRecv;
  video_init.stream_ids = {"ce_stream"};
  auto video_tr = pc_->AddTransceiver(video_track_, video_init);
  video_sender_ = video_tr.value()->sender();

  ApplyVideoQualityProfile(QualityProfile::kMedium);  // §7
}

// Mid-call audio → video upgrade. No renegotiation, no 0x0 texture.
void CallEngine::EnableVideo(bool enable) {
  signaling_thread_->PostTask([this, enable] {
    video_track_->set_enabled(enable);
    if (enable) camera_->Start(); else camera_->Stop();
    SendMediaState();                       // notify peer over data channel
    NotifyUi(CallEvent::kLocalVideoChanged, enable);
  });
}
```

---

## 5. Video capture — Camera2 NDK → WebRTC

Implement `rtc::AdaptedVideoTrackSource`. The `Adapted` base gives you free CPU/bandwidth downscaling — WebRTC calls `AdaptFrame()` and tells you what resolution it actually wants.

```cpp
// camera_video_source.h
#include <media/base/adapted_video_track_source.h>
#include <api/video/i420_buffer.h>

class CameraVideoSource : public rtc::AdaptedVideoTrackSource {
 public:
  // Called from the Camera2 NDK capture callback thread.
  void OnCapturedFrame(const uint8_t* y, int y_stride,
                       const uint8_t* u, int u_stride,
                       const uint8_t* v, int v_stride,
                       int width, int height,
                       int64_t timestamp_us,
                       webrtc::VideoRotation rotation) {
    int adapted_w, adapted_h, crop_w, crop_h, crop_x, crop_y;
    // Lets WebRTC drop / downscale frames under CPU or bandwidth pressure.
    if (!AdaptFrame(width, height, timestamp_us,
                    &adapted_w, &adapted_h,
                    &crop_w, &crop_h, &crop_x, &crop_y)) {
      return;  // frame dropped on purpose
    }

    rtc::scoped_refptr<webrtc::I420Buffer> buffer =
        webrtc::I420Buffer::Create(adapted_w, adapted_h);
    // libyuv scale+crop from the captured planes into `buffer` here.
    // (webrtc::I420Buffer::CropAndScaleFrom or libyuv::I420Scale)

    webrtc::VideoFrame frame = webrtc::VideoFrame::Builder()
        .set_video_frame_buffer(buffer)
        .set_rotation(rotation)          // §8 — CVO depends on this
        .set_timestamp_us(timestamp_us)
        .build();

    OnFrame(frame);                       // hands it to the encoder
  }

  // --- VideoTrackSourceInterface ---
  bool is_screencast() const override { return is_screencast_; }
  absl::optional<bool> needs_denoising() const override { return false; }
  webrtc::MediaSourceInterface::SourceState state() const override {
    return webrtc::MediaSourceInterface::kLive;
  }
  bool remote() const override { return false; }

  void SetScreencast(bool v) { is_screencast_ = v; }

 private:
  bool is_screencast_ = false;   // true for screen share — changes encoder tuning
};
```

**Why `is_screencast()` matters:** when true, WebRTC's encoder switches to a screen-content tuning (favours detail over framerate). This is the C++ equivalent of the web's `contentHint = 'detail'`. For screen sharing, set it to `true` **and** use `MAINTAIN_RESOLUTION` (§7).

---

## 6. Rendering to Flutter — VideoSink → ANativeWindow

Flutter's `Texture` widget is backed by a `SurfaceTexture`. Pass the `Surface` across JNI, get an `ANativeWindow`, and write decoded frames into it directly. **Dart never touches a frame.**

```cpp
// video_renderer.h
#include <api/video/video_sink_interface.h>
#include <api/video/video_frame.h>
#include <android/native_window.h>
#include <android/native_window_jni.h>
#include <libyuv.h>

class SurfaceVideoRenderer
    : public rtc::VideoSinkInterface<webrtc::VideoFrame> {
 public:
  SurfaceVideoRenderer(JNIEnv* env, jobject surface) {
    window_ = ANativeWindow_fromSurface(env, surface);
  }
  ~SurfaceVideoRenderer() override {
    if (window_) ANativeWindow_release(window_);
  }

  void OnFrame(const webrtc::VideoFrame& frame) override {
    if (!window_) return;

    const int w = frame.width();
    const int h = frame.height();

    // Report size changes up to the UI — this is what drives aspect-ratio
    // handling in Flutter (see the aspect-ratio work item).
    if (w != last_w_ || h != last_h_) {
      last_w_ = w; last_h_ = h;
      NotifyUiFrameSize(w, h, frame.rotation());
      ANativeWindow_setBuffersGeometry(window_, w, h, WINDOW_FORMAT_RGBA_8888);
    }

    ANativeWindow_Buffer buf;
    if (ANativeWindow_lock(window_, &buf, nullptr) != 0) return;

    auto i420 = frame.video_frame_buffer()->ToI420();
    libyuv::I420ToABGR(
        i420->DataY(), i420->StrideY(),
        i420->DataU(), i420->StrideU(),
        i420->DataV(), i420->StrideV(),
        static_cast<uint8_t*>(buf.bits), buf.stride * 4,
        w, h);

    ANativeWindow_unlockAndPost(window_);
  }

 private:
  ANativeWindow* window_ = nullptr;
  int last_w_ = 0, last_h_ = 0;
};
```

Attach it when the remote track arrives:

```cpp
void CallEngine::OnTrack(
    rtc::scoped_refptr<webrtc::RtpTransceiverInterface> transceiver) {
  auto track = transceiver->receiver()->track();
  if (track->kind() == webrtc::MediaStreamTrackInterface::kVideoKind) {
    remote_video_track_ =
        static_cast<webrtc::VideoTrackInterface*>(track.get());
    remote_video_track_->AddOrUpdateSink(remote_renderer_.get(), rtc::VideoSinkWants());
  }
  // Audio needs no sink — the ADM plays it automatically.
}
```

> **Note (perf):** the `I420ToABGR` + `ANativeWindow_lock` path is a CPU copy. It works and it's simple, but on weak devices consider an OpenGL ES / Vulkan path that uploads the Y/U/V planes as textures and converts in a fragment shader. Start with the CPU path, measure, optimise if needed.

---

## 7. Adaptive quality — the real fix for blurry video

Three separate knobs. People usually turn only one and wonder why nothing improves.

```cpp
enum class QualityProfile { kLow, kMedium, kHigh, kScreenShare };

void CallEngine::ApplyVideoQualityProfile(QualityProfile p) {
  auto params = video_sender_->GetParameters();
  if (params.encodings.empty()) return;
  auto& enc = params.encodings[0];

  switch (p) {
    case QualityProfile::kLow:      // 320x240 @15
      enc.max_bitrate_bps = 150'000;
      enc.max_framerate = 15;
      enc.scale_resolution_down_by = 4.0;
      params.degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_FRAMERATE;
      break;

    case QualityProfile::kMedium:   // 640x480 @24
      enc.max_bitrate_bps = 500'000;
      enc.max_framerate = 24;
      enc.scale_resolution_down_by = 2.0;
      params.degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_FRAMERATE;
      break;

    case QualityProfile::kHigh:     // 1280x720 @30
      enc.max_bitrate_bps = 1'500'000;
      enc.max_framerate = 30;
      enc.scale_resolution_down_by = 1.0;
      params.degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_FRAMERATE;
      break;

    case QualityProfile::kScreenShare:
      // Text must stay readable. 5 sharp fps beats 30 blurry fps.
      enc.max_bitrate_bps = 2'500'000;
      enc.max_framerate = 15;
      enc.scale_resolution_down_by = 1.0;
      params.degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_RESOLUTION;   // <-- the key line
      video_source_->SetScreencast(true);
      video_track_->set_content_hint(
          webrtc::VideoTrackInterface::ContentHint::kDetailed);
      break;
  }

  webrtc::RTCError err = video_sender_->SetParameters(params);
  if (!err.ok()) LOGE("SetParameters failed: %s", err.message());
  current_profile_ = p;
}
```

**No renegotiation happens here.** `SetParameters` changes the encoder live — that's why this is the right mechanism for mid-call adaptation, and why an SDP-renegotiation approach would tear the call down.

### Quality controller (hysteresis)

Poll stats, decide, but **never oscillate**. Asymmetric thresholds + a hold-down timer.

```cpp
class QualityController {
 public:
  // Called every ~1s with fresh stats.
  void OnStats(const CallStats& s) {
    // Down: react fast (1–2s).
    if (s.quality_limitation == "bandwidth" || s.quality_limitation == "cpu" ||
        s.available_outgoing_bitrate < DownThreshold(current_)) {
      if (++down_ticks_ >= 2) { StepDown(); Reset(); }
      up_ticks_ = 0;
      return;
    }

    // Up: react slowly (5–10s of sustained headroom).
    if (s.quality_limitation == "none" &&
        s.available_outgoing_bitrate > UpThreshold(current_) &&
        s.packets_lost_fraction < 0.02) {
      if (++up_ticks_ >= 8) { StepUp(); Reset(); }
      down_ticks_ = 0;
      return;
    }
    Reset();
  }

 private:
  // Asymmetric: go up only above 1.2x, come down below 0.8x. Prevents flapping
  // at the boundary.
  int UpThreshold(QualityProfile p)   { return BitrateOf(Next(p)) * 1.2; }
  int DownThreshold(QualityProfile p) { return BitrateOf(p) * 0.8; }
  void Reset() { up_ticks_ = down_ticks_ = 0; }

  QualityProfile current_ = QualityProfile::kMedium;
  int up_ticks_ = 0, down_ticks_ = 0;
};
```

### Reading stats

```cpp
class StatsCallback : public webrtc::RTCStatsCollectorCallback {
 public:
  void OnStatsDelivered(
      const rtc::scoped_refptr<const webrtc::RTCStatsReport>& report) override {
    CallStats out;
    for (const auto& s : *report) {
      if (s.type() == std::string("outbound-rtp")) {
        auto& r = s.cast_to<webrtc::RTCOutboundRtpStreamStats>();
        if (r.kind.value_or("") != "video") continue;
        out.frame_width  = r.frame_width.value_or(0);
        out.frame_height = r.frame_height.value_or(0);
        out.fps          = r.frames_per_second.value_or(0);
        // "none" | "cpu" | "bandwidth" | "other"
        out.quality_limitation = r.quality_limitation_reason.value_or("none");
      } else if (s.type() == std::string("candidate-pair")) {
        auto& r = s.cast_to<webrtc::RTCIceCandidatePairStats>();
        if (!r.nominated.value_or(false)) continue;
        out.available_outgoing_bitrate = r.available_outgoing_bitrate.value_or(0);
        out.rtt_ms = r.current_round_trip_time.value_or(0) * 1000;
      } else if (s.type() == std::string("remote-inbound-rtp")) {
        auto& r = s.cast_to<webrtc::RTCRemoteInboundRtpStreamStats>();
        out.packets_lost_fraction = r.fraction_lost.value_or(0.0);
      }
    }
    quality_->OnStats(out);
    NotifyUiStats(out);              // dev overlay
  }
};

// Poll every second:
void CallEngine::PollStats() {
  pc_->GetStats(stats_callback_.get());
}
```

> **`quality_limitation_reason` is the single most useful field in this whole document.** If it says `none` and the video is still blurry, the bottleneck is *your own* constraints, not the network or the CPU — stop tuning bitrate and go look at your capture constraints and your renderer's aspect-ratio handling.

---

## 8. Offer / answer / ICE — wiring to your existing WebSocket signaling

Your server relays only `webrtc.offer | webrtc.answer | webrtc.ice`. Keep that contract.

```cpp
// --- Caller: create offer ---
class CreateSdpObserver : public webrtc::CreateSessionDescriptionObserver {
 public:
  explicit CreateSdpObserver(CallEngine* e) : engine_(e) {}
  void OnSuccess(webrtc::SessionDescriptionInterface* desc) override {
    std::string sdp;
    desc->ToString(&sdp);
    engine_->pc()->SetLocalDescription(
        SetSdpObserver::Create(), desc);          // takes ownership
    engine_->signaling()->Send("webrtc.offer",
        {{"sdp", sdp}, {"type", desc->type()}, {"call_id", engine_->call_id()}});
  }
  void OnFailure(webrtc::RTCError err) override {
    LOGE("CreateOffer failed: %s", err.message());
    engine_->SetState(CallState::kFailed);
  }
 private:
  CallEngine* engine_;
};

void CallEngine::CreateOffer() {
  webrtc::PeerConnectionInterface::RTCOfferAnswerOptions opts;
  opts.offer_to_receive_audio = 1;
  opts.offer_to_receive_video = 1;   // always, even for audio-only (see §4)
  pc_->CreateOffer(new rtc::RefCountedObject<CreateSdpObserver>(this), opts);
}

// --- Callee: got offer, answer it ---
void CallEngine::OnRemoteOffer(const std::string& sdp) {
  signaling_thread_->PostTask([this, sdp] {
    webrtc::SdpParseError err;
    auto desc = webrtc::CreateSessionDescription(
        webrtc::SdpType::kOffer, sdp, &err);
    if (!desc) { LOGE("bad offer sdp: %s", err.description.c_str()); return; }

    pc_->SetRemoteDescription(
        std::move(desc),
        rtc::make_ref_counted<SetRemoteObserver>([this] {
          DrainBufferedIce();                    // your FIX-3, but in C++
          pc_->CreateAnswer(
              new rtc::RefCountedObject<CreateAnswerObserver>(this), {});
        }));
  });
}

// --- ICE ---
void CallEngine::OnIceCandidate(
    const webrtc::IceCandidateInterface* candidate) {     // PeerConnectionObserver
  std::string cand;
  candidate->ToString(&cand);
  signaling_->Send("webrtc.ice", {
      {"candidate",     cand},
      {"sdpMid",        candidate->sdp_mid()},
      {"sdpMLineIndex", candidate->sdp_mline_index()},
      {"call_id",       call_id_},
  });
}

void CallEngine::OnRemoteIce(const std::string& cand,
                             const std::string& mid, int mline) {
  // Remote description may not be set yet — buffer, don't drop. (Your FIX-3.)
  if (!pc_ || !pc_->remote_description()) {
    buffered_ice_.push_back({cand, mid, mline});
    return;
  }
  webrtc::SdpParseError err;
  std::unique_ptr<webrtc::IceCandidateInterface> ice(
      webrtc::CreateIceCandidate(mid, mline, cand, &err));
  if (ice) pc_->AddIceCandidate(std::move(ice), [](webrtc::RTCError e) {
    if (!e.ok()) LOGE("AddIceCandidate: %s", e.message());
  });
}
```

**Signaling token rule (your open 401 bug):** the WebSocket client in C++ must fetch a **fresh token on every connect attempt**, not reuse a snapshot. Expose a token getter callback into the UI process over AIDL — or read from shared storage — but never cache.

```cpp
class SignalingClient {
 public:
  using TokenGetter = std::function<std::string()>;
  void SetTokenGetter(TokenGetter g) { token_getter_ = std::move(g); }

 private:
  void OpenConnection() {
    const std::string token = token_getter_ ? token_getter_() : "";  // FRESH
    const std::string url = base_url_ + "?token=" + token;
    LOGI("[WS] connecting (token len=%zu)", token.length());  // never log the token
    // ...
  }
  void OnHttpStatus(int code) {
    if (code == 401) {
      RequestTokenRefresh();     // don't blind-backoff into a 401 loop
      return;
    }
    ScheduleReconnect();
  }
  TokenGetter token_getter_;
};
```

---

## 9. CPU / thread priority (your original requirement)

```cpp
#include <pthread.h>
#include <sched.h>
#include <sys/resource.h>

// Best effort — SCHED_FIFO is usually denied to unprivileged apps on Android.
// Always log the outcome; do not assume it worked.
void RaiseThreadPriority(rtc::Thread* t) {
  t->PostTask([] {
    sched_param param{};
    param.sched_priority = sched_get_priority_max(SCHED_FIFO);
    int rc = pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
    if (rc != 0) {
      LOGW("[CE] SCHED_FIFO denied (rc=%d) — falling back to nice()", rc);
      // Fallback that actually works on Android:
      setpriority(PRIO_PROCESS, 0, -19);   // ANDROID_PRIORITY_AUDIO
    } else {
      LOGI("[CE] SCHED_FIFO applied");
    }
  });
}
```

> Reality check: on stock Android, `SCHED_FIFO` from an unprivileged app is normally rejected. The real low-latency win comes from **AAudio's** `AAUDIO_PERFORMANCE_MODE_LOW_LATENCY` + exclusive mode, which the ADM already requests. Keep the code, log the result, don't build your design on it succeeding.

---

## 10. JNI bridge

```cpp
// jni_bridge.cc
#include <jni.h>

static JavaVM* g_vm = nullptr;
static jobject g_service = nullptr;   // global ref to CallEngineService

extern "C" JNIEXPORT jint JNICALL
JNI_OnLoad(JavaVM* vm, void*) {
  g_vm = vm;
  LOGI("[CE] JNI_OnLoad ok");
  return JNI_VERSION_1_6;
}

extern "C" JNIEXPORT jboolean JNICALL
Java_uz_udevs_xlogistic_1driver_1mobile_callengine_CallEngineNative_nativeInit(
    JNIEnv* env, jobject thiz) {
  g_service = env->NewGlobalRef(thiz);
  return CallEngine::Instance().Initialize() ? JNI_TRUE : JNI_FALSE;
}

extern "C" JNIEXPORT void JNICALL
Java_uz_udevs_xlogistic_1driver_1mobile_callengine_CallEngineNative_nativeStartCall(
    JNIEnv* env, jobject, jstring j_call_id, jboolean with_video) {
  const char* id = env->GetStringUTFChars(j_call_id, nullptr);
  CallEngine::Instance().StartCall(id, with_video == JNI_TRUE);
  env->ReleaseStringUTFChars(j_call_id, id);
}

extern "C" JNIEXPORT void JNICALL
Java_uz_udevs_xlogistic_1driver_1mobile_callengine_CallEngineNative_nativeSetRemoteSurface(
    JNIEnv* env, jobject, jobject surface) {
  CallEngine::Instance().SetRemoteSurface(env, surface);
}

// C++ → Java (state events). Must attach the thread — we're on a WebRTC thread.
void NotifyUi(CallState state) {
  JNIEnv* env = nullptr;
  bool attached = false;
  if (g_vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
    g_vm->AttachCurrentThread(&env, nullptr);
    attached = true;
  }
  jclass cls = env->GetObjectClass(g_service);
  jmethodID m = env->GetMethodID(cls, "onNativeState", "(I)V");
  env->CallVoidMethod(g_service, m, static_cast<jint>(state));
  if (attached) g_vm->DetachCurrentThread();
}
```

---

## 11. State machine

Single source of truth, lives in C++. Dart only renders it.

```cpp
enum class CallState : int {
  kIdle = 0,
  kRinging,          // incoming, CallKit UI is up
  kAccepting,        // accept API in flight
  kConnectingWebrtc, // ICE / DTLS
  kActive,
  kEnded,
  kFailed,
};

void CallEngine::SetState(CallState s) {
  if (state_ == s) return;
  LOGI("[CE] state %d -> %d", (int)state_, (int)s);
  state_ = s;
  NotifyUi(s);
}

// PeerConnectionObserver
void CallEngine::OnIceConnectionChange(
    webrtc::PeerConnectionInterface::IceConnectionState s) {
  switch (s) {
    case webrtc::PeerConnectionInterface::kIceConnectionConnected:
    case webrtc::PeerConnectionInterface::kIceConnectionCompleted:
      SetState(CallState::kActive);
      StartStatsPolling();
      break;
    case webrtc::PeerConnectionInterface::kIceConnectionDisconnected:
      // Transient — do NOT end the call. GATHER_CONTINUALLY may recover it.
      NotifyUi(CallEvent::kNetworkUnstable);
      break;
    case webrtc::PeerConnectionInterface::kIceConnectionFailed:
      SetState(CallState::kFailed);
      break;
    default: break;
  }
}
```

---

## 12. Media-state data channel (camera on/off, screen share, aspect)

```cpp
void CallEngine::CreateMediaStateDataChannel() {
  webrtc::DataChannelInit init;
  init.ordered = true;
  auto res = pc_->CreateDataChannelOrError("media-state", &init);
  if (res.ok()) {
    media_dc_ = res.MoveValue();
    media_dc_->RegisterObserver(this);
  }
}

void CallEngine::SendMediaState() {
  if (!media_dc_ || media_dc_->state() != webrtc::DataChannelInterface::kOpen)
    return;

  // Tell the peer what we're sending, so their UI can lay out BEFORE the
  // first frame arrives (no more layout jump in the first seconds).
  nlohmann::json j = {
      {"audio",       audio_track_->enabled()},
      {"video",       video_track_->enabled()},
      {"source",      is_screen_sharing_ ? "screen" : "camera"},
      {"orientation", is_landscape_ ? "landscape" : "portrait"},
      {"aspect",      current_aspect_},   // e.g. 1.777 for 16:9
  };
  std::string payload = j.dump();
  media_dc_->Send(webrtc::DataBuffer(
      rtc::CopyOnWriteBuffer(payload.data(), payload.size()), false));
}
```

The UI process uses `source`/`orientation`/`aspect` to pick `contain` vs `cover` — a 16:9 laptop stream on a 9:16 phone must be `contain` (letterbox with a blurred backdrop), never `cover`, or you crop the frame to a zoomed strip.

---

## 13. CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22)
project(callengine CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(WEBRTC_ROOT ${CMAKE_SOURCE_DIR}/third_party/webrtc)

add_library(webrtc STATIC IMPORTED)
set_target_properties(webrtc PROPERTIES
    IMPORTED_LOCATION
      ${WEBRTC_ROOT}/lib/${ANDROID_ABI}/libwebrtc.a)

add_library(callengine SHARED
    src/call_engine.cc
    src/camera_video_source.cc
    src/video_renderer.cc
    src/signaling_client.cc
    src/quality_controller.cc
    src/jni_bridge.cc)

target_include_directories(callengine PRIVATE
    ${WEBRTC_ROOT}/include
    ${WEBRTC_ROOT}/include/third_party/abseil-cpp
    ${WEBRTC_ROOT}/include/third_party/libyuv/include)

target_compile_definitions(callengine PRIVATE
    WEBRTC_POSIX WEBRTC_ANDROID WEBRTC_LINUX
    ABSL_ALLOCATOR_NOTHROW=1)

target_link_libraries(callengine
    webrtc
    android log camera2ndk mediandk aaudio OpenSLES EGL GLESv2)
```

`build.gradle`:

```gradle
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++17 -fexceptions -frtti"
                arguments "-DANDROID_STL=c++_static"
            }
        }
        ndk { abiFilters "arm64-v8a", "armeabi-v7a" }
    }
    externalNativeBuild {
        cmake { path "src/main/cpp/CMakeLists.txt"; version "3.22.1" }
    }
}
```

---

## 14. Manifest — the separate process

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.CAMERA" />

<service
    android:name=".callengine.CallEngineService"
    android:process=":callengine"
    android:exported="false"
    android:foregroundServiceType="microphone|camera" />
```

> **Play Console:** `foregroundServiceType="camera"` requires a Foreground Service declaration + a prominent-disclosure video. You have real video calling, so this is a legitimate use — but budget review time. A staged path is: ship `microphone|phoneCall` first (native audio), add `camera` once the declaration is approved.

**Android 14+:** the FGS must be started **before** you open the camera, and the runtime permission must already be granted, or you get a `SecurityException` / `ForegroundServiceStartNotAllowedException`.

---

## 15. Verification checklist

```bash
adb logcat -c && adb logcat | grep -E "\[CE\]|\[CE-UI\]|\[WS\]"
adb shell ps -A | grep callengine
adb shell dumpsys activity services | grep -i callengine
```

| # | Scenario | Pass criteria |
|---|---|---|
| 1 | Audio call web → android | `[CE] state 3 -> 4` (kActive), audio both ways |
| 2 | Mid-call audio → video upgrade | Video appears, **no renegotiation**, renderer size ≠ 0×0 |
| 3 | **Kill the UI process during an active call** | `:callengine` survives, audio/video keep flowing (the whole point) |
| 4 | Cold start accept after app kill | Call page opens, no `event=end` right after accept |
| 5 | Peer turns camera off | Placeholder shown, no frozen last frame |
| 6 | Bad network | `quality_limitation_reason=bandwidth`, profile steps down within ~2s, back up after ~8s of headroom |
| 7 | Screen share from laptop (16:9) on phone | `contain` fit, text readable, `MAINTAIN_RESOLUTION` active |
| 8 | Token refresh mid-reconnect | WS reconnects with a fresh token immediately — no 401 loop |
| 9 | Low-end device (Android 10, weak CPU) | `quality_limitation_reason=cpu` → steps down instead of stuttering |

---

## 16. Keep it portable for iOS

Put everything platform-specific behind an interface. Then the iOS port replaces implementations, not logic.

```cpp
// platform/audio_capture.h
class IAudioCapture {
 public:
  virtual ~IAudioCapture() = default;
  virtual bool Start() = 0;
  virtual void Stop() = 0;
};
// android/aaudio_capture.cc   → AAudio
// ios/audio_unit_capture.mm   → AudioUnit / Core Audio

// platform/video_capture.h  → Camera2 NDK  |  AVCaptureSession
// platform/renderer.h       → ANativeWindow |  CVPixelBuffer / Metal
// platform/socket.h         → your WS client (portable already)
// platform/logger.h         → __android_log_print | os_log
```

`CallEngine`, `QualityController`, the state machine, and the signaling protocol stay **100% shared C++**. That is the payoff for doing this in C++ rather than Kotlin.

---

## 17. Order of work

1. Skeleton compiles, `JNI_OnLoad ok` in logcat, `:callengine` process appears on bind. ← *you are here*
2. `fetch_libwebrtc.sh` — pin a commit, build/fetch `.a` for both ABIs, link it.
3. **Audio-only call end to end.** Don't touch video until audio works over a real network.
4. Video: capturer + renderer + the always-on transceiver (kills the 0×0 bug).
5. Stats + `QualityController`.
6. Screen share (`is_screencast` + `MAINTAIN_RESOLUTION` + `kDetailed`).
7. Kill-switch via Remote Config; `flutter_webrtc` stays as the fallback path until native proves itself in production.