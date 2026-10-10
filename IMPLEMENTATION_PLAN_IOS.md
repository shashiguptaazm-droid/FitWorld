# FitWorld 360 — iOS (SwiftUI) Implementation Plan
**Version 1.0 · 10 Oct 2026 · Scope: Phase 1 "Core Engine" (see decisions below)**

## 0. Locked-in decisions (confirmed with product owner)

| Decision | Choice |
|---|---|
| Platform | iOS / iPadOS only, native **SwiftUI** (no React Native / Flutter) |
| Backend | **Existing VPS** — `medicscholar.medigyaan.com` (213.199.61.156, Tailscale 100.71.127.19) |
| Animation | **Lottie** (`lottie-ios` via SPM) — male/female demonstrators as `.lottie`/`.json` assets; Rive explored later if needed |
| Phase 1 scope | Core engine first: app shell, auth, onboarding, home dashboard, exercise library, workout player + timer, workout history, program generator, Lottie player |
| Monetization | **Deferred** — data model will carry `entitlements`/`isPremium` flags from day 1; StoreKit 2 lands in a later phase |
| HealthKit | Deferred to later phase; permissions not requested in Phase 1 |

---

## 1. Environment inventory (verified)

**VPS `medicscholar`** (SSH key `~/.ssh/id_ed25519_medicscholar`, `root@213.199.61.156`, works today):
- Ubuntu, x86_64, **8 GB RAM (only ~1.4 GB available)**, 290 GB disk at **86% used (42 GB free)**.
- Already running: nginx (80/443 + many internal vhosts), MariaDB (native + docker), LiveKit, Nextcloud, node services on 9100/9200/9000, python on 8888, **llama-server (local LLM) and ollama**.
- ⚠️ Implication: the FitWorld backend must be **lightweight** (FastAPI or a small Node service, SQLite or the existing MariaDB, low RAM footprint). No new heavy DB container. Assume ~200 MB budget.

**Build host:** iOS builds require Xcode → the MacBook Air on this Tailscale net (`drishtis-macbook-air`; currently offline). Swift source files can be authored from Windows; Xcode project will be generated on the Mac.

## 2. High-level architecture

```
┌─────────────────────── iPhone / iPad (SwiftUI) ───────────────────────┐
│  FitWorld-iOS (Swift Package-based app, MVVM)                         │
│  ├─ AppShell (TabView: Home · Train · Library · Me)                    │
│  ├─ Onboarding flow → Profile store                                    │
│  ├─ ExerciseLibrary (search/filter/detail)                             │
│  ├─ WorkoutPlayer (Lottie demo + IntervalTimer engine + set tracking)  │
│  ├─ History/Progress (local, synced)                                   │
│  ├─ Cache layer: local JSON content (offline-first) + API sync         │
│  └─ Networking: URLSession + async/await, token auth, ETag caching     │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ HTTPS (nginx reverse proxy, subdomain)
┌───────────────────────────────▼───────────────────────────────────────┐
│ VPS medicscholar                                                       │
│  nginx vhost  fitworld.medigyaan.com  →  127.0.0.1:8400                │
│  fitworld-api (FastAPI, uvicorn, systemd)                              │
│   ├─ Auth (email+password, Argon2/bcrypt JWT v1)                       │
│   ├─ Content API (exercises, categories, programs — versioned)         │
│   ├─ Sync API (workout sessions, streaks)                              │
│   └─ DB: SQLite file /opt/fitworld/data/fitworld.db (WAL)              │
│  Content CMS: git-tracked JSON seed + a minimal admin editor (later)   │
└────────────────────────────────────────────────────────────────────────┘
```

**Offline-first rule:** the app ships/bundles a full content snapshot; server updates merge in. Workouts are recorded locally first and synced opportunistically — losing network never loses a workout.

## 3. Technology pinned

- **iOS:** iOS 17.4+ baseline, SwiftUI, `@Observable` models, MVVM, Swift Concurrency. Project layout: `FitWorld/UI`, `FitWorld/Features/*`, `FitWorld/Data`, `FitWorld/Network`, `FitWorld/Domain` (struct-first models).
- **Package deps (SPM):** `lottie-ios` (Lottie 4), `KeychainAccess` (token storage).
- **Backend:** Python 3 + FastAPI + uvicorn behind nginx, systemd unit `fitworld-api.service`, port 8400 (local only; nginx TLS-terminates). Pydantic v2 schemas. SQLite in WAL mode (fits the low-RAM constraint; MariaDB swap possible later via same repository layer).
- **Media:** Lottie JSON + Lottie-rigged characters served from `/opt/fitworld/media`, streamed through nginx with cache headers. Videos only if we later license real footage.

