---
name: user-profile-johnny
description: "Johnny's dev setup, device, workflow for scoreboard project"
metadata: 
  node_type: memory
  type: user
  originSessionId: 7009ff0c-5cd2-47fc-a4c6-cf2b7acb26e0
---

- Name: Johnny, email: epyosstr@gmail.com
- Device: iPhone 13 Pro, iOS 26.5
- Dev machine: Windows 11 Pro, Flutter 3.32.1 installed
- Workflow: code on Windows → git push → GitHub Actions (macos-latest) compiles iOS → download .ipa → sideload via AltStore
- Sideloading: AltStore (NOT Sideloadly — had persistent errors -29004/-22406/-20101)
- iCloud: installed legacy Apple.iCloud via winget (needed for AltServer)
- YouTube streaming now uses OAuth (google_sign_in) + YouTube Data API to auto-create the broadcast per session (no hardcoded key, no manual Studio setup). iOS client id in youtube_service.dart.
- Prefers direct answers, gets frustrated with repeated wrong guesses — always verify from source (HaishinKit GitHub, etc.) before suggesting. Responds very well to a disciplined "isolate one variable per build" approach since each build = CI round-trip + AltStore (expensive).
- Sometimes mixes up YouTube privacy terms: said "私人" but meant "不公開/unlisted" (link-accessible). Confirm intent by behavior, not the label.
- Reads on-device debug status + YouTube Studio stream health when asked — good at providing diagnostic observations.
