# Refyl Android Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Refyl for Android with feature parity to Refyl iOS 1.0, built in Kotlin/Compose on Google Maps and reading the same public `stations.json`.

**Architecture:** Single `:app` module.
- **Layers:** MVVM with unidirectional data flow.
  - `domain/` holds pure Kotlin models and rules, ported 1:1 from the iOS `Domain/`.
  - `data/` holds OkHttp + ETag + file cache + repository.
  - `service/` holds location and preferences.
  - `ui/` holds one home screen: a map, a persistent three-height sheet carrying the card, the list and the station detail, and an Info modal sheet.
- **Wiring:** a manual `AppContainer`, with an injectable clock and transport for tests.

**Tech Stack:**
- Kotlin 2.x, Jetpack Compose, Material 3, minSdk 26;
- kotlinx-serialization-json, OkHttp;
- lifecycle-viewmodel-compose, DataStore Preferences;
- play-services-location, maps-compose / play-services-maps, Secrets Gradle Plugin;
- tests: JUnit 4, kotlinx-coroutines-test, Turbine, Compose UI test.

**Spec:** `docs/superpowers/specs/2026-11-01-android-app-design.md`. The behavioural authority is the iOS repo `../FuelFinder_IOS` (code under `Fuel/`, tests under `FuelTests/`).

## Global Constraints

- applicationId and namespace `com.binarybuilders.refyl`, app name "Refyl", versionName `1.0.0`, versionCode 1.
- minSdk 26. compileSdk and targetSdk latest stable. Gradle Kotlin DSL and `gradle/libs.versions.toml`, using the latest stable versions at bootstrap.
- Gradle runs with Android Studio's JBR: prefix every command with `export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"`.
- Data URL: `https://leixien.github.io/FuelFinder_DATA/stations.json` as `BuildConfig.DATA_URL`.
- Maps key: `MAPS_API_KEY` from `local.properties` through the Secrets Gradle Plugin, with a defaults file holding a placeholder. The key is never committed.
- No XML layouts or Fragments. No Hilt, Koin, Retrofit, Room or Firebase. Network code only in `data/remote`, no `Context` in ViewModels, no work on the main thread.
- Every user-facing string goes in `res/values/strings.xml` (IT) and `res/values-en/strings.xml` (EN). Copy wording from the iOS `Fuel/Resources/Localizable.xcstrings`.
- MIMIT attribution in full: "Fonte dati: MIMIT – Osservaprezzi Carburanti" with its link.
- Semantic colours, dynamic colour off:
  - best: `#248A3D` light / `#30D158` dark, on-best white / black;
  - stale: `#C93400` light / `#FF9F0A` dark.
- Domain rule values:
  - radii 5000 / 10000 / 25000 m;
  - map pin limit 150;
  - price freshness 7 days;
  - stale data: more than 72 h after midnight Europe/Rome of `prices_reference_date`;
  - location distance filter 200 m.
- Defaults: fuel `benzina`, mode self, sort price, theme system.
- Commits: English, `type(scope): subject`, no `Co-Authored-By` trailers. One branch and one PR per task group (§Execution), CI green before merge, merge only on the user's OK.
- Licence Apache 2.0. The repo currently has MIT; replace it.

## Review Focus

1. **Unknown values in a payload from a newer pipeline:** an unknown `fuel` (e.g. `"idrogeno"`) or `type` (e.g. `"lago"`) must drop that one price, or map to `altro`, never fail the parse. Test: `StationsDecoderTest.unknownFuelAndTypeDoNotFailParsing` in Task 2.
2. **Large payload on a slow phone:** the real file is 13 MB and about 21,500 stations. Parsing must stream from disk off the main thread and keep the UI responsive. Test: `StationsRepositoryTest.decodesRealSizedPayloadOffMainThread` in Task 4 (a generated 22k-station payload on `StandardTestDispatcher`, asserting `Dispatchers.Default` is used through an injected dispatcher).
3. **Configuration change or process death with a station open:** rotating or folding must keep the selected station, the sheet height and the filters. Test: `HomeViewModelTest.selectionSurvivesRecreation` in Task 6 (selection stored in `SavedStateHandle`).
4. **Permission changes from outside the app:** location revoked in Settings, or "approximate only". On return, approximate counts as normal and revoked shows the "Posizione disattivata" card. Test: `HomeViewModelTest.revokedPermissionOnResumeShowsDenied` in Task 6.
5. **Empty names:** about 10 stations have an empty `name` and some have an empty `brand`, so the label falls back brand → name → "Distributore". Test: `StationDisplayTest.emptyBrandAndNameFallBack` in Task 7.

