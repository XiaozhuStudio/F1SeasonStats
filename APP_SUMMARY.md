# F1 Season Stats — App Summary

## Overview

**F1 Season Stats** is an iOS app for tracking Formula 1 season statistics. It provides a read-only viewer for race results, standings, and detailed charts — all synced automatically from a Firebase Firestore backend. Data is entered via a companion app, **F1 Season Editor**, by the app author.

- **Current season:** 2026
- **Platform:** iOS
- **Tech stack:** SwiftUI, SwiftData, Firebase Firestore, StoreKit 2, Swift Charts, FlagKit

---

## Targets

| Target | Role |
|--------|------|
| `F1 Season Stats` | Main consumer app — read-only stats viewer, auto-syncs from Firestore on launch |
| `F1 Season Editor` | Companion editor — used by the developer to enter race results and upload to Firestore |

---

## App Flow

1. **First launch** — `WelcomeView` displays a 5-screen animated iOS 26-style onboarding (dark-forced), then saves `hasSeenWelcome = true`.
2. **Main app** — a 4-tab `TabView` with Rounds, Standings, Statistics, and Settings.
3. **On every launch** — `ContentView` silently downloads the latest Firestore snapshot and applies it to the local SwiftData store.
4. **Review prompt** — after 5+ launches, StoreKit's native review prompt is shown once per app version.

---

## Data Model

### Models (SwiftData `@Model`)

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Driver` | `lastName` (UPPERCASE), `firstName`, `nickname` (3-char), `f1Number`, `countryCode`, `birthdate`, `penaltyPoints`, `car?` | A Formula 1 driver |
| `Car` | `name`, `colorHex`, `countryCode`, `dateOfFirstRace` | A constructor / team |
| `Round` | `roundID`, `date`, `city`, `country`, `countryCode`, `numberOfLaps`, `circuitLength`, `raceDistance`, `isSprintWeekend`, `isCancelled` | A race weekend |
| `RaceDriver` | `position`, `status: RaceStatus`, `driver?`, `car?`, `round?` | One driver's main race result |
| `SprintDriver` | Same shape as `RaceDriver` | One driver's sprint race result |
| `RaceQualifDriver` | `position`, `driver?`, `car?`, `round?` | Qualifying grid position |
| `SprintQualifDriver` | Same shape as `RaceQualifDriver` | Sprint qualifying position |

**`RaceStatus` enum:** `classified` ("F"), `dnf`, `dns`, `dsq` — only `classified` earns points.

### Naming conventions
- `Driver.fullName` → `"LECLERC Charles"`, `shortName` → `"LECLERC C."`
- `Car.colorHex` → hex string e.g. `"#27508C"`, parsed via `Color+Hex.swift`
- Country codes → ISO 3166-1 alpha-2 (e.g. `"MC"`, `"GB"`), rendered as flags via `FlagKit`

---

## Points System

Handled by the stateless `PointsCalculator`:

| Session | Scoring positions |
|---------|-----------------|
| Race | 25-18-15-12-10-8-6-4-2-1 (top 10) |
| Sprint | 8-7-6-5-4-3-2-1 (top 8) |

---

## Navigation Structure

```
ContentView
├── WelcomeView (first launch only, dark-forced)
└── TabView
    ├── Rounds tab        → RoundsListView
    │   └── RoundDetailView
    ├── Standings tab     → StandingsView
    │   ├── DriverStandingsView  → DriverDetailView (stats + info, premium)
    │   └── ConstructorStandingsView → ConstructorDetailView (premium)
    ├── Statistics tab    → StatisticsView (premium only)
    └── Settings tab      → SettingsView
