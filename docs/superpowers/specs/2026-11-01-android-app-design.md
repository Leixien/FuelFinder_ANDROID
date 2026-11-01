# Refyl Android: design

Date: 2026-11-01. Status: approved by the user on 2026-11-01 ("vai in native": go ahead, implemented natively in this session). Map: Google Maps, the user's earlier choice; the Data safety disclosure in §2 is the consequence, to be confirmed with the user at release.

## 1. Intent

Refyl for Android is the same product as Refyl for iPhone, which was submitted to App Review on 2026-11-01: the same screens, the same behaviour, the same data, in Android's own visual language. The user asked for exactly this ("uguale all'iPhone, stesse schermate ma stile Android") and for nothing more or less in the first version.

The user opens the app to know **immediately where the cheapest fuel near them is**. There are two moments of use, and they are the same as on iOS:
- in the car, phone on the holder: one glance, zero or one tap, a big number and a "Directions" button;
- before leaving, calmly: compare stations, open a station's details.

**The behavioural reference is the iOS app, not this document.** Wherever this document is silent, do what `FuelFinder_IOS` does. Read these in that repo:
- `docs/superpowers/specs/2026-10-29-fase2-ui-design.md`: the full behaviour spec, including the user's decisions after testing on a device;
- `Fuel/`: about 2,000 lines of Swift;
- `FuelTests/`: unit tests to port.

Where the two disagree, the iOS code wins over the iOS spec: the spec marks some sections "Superato".

### Goals
- **Feature parity with iOS 1.0.**
  - Map-first home with a persistent bottom sheet holding the "cheapest" card and the list.
  - Filter chips: fuel, self/attended, sort order.
  - Station detail screen.
  - Info screen.
  - Quick light/dark toggle.
  - Stale-data and offline banners.
  - Empty and error states.
  - Italian and English.
  - Dark theme, large font scale, TalkBack.
- The lowest price within the radius is readable within one second of opening the app, with no taps.
- Native Android look and feel (Material 3), polished in light and dark.
- **Published on Google Play.** That includes the steps only the user can do: creating the Play Console account, the closed test and the store listing.

