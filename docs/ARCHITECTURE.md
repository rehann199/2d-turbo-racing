# Architecture

All diagrams below are original and conceptual. Component names describe responsibilities; they are not the names of production classes or files.

## 1. High-level system architecture

```mermaid
flowchart LR
    subgraph Device
        direction TB
        UI[Screens and widgets]
        ST[UI state<br/>Provider controllers]
        RACE[Race module<br/>input, loop, painter]
        MODELS[Models<br/>progress, wallet, cars, tracks]
        LOCAL[(Local storage<br/>prefs + files)]
        SVC[Services<br/>audio, analytics, ads, billing, auth]
        NATIVE[Native bridges<br/>Kotlin / Swift]
    end
    subgraph Cloud
        AUTH[Firebase Auth]
        FS[(Cloud Firestore)]
        AN[Analytics + Crashlytics]
        STORE[Play Billing / StoreKit]
        ADN[Ad networks]
    end
    UI --> ST --> MODELS
    UI --> RACE
    RACE --> MODELS
    MODELS <--> LOCAL
    MODELS <--> SVC
    SVC --> NATIVE
    NATIVE --> STORE
    SVC <--> AUTH
    SVC <--> FS
    SVC --> AN
    SVC --> ADN
```

Responsibilities:

- **Screens and widgets** render menus, store, garage and dialogs. Heavy screens scope rebuilds with `Selector`.
- **Race module** owns the gameplay simulation and the canvas painter.
- **Models** hold player progress, wallet, catalogs (cars, tracks) and cloud-save documents; they are the source of truth the services read and write.
- **Services** are singletons that wrap external concerns. The game loop does not call SDKs directly.
- **Native bridges** exist for billing (and for ad platform views on TV).

## 2. Gameplay and input flow

<p align="center">
  <img src="../assets/diagrams/input-pipeline.svg" alt="Input to intent to simulation to output pipeline" width="900" />
</p>

The sequence below shows the same flow over time.

```mermaid
sequenceDiagram
    participant In as Input source
    participant IL as Input layer
    participant GL as Gameplay loop
    participant PN as Painter
    participant PS as Persistence
    In->>IL: key / pointer / sensor event
    IL->>IL: normalise, exclusivity, timestamp
    loop every tick
        GL->>IL: read steering intent
        GL->>IL: release stale input if too old
        GL->>GL: clamp dt, update car, road, spawns
        GL->>GL: collisions, pickups, fuel, laps
        GL-->>PN: notify repaint
    end
    GL->>PS: results, rewards, stats
    PS->>PS: save locally, schedule cloud push
```

## 3. Mobile architecture

```mermaid
flowchart TD
    subgraph Mobile app
        S1[Splash + parallel startup pipeline]
        S2[Home and menus]
        S3[Race: buttons or tilt]
        S4[Mobile-only services]
    end
    S4 --> A1[Ad mediation + remote placement gate]
    S4 --> A2[Consent / tracking transparency]
    S4 --> A3[Push + local notifications]
    S4 --> A4[In-app update]
    S4 --> A5[Connectivity monitor]
    S4 --> A6[Google + Apple sign-in]
    S3 --> H[Haptics]
    S3 --> T[Accelerometer]
    subgraph Native
        K[Kotlin: Play Billing, splash, WebView]
        W[Swift: StoreKit billing]
    end
    S1 --> K
    S1 --> W
```

## 4. Android TV architecture

```mermaid
flowchart TD
    subgraph TV app
        T1[Native splash + startup]
        T2[Focus-driven menus]
        T3[Race: remote / D-pad keys]
        T4[TV services]
    end
    T2 --> F1[TV buttons, cards, tabs]
    T2 --> F2[Focus graphs<br/>top bar, mode select]
    T2 --> F3[Back-navigation guard]
    T3 --> G1[Stale-key release]
    T3 --> G2[Focus watchdog]
    T3 --> G3[Back = pause]
    T4 --> M1[Unity Ads + InMobi]
    T4 --> M2[Focus telemetry throttling]
    T4 --> M3[Image-cache limits]
    subgraph Native
        K2[Kotlin: Play Billing, InMobi views, WebView]
    end
    T1 --> K2
```

## 5. Shared vs platform-specific

```mermaid
flowchart LR
    subgraph Shared lineage
        C1[Cars, tracks, game models]
        C2[Race simulation approach]
        C3[Store, garage, rewards UI]
        C4[Cloud-save schema]
        C5[Analytics taxonomy]
    end
    subgraph Mobile only
        M1[Tilt + touch buttons]
        M2[Haptics]
        M3[Ad mediation + gating]
        M4[Push, in-app update, consent]
        M5[Apple sign-in, StoreKit]
    end
    subgraph TV only
        V1[Leanback manifest + banner]
        V2[Focus telemetry]
        V3[Ad SDK integrations for TV]
        V4[Stricter memory policy]
    end
```

## Startup pipeline

Both apps launch through a lightweight splash app that renders the first frame immediately, then runs initialisation in parallel (Firebase, audio, asset preload for the home screen). Mobile additionally gates the app-open ad on confirmed remote config and connectivity, with a timeout so launch is never blocked on an ad. Only after this does the main app mount.

## Data and sync

```mermaid
flowchart LR
    G[Gameplay and store events] --> M[Progress / wallet models]
    M --> L[(Local save)]
    M -->|debounced push| C[(Firestore user doc)]
    C -->|live listener| M
    C -.->|merge on sign-in / resume| M
```

The cloud document carries a schema version, wallet, progress, purchases, settings and a capped race history. Merge rules combine local and cloud data rather than overwriting one with the other.

## Conceptual project layout

A sanitised view of how the code is organised, not the real directory tree:

```text
app
├── entry & startup pipeline     # splash, parallel init, deferred services
├── screens                      # home, modes, garage, tracks, race, store, settings, records
├── race                         # input handling, simulation, painter, overlays
├── models                       # cars, tracks, progress, wallet, cloud save
├── services                     # auth, cloud save, billing, ads, analytics, audio, config
├── widgets                      # shared UI, dialogs, TV focus widgets
└── platform                     # Android (Kotlin) and iOS (Swift) bridges, benchmark modules
```
