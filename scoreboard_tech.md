---
name: scoreboard-tech-decisions
description: "Technical decisions, failures, and current implementation for scoreboard app overlay"
metadata: 
  node_type: memory
  type: project
  originSessionId: 7009ff0c-5cd2-47fc-a4c6-cf2b7acb26e0
---

## ⚠️ UPDATE 2026-05-27 — verified current truth (supersedes older notes below)

**Streaming flow is now OAuth + YouTube Data API**, not manual Studio setup:
- `lib/youtube_service.dart`: OAuth (google_sign_in) → `setupLive()` creates liveBroadcast + liveStream (cdn.resolution toggles with profile) + bind → returns per-stream rtmpUrl + streamKey.
- `lib/main.dart _startStream()` order: setupLive → native `startStream(url,key)` → `waitUntilStreamActive(streamId)` (polls liveStreams status, **60s timeout** = 30×2s, waits for `streamStatus=='active'`) → `transitionToLive`. The "等待 YouTube 確認串流" hang = waitUntilStreamActive timing out because YouTube gets no data.
- IMPORTANT: YouTube ALWAYS requires receiving real RTMP video before a broadcast can go live — independent of API vs Studio. transitionToLive fails (errorStreamInactive) if stream not active. This is mandatory, not legacy.

**Current iOS architecture = HaishinKit VideoEffect + `.offscreen` + attachCamera** (NOT the custom AVCaptureSession the notes below describe — that was reverted in commit 153c982).

**THE 1080p BUG (root cause confirmed via YouTube Studio):** at 1080p, Studio shows "沒有資料 / no data received" — YouTube gets ZERO bytes (not even audio → whole RTMP publish blocked waiting for a first video keyframe that never comes). RTMP connect + publish succeed; the offscreen pipeline produces no encoded frames.

**KEY FINDING: 1080p has NEVER successfully streamed in this project, in ANY architecture.** Only 720p passthrough ever reached YouTube — and passthrough does NOT apply the score overlay. So "1080p" and "overlay pipeline" were never isolated. The memory's "極佳 quality works" was 720p custom-session, not 1080p.

**Verified HaishinKit 1.9.9 facts (from source):**
- `IOVideoMixer.append`: `.passthrough` → `outputSampleBuffer(raw)` NO effects; `.offscreen` → enqueues to screen objects where `screen.track == track`, effects applied there. **VideoEffect REQUIRES .offscreen.**
- Offscreen needs a VideoTrackScreenObject whose track matches the camera track (0) or frames are silently discarded. Lazy `screen` adds one default object.
- `Choreographer` DisplayLink is added to `RunLoop.main` (.common); screen compositing is synchronous on main thread.
- IOStream does NOT auto start/stop the screen on readyState; only `mixer.startRunning()`. So screen.startRunning() must be manual for offscreen — but placing it has a cursed history (crashes at init AND at startStream per older notes; now no crash but zero output).