### Non-goals (v1)
These are all absent from iOS 1.0 too:
- town/address search;
- "search this area";
- favourites;
- price history;
- widgets;
- Android Auto;
- Wear OS;
- a no-Google-services build for F-Droid;
- tablet-specific layouts (a phone layout that stretches is fine);
- community prices (iOS issue #10).

### Anti-goals (failure even if it technically works)
- A crowded screen: more than one highlighted number, decorative cards, ads.
- Having to tap anything to see the best price.
- Low-contrast or small text: it is read from a car holder.
- Diverging from iOS behaviour (a different "cheapest" rule, different radius logic, different stale rules). A user with two phones must get the same answer.
- Any account, analytics, tracking or ads SDK.

## 2. Constraints

- **Platform.** Kotlin 2.x, Jetpack Compose, Material 3, minSdk 26, target/compile SDK latest stable. No XML layouts or Fragments. Coroutines and Flow. Parsing and distance maths on `Dispatchers.Default`, I/O on `Dispatchers.IO`.
- **Build.** Gradle Kotlin DSL with a version catalog, single `:app` module, manual `AppContainer` (no Hilt/Koin).
- **Data contract.** `FuelFinder_DATA/CONTRACT.md`, schema v1. Production URL `https://leixien.github.io/FuelFinder_DATA/stations.json`. Parsing rules:
  - `ignoreUnknownKeys`;
  - unknown `fuel` values are dropped per price, never failing the parse;
  - reject `schema_version` greater than 1 with an explicit error;
  - ETag / `If-None-Match`;
  - last payload cached on disk.
- **Map: Google Maps (user's decision).** Use Maps SDK for Android through `maps-compose`. Consequences:
  - The API key lives in `local.properties` and reaches the manifest through the Secrets Gradle Plugin. It is never committed, and the key is restricted to the app's package and SHA-1 fingerprints.
  - The repo must build without a key: CI and contributors use a placeholder and get a blank map.
  - **The Maps SDK collects some technical data itself:** device metadata, SDK version, crash data, IP address, a pseudonymous SDK identifier and, depending on configuration, map interaction events (Google's "Prepare for Google Play's data disclosure requirements" page). The Play "Data safety" form and the privacy policy must disclose this. The iOS promise "no data collected" therefore does not hold word-for-word on Android; see §6.
- **Location:** `FusedLocationProviderClient` from Play services. Google Maps already requires Play services, so this adds no new dependency class.
- **Allowed dependencies:**
  - kotlinx-serialization-json;
  - OkHttp;
  - lifecycle-viewmodel-compose;
  - datastore-preferences;
  - play-services-location;
  - maps-compose with play-services-maps;
  - the Secrets Gradle Plugin;
  - for tests: JUnit, kotlinx-coroutines-test, Turbine and Compose UI test.

  Anything else needs a written reason in `specificheTecniche.md`.
- **Strings.** Every string goes in `res/values/strings.xml` (Italian, the base) and `res/values-en/strings.xml`. The MIMIT attribution "Fonte dati: MIMIT – Osservaprezzi Carburanti" with its link is mandatory and must never be shortened (IODL 2.0).
- **Identity.** Name **Refyl**. applicationId `com.binarybuilders.refyl`, the same as the iOS bundle id. Version 1.0.0 (versionCode 1). Licence Apache 2.0.
- **No secrets in the repo:** no keystore, no `google-services.json`, no API key. Google Play App Signing holds the app key, and the upload key stays only on the maintainer's machine.
- **Toolchain on this Mac:**
  - Android Studio is installed and `~/Library/Android/sdk` is present;
  - the only system JDK is 25, so Gradle must use Android Studio's bundled JBR (JDK 21);
  - test device: the user's Pixel 9 Pro XL, plus an emulator.

## 3. Decisions already taken

1. **Same as iPhone, Android style (user).** Every behaviour copies iOS. Every visual is Material 3:
   - top app bar / floating controls over the map;
   - `FilterChip`s and an `ExposedDropdownMenu`-style fuel menu;
   - a Material bottom sheet with three heights;
   - Material icons;
   - edge-to-edge with system bar insets, predictive back.
2. **Google Maps (user)**, with the consequences in §2.
3. **Brand colours stay fixed (my call).** Dynamic colour (Material You) is off, so the semantic colours match iOS exactly:
   - cheapest/best green `#248A3D` light, `#30D158` dark, with white/black text on it;
   - stale orange `#C93400` light, `#FF9F0A` dark.

   Other Material roles derive from the icon palette, and the theme lives in one place (`ui/theme`).
4. **Icon (my call).** An adaptive icon built from the iOS layers in `FuelFinder_IOS/docs/brand/icon/layers/`: the road as background, the R and the pump as foreground. A monochrome layer for Android 13+ themed icons.
5. **Directions (my call).** "Directions" opens turn-by-turn navigation in Google Maps (`google.navigation:q=lat,lon`). If Google Maps isn't installed, it falls back to a `geo:` intent so any maps app can handle it.
6. **The repo docs are brought into the repo and updated.**
   - Today `CLAUDE.md`, `infoProgetto.md`, `specificheTecniche.md` and `TODO.md` live outside it, as `../fuel-android-*.md`, and still say "Fuel", MapLibre, `com.leixien.fuel` and "fuel-data doesn't exist yet".
   - They move into the repo root and are aligned with this design, the way the iOS repo is organised.

## 4. Behaviour (summary; the iOS code is the authority)

### 4.1 Data
- On start, show the cache immediately, then run one background refresh. Like iOS, refresh again whenever the app returns to the foreground, with only one refresh in flight at a time.
- First run without cache: a loading state in the card. On failure, an error state with a typed, localised message and "Retry".
- Cache present but refresh failed: use the cache, with no blocking error.
- **Stale banner:** shown when `prices_reference_date` is more than 48 h old, counted from the end of that Italian day (iOS `DataFreshness`). Text: "Dati di N giorni fa", plus "offline" if the last refresh failed for network reasons. Never use `generated_at` or the download time for this.

### 4.2 Location, radius, map
- Ask for location ("while in use", precise or approximate; approximate counts as normal) on first launch, because the map is the first screen. Show a rationale only where Android requires one.
- Recompute when the user moves about 200 m or more, not on every fix.
- **Search radius:** 5 km, widening automatically to 10 and then 25 km when there's nothing in the chosen mode. The card always states the radius it used.
- **Mode fallback:** if nothing within 25 km has a recent price in the chosen mode, search again from 5 km accepting the other mode, and badge the card. This is the user's iOS decision, for LPG/LNG, where self-service barely exists.
- **Map pins:**
  - show the stations in the **visible map area**, at most 150, recent prices first and then the cheapest (the user's iOS decision after device testing);
  - card and list stay on the search around the user;
  - each pin shows the selected fuel's price to 3 decimals;
  - the cheapest is green, bigger and always on top;
  - no clustering.
- A standard "my location" control.
- **Location denied:** the map centres on Italy, the card says "Posizione disattivata" with "Apri Impostazioni", and the chips stay usable.

### 4.3 Chips
- **Fuel:** every `FuelType` with readable localised names and short subtitles, as in the iOS menu. Default: petrol.
- **Self/attended:** a toggle, default self. If a station lacks the chosen mode it shows the other mode with a badge, in the list and on the map only.
- **Sort:** "Più economici" (default) or "Più vicini". It changes the list order only; the card always shows the cheapest.
- All three choices persist in DataStore.

### 4.4 Sheet and card
- **The sheet:** always present and never dismissable; the map stays interactive behind it. It has three heights: low (card only, the car view), medium (card plus the start of the list) and high (full list).
- **Card:**
  - price shown big, with `€/l`, or `€/kg` for methane and LNG;
  - name is the `brand`, otherwise the `name`, otherwise "Distributore";
  - then the short address, the distance, self/attended and a relative age ("aggiornato 3 h fa");
  - a "Indicazioni" button.
- **Card rules:**
  - Tie on price: the nearest wins.
  - A price older than 7 days cannot win the card or the green pin. It stays in the list and on the map with its age in orange.
  - Radius widening counts only prices within 7 days.
- **List rows:** name, distance · age, and the price on the right in tabular figures; the best one is in green.
- Tapping a pin or a row opens the detail **inside the sheet**, as on iOS (my call, from "same as iPhone"). The sheet content becomes the detail and rises to the high position; the map stays live behind it with the selected pin highlighted. System back (predictive back through `BackHandler`) or ✕ returns to the list. Info is a second modal sheet on top. There are no separate navigation destinations, so navigation-compose is not needed.

### 4.5 Detail
- Shows name, the full address `address, city (province)`, distance, type (road/motorway), "Indicazioni" and the age of the data.
- Price table: fuel × self/attended, one row per (fuel, `label`) pair, in `FuelType` order, empty cells "—". The chosen fuel is highlighted.
- When the chosen fuel is a special, the card, rows and detail show the winning product's `label`.

### 4.6 Theme
- Three states, saved: system (the default), light and dark.
- The quick toggle on the map shows a moon in light mode and a sun in dark mode; tapping it forces the opposite.
- Info has "Usa impostazioni di sistema".

### 4.7 Info
- "Fonte dati: MIMIT – Osservaprezzi Carburanti" with its link, in full, and IODL 2.0.
- Apache 2.0, the repo link and the version and build.
- The data dates (`generated_at` / `prices_reference_date`) and the system-theme option.
- New on Android: a line saying the map is provided by Google Maps, with links to Google's terms and privacy policy. Google's terms require this.

### 4.8 Accessibility
- Font scale up to 200% without clipping the card price.
- AA contrast, using the same light/dark green rule as iOS.
- TalkBack labels on pins and rows, e.g. "Q8, 1,729 euro al litro, a 1,2 chilometri, self".
- Locale-formatted prices (a decimal comma in Italian) in tabular figures.

## 5. Tests and CI

- **Unit tests**, ported from `FuelTests/` and kept to their intent:
  - parsing, including unknown fuel and an unsupported `schema_version`;
  - ETag cache;
  - distance;
  - filter and ranking (tie means nearest, the 7-day rule, radius widening, mode fallback, sorting);
  - freshness at 48 h;
  - ViewModel states (denied location, banners).
- **Fixture:** a reduced real `stations_sample.json`, which can be copied from the iOS fixtures.
- **Compose UI smoke test:** launch, card visible, tap a row, detail, back. It must be deterministic: a debug-only test hook injects the fixture, a fixed location (Naples) and a frozen clock, like iOS `-uiTesting`.
- Coverage of at least 80% on `domain/` and `data/`.
- **GitHub Actions on every PR:** `lint testDebugUnitTest assembleDebug` with a Gradle cache and a placeholder Maps key, plus ktlint. No signing in CI.

## 6. Release (user steps in bold)

1. **The user creates a Google Play Console developer account** (US$25, one-off, with identity verification). New personal accounts must run a **closed test with at least 12 testers for 14 continuous days** before production access. Organisation accounts are exempt. Some third-party sources mention engagement checks; verify the rules on Google's help page when the time comes.
2. **The user creates a Google Cloud project and a Maps SDK for Android key.** A billing account is required, though mobile dynamic maps are free. The key is restricted to `com.binarybuilders.refyl` with the debug, upload and Play-signing SHA-1s, which are added once Play App Signing exists.
3. Internal testing track first, on the user's Pixel, then the closed test.
4. **Store listing:** the iOS IT/EN texts adapted to Android; phone screenshots from the Pixel or emulator at Play's sizes; the feature graphic (1024×500); category Maps & Navigation.
5. **Data safety:**
   - **App data:** no data collected by the app itself.
   - **Google Maps SDK:** disclose its collection following Google's guidance page at release time.
   - **Privacy policy:** the website's privacy page (`leixien.github.io/refyl-site/privacy/`) gets an Android paragraph for Google Maps, keeping the iOS statements unchanged.
6. **Website:** once the app is live, add a Google Play link next to the App Store one.

## 7. Open release decisions (the user's, at release time)

- Play Console account: personal (closed test with 12 testers for 14 days) or organisation for Binary Builders (needs a D-U-N-S number, no closed test).
- What counts as "done": live in production, or a running closed test.

## 8. Workflow (my calls, from the user's standing preferences)

- One branch and one pull request per step; CI must be green before merging, and the user asks to merge (as in the iOS repo).
- Commit messages in English, `type(scope): subject`, with no `Co-Authored-By` trailers (a rule in the user's Android `CLAUDE.md`).
- Gradle runs with Android Studio's bundled JBR (`JAVA_HOME` set per command), not the system JDK 25.

## 9. Left to the builder

- Type and file names.
- How the ViewModels are split.
- The sheet implementation and its exact heights.
- Icon choices within Material Symbols.
- Animations, exact string wording (IT and EN), and the exact movement threshold.
- How pins are drawn (composable markers or bitmaps), as long as 150 pins stay smooth on the Pixel.
- Gradle, AGP and Kotlin versions (latest stable at bootstrap).
