# Technical Case Study: 2D Turbo Racing

A detailed write-up of the engineering behind a two-platform Flutter racing game. It is based on read-only inspection of the production Mobile and Android TV codebases. No production source is reproduced; descriptions are deliberately high level.

## Contents

1. [Project context](#project-context)
2. [Objectives and requirements](#objectives-and-requirements)
3. [System architecture](#system-architecture)
4. [Rendering and game systems](#rendering-and-game-systems)
5. [Input architecture](#input-architecture)
6. [Mobile implementation](#mobile-implementation)
7. [Android TV implementation](#android-tv-implementation)
8. [Cross-platform considerations](#cross-platform-considerations)
9. [Persistence and data](#persistence-and-data)
10. [Integrations](#integrations)
11. [Performance](#performance)
12. [Testing and release engineering](#testing-and-release-engineering)
13. [Design decisions and lessons](#design-decisions-and-lessons)
14. [Maintainability](#maintainability)
15. [Future technical improvements](#future-technical-improvements)

---

## Project context

2D Turbo Racing is a free-to-play top-down racing game with a garage, store, daily rewards and ads. It had to run on touch-first phones and on Android TV, where there is no touchscreen and the user holds a remote. The project is a commercially oriented product, which shaped many decisions: billing, ad policy, analytics and cloud save are first-class concerns rather than add-ons.

The two apps are Flutter projects. The Mobile lib is about 55k lines of Dart; the TV lib about 47k.

## Objectives and requirements

**Product**
- Fast-to-learn arcade racing with a collection and upgrade loop
- Same content and economy on every platform
- Progress that follows the player across devices
- Monetisation through purchases and ads, without ads during gameplay

**Technical**
- Playable with touch, tilt, remote or keyboard keys
- Predictable frame-to-frame behaviour on a wide range of devices, including low-memory TV hardware
- Offline-tolerant: local save first, cloud merge later
- Remote control over offers and ad placements without an app update
- Observability: analytics funnels and crash reporting

## System architecture

The app layers are:

1. **Presentation**: screens and widgets, with scoped rebuilds through Provider selectors.
2. **Race module**: input handling, simulation and the canvas painter.
3. **Domain models**: cars, tracks, progress, wallet, race results, cloud-save documents.
4. **Services**: authentication, cloud save, billing, ads, analytics, audio, remote config, ratings prompts, notifications.
5. **Platform bridges**: Kotlin (billing, splash, WebView, TV ad views) and Swift (StoreKit billing).

See [ARCHITECTURE.md](ARCHITECTURE.md) for diagrams.

## Rendering and game systems

**No game engine.** The race scene is drawn by a `CustomPainter` on Flutter's canvas. For a 2D top-down game with scrolling backgrounds, sprites and simple effects this keeps the dependency surface small and keeps UI and game in one toolkit. The cost is that features an engine gives for free (scene graph, asset pipeline, physics) are hand-written.

**Loop.** A periodic timer drives the simulation. Each tick measures elapsed time, clamps it so a pause or slow frame cannot produce a huge step, and scales movement by a factor relative to a 60 fps baseline. The painter repaints from a lightweight notifier, so the whole widget tree is not rebuilt each tick.

**Physics model.** Steering is an impulse-and-friction model: key or button input adds lateral velocity, friction decays it, and lateral speed is capped by the car's handling. Tilt input eases toward a target velocity proportional to the tilt. When no input is present the car glides to a stop. Road speed comes from the car's speed stat, a per-track difficulty multiplier and a nitro multiplier.

**Entities and spawning.** Obstacles, traffic and pickups are placed on lanes. Spawn logic checks separation from existing entities, caps pickups per race, and uses per-theme obstacle art. Race difficulty and the free-play traffic level adjust pacing.

**Collision.** Rectangle-overlap tests in canvas coordinates, aligned with how the car is painted; entities clearly off-screen are skipped. Hits consume a shield if active, otherwise crash.

**Progress inside a race.** Distance converts to laps. Fuel drains on a timer. Nitro, shield and magnet apply timed effects. Collection triggers particle animations.

**Audio and haptics.** Background music loops through `just_audio`, responds to app lifecycle and to ads (paused while an ad is on screen), and respects volume and music settings. Mobile adds light and heavy haptics.

## Input architecture

Input sources are normalised into a small set of steering and action intents:

| Source | Platform | Notes |
|---|---|---|
| Pointer buttons | Mobile | Down, up and cancel all handled |
| Accelerometer | Mobile | Orientation-aware, calibrated, smoothed |
| Raw key events | TV (and keyboards) | Arrow/WASD, OK/Select/Enter, Back |

The input layer enforces left/right exclusivity, timestamps presses, and releases stale input. Directional events are accepted only while gameplay is actually active so dialog navigation cannot steer the car. See [Engineering challenges](ENGINEERING_CHALLENGES.md) for the TV-specific work.

## Mobile implementation

- Landscape-locked, sensor-based so both landscape orientations work (including inside ad activities).
- Splash activity natively, then a Flutter loading screen, then the app. Startup runs Firebase, audio, home asset preload and connectivity in parallel.
- Ads: a mediation SDK with several adapters. Ad placements are toggled via remote config; decisions are centralised, audited and throttled. App-open, interstitial and rewarded flows are tied to specific game moments.
- Consent and tracking-transparency handling before ad personalisation.
- Sign-in with Google and with Apple (including nonce handling and resilience to Android lifecycle churn after the browser round-trip).
- Billing: a Kotlin Play Billing manager and a Swift StoreKit manager behind platform channels, with connectivity-aware initialisation.
- Push and local notifications, in-app update prompts.
- Release: R8 and resource shrinking, ABI filters, baseline profile and macrobenchmark modules.

## Android TV implementation

- Leanback manifest declarations, launcher banner, touchscreen optional.
- A TV widget kit: button, card, tab, badge widgets with built-in focus visuals and no ripple.
- Focus graphs for the top bar and mode-select screens, so D-pad order is deterministic.
- A back-navigation guard to prevent double pops.
- Race-screen key handling with stale-input release and a focus watchdog.
- Throttled focus and key telemetry implemented as pure Dart so that it is unit-testable.
- Ads through Unity Ads and InMobi; native platform views for InMobi banner and native formats. Double taps from remote presses are blocked while an ad is loading.
- Memory policy: a capped image cache, trimmed before heavy screens.

## Cross-platform considerations

- **Shared design, separate apps.** Catalogs, models and most screens match; the platforms diverge where it matters (input, ads, services, manifests).
- **Platform code stays at the edges.** Billing, ads, notifications, update checks and sign-in are services behind small interfaces, so the race module has a few well-defined dependencies.
- **Scaling.** UI is scaled from a 1080p design size; Mobile adds a minimum scale to keep phones legible.
- **Parity cost.** Because the apps are separate projects, fixes to shared logic need to be applied twice.

## Persistence and data

- **Local:** `shared_preferences` for settings and small state; file storage for race history, with a preferences fallback if file writes fail.
- **Cloud:** a per-user Firestore document with a schema version, wallet, progress, purchases, settings and a capped race history. Pushes are debounced after local saves; a live listener applies remote changes; a flag prevents the listener/push feedback loop. Merge rules combine local and cloud state; a unit test covers the save merge.
- **Remote catalogs and config:** a public read-only Firestore collection holds limited-time-offer catalogs; another holds ad configuration. User documents are readable and writable only by their owner according to the rules in the project.
- **Daily rewards:** a 1-to-7-day streak with reset on a missed day, computed from stored timestamps.

## Integrations

| Concern | Mobile | TV |
|---|---|---|
| Auth | Firebase Auth, Google, Apple | Firebase Auth, Google |
| Data | Firestore | Firestore |
| Analytics | Firebase Analytics with a validated event taxonomy | Same, plus focus telemetry |
| Crashes | Crashlytics | Crashlytics |
| Billing | Play Billing, StoreKit | Play Billing |
| Ads | Mediation SDK | Unity Ads, InMobi |
| Messaging | FCM, local notifications | None observed |

Analytics events are documented in the project, cover navigation, purchases, ads, races and platform-specific input issues, and are covered by validation tests that check parameters against GA4 constraints.

## Performance

- Delta-time updates with clamping
- Pre-decoded sprites, explicit disposal, no per-frame decodes
- Early-outs for off-screen entities in collision checks
- Selector-scoped rebuilds and a separate repaint notifier for the canvas
- Image-cache limits and trimming
- Parallel, timeout-guarded startup with a native splash
- Release: shrinking, ABI filters, baseline profile, macrobenchmark startup and jank benchmarks
- Ads load on demand; request throttling prevents bursts

No benchmark figures are published here because none were collected for this showcase.

## Testing and release engineering

Automated tests exist for: ad-gate decisions, ad-placement analytics, analytics parameter validation, player-progress save merging, and TV focus telemetry throttling. A default widget test also exists. Diagnostic tooling includes an ad-debug surface and helper scripts. The race loop itself and the UI are not covered by automated tests in the inspected projects, which is typical for a canvas game of this shape but a real gap.

## Design decisions and lessons

1. **Fail closed for anything money- or policy-related.** Ads and rewards do nothing when config or state is uncertain.
2. **Defend against hardware, not just code.** The TV input work exists because real remotes behave differently from simulated key events.
3. **Put platform differences at the boundary.** Differences in input, ads and manifest are cheaper to maintain than conditionals inside the game loop.
4. **Prefer simple blunt recovery.** A focus watchdog is less elegant than precise focus bookkeeping, but it works across every way focus can be lost.
5. **Make diagnostics a product feature.** Throttled telemetry for input and ad decisions made hard-to-reproduce device issues visible.

## Maintainability

Strengths: clear separation of services from UI; models with JSON round-tripping; documentation of the analytics taxonomy; platform-specific code isolated in services.

Weaknesses: the race screen is a very large single file combining state, input, simulation and parts of the UI; shared code is duplicated between two repositories; some dead or disabled code paths remain (for example a disabled staged-precache service and a no-op ad preload coordinator kept for API compatibility).

## Future technical improvements

- **Extract the simulation** from the race widget into a plain Dart class with a fixed timestep, making it unit-testable and shareable.
- **Create a shared package** for models, catalogs, persistence, sync and simulation, with thin Mobile and TV shells.
- **Typed identifiers and enums** in place of string-based difficulty, rarity and theme values.
- **Replace polling** in the focus watchdog with a focus-scope strategy where feasible, keeping the watchdog as a fallback.
- **Add golden and integration tests** for key screens and the race flow.
- **Add on-device frame-time telemetry** to complement the existing benchmark modules.
- **Retire dead code** and finish migrating deprecated keyboard APIs.