## 4. Domain data model (canonical, single source of truth — mirrors API schemas)

```swift
// Exercise
Exercise {
  id, slug, names:[(lang, name)], altNames:[String],
  primaryMuscles:[MuscleID], secondaryMuscles:[MuscleID],
  movementPattern, category, objective,
  difficulty: .beginner|.intermediate|.advanced,
  equipment:[EquipmentID], spaceRequired, impactLevel,
  steps:[String], startPosition, formCues:[String],
  breathing, setsRepGuidance per level, restSeconds,
  commonMistakes:[(mistake, correction)],
  easierVariantID, harderVariantID,
  contraindications:[String], substitutions:[ExerciseID],
  demo: { maleAssetID, femaleAssetID, view:.front|.side|.rear },
  reviewStatus, reviewedBy, reviewDate      // safety governance from day 1
}

// Workout
Workout { id, name, level, sessionLengthMin, blocks:[ExerciseBlock] }
ExerciseBlock { exerciseID, sets, reps?, durationSec?, restSec, tempo?, notes }

// Executed
WorkoutSession { id, workoutID, startedAt, endedAt?, entries:[SetEntry], felt: RPE? }
SetEntry { blockIndex, setNumber, reps, weightKg?, durationSec?, completed }

// User profile (onboarding output)
Profile { ageRange?, country, units, heightCm, weightKg, goal, level,
          location: .home|.gym|.outdoor, equipmentAvailable:[EquipmentID],
          weeklyAvailability:[Day:{durationMin}], danceStyles, dietary:[String],
          accessibility:[seated, lowImpact, captions...], entitlements }

// Program
Program { id, name, goal, level, durationDays(7|14|30|60|90),
          weeks:[ { days:[ { blocks } ], deload? } ], progressionRules }
```

Lottie asset naming contract: `{slug}-{male|female}-{front|side|rear}-{b|i|a}.lottie.json`. Player must render **unknown assets gracefully** with a labeled "demo pending" placeholder — never a fake animation.

## 5. API contracts (v1, JSON, bearer-token)

```
POST /api/v1/auth/register      {email, password, name}          → {token, user}
POST /api/v1/auth/login          {email, password}                → {token, user}
GET  /api/v1/me                                                   → user+profile
PUT  /api/v1/me/profile          {profile partial}                → profile
GET  /api/v1/content/manifest                                     → {version, changelog, counts}
GET  /api/v1/content/exercises?since=<ver>&muscle=&level=&equipment=  → [Exercise] (paginated)
GET  /api/v1/content/workouts?since=…&level=&goal=…                  → [Workout]
GET  /api/v1/content/programs?since=…                                 → [Program]
POST /api/v1/sync/sessions       [WorkoutSession]                 → {accepted:[ids], conflicts}
GET  /api/v1/sync/sessions?since=…                                → [WorkoutSession]
```
- All content responses carry `server_version`; app only re-fetches when manifest version bumps.
- JWT (short-lived) + refresh; revoke on password change.
- OpenAPI doc auto-served at `/docs` (FastAPI built-in) — future developers get the contract free.

## 6. Screen inventory (Phase 1)

1. Splash / sign-in / register (Apple Sign-In deferred, email first)
2. **Onboarding** — 8 steps (goal → level → location+equipment → availability → body → preferences → accessibility → summary), skippable optional questions
3. **Home dashboard** — today's plan, continue-last, quick-start timer, streak, weekly calendar strip, recovery nudge (customizable order stored locally)
4. **Library** — muscle-group browser, search, filters (level/equipment/impact/location), exercise detail page (Lottie male/female toggle + speed/loop + steps/mistakes/breathing + timer entry)
5. **Workout player** — block list + live screen (animation, timer ring, set/rep, rest countdown, RPE tag), pause/resume/skip, survives lock/call/background (timer engine restores from wall-clock `startedAt`)
6. **Programs** — generator flow (goal/level/days) → program detail → calendar → start session
7. **History & progress** — session list, volume/streak charts, PR list
8. **Me / Settings** — profile edit, units, accessibility prefs, account (delete/export via API), about + safety disclaimers page

