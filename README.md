# TerraOffline: Autonomous On-Device Travel Companion

[![Flutter Version](https://img.shields.io/badge/Flutter-3.x-02569B?logo=flutter)](https://flutter.dev)
[![Architecture](https://img.shields.io/badge/Architecture-Feature--First%20%2F%20Riverpod-teal)](#architecture)
[![Network Mode](https://img.shields.io/badge/Connectivity-Zero--Network%20Offline--First-success)](#project-vision)

An offline-first mobile travel companion built with Flutter. TerraOffline empowers travelers to explore unfamiliar cities without internet connectivity or cellular roaming by packaging vector maps, on-device machine learning models for live signage translation, spatial landmark discovery, and multicurrency ledgering into self-contained destination bundles.

---

## Table of Contents

- [Project Vision](#project-vision)
- [Core Objectives](#core-objectives)
- [Navigation Flow & App Hierarchy](#navigation-flow--app-hierarchy)
- [Technical Architecture](#technical-architecture)
- [Key Features](#key-features)
- [Tech Stack & Package Dependencies](#tech-stack--package-dependencies)
- [Getting Started & Local Setup](#getting-started--local-setup)
- [Project Directory Structure](#project-directory-structure)

---

## Project Vision

Modern travel applications assume continuous high-speed cellular access. When travelers cross international borders, enter subway tunnels, or hike remote trails, standard apps fail—leaving them stranded without maps, language aids, or budget tracking.

**TerraOffline** reverses this paradigm:
1. **Zero-Trust Network Model:** All critical travel utilities (map rendering, camera translation, routing, and expense conversion) run 100% on-device in airplane mode.
2. **Deterministic Pre-Caching:** Destination packs (maps, spatial databases, and translation weights) are bundled and downloaded upfront over Wi-Fi.
3. **Battery-Aware Edge Computing:** Heavy computational tasks (OCR parsing, coordinate distance sorting, TSP routing) are delegated to background Dart isolates, keeping the UI at a continuous 60/120 fps.

---

## Core Objectives

| Objective | Target Metric | Engineering Strategy |
| :--- | :--- | :--- |
| **Complete Network Autonomy** | 0 network requests during active navigation | Local SQLite/Drift databases and `.mbtiles` raster/vector rendering. |
| **Real-Time On-Device Translation** | $\le 120\text{ ms}$ bounding & text replacement | `google_mlkit_text_recognition` + offline `google_mlkit_translation` language models. |
| **Instant Spatial POI Search** | $\le 15\text{ ms}$ discovery radius query | SQLite R-Tree spatial indexing with haversine distance filtering. |
| **Battery Conservation** | $< 8\%$ battery drain per active hour | Geofence distance filters ($> 25\text{ m}$ threshold) over continuous GPS polling. |
| **Pedestrian Waypoint Sequencing** | Sub-second route generation | Client-side 2-opt heuristic Traveling Salesperson Problem (TSP) solver. |

---

## Navigation Flow & App Hierarchy

The application employs a bottom-navigation shell with a global modal overlay system for camera-driven tasks.

```
                      ┌────────────────────────────┐
                      │    First Launch Check      │
                      └─────────────┬──────────────┘
                                    │
                    [No Active City Pack Downloaded]
                                    │
                                    ▼
                      ┌────────────────────────────┐
                      │ City Pack Downloader View  │
                      │  (Search, Fetch & Seed)   │
                      └─────────────┬──────────────┘
                                    │
                     [City Pack Cached & Validated]
                                    │
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │             Main Navigation Shell (Scaffold)            │
       └────┬─────────────────┬────────────────────┬────────────┬┘
            │                 │                    │            │
            ▼                 ▼                    ▼            ▼
     ┌─────────────┐   ┌─────────────┐      ┌────────────┐┌────────────┐
     │ Tab 1:      │   │ Tab 2:      │      │ Tab 3:     ││ Tab 4:     │
     │ Discovery   │   │ Interactive │      │ Multicur-  ││ City Pack  │
     │ Radar (Feed)│   │ Vector Map  │      │ rency Log  ││ Manager    │
     └──────┬──────┘   └──────┬──────┘      └─────┬──────┘└────────────┘
            │                 │                   │
            │                 ▼                   │
            │          ┌──────────────┐           │
            ├─────────►│ Landmark     │           │
            │          │ Detail Sheet │           │
            │          └──────┬───────┘           │
            │                 │                   │
            ▼                 ▼                   ▼
     ┌───────────────────────────────────────────────────┐
     │             Quick Action Overlay HUD              │
     │   ┌─────────────────────────┬─────────────────┐   │
     │   │ Live AR Camera Lens     │ Receipt OCR     │   │
     │   │ (Real-Time Translation) │ Scanner Form    │   │
     │   └─────────────────────────┴─────────────────┘   │
     └───────────────────────────────────────────────────┘
```

### Screen Flow Breakdown

1. **City Pack Downloader (Onboarding/Setup):**
   - Discovered when no destination pack exists locally.
   - Allows users to search and download combined city packages (MBTiles, POI database, translation weights) over Wi-Fi.

2. **Tab 1: Discovery Radar (Home Feed):**
   - Displays landmarks, transit hubs, and cultural points dynamically sorted by haversine distance from the user's current GPS fix.
   - Features heading compass indicators directing toward selected landmarks.

3. **Tab 2: Interactive Vector Map (`flutter_map`):**
   - Offline tile exploration with custom layers for walking paths, viewpoints, and transit stations.
   - Tapping an entity opens the **Landmark Detail Sheet** with historical context and visiting tips.
   - Multi-waypoint selection to trigger the **Offline Waypoint Sequencer**.

4. **Tab 3: Multicurrency Ledger:**
   - Tracks expenses against daily budgets.
   - Automatically converts local prices to the traveler's home currency using cached rates.
   - Quick FAB opens the **Receipt OCR Scanner** to parse totals automatically.

5. **Tab 4: Pack Manager & Settings:**
   - Manage installed regional packs, inspect storage footprints, delete unneeded cities, and download additional offline language packs.

6. **Camera Action HUD (Global Modal):**
   - **AR Sign Translator:** Live camera stream passing frames to ML Kit; paints translated text bounding boxes directly over detected foreign words.
   - **Receipt Scanner:** Single-shot capture that crops text, isolates numeric currency patterns, and populates the expense entry modal.

---

## Technical Architecture

The codebase follows a **Feature-First / Layered Architecture** powered by Riverpod:

```
lib/
├── app/                        # App config, routing, themes
├── core/
│   ├── database/               # Drift SQLite schema, migrations, R-Tree virtual tables
│   ├── errors/                 # Domain failures & custom exceptions
│   ├── isolates/               # Heavy computation workers (TSP, OCR, Spatial Math)
│   ├── services/               # Device sensors (GPS, Magnetometer, File System)
│   └── utils/                  # Coordinate math, formatters, currency helpers
└── features/
    ├── camera_vision/          # Live frame stream, ML Kit OCR, on-device translation
    ├── city_packs/             # Bundle extraction, archive validation, download manager
    ├── expenses/               # Ledger domain, receipt parser, FX conversion
    ├── map/                    # flutter_map view, local tile provider, polyline rendering
    └── radar/                  # Proximity calculations, compass bearing stream
```

---

## Key Features

* **Zero-Network Street Sign Translation:** Real-time on-device text recognition and translation using downloadable Google ML Kit language models (~30 MB per pair).
* **Offline Vector/Raster Map Canvas:** Smooth zooming and panning using `flutter_map` backed directly by local `.mbtiles` or `.pmtiles` archives.
* **Spatial R-Tree POI Discovery:** Local SQLite R-tree indexing enables rapid nearest-neighbor queries without scanning entire POI tables.
* **Waypoint Tour Sequencer:** Locally arranges up to 10 daily stops into the most efficient walking sequence using a 2-opt heuristic algorithm.
* **Smart Currency & Receipt Ledger:** Logs transactions, extracts totals from paper receipts via OCR, and converts amounts based on pre-cached exchange rates.

---

## Tech Stack & Package Dependencies

| Category | Library | Purpose |
| :--- | :--- | :--- |
| **Framework** | `Flutter (Dart 3)` | Cross-platform mobile development (iOS & Android). |
| **State Management** | `flutter_riverpod` | Dependency injection and reactive state streams. |
| **Navigation** | `go_router` | Declarative routing and deep link handling. |
| **Persistence** | `drift` + `sqlite3_flutter_libs` | Type-safe SQLite database with custom spatial query support. |
| **Mapping Engine** | `flutter_map` + `flutter_map_mbtiles` | Rendering local map tiles without third-party API keys. |
| **Vision & ML** | `google_mlkit_text_recognition`<br>`google_mlkit_translation` | Edge text parsing and offline language translation. |
| **Hardware & Sensors**| `geolocator`<br>`flutter_compass`<br>`camera` | GPS telemetry, directional azimuth, and raw camera frames. |

---

## Getting Started & Local Setup

### Prerequisites

- [Flutter SDK](https://docs.flutter.dev/get-started/install) (`>= 3.19.0`)
- Android Studio / Xcode configured for native builds
- Physical test device recommended (camera streaming and compass sensors do not fully run in desktop simulators)

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-username/terra_offline.git
   cd terra_offline
   ```

2. **Install Flutter packages:**
   ```bash
   flutter pub get
   ```

3. **Run code generation (Drift & Riverpod):**
   ```bash
   dart run build_runner build --delete-conflicting-outputs
   ```

4. **Verify device setup:**
   Ensure your physical device has camera and location permissions granted:
   ```bash
   flutter doctor -v
   ```

5. **Run the project:**
   ```bash
   flutter run
   ```

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.