---

## File Structure

```
app/src/main/kotlin/com/binarybuilders/refyl/
├── RefylApplication.kt, MainActivity.kt, AppContainer.kt
├── domain/model/        FuelType.kt, Station.kt (Station, FuelPrice, StationType), StationsPayload.kt
├── domain/logic/        DistanceCalculator.kt, StationFilter.kt (ServiceMode, StationMatch), StationRanking.kt (StationSort, NearbyResult, MapArea), PriceTable.kt, DataFreshness.kt
├── data/                DataError.kt
├── data/remote/         StationsDecoder.kt, StationsDataSource.kt (HttpTransport), OkHttpTransport.kt
├── data/cache/          FileCache.kt
├── data/repository/     StationsRepository.kt (StationsSnapshot)
├── service/             LocationService.kt (LocationUpdate, LocationAuthorization), Preferences.kt (ThemeMode)
├── ui/theme/            Theme.kt, Color.kt, Type.kt
├── ui/format/           Formatters.kt (price, unit, distance, relative age), StationDisplay.kt
└── ui/home/             HomeViewModel.kt, HomeScreen.kt, StationMap.kt, ResultsSheet.kt, CheapestCard.kt, StationRow.kt, FilterChips.kt, StaleBanner.kt, StationDetail.kt, InfoSheet.kt, Directions.kt
app/src/test/...         one test file per production file in domain/, data/, ui/format, HomeViewModelTest
app/src/test/resources/stations_sample.json   (copied from ../FuelFinder_IOS/FuelTests/Fixtures/)
app/src/androidTest/...  HomeSmokeTest.kt
app/src/debug/kotlin/... TestHooks.kt (fixture + Naples + frozen clock when launched with the extra "uiTesting")
```

## Execution

The tasks group into four PRs, each on its own branch:
- **A:** Tasks 1–4, `feat/foundation`;
- **B:** Tasks 5–6, `feat/state`;
- **C:** Tasks 7–11, `feat/ui`;
- **D:** Task 12, `chore/release`.

After each group: run `./gradlew lint testDebugUnitTest assembleDebug ktlintCheck`, open the PR, wait for CI, and ask the user to merge.

---

### Task 1: Bootstrap the project, docs and CI

