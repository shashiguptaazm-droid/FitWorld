# FitWorld 360 — iOS (SwiftUI)

All-in-one fitness & wellness app per `MASTER_PROMPT.md`. This repo is the
**Phase 1 "Core Engine"** SwiftUI implementation (see `IMPLEMENTATION_PLAN_IOS.md`).

## What's here

- `project.yml` — XcodeGen definition (source of truth; `.xcodeproj` is generated, never committed)
- `FitWorld/` — app: shell, onboarding, exercise library, Lottie demo player,
  wall-clock interval timer, workout player with interruption-safe session persistence,
  workouts/programs, history, settings
- `FitWorldTests/` — timer engine unit tests (wall-clock accuracy, pause/resume, phases)
- `.github/workflows/ios.yml` — macOS CI: XcodeGen → resolve SPM → `xcodebuild test` on iPhone simulator

## Build (macOS)

```bash
brew install xcodegen
xcodegen generate
xcodebuild test -scheme FitWorld -destination 'platform=iOS Simulator,name=iPhone 16'
```

## Backend

Phase 1 targets `https://fitworld.medigyaan.com` (VPS FastAPI + SQLite behind nginx).
The app is offline-first: auth/sync fail gracefully until the backend ships.

## Status

Code authored on Windows; first Xcode compile pending on a Mac.
Seed content JSON and Lottie demo assets are pending — the UI degrades to honest
empty states and a labeled "demo pending" placeholder.