```

---

## Tab Details

### Rounds (`RoundsListView`)
- Displays a **season progress bar** (colour changes: blue → green → orange based on % complete).
- Sections: **Next Race**, **Upcoming** (up to 3, with "X more…" link), **Completed** (up to 3, with "X more…" link).
- Pull-to-refresh syncs from Firestore.
- Empty state includes a **Fetch Data** button for first-time use.

### Standings (`StandingsView`)
- Segmented picker: **Drivers** / **Constructors**.
- Lists standings sorted by points (wins as tiebreaker).
- Tapping a row opens `DriverDetailView` or `ConstructorDetailView` (both require premium for full detail).

#### DriverDetailView (premium)
Two tabs — **Statistics** and **Information**:
- Statistics: best positions (race/qualif/sprint/sprint-Q), race stats grid (wins, podiums, poles, DNF/DNS/DSQ), sprint stats grid, plus 4 interactive line charts (points evolution, race positions, qualifying positions, sprint/sprint-Q positions).
- Information: personal details (name, number, DOB, age, nationality, penalty points) and team.

#### ConstructorDetailView (premium)
Similar layout for constructors — points, wins, best positions, and evolution/position charts for both drivers in the team.

### Statistics (`StatisticsView`) — premium only
Segmented picker with 3 tabs:

| Tab | Content |
|-----|---------|
| **Driver** | Cumulative points line chart (filterable by driver), race positions line chart, per-round points bar chart for featured/favourite driver |
| **Constructor** | Same charts but for constructors |
| **Wins & Poles** | Leaderboards for: podiums, race wins, sprint wins, race poles, sprint poles, DNFs, DNSs, DSQs, sprint DNFs/DNSs/DSQs, super licence penalty points |

All line charts have a **toggle** to switch between showing all season rounds vs. completed rounds only. Charts use country flag labels on the x-axis.

A "favourite" driver/constructor can be pinned to always appear as the featured bar chart subject (persisted via `@AppStorage`).

### Settings (`SettingsView`)
| Section | Options |
|---------|---------|
| Appearance | Color scheme picker (System / Light / Dark), Language (opens iOS Settings) |
| About | View Welcome Screens again |
| Feedback | Rate the App (App Store), Send Feedback (email) |
| Season Pass | Unlock Full Season (paywall), Restore Purchases |
| Support the Developer | Leave a Tip (TipsView with 4 tiers) |
| App Info | Current season, app version + build number |
| Danger zone | Reset All Data (destructive, with confirmation alert) |

---

## Services

### `StandingsService`
Static methods taking `[Round]` as input:
- `driverStandings(from:)` — sorted `[DriverStanding]` (wins as tiebreaker)
- `constructorStandings(from:)` — sorted `[ConstructorStanding]`
- `cumulativeDriverPoints(from:)` — per-round cumulative points for line charts
- `cumulativeConstructorPoints(from:)` — same for constructors
- `driverRacePositions(from:)` — per-round finishing positions
- `constructorRacePositions(from:)` — best position per constructor per round
- `driverPointsPerRound(from:)` — non-cumulative points per round
- `constructorPointsPerRound(from:)` — same for constructors

### `ExportService`
Handles all Firestore communication:
- `uploadToFirestore()` — encodes all SwiftData objects to JSON and uploads (editor target only)
- `downloadFromFirestore()` → `ExportData` — async fetch
- `applyImport(_:into:)` — wipes local SwiftData and reconstructs from payload

Firestore: database `f1seasonstats`, collection `f1seasons`, document ID = season string (e.g. `"2026"`).

### `SeedDataService`
- `load2025Seed(into:)` / `load2026Seed(into:)` — idempotent seed with initial calendar and driver data.

### `PurchaseManager`
`@Observable @MainActor` class, injected into the environment:
- `isPremium: Bool` — persisted in `UserDefaults` as `season2026Unlocked`
- `seasonProduct: Product?` — the season pass (`com.xiaozhuStudio.F1_season_stats.season2026`)
- `tipProducts: [Product]` — 4 tip tiers
- Uses StoreKit 2 (`Transaction.updates` listener, `Transaction.currentEntitlements` on launch)

---

## Monetisation

| Product | ID | Purpose |
|---------|----|---------|
| Season 2026 Pass | `com.xiaozhuStudio.F1_season_stats.season2026` | Unlocks Statistics tab + detailed driver/constructor views |
| Tip Tier 1–4 | `…tip_tier_1` … `…tip_tier_4` | Optional tips for the developer |

Free users can view Rounds and basic Standings (name + points). Premium content is gated behind `PaywallView`.

---

## Persistence & Sync

| Layer | Technology | Notes |
|-------|-----------|-------|
| Local | SwiftData | 7 `@Model` types in shared `ModelContainer` |
| Cloud | Firebase Firestore | Single JSON document per season; downloaded on every launch |
| Preferences | `@AppStorage` / `UserDefaults` | Color scheme, welcome flag, launch count, premium unlock, favourites |

`ModelContainer` is configured in `F1_Season_StatsApp.swift`.

---

## Key Utilities

| File | Purpose |
|------|---------|
| `AppConstants` | Season string (`"2026"`), slot counts for race/qualifying/sprint |
| `ChartRangeToggleIcon` | SF Symbol names + styling for the all-rounds / completed-only toggle |
| `Color+Hex` | `Color(hex:)` initialiser from hex strings |
| `FlagKitManager` | Resolves ISO country codes to `FlagKit` flag images |

---

## File Map

```
Models/          Driver, Car, Round, RaceDriver, SprintDriver, RaceQualifDriver, SprintQualifDriver
Services/        PointsCalculator, StandingsService, SeedDataService, ExportService, PurchaseManager
Utilities/       AppConstants, Color+Hex
Manager/         FlagKitManager
Views/
  welcome/       WelcomeView, iOS26StyleOnBoarding, exampleLogoView
  Rounds/        RoundsListView, RoundDetailView
  Standings/     StandingsView, DriverStandingsView, ConstructorStandingsView,
                 DriverDetailView, ConstructorDetailView, StatBox, CompactStatBox,
                 DriverPointsEvolutionChart, DriverPositionChart,
                 ConstructorPointsEvolutionChart, ConstructorPositionChart
  Statistics/    StatisticsView, StatFullListView, PointsPerRoundChart,
                 StatisticsDriverPointsChart, StatisticsDriverPositionChart,
                 StatisticsConstructorPointsChart, StatisticsConstructorPositionChart,
                 DriverPointsPerRoundListView, ConstructorPointsPerRoundListView,
                 DriverFilterSheet, ConstructorFilterSheet, ChartPopoverAnnotation
  Settings/      SettingsView, SettingRowView, TipsView
  Editor/        RoundEditorView, ResultEntryView, DriverListView, CarListView, DriverPickerView
  (root)         PaywallView, ContentView
```

---

## Firebase Setup

- `GoogleService-Info.plist` present in main target
- `FirebaseApp.configure()` called in `AppDelegate.application(_:didFinishLaunchingWithOptions:)`
- Non-default Firestore database name: `f1seasonstats` (passed explicitly in `ExportService`)

---

## Testing

- **Unit tests:** Testing framework (`import Testing`, `@Test` macro)
- **UI tests:** XCUIAutomation framework