**Files:**
- Create:
  - `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, the Gradle wrapper;
  - `app/build.gradle.kts`, `app/src/main/AndroidManifest.xml`, `secrets.defaults.properties`;
  - `RefylApplication.kt`, `MainActivity.kt`, `AppContainer.kt`;
  - `.github/workflows/ci.yml`, `.editorconfig`.
- Modify:
  - `LICENSE` → Apache 2.0;
  - `.gitignore`: Android Studio, `local.properties`, `*.jks`, `*.keystore`, `build/`, `.gradle/`, `.idea/`.
- Move and update:
  - `../fuel-android-CLAUDE.md` → `CLAUDE.md`
  - `../fuel-android-infoProgetto.md` → `infoProgetto.md`
  - `../fuel-android-specificheTecniche.md` → `specificheTecniche.md`
  - `../fuel-android-TODO.md` → `TODO.md`

  Rewrite every mention of "Fuel", MapLibre, `com.leixien.fuel`, navigation-compose and "fuel-data doesn't exist" to match the spec. Mark Fase 0 done in `TODO.md`. Update `README.md` with a description, build steps (Android Studio, then set `MAPS_API_KEY` in `local.properties`) and the MIMIT attribution.
- Test: `app/src/test/kotlin/com/binarybuilders/refyl/BuildConfigTest.kt`

**Interfaces:**
- Produces:
  - `BuildConfig.DATA_URL: String`;
  - `AppContainer(context: Context)`, filled in by later tasks;
  - `MainActivity` hosting `setContent { }` with edge-to-edge (`enableEdgeToEdge()`).

- [ ] **Step 1:** Scaffold the Gradle project, using the latest stable AGP, Kotlin and Compose BOM.
  - Plugins: kotlin-serialization, compose compiler, secrets-gradle-plugin, ktlint (jlleitschuh).
  - Manifest: `ACCESS_COARSE_LOCATION`, `ACCESS_FINE_LOCATION`, `INTERNET`, and the Maps meta-data `com.google.android.geo.API_KEY = ${MAPS_API_KEY}`.
  - `secrets.defaults.properties` holds `MAPS_API_KEY=DEFAULT_API_KEY`.
- [ ] **Step 2:** Write `BuildConfigTest.dataUrlPointsToPublishedFile`: `assertEquals("https://leixien.github.io/FuelFinder_DATA/stations.json", BuildConfig.DATA_URL)`.
- [ ] **Step 3:** Run `./gradlew testDebugUnitTest assembleDebug ktlintCheck lint`. Expected: BUILD SUCCESSFUL, with an empty "Refyl" activity installable.
- [ ] **Step 4:** Add `ci.yml` on `ubuntu-latest`, triggered on pull_request and push to main.
  - Steps: setup-java 21, gradle cache, `./gradlew lint testDebugUnitTest assembleDebug ktlintCheck`.
  - No key needed: the defaults file covers it.
- [ ] **Step 5:** Commit `chore: bootstrap Android project, docs and CI`.

### Task 2: Domain models and decoder

**Files:**
- Create:
  - `domain/model/FuelType.kt`, `domain/model/Station.kt`, `domain/model/StationsPayload.kt`;
  - `data/DataError.kt`, `data/remote/StationsDecoder.kt`.
- Test:
  - `domain/model/FuelTypeTest.kt`, `data/remote/StationsDecoderTest.kt`;
  - `test/resources/stations_sample.json`.

**Interfaces:**
- Produces:
  - `enum class FuelType { BENZINA, GASOLIO, GPL, METANO, GNL, HVO, BENZINA_SPECIAL, GASOLIO_SPECIAL }`, with `@SerialName` values per the contract and `val perKg: Boolean` (METANO, GNL).
  - `enum class StationType { STRADALE, AUTOSTRADALE, ALTRO }`. Unknown values map to ALTRO.
  - `data class FuelPrice(val fuel: FuelType, val price: Double, val isSelf: Boolean, val updatedAt: Instant, val label: String?)`
  - `data class Station(val id: Int, val name: String, val brand: String, val operatorName: String, val type: StationType, val address: String, val city: String, val province: String, val lat: Double, val lon: Double, val prices: List<FuelPrice>)`
  - `data class StationsPayload(val schemaVersion: Int, val generatedAt: Instant, val pricesReferenceDate: String, val source: Source, val stations: List<Station>)` with `Source(name, license, url)` and `companion const val SUPPORTED_SCHEMA_VERSION = 1`.
  - `sealed interface DataError { Network; HttpStatus(code: Int); Decoding; UnsupportedSchema(version: Int) }`.
  - `object StationsDecoder { fun decode(input: InputStream): StationsPayload }`. It throws `DataErrorException(error: DataError)`, streams with `Json.decodeFromStream`, uses `ignoreUnknownKeys = true`, and parses prices leniently.

- [ ] **Step 1:** Port the `StationsDecoderTests.swift` cases to `StationsDecoderTest`:
  - the fixture decodes with the expected station count and the first station's fields;
  - `schema_version` 2 → `UnsupportedSchema(2)`, including when the rest of the body doesn't decode;
  - garbage → `Decoding`;
  - `unknownFuelAndTypeDoNotFailParsing`: one price with `"fuel":"idrogeno"` is dropped and the others are kept; `"type":"lago"` → `ALTRO`.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests '*StationsDecoderTest'`. Expected: FAIL, unresolved references.
- [ ] **Step 3:** Implement the models with serial DTOs or custom serializers:
  - the price list decodes element by element, dropping elements whose `fuel` isn't known;
  - `updated_at` and `generated_at` are ISO-8601 instants;
  - on a decode failure, peek `schema_version` before mapping to `Decoding`, as iOS does.
- [ ] **Step 4:** Run the same command. Expected: PASS.
- [ ] **Step 5:** Commit `feat(domain): models and stations.json decoder`.

### Task 3: Domain rules

**Files:**
- Create: `domain/logic/DistanceCalculator.kt`, `StationFilter.kt`, `StationRanking.kt`, `PriceTable.kt`, `DataFreshness.kt`.
- Test: one `*Test.kt` per file, porting `../FuelFinder_IOS/FuelTests/Domain/{DistanceCalculator,StationFilter,StationRanking,PriceTable,DataFreshness}Tests.swift` case by case. Keep the same inputs and expected values.