Tab bar: **Home · Train · Library · Me** (nav-design note: 4 tabs, deeper sections reachable from them — matches spec's main navigation).

## 7. Backend build plan (Ph 1B, done on VPS)

1. `/opt/fitworld/{app,data,media,seed}` layout; `fitworld` system user (no root); systemd unit, `Restart=on-failure`, listen 127.0.0.1:8400.
2. FastAPI app: auth (passlib bcrypt), JWT, profile CRUD, content routers (manifest/exercises/workouts/programs), sync router, ETag/static media route via nginx.
3. **Content seed:** author 60–120 real exercises across chest/back/shoulders/arms/core/legs/full-body with beginner/intermediate/advanced variants, structured per §4 (steps, cues, mistakes, breathing, contraindications). Seed as git-tracked JSON → loader applies to DB. Sample content is labeled `review_status: draft` until a qualified reviewer signs off — per master spec, no invented medical claims.
4. nginx vhost `fitworld.medigyaan.com` (Let's Encrypt if DNS permits; else reuse SAN/wildcard of existing cert) → proxy 8400 + `/media/` cache headers.
5. Ops: logrotate, fail2ban-safe (rate-limited login), nightly SQLite backup (existing disk is 86% full → backups compressed <50 MB).

## 8. iOS build plan (Ph 1A, Swift code authored now, Xcode build on Mac)

Milestones in buildable order (each ends with running code):

- **M1 Project skeleton** — Xcode project (generated with XcodeGen on the Mac: `project.yml` kept in repo so Windows-side edits stay authoritative), SPM deps, app shell with 4 empty tabs, dark/light theming, design tokens.
- **M2 Onboarding + profile** — flow UI, validation, local profile store (SwiftData), settings screens.
- **M3 Auth + networking** — register/login against deployed API, keychain tokens, offline fallback mode.
- **M4 Content domain + library** — bundled JSON snapshot loader + API incremental sync; exercise list, filters, detail template incl. safety fields and review-status badge.
- **M5 Lottie player** — male/female switch, front/side view, speed 0.5–2×, loop, "demo pending" fallback.
- **M6 Timer engine (critical)** — `IntervalTimerEngine` using `Date`-anchored wall clock (immune to pausing/backgrounding); work/rest/transition/warm-up/cooldown phases; voice cues (AVSpeechSynthesizer) + haptics; **lock-screen/lifecycle tests**; set/rep tracking.
- **M7 Workout player + session recording** — combine M5+M6; save sessions locally (SwiftData), RPE, streak calc.
- **M8 Sync** — push/pull sessions, conflict policy (last-write-wins per entry), empty states, error toasts.
- **M9 Program generator** — server-defined programs + client personalization rules (equipment filter, duration fit); deload handling.
- **M10 History & charts** — volume/streak/PR visualizations, weekly dashboard.

Each milestone ships with unit tests + a "completed vs pending" checklist against §24 acceptance criteria of the master prompt.

## 9. Explicitly out of Phase 1 (tracked, not forgotten)

Bhangra/dance academy, yoga, stretching, meditation, nutrition/meal planner, physio/rehab + remedies, AI coach, doctor library, challenges/leaderboards, consistency titles, intro offers/coupons, HealthKit, Apple Watch, admin CMS UI, photo food recognition. **The schemas above already reserve fields for these** (categories, reviewStatus, evidence tags, entitlements) so Phase 3+ content plugs in without migrations. Monetization deferred per decision — premium flag only.

## 10. Risks & mitigations

| Risk | Mitigation |
|---|---|
| VPS RAM 1.4 GB free | SQLite over a DB server; uvicorn 1 worker; monitor with existing tooling |
| Disk 86% | Media budget cap; compressed backups; audit before phase start |
| No Mac currently online for builds | Author Swift + Backend fully on Windows; validate Swift syntax via swiftlint in CI later; Mac sync over Tailscale for builds |
| Lottie asset authoring effort (male+female × views × levels) | Start with ~20 hero exercises fully animated; graceful placeholder elsewhere; track coverage indicator in-app (spec-compliant) |
| Health/fitness claims governance | All seeded content `draft` until physio/trainer review; red-flag disclaimer screens included from M4 |
| iOS-only app w/o App Store account | M1 targets simulator first; TestFlight when account is ready |

## 11. Acceptance gate for Phase 1 (maps to master spec §24 subset)

1. Register/login/manage profile against the live VPS API. ✅ → M3
2. Search + filter real exercise records, structured detail page with demo slots. ✅ → M4
3. Timer survives pause/background/lock; sessions complete & save; sync works. ✅ → M6–M8
4. Programs with 3 difficulty levels + equipment modifications + deloads. ✅ → M9
5. Safety notices/disclaimers present wherever exercise instruction shows. ✅ → M4
6. All above verified on device (Mac + iPhone), not just simulator.

**Next action on approval:** scaffold `/opt/fitworld` backend on the VPS + open the iOS repo skeleton (M1/M2 code) — backend and app can be built in parallel.
