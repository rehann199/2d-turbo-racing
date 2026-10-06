<div align="center">

<img src="assets/branding/app-icon.png" alt="2D Turbo Racing app icon" width="110" />

# 2D Turbo Racing

**A top-down 2D racing game with a garage, store and cloud-synced progress, engineered as two apps: a touch/tilt Mobile app and a remote-controlled Android TV app.**

**Mobile · Android TV**<br/>
Flutter · Dart · Firebase · Custom Canvas Rendering (no game engine)

![Flutter](https://img.shields.io/badge/Flutter-Dart-02569B?logo=flutter&logoColor=white)
![Android](https://img.shields.io/badge/Android-3DDC84?logo=android&logoColor=white)
![iOS](https://img.shields.io/badge/iOS-000000?logo=apple&logoColor=white)
![Android TV](https://img.shields.io/badge/Android%20TV-D9822B?logo=androidtv&logoColor=white)
![Firebase](https://img.shields.io/badge/Firebase-FFCA28?logo=firebase&logoColor=black)

<table>
  <tr>
    <td><img src="assets/screenshots/mobile/mobile-main-menu.webp" width="440" alt="Mobile main menu: Play Now, Garage, Store, Daily Rewards, limited-time offer" /></td>
    <td><img src="assets/screenshots/mobile/mobile-gameplay.webp" width="440" alt="Mobile gameplay on the Sunset Beach track with speed, fuel, score and collectible HUD" /></td>
  </tr>
</table>
<sub>Real screenshots from the Mobile build.</sub>

</div>

---

## Get the Game

<p>
  <a href="https://play.google.com/store/apps/details?id=co.futurewatch.turborg2d.racing&hl=en"><img alt="Get it on Google Play" src="https://img.shields.io/badge/Google%20Play-Get%20the%20game-414141?style=for-the-badge&logo=googleplay&logoColor=white" /></a>
  <a href="https://apps.apple.com/au/app/turbo-racing-arcade-car-game/id6764758733"><img alt="Download on the App Store" src="https://img.shields.io/badge/App%20Store-Download-0D96F6?style=for-the-badge&logo=apple&logoColor=white" /></a>
</p>

## Product at a Glance

| | |
|---|---|
| **What it is** | Landscape, lane-based arcade racing with 10 themed tracks, 14 cars and four difficulty tiers |
| **Meta-game** | Garage, coin/gem economy, fuel, power-ups, 7-day daily rewards, limited-time offers, records |
| **Platforms** | Mobile app (Android and iOS) and an Android TV app, built from one game design |
| **Rendering** | Hand-written `CustomPainter` canvas renderer with a delta-time loop; no game engine |
| **Services** | Firebase Auth, Firestore, Analytics, Crashlytics; native Play Billing (Kotlin) and StoreKit (Swift) bridges |
| **Code size** | About 55k lines of Dart in the Mobile app and 47k in the TV app (`lib/` only) |

## Live Product Showcase

<table>
  <tr>
    <td align="center"><img src="assets/screenshots/mobile/mobile-game-modes.webp" width="300" alt="Game modes screen"/><br/><sub><b>Game modes</b><br/>Drive with Traffic; a second mode is marked coming soon</sub></td>
    <td align="center"><img src="assets/screenshots/mobile/mobile-car-select.webp" width="300" alt="Car selection with per-car stats"/><br/><sub><b>Car selection</b><br/>Per-car stats, default car, coin prices</sub></td>
    <td align="center"><img src="assets/screenshots/mobile/mobile-track-select.webp" width="300" alt="Track selection with locked tracks and coin prices"/><br/><sub><b>Track selection</b><br/>Free and purchasable tracks</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/mobile/mobile-free-play-setup.webp" width="300" alt="Free play setup with traffic level selector"/><br/><sub><b>Free play setup</b><br/>Traffic level selector</sub></td>
    <td align="center"><img src="assets/screenshots/mobile/mobile-turbo-store.webp" width="300" alt="Turbo Store with bundles, coins, gems, cars, tracks, nitros and fuel"/><br/><sub><b>Turbo Store</b><br/>Bundles, currencies, cars, tracks, boosters</sub></td>
    <td align="center"><img src="assets/screenshots/mobile/mobile-controls.webp" width="300" alt="Controls screen with Gyro and Buttons steering modes"/><br/><sub><b>Controls</b><br/>Gyro (tilt) or on-screen buttons</sub></td>
  </tr>
</table>

## One Game. Two Interaction Models.

The same racing product runs in two very different environments. Mobile is touch- and sensor-driven. Android TV has no touchscreen and is driven entirely by a remote, so focus handling and input reliability become first-order engineering concerns.

<p align="center">
  <img src="assets/diagrams/platform-overview.svg" alt="Platform overview: Mobile app, shared game design, Android TV app, cloud services" width="900" />
</p>

<table>
  <tr>
    <td width="50%" align="center"><img src="assets/screenshots/mobile/mobile-settings-hub.webp" width="400" alt="Mobile settings hub"/><br/><sub><b>Mobile</b>: real build screenshot</sub></td>
    <td width="50%" align="center"><img src="assets/branding/tv-launcher-banner.jpg" width="400" alt="Android TV launcher banner artwork"/><br/><sub><b>Android TV</b>: launcher banner artwork shipped with the TV app (artwork, not a screenshot)</sub></td>
  </tr>
</table>

| | Mobile | Android TV |
|---|---|---|
| **Platform** | Android and iOS | Android TV (Leanback launcher, touchscreen optional) |
| **Primary input** | On-screen buttons or accelerometer tilt | Remote / D-pad / key events |
| **Navigation model** | Touch, with focus support retained | Focus traversal with explicit focus graphs |
| **Screen environment** | Handheld, landscape, either orientation | Living-room display, landscape |
| **Control architecture** | Pointer down/up/cancel; calibrated, smoothed tilt | Raw key events with stale-input release and focus watchdog |
| **UI considerations** | Scaled from a 1080p baseline with a minimum scale so phones stay legible | Scaled from a 1080p baseline; custom focus-state widgets, no touch ripples |
| **Performance** | Image-cache cap, trimming before heavy screens, R8, baseline profile | Same cache policy, justified in code by TV heap limits |
| **Platform services** | Push, in-app update, consent, Apple/Google sign-in, ad mediation | Google sign-in, Unity Ads and InMobi integrations |

Full comparison: [docs/MOBILE_VS_TV.md](docs/MOBILE_VS_TV.md).

## Engineering Highlights

### TV remotes lose key-up events
- **Problem:** a dropped key-up leaves the car steering in one direction forever.
- **Approach:** timestamped key state, auto-release timers plus a per-frame stale check, left/right mutual exclusion, and input accepted only while gameplay is active.
- **Outcome:** steering fails safe. If input goes quiet the car stops steering, and input recovers after dialogs or overlays steal focus (focus watchdog, back-navigation guard).

### Unknown ad configuration
- **Problem:** ad placements are controlled remotely, and showing ads on an unconfirmed config risks unintended placements.
- **Approach:** a single decision gate that fails closed until the remote config is confirmed, with background retry, per-zone request throttling, a per-race cap on rewarded ads and an audit trail.
- **Outcome:** no ad loads or shows on an unknown config, and the game loop has no direct dependency on ad SDKs.

### Mobile tilt steering across orientations
- **Problem:** raw accelerometer data is noisy, depends on how the player holds the phone, and flips sign when the device is rotated 180° between landscape orientations.
- **Approach:** detect the landscape flip and correct the sensor axis, calibrate the neutral point from an averaged sample, then smooth and apply a dead zone, scaled by a user sensitivity setting.
- **Outcome:** tilt direction stays correct in both landscape orientations, and the player's natural hold angle becomes neutral.

### More
| Area | What was built |
|---|---|
| **Rendering** | Single `CustomPainter` repainted from a lightweight notifier; sprites pre-decoded during a loading phase and disposed explicitly; delta-time loop with clamping; rectangle-overlap collision |
| **Persistence** | Local-first save with a schema-versioned Firestore document, merge and de-duplication, capped history, live listener with a write-echo guard |
| **Billing** | Kotlin Play Billing and Swift StoreKit managers behind platform channels, connectivity-aware initialisation |
| **Resources** | Capped image cache, trimming before heavy screens, race sprites loaded only for the selected car and track |
| **Release engineering** | R8 and resource shrinking, ABI filters, baseline-profile and macrobenchmark (startup, jank) modules |
| **Tests** | Ad-gate decisions, analytics validation, progress-save merge, TV focus telemetry throttling |

Details and trade-offs: [docs/ENGINEERING_CHALLENGES.md](docs/ENGINEERING_CHALLENGES.md).

## Architecture

```mermaid
flowchart TD
    subgraph Input
        T[Touch buttons] --> N
        A[Tilt sensor] --> N
        R[Remote / D-pad / keys] --> N
    end
    N[Input normalisation] --> L[Gameplay loop<br/>delta-time update]
    L --> P[Canvas painter]
    L --> S[Audio / haptics]
    L --> G[Progress + wallet models]
    G --> LS[(Local storage)]
    G <--> CS[Cloud save<br/>Firestore merge]
    UI[Screens + widgets<br/>Provider state] <--> G
    UI --> L
    SV[Services: ads, billing, analytics,<br/>auth, remote config] <--> UI
```

More diagrams (input pipeline, Mobile, TV, shared vs platform-specific, sync): [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Technology Stack

| Layer | Technology |
|---|---|
| App framework | Flutter / Dart 3 |
| Rendering | `CustomPainter` canvas, `dart:ui` images |
| UI state | `provider` with `ChangeNotifier` controllers and scoped `Selector`s |
| Audio / sensors | `just_audio`, `sensors_plus` (Mobile) |
| Backend | Firebase Auth, Cloud Firestore, Analytics, Crashlytics, Messaging (Mobile) |
| Sign-in | Google; Apple (Mobile) |
| Billing | Native Kotlin (Play Billing) and Swift (StoreKit, Mobile) via platform channels |
| Ads | Mediation SDK (Mobile); Unity Ads and InMobi (TV) |
| Tooling | R8, Android baseline profiles, macrobenchmark |

## Project Scope

2D Turbo Racing is a production software project spanning Mobile and Android TV. This portfolio focuses on the engineering architecture, cross-platform implementation and technical challenges represented in the application.

## Documentation

| Read | For |
|---|---|
| [Technical case study](docs/TECHNICAL_CASE_STUDY.md) | End-to-end technical write-up |
| [Architecture](docs/ARCHITECTURE.md) | Diagrams and component responsibilities |
| [Mobile vs Android TV](docs/MOBILE_VS_TV.md) | Platform comparison in depth |
| [Engineering challenges](docs/ENGINEERING_CHALLENGES.md) | Problem, approach, trade-offs, outcome |
| [Product overview](docs/PRODUCT_OVERVIEW.md) | Gameplay, content, screens |
| [Privacy and source](docs/PRIVACY_AND_SOURCE.md) | What is and is not published |

## Source Code

This repository is a portfolio showcase for a production application. Production source code is maintained privately. See [RIGHTS.md](RIGHTS.md) for usage terms.
