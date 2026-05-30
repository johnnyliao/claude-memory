---
name: scoreboard-app-project
description: Flutter/iOS soccer scoreboard app with YouTube RTMP streaming — full project state
metadata: 
  node_type: memory
  type: project
  originSessionId: 7009ff0c-5cd2-47fc-a4c6-cf2b7acb26e0
---

## Repo
GitHub: https://github.com/johnnyliao/scoreboard_app
Local: C:\Users\Johnny\Documents\scoreboard_app

## App Purpose
Soccer scoreboard app that:
1. Shows live score (home/away team names, scores, +/- buttons, reset)
2. Forces landscape orientation, dark theme
3. Streams to YouTube via RTMP (HaishinKit ~1.9)
4. Burns score overlay into the video stream (viewers see the score)

## Key Files
- `lib/main.dart` — Full Flutter UI: scoreboard, match-clock controls, goal-scorer picker dialog, stream controls, player roster constant
- `lib/youtube_service.dart` — OAuth + YouTube Data API (setupLive, waitUntilStreamActive, transitionToLive, stopBroadcast)
- `ios/Runner/AppDelegate.swift` — MethodChannel handler, CameraPreviewFactory registration, AVAudioSession setup
- `ios/Runner/StreamingService.swift` — **MAIN FILE**: HaishinKit RTMP setup, score overlay (`makeScoreOverlay`), celebration overlay (`makeCelebrationOverlay`), `ScoreboardOverlayEffect` VideoEffect subclass (defined inline at top of file — there is NO separate ScoreOverlayEffect.swift, that file was removed)
- `ios/Runner/CameraPreviewFactory.swift` — FlutterPlatformView returning StreamingService.previewView (MTHKView)
- `ios/Podfile` — `pod 'HaishinKit', '1.9.9'`, `platform :ios, '13.0'`
- `.github/workflows/build-ios.yml` — GitHub Actions CI/CD (push to main → build IPA artifact)

## Current State (latest commit: a97cfac, 2026-05-28)
**Status: ✅ WORKING — 1080p30 streaming + broadcast-style overlay + match clock + goal-scorer picker (14 players incl. 2 loaned) + selectable team colors. Screen stays awake while live. Confirmed live by user.**

Additions this session (2026-05-28): selectable team colors synced to overlay (home 橘/紫, away 10 colors via palette icon beside edit-name; luminance-based text contrast on score cells); goal-picker landscape overflow fix (Flexible + scrollable — bottom row had been untappable); 2 loaned players (簡/林, no jersey number → tile shows surname, celebration shows name 以諾/言瑀); player #2 corrected 秉彥→秉諺; REAL 30fps via Screen.frameRate fix + keep-screen-awake during stream (both detailed in [[scoreboard-tech-decisions]]).

Architecture: HaishinKit `RTMPStream` + `attachCamera`/`attachAudio` + `.offscreen` videoMixer mode + `registerVideoEffect(ScoreboardOverlayEffect)` for the score & celebration overlays. YouTube broadcast is created programmatically via OAuth + YouTube Data API (`lib/youtube_service.dart`), no manual Studio setup.

### What works
- 1080p RTMP streaming to YouTube (after the H.264 profileLevel fix — see [[scoreboard-tech-decisions]])
- **Broadcast-style score overlay**: horizontal pill in top-left at (48, 48), 1920x1080 canvas. Layout: `[ MM:SS | HOME NAME | H | A | AWAY NAME ]`. Dark-navy bar, white text, score cells filled with team colors (#2196F3 home, #E53935 away), thin colored bottom-accent stripes under name cells, monospaced tabular digits for clock & scores. Sized for TV-graphic prominence (barH=90, name font 40, score font 48, clock font 40).
- **Match clock**: wall-clock-anchored (drift-free pause/resume). State: `_clockAccumMs` + `_clockRunStart`; `_matchTimer` ticks 1Hz to refresh display and push clock to overlay via `_syncScore`. Format MM:SS, allows >59 min. Controls in center panel: play/pause toggle (green/amber) + reset (gray).
- **Goal scorer picker**: tap GOAL → `_GoalPickerDialog` (4x3 grid of jersey numbers, gold-on-dark, "取消" to dismiss) → on tap, fires `triggerGoal` with `playerName`. Celebration shows two-line text: player name (white) above big yellow "GOAL!" (Impact font, black stroke).
- **Celebration animation**: `celebrationScale = 1.2` (+20% from original). Particles, text (140→168pt baseline, ~218pt at rest), pulse border (20→24px), beam half-width (90→108) all scaled. Positions/velocities NOT scaled (frame-bound). Two-line layout: PingFangTC-Semibold for Chinese name, Impact for GOAL!.
- Landscape capture (`stream.videoOrientation = .landscapeRight`).
- OAuth YouTube login → auto-create unlisted broadcast → push → `waitUntilStreamActive` → `transitionToLive`.
- AltStore sideloading, GitHub Actions build pipeline.

### Settings / decisions
- Broadcast privacy = **unlisted (不公開)** = anyone with the link can watch ([youtube_service.dart:60](lib/youtube_service.dart#L60)). User confirmed this is the desired behavior (NOT 'private', which would block link access). Do NOT change to 'private'.
- Video: 1920x1080, **H.264 High AutoLevel** (critical — see [[scoreboard-tech-decisions]]), 4.5 Mbps, 30fps, keyframe interval 2s. Audio: AAC 128kbps.
- Score cell colors match Flutter UI team accents (#2196F3 / #E53935) — keep consistent if either side changes.

### Player roster (12 players, drives the goal picker 4x3 grid order)
```
1 胤丞   2 秉彥   4 浩宇   5 胤銘
6 梓敬   7 祐翼   8 學濬   9 宥愷
10 岳辰  11 翰墨  14 祥宇  15 祐瑀
```
Non-contiguous jersey numbers (no 3/12/13). Stored as `const List<(int, String)> _players` in [lib/main.dart](lib/main.dart). Order in the list = order in the grid (left→right, top→bottom).

### Performance note (2026-05-27)
Per architecture analysis: heat during long streams is dominated by (1) sustained radio upload at 4.5 Mbps and (2) HaishinKit's CPU-based screen compositing at 30Hz/1080p (`ScreenRendererByCPU` per source). Hardware H.264 encode (VideoToolbox) is cheap. iPhone 13 Pro should handle 45-min halves fine; 90 min + charging + sun may trigger thermal throttling. Mitigations: WiFi > cellular, don't charge while streaming, shade. User chose NOT to implement code-side optimizations (offscreen start/stop too risky given prior crash history; dedup ineffective with 1Hz clock changes).

### What's pending / future
- (If reported) landscape upside-down → flip to `.landscapeLeft`
- Auto-stop / cleanup edge cases

## GitHub Actions Workflow
```yaml
# .github/workflows/build-ios.yml
# Triggers on push to main
# Steps: checkout → flutter setup → flutter pub get → pod install --repo-update → flutter build ios --release --no-codesign → package IPA
# IPA packaging: mkdir Payload → cp -rL Runner.app Payload/ → zip -r -y app.ipa Payload
# Artifact: app.ipa (download from Actions → build → Artifacts)
```

## Podfile
```ruby
platform :ios, '13.0'
pod 'HaishinKit', '~> 1.9'
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
      config.build_settings['SWIFT_VERSION'] = '5.0'
    end
  end
end
```