**✅ ROOT CAUSE CONFIRMED (2026-05-27) — THE 1080p BUG WAS H.264 PROFILE LEVEL:**
HaishinKit 1.9.9 `VideoCodecSettings.profileLevel` DEFAULTS to `kVTProfileLevel_H264_Baseline_3_1`. **H.264 Baseline Level 3.1 max frame size = 1280x720 (3600 macroblocks).** 1920x1080 needs 8160 MBs → exceeds Level 3.1 → VTCompressionSession silently produces ZERO frames (VideoCodec.swift error path is `status != noErr -> return`, no log) → muxer never gets a keyframe → YouTube "no data". The code NEVER set profileLevel, so it always used the 3.1 default.
- Explains EVERYTHING: 720p works (1280x720 is exactly Level 3.1's max); 1080p fails in EVERY architecture (passthrough/offscreen/custom-append all used the default 3.1); "no data" not "poor data"; bandwidth-independent. Build 1 (1080p passthrough, no overlay) STILL showed "no data" in Studio → proved it was NOT offscreen, it was the encoder level. offscreen was never independently broken — it was collateral damage from the same profileLevel.
- **FIX (1 line):** `videoSettings.profileLevel = kVTProfileLevel_H264_High_AutoLevel as String` (needs `import VideoToolbox`). High profile + AutoLevel → VideoToolbox picks level 4.0+ for 1080p (= YouTube's recommended 1080p profile).
- Build with the fix (commit after 24485e2): restored offscreen + overlay + profileLevel fix + sessionPreset/frameRate/1080p/4.5M. Expectation: 1080p WITH score overlay finally streams.
- LESSON: if YouTube shows "no data" (zero bytes, incl. no audio) and connect/publish succeeded, suspect the video ENCODER producing nothing before suspecting network/offscreen. Check profileLevel vs resolution first.
- ✅ VERIFIED WORKING (2026-05-27, commit 8a4e416): user confirmed 1080p streaming to YouTube with the burned-in score overlay ("安和 2 / 客隊 0") visible and "直播中". The long-standing 1080p timeout is SOLVED. offscreen overlay works fine once profileLevel is correct.
- ORIENTATION: HaishinKit camera defaults to PORTRAIT → stream came out vertical. Fixed in attachDevicesIfNeeded with `stream.videoOrientation = .landscapeRight` (set AFTER attachCamera). commit ab1f76b. If image is ever upside-down, switch to .landscapeLeft. Flutter UI allows both landscapeLeft+landscapeRight; camera orientation is fixed.
- ⚠️ FRAME RATE TRAP (2026-05-28, commit a97cfac): stream looked ~15-20fps despite 1080p30. ROOT CAUSE: HaishinKit 1.9.9 `Screen.frameRate` DEFAULTS to 15, and in `.offscreen` mixer mode the ENCODED output fps is driven by the screen's choreographer (`preferredFramesPerSecond = Screen.frameRate`), NOT by `stream.frameRate` (that only sets CAMERA CAPTURE rate). We never set `stream.screen.frameRate`, so output ran at 15fps. FIX: `stream.screen.frameRate = 30` set BEFORE `stream.screen.startRunning()` (startRunning copies the value into the choreographer). This is the SECOND HaishinKit default-value trap after profileLevel — pattern: "you set a plausible-looking param but the one that actually drives output is a different one." When tuning offscreen output, always configure `stream.screen.*` not just `stream.*`.
- SCREEN AUTO-LOCK (2026-05-28, commit a97cfac): phone auto-locked mid-stream (user watching match without touching screen → iOS idle timer). FIX: drive `UIApplication.shared.isIdleTimerDisabled` from an `isStreaming` didSet (true on succeedStart, false on failStart/stopStream), dispatched to main. didSet covers every transition in one place and doesn't fire on the initial false (launch unaffected). Restores normal auto-lock when streaming stops (saves battery).
- TOOLING NOTE (2026-05-28): during this edit session the Bash/Read/PowerShell tool RESULTS came back severely delayed and out-of-order, which made the file look corrupted mid-edit (it wasn't). Lesson: when results look wrong, verify with ONE sequential command (e.g. `wc -l` + targeted `grep -c`) before reacting; git checkout is the safe reset. Don't re-apply edits based on stale/empty results.

**(historical) build 1 isolation experiment:** setupStream → pure 1080p passthrough, no overlay, 4.5M — used to prove encoder-vs-overlay. Result: still "no data" → pointed at encoder → led to the profileLevel discovery above.
Also fixed in main.dart: on start failure, native left isStreaming=true (Dart never called stopStream) → "start blocked: already starting" on retry. Now `finally` calls native stopStream when `!_isStreaming`.

---

## Current StreamingService Architecture (f7ade06, latest)

**Why:** HaishinKit's VideoEffect/offscreen system caused crashes. Custom pipeline bypasses it entirely.

```swift
// Key design:
// 1. Own AVCaptureSession at 1280x720 (video only)
// 2. AVCaptureVideoDataOutput delegate: apply overlay to CVPixelBuffer via CIContext
// 3. Feed modified frame: stream.append(sampleBuffer, track: 0)  ← public HaishinKit API
// 4. HaishinKit handles: audio (attachAudio), RTMP, H.264 encoding
// 5. Preview: MTHKView attached to stream (receives frames from stream.append())
// 6. Overlay: built on main thread in updateScore(), cached CIImage, read on capture queue
// 7. No attachCamera(), no offscreen mode, no screen.startRunning()
```

## HaishinKit 1.9.9 Key Facts (verified from source)
- `VideoEffect` is a CLASS (not protocol), inherit with `class MyEffect: VideoEffect`
- `execute(_ image: CIImage, info: CMSampleBuffer?) -> CIImage` — override this
- `stream.registerVideoEffect(effect)` → `IOVideoMixer.registerEffect()` → `VideoTrackScreenObject.effects[]`
- **Default mode is `.passthrough`** — VideoEffect.execute() is NEVER called in passthrough
- **Only `.offscreen` mode** calls execute() via VideoTrackScreenObject.makeImage()
- `Screen.startRunning()` — MUST be called explicitly to start display link + pixel buffer pool
- `stream.screen` is the public Screen object
- `stream.videoMixerSettings` — change mode here: `var s = stream.videoMixerSettings; s.mode = .offscreen`
- `stream.append(_ sampleBuffer: CMSampleBuffer, track: UInt8 = 0)` — PUBLIC, feeds video manually
- `stream.append(_ audioBuffer: AVAudioBuffer, when: AVAudioTime, track: UInt8 = 0)` — PUBLIC, feeds audio

## History of Failures

### VideoEffect approach — FAILED
- Issue 1: `class ScoreOverlayEffect: NSObject, VideoEffect` → "Multiple inheritance from classes" error
- Fix: `class ScoreOverlayEffect: VideoEffect` (VideoEffect already extends NSObject)
- Issue 2: `override func execute(_ buffer: CVPixelBuffer, ...)` → "Method does not override" error  
- Fix: correct signature is `execute(_ image: CIImage, info: CMSampleBuffer?) -> CIImage`
- Issue 3: overlay never appeared → root cause: DEFAULT MODE IS PASSTHROUGH, execute() never called
- Issue 4: switching to offscreen + `stream.screen.startRunning()` in `setupStream()` → APP CRASHES ON LAUNCH
  - Root cause: Display Link fires ~16ms after init, codec/mixer not fully initialized
- Issue 5: moving `screen.startRunning()` to `startStream()` → APP CRASHES ON PRESS START
  - Root cause: unknown (couldn't get crash log), likely same issue with display link
- **Decision: Abandon VideoEffect/offscreen system entirely**

### Overlay coordinate system
- CIImage uses bottom-left origin, UIKit uses top-left
- When drawing bar at UIKit bottom (y = height - barH): maps to CIImage y=0 (bottom) — NO FLIP NEEDED
- `CIImage(image: uiImage)` from UIKit-drawn UIImage: bar appears at CIImage bottom automatically

### Compositing
- Use `CIFilter(name: "CISourceOverCompositing")`
- `filter.setValue(overlay, forKey: kCIInputImageKey)` — overlay is FOREGROUND
- `filter.setValue(videoFrame, forKey: kCIInputBackgroundImageKey)` — video is BACKGROUND

## ScoreOverlayEffect.swift (currently in project but UNUSED)
- File exists in project.pbxproj, kept to avoid changing build config
- NOT registered with stream (no stream.registerVideoEffect call)
- Safe to ignore

## project.pbxproj Custom Entries
Three Swift files were manually added with these UUIDs:
- StreamingService.swift: FileRef `74858FC01ED2DC5600515810`, BuildFile `74858FC11ED2DC5600515810`
- CameraPreviewFactory.swift: FileRef `74858FC21ED2DC5600515810`, BuildFile `74858FC31ED2DC5600515810`
- ScoreOverlayEffect.swift: FileRef `74858FC41ED2DC5600515810`, BuildFile `74858FC51ED2DC5600515810`
- iOS deployment target: 13.0

## Info.plist Permissions Added
```xml
<key>NSCameraUsageDescription</key><string>需要攝影機進行直播</string>
<key>NSMicrophoneUsageDescription</key><string>需要麥克風進行直播</string>
```

## Flutter Platform Channel
- Channel name: `com.scoreboard/streaming`
- Methods: `startStream({url, key})`, `stopStream`, `updateScore({homeName, homeScore, awayName, awayScore})`
- PlatformView: `com.scoreboard/camera_preview` → CameraPreviewFactory → StreamingService.previewView (MTHKView)

## Pending Issues
1. **Verify overlay appears in YouTube stream** with latest approach (f7ade06)
2. **Auto-start YouTube live** from phone (needs YouTube Data API v3 + OAuth2 — future feature)
3. If overlay still doesn't appear: next step is to verify `stream.append(sampleBuffer, track: 0)` properly routes through encoder AND preview
