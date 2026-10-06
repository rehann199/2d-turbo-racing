# Engineering Challenges

Six problems that stand out in the production code. Outcomes describe observable behaviour in the code, not measured metrics; none are claimed.

---

## 1. Reliable steering from a TV remote

**Problem**
Remote controls and TV key pipelines can drop key-up events or deliver repeated key-downs. A missed key-up leaves the car steering in one direction indefinitely.

**Context**
The Android TV app is played with a remote and no touchscreen. The race screen reads raw key events, and dialogs or system overlays can interrupt key delivery.

**Engineering approach**
- Timestamp every direction press and treat input older than a short threshold as stale.
- Release stale input both from a per-press timer and from the main loop's per-frame check.
- Enforce mutual exclusion between left and right and clear both if they conflict.
- Apply a small steering impulse on key-down so a tap produces a deterministic nudge even if the key-up never arrives.
- Gate directional input by game state so presses meant for dialogs cannot leak into gameplay.
- Emit throttled telemetry when a stale release occurs so field issues are visible.

**Trade-offs**
A hard release threshold means a user deliberately holding a direction relies on key-repeat to keep steering. That is acceptable on remotes (which repeat), but would feel wrong on a controller that does not repeat.

**Outcome**
Steering fails safe: if input goes quiet, the car stops steering instead of drifting.

---

## 2. Keeping gameplay focus on a focus-driven platform

**Problem**
Flutter's focus system can move primary focus to a dialog, overlay or another widget. When that happens during a race on TV, the game stops receiving key events.

**Context**
The race screen shows pause, crash, refuel, reward and ad-related dialogs that open and close frequently, and Back must pause rather than leave.

**Engineering approach**
- A dedicated focus node for gameplay with listeners for focus loss.
- On focus loss: clear directional input immediately, then request focus again after a short delay if the race is still active.
- A watchdog timer that runs several times per second and re-requests focus when the game is active, not paused, no dialog is open and the app is in the foreground.
- Dialog-open flags that prevent duplicate dialogs from key-repeat on Back.
- A separate navigation guard that stops a single Back press from popping both a nested route and the root.

**Trade-offs**
A polling watchdog is blunt and costs a little work each tick, but it is far simpler and more robust than trying to enumerate every way focus can be taken.

**Outcome**
Gameplay input recovers automatically after dialogs, ads and overlays.

---

## 3. Tilt steering that behaves in both landscape orientations

**Problem**
Raw accelerometer values are noisy, differ per user's resting hold angle, and flip sign when the device is rotated 180° between the two landscape orientations.

**Context**
The Mobile app is landscape-only and allows both landscape directions.

**Engineering approach**
- Detect which landscape orientation is active from safe-area padding asymmetry and flip the sensor axis accordingly.
- Calibrate the neutral point from the average of the first samples instead of a single reading.
- Recalibrate after an orientation change.
- Low-pass smooth the signal, apply a dead zone, and scale by a user sensitivity setting.
- Ignore tilt entirely in button-steering mode.

**Trade-offs**
Using padding asymmetry to infer orientation is indirect, but avoids an extra platform channel; it assumes the device has asymmetric insets, and falls back to a default otherwise.

**Outcome**
Tilt direction stays correct when the device is flipped, and a user's natural hold angle becomes the neutral point.

---

## 4. Local-first progress with cloud sync

**Problem**
Players can progress offline, on more than one device, and across reinstalls. Naive "last write wins" would lose purchases and progress.

**Context**
Progress, wallet, purchases, settings and race history live in local storage and, for signed-in users, in a per-user Firestore document.

**Engineering approach**
- A schema-versioned cloud document and a defined merge procedure (union-style) for local and cloud data.
- De-duplication of race history with a derived key and a cap on stored entries to stay far under document size limits.
- Debounced pushes after local saves.
- A live listener so changes on another device appear without waiting for sign-in or resume.
- A guard flag so applying a remote snapshot does not trigger a push that re-fires the same listener.
- A unit test around the save-merge behaviour.

**Trade-offs**
The merge rules are more complex than overwriting and some fields necessarily favour one side; capping history drops the oldest entries.

**Outcome**
Progress converges across devices without a feedback loop between listener and writer.

---

## 5. Advertising that must never run on unknown configuration

**Problem**
Ad placements are controlled remotely. If the app cannot read that configuration, showing ads anyway can violate intended placement rules, and aggressive re-requesting can cause ad networks to throttle fills.

**Context**
The Mobile app uses a mediation SDK with banner, native, interstitial, rewarded and app-open formats. The game also has strict moments where an ad must not appear (active gameplay).

**Engineering approach**
- A single decision function that every "should this ad load or show" check routes through, with an audit record.
- Fail-closed behaviour: without a confirmed remote config for the session, no ad loads or shows. A bounded background retry picks the config up later and re-evaluates placements.
- A hard minimum interval per ad zone, added after bursts of overlapping requests were observed to suppress fills.
- Ads load on demand rather than being pre-warmed.
- A cap on rewarded ads watched in one race.
- Unit tests for the gate and for ad-placement analytics.

**Trade-offs**
Fail-closed protects policy compliance but gives up revenue on launches where the config read fails. On-demand loading adds latency at show time compared with preloading.

**Outcome**
Ad behaviour is explainable and testable, and the game loop has no direct dependency on ad SDKs.

---

## 6. Staying inside TV-class memory

**Problem**
Menu and store screens use large artwork. Flutter's default image cache and unmanaged decodes can exhaust the heap on TV hardware.

**Context**
Code comments in the project cite out-of-memory behaviour on Android TV.

**Engineering approach**
- Cap the image cache by entry count and bytes at startup.
- Trim the cache before entering heavy screens (garage, race, store).
- Pre-decode and hold only the sprites a race needs, and dispose them explicitly afterwards.
- Load assets per screen rather than preloading everything: only home-screen assets are preloaded at startup, and race sprites are preloaded during the pre-race loading screen.

**Trade-offs**
A small cache means images are re-decoded more often on navigation; the design accepts that to avoid spikes.

**Outcome**
Memory use is bounded by policy rather than by whatever the screens happen to load.