**Interfaces:**
- Consumes: Task 2 models.
- Produces:
  - `object DistanceCalculator { fun distance(fromLat: Double, fromLon: Double, toLat: Double, toLon: Double): Double }`: Haversine, metres, earth radius 6_371_000.
  - `enum class ServiceMode { SELF_SERVICE, FULL_SERVICE }`.
  - `data class StationMatch(val station: Station, val price: FuelPrice, val distance: Double)`.
  - `object StationFilter`:
    - `fun price(fuel: FuelType, mode: ServiceMode, station: Station, now: Instant): FuelPrice?`, using the tiers fresh∧inMode → fresh → inMode → any, cheapest in the first non-empty tier;
    - `fun nearby(stations, fuel, mode, originLat, originLon, radius: Double, now): List<StationMatch>`, nearest first.
  - `enum class StationSort { PRICE, DISTANCE }`.
  - `data class NearbyResult(val radius: Double, val matches: List<StationMatch>, val cheapest: StationMatch?)`.
  - `data class MapArea(val centerLat: Double, val centerLon: Double, val latDelta: Double, val lonDelta: Double) { fun contains(lat: Double, lon: Double): Boolean }`.
  - `object StationRanking`:
    - `RADII = listOf(5000.0, 10000.0, 25000.0)`, `MAP_LIMIT = 150`, `MAX_PRICE_AGE = 7.days`;
    - `isFresh(price, now)` and `qualifiesForCheapest(match, mode, now)`;
    - `search(stations, fuel, mode, originLat, originLon, sort, now): NearbyResult`, using rules [qualifies, fresh] × radii with the first hit winning, the cheapest by (price, distance), and matches sorted by (price, distance) or by distance;
    - `inArea(stations, fuel, mode, area, originLat: Double?, originLon: Double?, now, limit = MAP_LIMIT): List<StationMatch>`, ranked by (fresh ? 0 : 1, price, distance), distance from the origin or else the area centre.
  - `data class PriceTableRow(val fuel: FuelType, val label: String?, val selfPrice: Double?, val servedPrice: Double?, val updatedAt: Instant)`, with `object PriceTable { fun rows(station: Station): List<PriceTableRow> }`: one row per (fuel, label), an empty label counting as null, in `FuelType` order and then by label.
  - `object DataFreshness { fun staleDays(pricesReferenceDate: String, now: Instant): Int? }`: null while `now - midnight(Europe/Rome, date) ≤ 72 h`, otherwise whole days.

- [ ] **Step 1:** Write the five ported test files, every iOS case with the same numbers. That includes:
  - ties going to the nearer station;
  - stale prices excluded from the cheapest;
  - widening 5 → 10 → 25 km;
  - fallback to the other mode after 25 km;
  - the `inArea` cap and ordering;
  - the Rome-midnight edge across DST.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests 'com.binarybuilders.refyl.domain.*'`. Expected: FAIL.
- [ ] **Step 3:** Implement the five files to the signatures above (the bodies mirror `Fuel/Domain/Logic/*.swift`).
- [ ] **Step 4:** Run the same command. Expected: PASS, with domain coverage ≥ 80% (`./gradlew koverReport`, or the JaCoCo report if Kover isn't wanted; one of them, note the choice in `specificheTecniche.md`).
- [ ] **Step 5:** Commit `feat(domain): ranking, filters, price table and freshness ported from iOS`.

### Task 4: Data layer

**Files:**
- Create: `data/remote/StationsDataSource.kt`, `data/remote/OkHttpTransport.kt`, `data/cache/FileCache.kt`, `data/repository/StationsRepository.kt`.
- Test: `StationsDataSourceTest.kt`, `FileCacheTest.kt`, `StationsRepositoryTest.kt`, porting the iOS `FuelTests/Data/*`.

**Interfaces:**
- Consumes: `StationsDecoder`, `DataError`.
- Produces:
  - `fun interface HttpTransport { suspend fun get(url: String, ifNoneMatch: String?): HttpResponse }`, with `data class HttpResponse(val code: Int, val etag: String?, val body: ByteArray?)`. It throws `IOException` on network failure.
  - `class StationsDataSource(url: String, transport: HttpTransport)` with `suspend fun fetch(etag: String?): FetchResult`:
    - `sealed interface FetchResult { NotModified; Modified(bytes: ByteArray, etag: String?) }`;
    - IOException → `Network`; non-2xx except 304 → `HttpStatus`.
  - `class OkHttpTransport(client: OkHttpClient)`: bypasses OkHttp's cache so 304s reach us.
  - `class FileCache(dir: File)`:
    - `fun load(): Entry?` and `fun save(entry: Entry)`, with `data class Entry(val file: File, val etag: String?, val downloadedAt: Instant)`;
    - the payload is written first, atomically (tmp + rename), then `metadata.json`;
    - `dir = context.noBackupFilesDir/stations`.
  - `class StationsRepository(dataSource, cache, decodeDispatcher: CoroutineDispatcher = Dispatchers.Default)`:
    - `suspend fun cached(): StationsSnapshot?` and `suspend fun refresh(): StationsSnapshot`;
    - single-flight: a `Mutex`, so concurrent callers share one download;
    - `data class StationsSnapshot(val payload: StationsPayload, val downloadedAt: Instant)`.

- [ ] **Step 1:** Port the iOS tests, using a fake `HttpTransport`:
  - 304 with a cache returns the cached snapshot; 304 without a cache → `HttpStatus(304)`;
  - the ETag is sent only when a cache exists;
  - a failed cache write still returns fresh data;
  - corrupt cache → `cached()` is null.

  Add:
  - `concurrentRefreshesDownloadOnce`;
  - `decodesRealSizedPayloadOffMainThread`: a generated 22,000-station payload decodes on the injected dispatcher, not the caller's.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests 'com.binarybuilders.refyl.data.*'`. Expected: FAIL.
- [ ] **Step 3:** Implement. Decode from `FileInputStream` on `decodeDispatcher`; file I/O on `Dispatchers.IO`.
- [ ] **Step 4:** Run the same command. Expected: PASS.
- [ ] **Step 5:** Commit `feat(data): ETag download, file cache and single-flight repository`. Then open PR A.

### Task 5: Location and preferences services

**Files:**
- Create: `service/LocationService.kt`, `service/Preferences.kt`.
- Test: `service/PreferencesTest.kt`, plus `service/LocationAuthorizationTest.kt` for the mapping function.

**Interfaces:**
- Produces:
  - `enum class LocationAuthorization { NOT_DETERMINED, GRANTED, DENIED }`. Approximate-only maps to GRANTED.
  - `sealed interface LocationUpdate { data class Authorization(val value: LocationAuthorization); data class Position(val lat: Double, val lon: Double) }`.
  - `interface LocationService`:
    - `val authorization: LocationAuthorization`;
    - `fun updates(): Flow<LocationUpdate>`: Fused, balanced power, 200 m minimum displacement;
    - `fun refreshAuthorization()`: called on resume.
  - `class FusedLocationService(context)` implements it.
  - `class Preferences(dataStore)`:
    - `val fuel: Flow<FuelType>`, `val mode: Flow<ServiceMode>`, `val sort: Flow<StationSort>`, `val theme: Flow<ThemeMode>`, each with a matching `suspend fun set…`;
    - `enum class ThemeMode { SYSTEM, LIGHT, DARK }`;
    - unknown stored values fall back to the defaults.

- [ ] **Step 1:** Write `PreferencesTest`:
  - defaults are benzina, self, price and system;
  - round-trip of each value;
  - a garbage stored string returns the default.

  Write `LocationAuthorizationTest.mapsPermissionStates`:
  - fine or coarse granted → GRANTED;
  - denied after asking → DENIED;
  - never asked → NOT_DETERMINED.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests '*Preferences*' --tests '*LocationAuthorization*'`. Expected: FAIL.
- [ ] **Step 3:** Implement. Keep the permission mapping in a pure function `fun authorizationOf(fineGranted: Boolean, coarseGranted: Boolean, askedBefore: Boolean): LocationAuthorization`, so it is testable.
- [ ] **Step 4:** Run the same command. Expected: PASS.
- [ ] **Step 5:** Commit `feat(service): location and persisted preferences`.

### Task 6: HomeViewModel

**Files:**
- Create: `ui/home/HomeViewModel.kt`, plus the wiring in `AppContainer` (repository, location, preferences, `clock: () -> Instant`).
- Test: `ui/home/HomeViewModelTest.kt`, porting `../FuelFinder_IOS/FuelTests/Features/HomeViewModelTests.swift` with fakes for the transport, location and clock.

**Interfaces:**
- Consumes: Tasks 3–5.
- Produces `class HomeViewModel(repository, location, preferences, clock, savedState: SavedStateHandle) : ViewModel`, exposing:
  - `val state: StateFlow<HomeUiState>`;
  - `data class HomeUiState(val content: Content, val staleBanner: StaleBanner?, val fuel: FuelType, val mode: ServiceMode, val sort: StationSort, val theme: ThemeMode, val origin: LatLon?, val mapMatches: List<StationMatch>, val selected: StationMatch?, val payload: StationsPayload?)`;
  - `sealed interface Content { Loading; Locating; LocationDenied; Results(NearbyResult); Failed(DataError) }`;
  - `data class StaleBanner(val days: Int, val offline: Boolean)`.

  Functions:
  - `fun start()` (cached, then refresh, then follow location);
  - `fun onResume()` (refresh plus `refreshAuthorization`);
  - `fun retry()`;
  - `fun setFuel/setMode/setSort/setTheme(...)`;
  - `fun onVisibleArea(area: MapArea)`;
  - `fun select(stationId: Int?)`.

  The selected id lives in `savedState`. A fuel or mode change clears a selection that no longer matches; moving or new data never does.

- [ ] **Step 1:** Port every iOS ViewModel case:
  - loading → results;
  - failed without cache, and cache kept on a failed refresh;
  - banner days, plus offline only for `Network`;
  - denied location;
  - selection kept on data refresh and cleared on fuel change;
  - retry ignored while refreshing;
  - map matches follow the area.

  Add:
  - `selectionSurvivesRecreation`: a new ViewModel on the same `SavedStateHandle` restores `selected` once the data loads;
  - `revokedPermissionOnResumeShowsDenied`.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests '*HomeViewModelTest'`. Expected: FAIL.
- [ ] **Step 3:** Implement, mirroring `Fuel/Features/Home/HomeViewModel.swift`. Run the heavy ranking calls on `Dispatchers.Default`.
- [ ] **Step 4:** Run the same command. Expected: PASS.
- [ ] **Step 5:** Commit `feat(home): view model with search, banners and saved selection`. Then open PR B.

### Task 7: Theme, formatters and strings

**Files:**
- Create: `ui/theme/{Theme,Color,Type}.kt`, `ui/format/Formatters.kt`, `ui/format/StationDisplay.kt`, `res/values/strings.xml`, `res/values-en/strings.xml`.
- Test: `ui/format/FormattersTest.kt` (port `FormattersTests.swift`), `ui/format/StationDisplayTest.kt`.

**Interfaces:**
- Produces:
  - `RefylTheme(themeMode: ThemeMode, content)`: Material 3, no dynamic colour, plus `RefylColors.best/onBest/stale` via a `CompositionLocal`.
  - Formatters:
    - `fun formatPrice(value: Double, locale: Locale): String`: 3 decimals with the locale separator, rendered with tabular figures in the UI;
    - `fun unitLabel(fuel: FuelType): Int`: the string res for €/l or €/kg;
    - `fun formatDistance(metres: Double, locale): String`: like iOS, metres under 1 km, else one decimal in km;
    - `fun relativeAge(updatedAt: Instant, now: Instant, res: Resources): String`: "aggiornato oggi / ieri / N h fa / N gg fa", mirroring iOS.
  - `fun displayName(station: Station, fallback: String): String`: brand, else name, else the fallback.
  - `fun fuelName(fuel)` and `fuelSubtitle(fuel)` string res: the same names and subtitles as the iOS fuel menu.

- [ ] **Step 1:** Port the formatter tests. Add `StationDisplayTest.emptyBrandAndNameFallBack` (brand "" with name "" → "Distributore"; brand "" with name "ESSO 12" → "ESSO 12").
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests 'com.binarybuilders.refyl.ui.format.*'`. Expected: FAIL.
- [ ] **Step 3:** Implement, and fill both `strings.xml` files with every string the UI needs, copied from the iOS catalog. That includes the `DataError` messages, verbatim from `Fuel/Data/DataError.swift`.
- [ ] **Step 4:** Run the same command, plus `./gradlew lint` with `MissingTranslation` as an error. Expected: PASS, with no missing translations.
- [ ] **Step 5:** Commit `feat(ui): theme, formatters and IT/EN strings`.

### Task 8: Home screen: map, chips, sheet, card, list

**Files:**
- Create:
  - `ui/home/{HomeScreen,StationMap,ResultsSheet,CheapestCard,StationRow,FilterChips,StaleBanner}.kt`;
  - `ui/home/HomeStates.kt`: the loading, locating, denied, failed and empty-radius views.
- Modify: `MainActivity.kt` hosts `HomeScreen`, the permission request on first launch, and `onResume → viewModel.onResume()`.

**Interfaces:**
- Consumes: `HomeViewModel.state` and its functions, the formatters and `RefylColors`.
- Produces:
  - `@Composable fun HomeScreen(state: HomeUiState, actions: HomeActions, modifier: Modifier = Modifier)`, stateless, with `data class HomeActions(...)` holding the lambdas for every ViewModel function plus `openDirections` and `openSettings`;
  - a `@Preview` for each state using the fixture.

- [ ] **Step 1:** **The map.**
  - `GoogleMap` from maps-compose.
  - Report `MapArea` from `cameraPositionState.projection.visibleRegion` when the camera is idle.
  - Pins for `mapMatches`: the price to 3 decimals on a pill; the cheapest in `best`, larger and with the highest z-index; the selected one highlighted.
  - My-location layer when granted; Italy centred when denied (41.9, 12.5, zoom 5.5).
  - A pin tap → `select(id)`.
  - TalkBack label: "{name}, {price} euro al {unità}, a {distanza}, {self/servito}".
- [ ] **Step 2:** **The chips over the map, top-start.**
  - A fuel dropdown with names and subtitles.
  - A self/served toggle chip.
  - A sort toggle: "Più economici" / "Più vicini".
- [ ] **Step 3:** **The controls over the map, top-end, stacked.**
  - A theme toggle: a moon when the effective theme is light, a sun when dark; a tap forces the opposite.
  - An Info button.
  - The my-location button.
- [ ] **Step 4:** **The sheet.**
  - Persistent, with three anchors: low (the card), medium (the card and the start of the list) and high (the full list).
  - The map stays interactive.
  - Inside the sheet: the stale banner, then the card or a state view, then the list.

  The card shows:
  - the large price and its unit;
  - the display name and `label` when the fuel is a special;
  - short address · distance · mode badge (when the price is in the other mode) · relative age (orange when older than 7 days);
  - "Indicazioni", a filled button in `best`;
  - "entro N km".

  List rows show name, distance · age and price, with the best in green; a tap → `select`.
- [ ] **Step 5:** Run `./gradlew assembleDebug lint`, install on the Pixel with the real key in `local.properties`, and check the starting state:
  - light and dark;
  - font scale 200% (the card price doesn't clip);
  - TalkBack on the card and pins.
- [ ] **Step 6:** Commit `feat(home): map with price pins, filter chips and results sheet`.

### Task 9: Station detail in the sheet and directions

**Files:**
- Create: `ui/home/StationDetail.kt`, `ui/home/Directions.kt`.
- Modify: `ResultsSheet.kt`. When `state.selected != null`, it shows `StationDetail`, animates to the high anchor, and calls `BackHandler { actions.select(null) }`.

**Interfaces:**
- Consumes: `PriceTable.rows`, `StationMatch`, the formatters.
- Produces:
  - `@Composable fun StationDetail(match: StationMatch, fuel: FuelType, now: Instant, onClose: () -> Unit, onDirections: () -> Unit)`;
  - `fun directionsIntent(lat: Double, lon: Double, pm: PackageManager): Intent`: `google.navigation:q=lat,lon` when Google Maps resolves, otherwise `geo:lat,lon?q=lat,lon`.

- [ ] **Step 1:** Write `DirectionsTest` (Robolectric isn't allowed, so make the choice a pure function):
  - `fun directionsUri(lat, lon, hasGoogleMaps: Boolean): Uri`;
  - assert both URIs, with a dot decimal separator whatever the locale.
- [ ] **Step 2:** Run `./gradlew testDebugUnitTest --tests '*DirectionsTest'`. Expected: FAIL, then implement it. Expected: PASS.
- [ ] **Step 3:** Build the detail UI:
  - the name;
  - `address, city (province)`;
  - distance and type;
  - "Indicazioni";
  - the price table, with self and served columns, "—" for empty cells, and the chosen fuel's row highlighted;
  - the age;
  - ✕.

  Predictive back and ✕ both return to the list.
- [ ] **Step 4:** Check on the Pixel: open from a pin and from a row, back gesture, rotate with the detail open (the selection persists), and Directions opens Google Maps navigation.
- [ ] **Step 5:** Commit `feat(detail): station detail inside the sheet with directions`.

### Task 10: Info sheet

**Files:**
- Create: `ui/home/InfoSheet.kt`.

**Interfaces:**
- Consumes: `state.payload` (source and dates), `BuildConfig.VERSION_NAME/VERSION_CODE`, `setTheme`.

- [ ] **Step 1:** Build a `ModalBottomSheet` containing:
  - "Fonte dati: MIMIT – Osservaprezzi Carburanti" (verbatim, linked to `payload.source.url`) and "Licenza IODL 2.0";
  - "Codice: Apache 2.0" and a repo link (`https://github.com/Leixien/FuelFinder_ANDROID`);
  - version (build);
  - "Dati del {prices_reference_date}, generati {generated_at}";
  - "Usa impostazioni di sistema", which sets `ThemeMode.SYSTEM`;
  - "Mappa: Google Maps", linked to Google's terms (`https://www.google.com/help/terms_maps/`) and privacy policy (`https://policies.google.com/privacy`);
  - the privacy policy link `https://leixien.github.io/refyl-site/privacy/` (or `/en/privacy/` in English).
- [ ] **Step 2:** Check on the Pixel that every link opens in the browser and that the theme reset works.
- [ ] **Step 3:** Commit `feat(info): attribution, licences, versions and theme reset`.

### Task 11: App icon and UI smoke test

**Files:**
- Create:
  - `res/mipmap-anydpi-v26/ic_launcher.xml` (adaptive, with the `monochrome` layer);
  - the vector drawables `ic_launcher_background.xml` and `ic_launcher_foreground.xml`, converted from `../FuelFinder_IOS/docs/brand/icon/layers/{1-road,2-r,3-pump}-light.svg`;
  - `ic_launcher_monochrome.xml`;
  - `app/src/debug/kotlin/.../TestHooks.kt`;
  - `app/src/androidTest/kotlin/.../HomeSmokeTest.kt`.
- Modify: `AppContainer`. When launched with the intent extra `uiTesting=true` (debug builds only), use the fixture transport, a fake location at Naples (40.8518, 14.2681) and the clock frozen at the iOS test date (copy it from `Fuel/App/AppEnvironment.swift`).

- [ ] **Step 1:** Write `HomeSmokeTest.launchShowsCardThenDetailAndBack`:
  - launch with `uiTesting`;
  - assert the card shows the fixture's cheapest price (same as the iOS UI test);
  - tap the first row;
  - assert the detail's "Indicazioni" and the price table exist;
  - press back;
  - assert the list is visible again.
- [ ] **Step 2:** Run `./gradlew connectedDebugAndroidTest` on the emulator. Expected: FAIL until the hook exists, then implement the hook. Expected: PASS.
- [ ] **Step 3:** Convert the icon layers to VectorDrawables, keeping the foreground inside the 66 dp safe zone. Check the launcher on the Pixel with the system and themed icons on and off.
- [ ] **Step 4:** Commit `feat(app): adaptive and themed icon, UI smoke test`. Then open PR C.

### Task 12: Release preparation

**Files:**
- Create:
  - `docs/release/play-listing.md`: IT/EN title (≤30), short description (≤80) and full description, adapted from `../FuelFinder_IOS/docs/release/app-store-listing.md`;
  - `docs/release/data-safety.md`: the answers, citing Google's Maps SDK disclosure page;
  - `docs/release/play-submission.md`: a checklist with the user's steps.
- Modify:
  - `app/build.gradle.kts`: a release `signingConfig` read from `keystore.properties` (gitignored), with R8 and shrinkResources on;
  - `../refyl-site/privacy/index.html` and `../refyl-site/en/privacy/index.html`: add an Android paragraph on Google Maps. This is a separate PR in that repo.

- [ ] **Step 1:** Run `./gradlew bundleRelease`. Expected: an `.aab` signed with the local upload key, which the user creates once with `keytool` on their machine. Store the keystore outside the repo.
- [ ] **Step 2:** Write the three docs. `play-submission.md` lists, in order:
  - **user:** create the Play Console account;
  - **user:** create the Google Cloud Maps key, restricted to `com.binarybuilders.refyl` plus the debug and upload SHA-1s, then add the Play-signing SHA-1 after the first upload;
  - **user:** create the app and opt in to Play App Signing;
  - **user:** upload the AAB to internal testing;
  - **user:** fill in the store listing (screenshots taken from the Pixel), the Data safety form, the content rating and the target audience;
  - **user:** run the closed test (12 testers for 14 days for a personal account);
  - **user:** go to production.
- [ ] **Step 3:** Commit `chore(release): signing config, Play listing and data safety notes`. Open PR D. Open the refyl-site privacy PR separately.

---

## Self-review notes

- **Spec coverage:**
  - §1 goals: Tasks 3–10.
  - Anti-goals: the Task 3 rules port, plus Task 8 (no extra highlights).
  - §2 constraints: Global Constraints and Task 1.
  - §4.1: Task 6.
  - §4.2: Tasks 3, 6 and 8.
  - §4.3: Tasks 5 and 8.
  - §4.4: Task 8.
  - §4.5: Task 9.
  - §4.6: Tasks 5, 8 and 10.
  - §4.7: Task 10.
  - §4.8: Tasks 7 and 8.
  - §5: Tasks 1–11.
  - §6: Task 12.
  - §7 (open release decisions): stay with the user, in `play-submission.md`.
- **Types:** `StationMatch`, `NearbyResult`, `MapArea`, `HomeUiState` and `DataError` are named identically across Tasks 3–11.
