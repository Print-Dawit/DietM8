# DietM8 plan: a privacy-first diet tracker built as a fork of Chompass

**DietM8 is built by forking Chompass (MIT, native Kotlin/Compose) and editing it toward the DietM8 architecture, not by writing a new app.** Every architectural decision from the DietM8 planning interview is kept: no INTERNET permission, `allowBackup=false`, a bundled layered UK-first food database, unknown-not-zero nutrients snapshotted into every entry, a versioned UUID-keyed export envelope shared with WorkM8, SAF folder export to Syncthing-Fork, UK reference values with a never-red gap view, adaptive targets that suggest and never overwrite, supplements, food exclusion tags, and a skippable first-run onboarding flow. Two decisions change because Chompass already solves them differently and the owner chose to keep its approach: **storage stays as Chompass's month-bucketed JSON files plus DataStore (not Room)**, and **nutrients keep Chompass's 23 typed fields plus a new registry-keyed map for everything else**. The fork's main gain is time to daily use: a stripped, offline Chompass is usable as a MyFitnessPal replacement after Phase F0 (about 10 to 16 hours, estimate), where the from-scratch plan needed about 46 to 70 hours. Total build is about 86 to 134 hours (estimate), including about 6 to 10 hours of groundwork so the owner's voice assistant can plug in later (optional wiring, Phase F7, is a further 10 to 16 hours). The main cost is removing Chompass's network features (cloud AI, Coach chat, live Open Food Facts, WebDAV sync, weather, Mealie, update checker, remote speech, model downloads) from a 107,000-line codebase without breaking its 35,000 lines of tests.

**Evidence legend.** **(V)** a source or the Chompass code states it (file paths are given; repo state is commit `3cf301d`, 23 Oct 2026). **(I)** my design inference or recommendation. **(U)** unverified or an unsourced starting point. Hour figures are my estimates. Nothing here is medical advice (see the boundary text).

**Scope.** This plan covers the Android app only. HealthM8 (the Umbrel dashboard, analytics, MCP and laptop model) is a separate plan. The links are the export contract this plan owns, described in "Export contract", and a small voice-assistant interface (section 5.6) that lets the owner's Sovereign Assistant log food and weight and read today's numbers. The assistant interface is designed and tested in the initial build but only wired to the assistant afterwards (Phase F7).

---

## 1. Key recommendations

| # | Recommendation | Why | Status |
|---|---|---|---|
| 1 | Hard fork Chompass at a recorded commit. Rename to DietM8 (`app.dietm8`). Cherry-pick upstream fixes selectively; never merge upstream wholesale. | DietM8 removes and reshapes too much for merges to stay clean. | (I) |
| 2 | Phase F0 strips every network path, the INTERNET and RECORD_AUDIO permissions, Health Connect, and sets `allowBackup=false`. CI asserts the release APK has no INTERNET permission. | The "cannot leak" claim stays checkable in the manifest. | (I) |
| 3 | Keep Chompass's JSON bucket store and DataStore. Add identity, `rev`, and `local_date` at the export boundary and in a revision ledger that is always on. | Owner decision. Avoids rewriting about 8,000 lines of persistence. | Owner decision |
| 4 | Keep the 23 typed micronutrient fields; add `nutrients: Map<String, Double>` with unit-in-key registry IDs (for example `selenium_ug`) beside them. Export flattens both into registry IDs. | Owner decision. Chompass screens keep working while the full panel is added. | Owner decision |
| 5 | Reuse Chompass's bundled SQLite food-index pattern (`OfflineFoodIndex`, USDA and Swiss assets, Python build scripts). Add CoFID and an Open Food Facts UK subset as new indexes; widen columns to the full panel. | The pattern is already in place, tested, and offline. | (V)+(I) |
| 6 | Keep zxing-cpp plus CameraX scanning. Switch the lookup from live Open Food Facts to the bundled OFF UK index; misses go to the custom-food form. | Camera barcode arrives on day one instead of "later". | (V)+(I) |
| 7 | Change Chompass's adaptive goals from "updates the target" to "suggests beside the manual target". | Chompass currently saves the new target to the profile (`services/AdaptiveGoalsService.kt`). DietM8 requires suggest-only. | (V)+(I) |
| 8 | Replace AI free-text logging with the rule-based parser, behind a `FragmentResolver` interface. Keep the on-device LLM code switched off for the later LLM-assisted phase. | Owner decision: rule-based now, LLM later. | Owner decision |
| 9 | Adopt WorkM8's CI standards (spotless/ktlint, detekt, lint, tests, branch and PR, signed GitHub Releases) using baselines so CI is green from day one. | Chompass has ktlint via a CLI only and no detekt or Android CI (V). | (I) |
| 10 | Build the assistant interface as contracts first: parser preview and commit use cases with idempotency keys, a `TodaySummary` use case with templated sentences, and a shared `assistant-card.v1` schema, all tested with `adb` and no assistant (F3, F4, F6). Wire it to the assistant through OACP in Phase F7, after the assistant's V1 is stable. | Owner decision (26 Oct): contracts now, wiring later; DietM8 first. Keeps DietM8 useful alone and costs about 6 to 10 hours inside F3, F4 and F6. | Owner decision |
| 11 | Voice to DietM8 is gated three ways: the assistant shows a confirm card for every write; DietM8 accepts calls only from the owner's own signed assistant build (signature-level permission); the interface exposes no history, supplements or exclusion tags. | Owner decisions (26 Oct). OACP's own trust model is undocumented (U). | Owner decision |

---

## 2. Context and constraints

These restate the owner's decisions so Claude Code sessions can start from this file alone.

| Area | Constraint or decision | Status |
|---|---|---|
| Device | OnePlus 12, degoogled LineageOS. Signed GitHub Release APKs installed via Obtainium. Personal, single user. | Owner |
| Build | Claude Code on the owner's laptop. No deadlines; phases are done as time allows. | Owner |
| Base | Fork of Chompass (fitguy, MIT, based on Fud AI). Keep all DietM8 architecture; reuse Chompass where it already fits. | Owner (26 Oct) |
| Stack (as inherited) | Kotlin 2.2, Compose with Material 3, AGP 9.2, minSdk 26, targetSdk 36, manual DI (`AppContainer` in `ChompassApp.kt`), ViewModel with StateFlow, kotlinx.serialization, DataStore, Vico, CameraX, zxing-cpp, Glance, WorkManager. | (V) |
| Storage | Chompass's `JsonBucketStore` (one JSON file per month, atomic tmp-and-rename writes, corrupt-file preservation) plus DataStore. Not Room. | Owner (26 Oct) |
| Network | No INTERNET permission, ever. Data leaves only as files. | Owner |
| Units | Metric only: kg, g, kcal, mg, µg. | Owner |
| MyFitnessPal | Replace completely; start fresh, no import. Pain points: too many taps, ads and paywalls, data ownership, clean export. | Owner |
| Build order | (1) fast logging, (2) nutrient-gap insight, (3) dashboard-ready export. | Owner |
| Logging | Food search, barcode scan, free text (rule-based now, LLM later). | Owner |
| Food data | Bundled offline database; barcode misses added by hand. No food API. | Owner |
| Nutrients | Everything the database has. | Owner |
| Targets | Calories and protein set manually; adaptive suggestion from weight trend; micronutrients default to UK reference intakes. | Owner |
| Also logged | Supplements, bodyweight and body fat % (DietM8 owns these; WorkM8 v1.1 bodyweight log is dropped), eating window and fasting. | Owner |
| Day and window | Day ends at midnight. Entries of 20 kcal or more count as eating. | Owner |
| Exclusions | Food exclusion tags from the start: pork, salmon, high-parasite foods, dairy except yoghurt and whey. | Owner |
| Watch list | None by default; all nutrients shown equally. | Owner |
| Onboarding | Skippable, re-runnable first-run flow that also collects the supplement list. | Owner |
| WorkM8 link | Apps stay separate. HealthM8 joins exports by `local_date`. | Owner |
| Voice assistant | The owner's Sovereign Assistant (a Hark fork speaking OACP) should be able to log food and weight into DietM8 and read today's numbers. Wanted: log food and weight, ask today's numbers. Not wanted in the first interface: WorkM8 control (separate), history or gap queries. Confirm card on every write; only the owner's own assistant build may call DietM8. Contracts in the initial build, wiring after the assistant's V1. DietM8 is built first. | Owner (26 Oct) |

---

## 3. Fork strategy

**Hard fork (I).** Fork from Codeberg (`codeberg.org/fitguy/Chompass`; the GitHub repo is a mirror) at a named commit and record it in `docs/UPSTREAM.md` with the date. DietM8 removes whole subsystems and changes the data contract, so tracking upstream would produce constant conflicts. Instead:

- Watch upstream releases occasionally. Cherry-pick fixes to code DietM8 still uses (search ranking, JSON store, barcode decode, Compose screens), each as its own PR noting the upstream commit.
- Never pull upstream changes to AI, sync, Health Connect or network code.

**Licence obligations (V).** Chompass is MIT: "Copyright (c) 2026 Apoorv Darshan" with "Modifications Copyright (c) 2026 fitguy". Keep `LICENSE` with both lines and add the owner's line. Keep `docs/NOTICE.md` and `docs/ASSET_CREDITS.md` and extend them. The About screen credits Chompass and Fud AI as well as the food data sources. MIT allows private modification and redistribution with the notice retained.

**Renaming.** Package `app.chompass` to `app.dietm8`, application id, app name, launcher icon (drop Chompass's alternative launcher icons via `AndroidAppIconManager` unless wanted), strings, file names under `filesDir` (for example `fudai-food-images/`), and the release signing key (a new DietM8 keystore; never reuse Chompass's).

**What is dropped from the repo:** `web/` (PWA), `website/` (Hugo site), `docs/fdroid` and F-Droid metadata, `scripts/deploy_pages.sh`, PWA parity contracts and scripts (`testdata/parity`, `scripts/check_parity*.py`), `devenv.*` (optional; keep if the owner wants Nix).

---

## 4. Component reuse map

Status: **Keep** (use as is), **Adapt** (keep and change), **Remove**, **Add** (new code).

### 4.1 Platform and infrastructure

| Chompass component | Path | Status | DietM8 change |
|---|---|---|---|
| Kotlin/Compose app shell, navigation, theme, components | `ui/navigation`, `ui/theme`, `ui/components` | Keep | Rename, rebrand. |
| Manual DI root | `ChompassApp.kt` (`AppContainer`) | Adapt | Remove AI, sync, weather, Mealie, speech, update, Health Connect wiring. |
| JSON bucket store | `data/JsonBucketStore.kt`, `PersistedJsonGuard.kt` | Keep | Owner decision. Add new stores for supplements, event flags, day status, target sets. |
| DataStore preferences | `data/PreferencesStore*.kt` | Adapt | Remove AI, WebDAV, weather, Mealie, Health Connect keys. Add DietM8 settings. |
| Sync revision ledger | `data/PreferencesStoreSync.kt` (`SyncRevision`), `sync/SyncRepository.kt` (`touch`, `tombstone`) | Adapt | Today it only records when WebDAV is configured (`syncConfigured()`). Make it always on, add `rev` and `created_at`, and use it as the source of `rev`, `updated_at` and `deleted_at` for export. Consider moving it from a single DataStore map to a bucket store if it grows (U). |
| WebDAV sync transport | `sync/WebDavClient.kt`, `WebDavAutoSync.kt`, `WebDavUrl.kt` | Remove | Network. |
| Sync merge logic | `export/SyncMerge.kt`, `SyncDocument.kt` | Adapt | Last-write-wins merge with tombstones is reused for **restore** (idempotent upsert on import). |
| Versioned JSON Schemas and validation | `contracts/`, `scripts/validate_parity_contracts.py` | Adapt | Keep the pattern. Add `contracts/dietm8-export-1.0.schema.json`. Drop PWA parity contracts. |
| Widgets | `widget/` (calorie, protein, all-metrics, water) | Keep | Remaining kcal and protein widget arrives early. |
| WorkManager | `services/*Worker`, `ModelDownloadWorker` | Adapt | Keep for scheduled export to the SAF folder; remove download and health-sync workers. |
| Notifications and reminders | `services/NotificationService.kt`, `ReminderReceiver` | Keep | Keep neutral wording; no streaks (anti-obsession rules). |
| Tests | Robolectric, Compose UI tests, screenshot tests, Maestro flows (`android/maestro`) | Adapt | Delete tests of removed features. Keep the rest green. |

### 4.2 Network features (all removed in Phase F0)

| Chompass component | Path | Lines (approx.) | Status |
|---|---|---|---|
| Cloud AI clients and food analysis (Anthropic, Gemini, OpenAI-compatible, Ollama), chat service | `services/ai/` | 7,500 | Remove |
| Coach chat UI | `ui/coach/` | 1,900 | Remove |
| Live Open Food Facts lookup and search | `services/OpenFoodFactsService.kt` | 700 | Remove the network path. Reuse its OFF field-mapping knowledge in the offline index builder. |
| Update checker | `services/update/AndroidUpdateChecker.kt` | 100 | Remove (Obtainium handles updates). |
| Remote speech-to-text, voice logging | `services/speech/` | 660 | Remove, with `RECORD_AUDIO`. |
| Weather | `services/weather/OpenMeteoClient.kt` | 90 | Remove. |
| Mealie import | `services/mealie/` | 290 | Remove. |
| On-device model download | `services/ondevice/ModelDownloadWorker.kt` and related | part of 1,100 | Remove the downloader. See 4.5 for the engine. |
| Grounded entry (model plus database tools) | `src/grounded`, `src/groundedStubs`, `services/grounding/GroundedEntryFeature.kt` | 1,800 | Remove the model-calling part. Keep `PortionResolver` ideas for the parser. |
| Health Connect read/write and permissions | `services/health/`, manifest `health.*` permissions | 1,550 | Remove (deferred, master D22). Keep a `HealthSource` interface stub. |
| Photo AI logging (up to 10 photos) | `services/FoodPhotoSession.kt`, `FoodImageComposer.kt` | | Remove analysis. Keep optional meal photos stored locally (excluded from export by default). |

### 4.3 Food data and logging

| Chompass component | Path | Status | DietM8 change |
|---|---|---|---|
| Offline food index abstraction and search ranking | `services/grounding/OfflineFoodIndex.kt`, `FoodDatabaseSearch.kt`, `FoodSuggestionRanker.kt`, `FoodSuggestions.kt`, `QueryNormalizer.kt` | Adapt | Add `CofidFoodIndex` and `OffUkFoodIndex`. Source order: own foods, CoFID, USDA, OFF UK, Swiss (off by default). |
| USDA index and build script | `assets/usda/usda_foods.sqlite` (FDC 2024-10-31 Foundation plus FNDDS, 5,808 foods, 26 nutrient columns, FTS5, CC0), `scripts/build_usda_food_index.py` | Adapt | Rebuild with the full nutrient panel and SR Legacy added; current FDC release. |
| Swiss index | `assets/swiss/swiss_foods.sqlite` (4,760 foods), `scripts/build_swiss_food_index.py` | Keep, off by default | Low relevance to a UK diet. Licence terms of naehrwertdaten.ch not checked (U). Drop if unclear. |
| Barcode scanning | CameraX plus zxing-cpp, `services/BarcodeImageDecoder.kt`, `BarcodeCodeNormalizer.kt` | Keep | Lookup goes to the bundled OFF UK index. |
| Food entry model | `models/FoodEntry.kt` | Adapt | Add `nutrients` map, `nutrientQual` map, `localDate`, `utcOffsetMin`, `tzId`, `origin`, `recordingMethod`, `calcVersion`, `exclusionTags` snapshot. |
| Micronutrient catalog | `models/MicronutrientCatalog.kt` (23 typed fields) | Adapt | Add a mapping table from each typed field to its registry ID and unit. |
| Saved foods, favourites, recents, frequents, recipes, saved meals | `data/FoodRepository.kt`, `SavedFoodIndex.kt`, `RecipeRepository.kt`, `models/Recipe.kt`, `MealCatalog.kt` | Keep | Covers pinned meals, go-to meals, recents and copy-from-day. Add versioning to saved meals (edit creates a new version). |
| Meal components (constituents) | `models/FoodConstituent.kt` | Keep | Matches "explode a meal into items". |
| Serving units and heuristics | `models/ServingUnit*.kt` | Keep | Portion chips and last-serving memory. |
| Manual entry and nutrition-label entry | `FoodSource.MANUAL`, `NUTRITION_LABEL` | Keep | Becomes the custom-food form for barcode misses. |
| Free-text entry | `FoodSource.TEXT_INPUT` via AI | Adapt | Rule-based parser replaces the AI call. |
| Gap filling | `models/FoodEntryGaps.kt`, `services/FillGapsEngine.kt` | Adapt | Keep the gap report (missing values per entry) for the data-problems list. Remove the AI fill. |
| Draft recovery | `models/Pending*Draft.kt`, `data/AnalysisQueueStore.kt` | Adapt | Keep drafts; drop the AI analysis queue. |

### 4.4 Body, targets, fasting and other trackers

| Chompass component | Path | Status | DietM8 change |
|---|---|---|---|
| Weight, body fat with method, measurements | `data/WeightRepository.kt`, `BodyFatRepository.kt`, `BodyMeasurementRepository.kt`, `models/BodyFatMethod.kt` | Keep | DietM8 owns bodyweight. Export joins weight and body fat into `body_measurement`. |
| Weight forecast and analysis | `services/WeightAnalysisService.kt`, `WeightForecastMath.kt` | Adapt | No network or LLM (V). Compare its trend method with the DietM8 algorithm; keep whichever passes the back-test. |
| Adaptive goals | `services/AdaptiveGoalsService.kt` | Adapt | Today it saves the new target to the profile (V). Change to write a suggestion record only. Remove the measured-TDEE path, which reads Health Connect energy burn. |
| Goal history | `models/GoalJournal.kt`, `GoalPhase*.kt`, `data/PreferencesStoreRecalcSheet.kt` | Adapt | Becomes `target_set` (effective-dated) and `adaptive_target_run` (reproducible suggestions). |
| Macro plans, day types | `models/MacroPlan*.kt`, `DayTypeActiveStats.kt` | Keep | Day types map to training upper, lower and rest. |
| Diet modes (keto and others) | `models/DietMode.kt`, `KetoCarbMode.kt` | Keep, unused by default | |
| Untracked days | `models/UntrackedDays.kt` | Adapt | Becomes `day_status` with an explicit `complete` flag. |
| Fasting sessions and planners | `data/FastingRepository.kt`, `services/Fasting*Planner.kt` | Adapt | DietM8 derives the eating window from entries of 20 kcal or more. Keep the fasting timer as optional. Remove streak-like features. |
| Water, caffeine, nicotine, notes | `data/WaterRepository.kt`, `CaffeineRepository.kt`, `NicotineRepository.kt`, `NotesRepository.kt` | Keep, off by default | Caffeine matches a suggested extra stream. Nicotine can be removed if unwanted. |
| Progress screens and charts | `ui/progress/` (Vico) | Keep | |
| Calculation methods screen | `ui/settings/CalculationMethodsScreen.kt` | Adapt | Document DietM8's formulas and sources here. |
| Onboarding | `ui/onboarding/` (`OnboardingStep`: WELCOME, GENDER, BIRTHDAY, HEIGHT_WEIGHT, BODY_FAT, ACTIVITY, GOAL, DIET_MODE, GOAL_WEIGHT, GOAL_SPEED, NOTIFICATIONS, HEALTH_CONNECT, HOME_LOOK, PROVIDER, BUILDING_PLAN, PLAN_READY, DISCLAIMERS; resumable draft) | Adapt | See section 9. |

### 4.5 New code (nothing in Chompass covers it)

| New component | Purpose |
|---|---|
| Nutrient registry (`nutrient_def` asset) and typed-field mapping | Unit-in-key IDs, cross-references, UK reference values. |
| CoFID and OFF UK index builders | Python scripts in `scripts/refdb/`, modelled on `build_usda_food_index.py`. |
| Rule-based free-text parser | Section 7.1. |
| Supplements (products, per-dose panels, tick list) | Section 6. |
| Food exclusion tags | Tags on foods and saved meals; a filter for later suggestion logic. |
| UK reference table and gap grading (green, amber, grey) | Section 7.5. |
| DietM8 export: envelope, zip, CSVs, data dictionary | Section 5. |
| SAF persisted folder export and "folder lost" state | Chompass has no SAF folder export (no `ACTION_OPEN_DOCUMENT_TREE` use found) (V). |
| `device_id`, `rev`, `local_date` derivation | Section 5. |
| Event flags | Freeze adaptive updates around creatine, illness, travel, scale change. |
| Manifest permission CI test | Section 11. |
| On-device LLM engine (later) | `services/ondevice/OnDeviceLlmEngine.kt` (LiteRT-LM, Gemma 4 E2B or E4B, needs arm64 and 6 GB+ RAM per Chompass's README) is kept in the repo but switched off. For the later LLM-assisted parser it loads a model file the owner copies to the phone by hand; no download code. APK size and RAM cost on the OnePlus 12 are unmeasured (U). |

---

## 5. Data model and the export contract

### 5.1 Principles (unchanged from the DietM8 design)

- **Snapshot, don't reference.** Each entry stores the food name, portion, grams and the full nutrient vector for the amount eaten. Chompass's `FoodEntry` already does this (V). Editing a food or refreshing a database never rewrites history.
- **Unknown is not zero.** A missing nutrient is null (typed fields) or an absent key (map). A known zero is 0. Chompass's micronutrient fields are already nullable (V). Daily totals report coverage.
- **Stable identity.** Every exported entity has an `id`, a `rev` that increases by 1 per edit, `created_at`, `updated_at`, `deleted_at` (tombstone) and `origin_device_id`. Chompass ids are random UUIDv4 (V). Decision: existing v4 ids stay valid; new entities created by DietM8 use UUIDv7. Readers treat ids as opaque strings (I).
- **Time.** Store the instant (Chompass `timestamp`), plus `utc_offset_min`, `tz_id` and a derived `local_date`. Joins use `local_date`. Use `java.time` only (`Instant.atZone(zone).toLocalDate()`); never date arithmetic. Day boundary is midnight (owner decision); the envelope still carries `day_boundary: "00:00"`.
- **Nutrient keys carry their unit** (`vitamin_d_ug`, `iron_mg`, `energy_kcal`). A key's meaning never changes; a new unit or definition gets a new key. A `nutrient_def` registry ships with each export.
- **Full exports with idempotent upsert.** `export_kind: incremental` is reserved for later.

### 5.2 How the two decisions shape storage

**JSON store, not Room (owner decision).** Each user-data type is a JSON bucket store or a DataStore entry, as in Chompass. Consequences (I):

- No SQL queries on the phone. Daily totals and 14-day grades are computed in Kotlin from month buckets (at about 10 entries a day, a 14-day window is about 140 objects; my estimate, not benchmarked).
- Schema evolution uses kotlinx.serialization defaults, which Chompass already relies on ("defaults keep older bucket JSON decodable", `models/FoodEntry.kt`). Every new field must have a default. A decode test per release reads fixtures from every earlier version.
- "Never destructive for user data" still applies: Chompass's `PersistedJsonGuard` copies a corrupt file aside before any overwrite (V). Keep it.
- The bundled food indexes are separate read-only SQLite assets, replaced wholesale on a version bump (the narrow exception to "never destructive", unchanged).

**Typed fields plus a nutrient map (owner decision).**

- `FoodEntry` keeps `calories`, `protein`, `carbs`, `fat` and the 23 typed micronutrients (sugar, added sugar, fibre, saturated, mono and polyunsaturated fat, cholesterol, sodium, potassium, trans fat, calcium, iron, magnesium, zinc, vitamins A, C, D, B12, E, K, folate, omega-3, caffeine) (V).
- New `nutrients: Map<String, Double>` holds every other nutrient by registry ID: thiamin, riboflavin, niacin, B6, pantothenic acid, biotin, choline, phosphorus, selenium, iodine, copper, manganese, chloride, EPA, DHA, vitamin K fractions, and so on.
- New `nutrientQual: Map<String, String>` holds quality per key (measured, trace, label, estimated, user).
- A fixed table maps each typed field to one registry ID. The map never duplicates a typed field. Export writes one flat `nutrients` object of registry IDs.
- **Form ambiguity (U):** Chompass's `vitaminA` (µg) and `folate` (µg) do not record form (retinol equivalents vs RAE; total folate vs DFE). The mapping must record which form each source supplies, and values of different forms are never summed. Check what the USDA build script writes before fixing the mapping.

### 5.3 Entities

| Entity | Storage (Chompass base) | Key fields added or kept | Notes |
|---|---|---|---|
| `nutrient_def` | Bundled asset (new) | `key`, `display_name`, `unit`, `group`, `decimals`, cross-references (`infoods`, `fdc_nutrient_nbr`, `off_key`, `typed_field`), `uk_ref_male`, `uk_ref_kind`, `ref_source`, `ref_verified_on`, `sort_order` | Cross-references are hints, not keys. |
| Custom food / saved food | Chompass favourites and saved foods | `barcode?`, `state`, `basis_qty`, `basis_unit`, `source`, `exclusion_tags[]` | Own foods rank first in search. |
| Saved meal (template) | Chompass saved meals and recipes | `version`, `pinned`, `slot_hint`, `exclusion_tags[]` | Editing creates a new version; past entries unaffected. |
| `food_log_entry` | `FoodEntry` in month buckets | `local_date`, `utc_offset_min`, `tz_id`, `origin` (food, recipe, supplement, quick_add), `recording_method` (manual, barcode, parsed, search, import), `nutrients{}`, `nutrient_qual{}`, `calc_version` | Chompass `mealType` stays as an optional label. |
| `supplement` | New bucket store | `name`, `brand`, `dose_qty`, `dose_unit`, `schedule`, `active`, per-dose `nutrients{}` (blank means unknown) | Product definition. |
| Supplement intake | `FoodEntry` with `origin='supplement'` | | Same table, so totals stay simple and food-only intake is recoverable. |
| `body_measurement` | Weight and body fat repositories, joined at export | `local_date`, `weight_kg`, `body_fat_pct?`, `method` (dexa, bia8, bia4, tape, other; chosen per reading), `fasted?` | DietM8 is the single owner of bodyweight. Readings from different methods are not directly comparable; trends should be shown per method or flagged when the method changes (I). |
| `target_set` | Adapted `GoalJournal` | `effective_from`, `kcal`, `protein_g`, `mode` (manual/adaptive), `day_type_overrides?`, `reason` | Effective-dated, never overwritten. |
| `micro_target` | New, seeded from `nutrient_def` | `effective_from`, `nutrient_key`, `value`, `kind`, `source` | UK defaults, overridable. |
| `adaptive_target_run` | Adapted recalc sheet | `run_at_utc`, `algorithm`, `params_json`, `trend_weight_kg`, `slope_kg_per_week`, `expenditure_est`, `suggested_kcal`, `confidence`, `accepted` | Reproducible suggestions. |
| `day_status` | Adapted `UntrackedDays` | `local_date`, `complete`, `day_type`, `note?` | Unlogged days never look like 0 kcal. |
| `eating_window_override` | New | `local_date`, start, end | Derived window is computed, not stored. |
| `event_flag` | New | `local_date`, `kind` (creatine_start, illness, travel, scale_change, other) | Freezes adaptive updates for 14 days. |
| `user_profile` | Chompass `UserProfile` | height, optional birth year (for the adaptive estimate's age term), onboarding state per step | Gender and goal-weight fields stay but are optional. |
| `app_setting` | DataStore | day boundary, eating threshold (20 kcal), exclusion-notice toggle, watch list, hide toggles, export-folder URI and state | Not exported. |

### 5.4 Export contract (DietM8 owns this; HealthM8 consumes it)

The envelope is a design, not a standard; it borrows Health Connect's id, version and origin ideas (V)+(I). WorkM8 adopts the same envelope.

```json
{
  "envelope": {
    "envelope_version": 1,
    "export_id": "0197f3b0-1a4e-7d10-8c3b-5a1d0e9f7b42",
    "app": { "id": "app.dietm8", "name": "DietM8", "version": "0.3.0", "build": 12 },
    "schema": { "name": "dietm8.export", "version": "1.0" },
    "exported_at": "2026-10-21T18:10:00Z",
    "timezone_at_export": { "id": "Europe/London", "utc_offset_min": 60 },
    "device_id": "5b0a4a3e-2d6a-4e0c-9a3f-0d4c7d9e8f11",
    "export_kind": "full",
    "since_updated_at": null,
    "date_range": { "from": "2026-09-01", "to": "2026-10-21", "basis": "local_date" },
    "day_boundary": "00:00",
    "units": { "mass": "kg", "energy": "kcal", "body_fat": "percent", "nutrients": "see nutrient_def[].unit" },
    "counts": { "food_log_entry": 412, "body_measurement": 48 },
    "files": [ { "path": "food_log_entries.csv", "entity": "food_log_entry", "rows": 412, "sha256": "..." } ]
  },
  "data": {
    "nutrient_def": [ { "key": "vitamin_d_ug", "name": "Vitamin D", "unit": "ug", "group": "vitamin", "decimals": 1,
                        "typed_field": "vitaminD", "uk_ref": { "male": 10, "kind": "RNI", "verified_on": null } } ],
    "food_log_entry": [
      { "id": "0197f3a2-7c1e-7b52-9a0d-3f6c2c7a1e11", "rev": 2,
        "created_at": "2026-10-21T11:42:03Z", "updated_at": "2026-10-21T11:44:10Z", "deleted_at": null,
        "origin_device_id": "5b0a4a3e-2d6a-4e0c-9a3f-0d4c7d9e8f11",
        "eaten_at_utc": "2026-10-21T11:41:00Z", "utc_offset_min": 60, "tz_id": "Europe/London",
        "local_date": "2026-10-21", "meal_label": "lunch", "origin": "food",
        "food_name_snapshot": "Greek yoghurt, 0% fat (illustrative)", "portion_qty": 200, "portion_unit": "g", "portion_grams": 200,
        "kcal": 118, "protein_g": 20.4, "carbs_g": 7.2, "fat_g": 0.4,
        "nutrients": { "energy_kcal": 118, "protein_g": 20.4, "calcium_mg": 220, "vitamin_b12_ug": 1.0, "iodine_ug": 60 },
        "nutrient_qual": { "calcium_mg": "measured", "vitamin_b12_ug": "label" },
        "source": "cofid", "recording_method": "search", "calc_version": 1 }
    ],
    "body_measurement": [ ... ], "target_set": [ ... ], "supplement": [ ... ], "day_status": [ ... ],
    "adaptive_target_run": [ ... ], "event_flag": [ ... ]
  }
}
```

All numbers are illustrative. Rules: a missing nutrient key means unknown; `kcal` equals `nutrients.energy_kcal`; tombstones carry identity columns plus `deleted_at` and `rev`; readers ignore unknown fields and enum values; readers refuse a higher MAJOR version and warn on a higher MINOR; minor bumps are additive only.

**Packaging:** one immutable zip per export, uniquely named (for example `dietm8_full_2026-10-21T1810Z_<uuid>.zip`), holding `manifest.json` (envelope), `backup.json` (envelope plus data), per-entity CSVs, `daily_totals.csv`, `daily_nutrients_long.csv` and `data_dictionary.json`. CSV: UTF-8, no BOM, comma, RFC 4180 quoting, `.` decimal, ISO 8601, empty means null, headers only appended to.

**`daily_totals.csv`** (one row per `local_date`): date, tz_id, day_complete, entries_n, energy and macros, every nutrient key as a column, micro_coverage_pct, targets in effect, first and last entry local time, eating_window_min, fast_before_min (assigned to the day the fast ends), weight_kg, weight_trend_kg, body_fat_pct, schema_version. **`daily_nutrients_long.csv`**: date, nutrient_key, unit, amount, origin, target, target_kind, pct_of_target, entries_with_data_n, entries_n.

**Relationship to Chompass's own exports.** Chompass's diary JSON (`format_version` 1.6, `export/DiaryExporter.kt`), Markdown and CSV exports and its body-metrics export stay available as human-readable extras. The DietM8 envelope export is the only contract HealthM8 reads. The JSON Schema lives in `contracts/dietm8-export-1.0.schema.json` and is validated in CI, reusing Chompass's contracts pattern.

### 5.5 Minimal WorkM8 changes (separate PR in the WorkM8 repo)

1. Wrap the JSON backup as `{ "envelope": {...}, "data": <current object> }`; restore still accepts the legacy shape.
2. Add a `uuid` column to sessions, sets, slots and exercises (backfilled), plus `rev`, `updated_at`, `deleted_at` where edits and deletes exist.
3. Export session times as an ISO instant plus `utc_offset_min`, `tz_id`, `local_date`.
4. CSV headers with units and appended columns `session_uuid`, `set_uuid`, `local_date`, `session_started_at_utc`, `utc_offset_min`, `exercise_uuid`.
5. Generate and persist a random `device_id` once.
6. Ship a `data_dictionary.json`.
7. Bodyweight: the v1.1 bodyweight log is dropped (owner decision); DietM8 owns bodyweight.

### 5.6 Assistant interface (contracts in the initial build, wiring in F7)

**What the owner asked for (26 Oct).** Voice logging of food and weight, and spoken answers to "how am I doing today". Seamless means the interface exists, is tested and matches what HealthM8 serves, so connecting the assistant later is wiring and not redesign. The assistant is a Hark fork (Flutter, Apache 2.0) that discovers apps through OACP, the Open App Capability Protocol.

**What OACP is, as read (V, from the Hark and OACP READMEs and the getting-started page, through a model-written extraction; spot-check before building).** An app ships `oacp.json` in `src/main/assets/` and a `ContentProvider` with authority `<applicationId>.oacp` that serves `/manifest`. Each capability has an id, description, aliases, examples, keywords, typed parameters, `confirmation`, `visibility`, `completionMode` and an `invoke.android` block that is either a **broadcast** (background, to an exported `BroadcastReceiver`) or an **activity** (foreground). The Android SDK offers `OacpReceiver.onAction(...)` returning `OacpResult.success(message, data)`, which is broadcast back by `requestId`. No INTERNET permission is mentioned anywhere, so a no-network app can take part. Maturity: manifest version 0.3 labelled preview; Hark showed 0 stars, no releases and 80 commits; the SDK is not on Maven Central yet. **Not documented, so treated as unknown (U):** the trust model (any app may call an exported receiver), how `confirmation` and `visibility` are enforced, and how the result broadcast is addressed.

**Design (I).**

| Piece | Rule |
|---|---|
| Use-case layer | The interface is a thin adapter over three domain use cases that exist and are unit-tested from F3 and F4 whether or not the assistant exists: `PreviewEntry` (parse, no write), `CommitPreview` (write, idempotent), `TodaySummary` (read). Receivers contain no logic. |
| Capabilities (v1) | `food_preview(text, meal?, when_local?)`, `body_preview(weight_kg, body_fat_pct?, method?, when_local?)`, `assistant_commit(preview_id)`, `today_summary(date?)`, and `open_quick_log(text)` (foreground, prefilled, the fallback for unresolved fragments). Nothing else is exposed: no history, no gap or grade queries, no supplements, no exclusion tags, no deletes. |
| Two-step writes | A preview returns `preview_id`, the parsed items (name, quantity, unit, grams, kcal, protein, confidence), `unresolved[]` and an expiry of 5 minutes. Only `assistant_commit` writes. The assistant shows the confirm card between the two calls (owner decision). Previews live in a small cache file so a killed process does not lose them. Commit is idempotent on `preview_id`; a repeat returns the same entry ids. |
| Parsing | DietM8 parses the raw text with its own rule-based parser (section 7.1), so a spoken and a typed phrase give identical results. The contract also accepts an optional `items[]` array of hints that is ignored until `FragmentResolver` has a real implementation. Trade-offs in the table below. |
| Body entries | `body_preview` validates ranges and converts nothing silently (kg only). A body-fat value requires a `method` from the DM-22 list, because the method is chosen per reading; a voice phrase without a method gets weight only. |
| `today_summary` | Returns an `assistant-card.v1` object: `sentences[]` built from fixed templates, `figures[]` with ids and units, `coverage`, `caveat`, `as_of`, `sensitivity: "health"`. It states what was logged and what remains against the manual target, says plainly when few items are logged ("only 2 items logged, so this is partial"), never uses red or "deficient" wording, and shows the adaptive suggestion only as a separate line if the owner enables it. |
| Caller trust | Receivers and the provider's data paths require a custom permission `app.dietm8.permission.ASSISTANT` with `protectionLevel="signature"`, so only an app signed with the same key (the owner's Hark fork) can call them. The result is sent as an explicit broadcast to the assistant's package, protected by the same permission, so other apps cannot read it. Fallback if the Hark fork cannot send with the permission: check `getSentFromPackage()` on Android 14 and later. The capability manifest itself (names and descriptions only) may be world-readable. (U): OACP's result path, signature-permission grant order after installs, and behaviour on LineageOS; all tested in F7. |
| Health-data handling | Every payload carries `sensitivity: "health"`. The assistant must not write such payloads to its memory, its Syncthing-synced files or its logs, and must not place them in a context that also holds web-search results or can send data out. These are requirements on the assistant plan, which the owner controls. |
| No network, no background | The receivers run only when called; DietM8 adds no service, no INTERNET permission and no exported component outside the permission-protected ones. CI asserts the exported-component list as well as the permission list. |
| Concurrency | A receiver hands work to the same repositories the UI uses, serialised by the existing store mutex, with `goAsync()` and a short timeout; if the app process is cold, the receiver starts the minimum of `AppContainer`. Tested with the UI writing at the same moment. |

**Who parses the sentence (owner asked for trade-offs).**

| Option | For | Against |
|---|---|---|
| DietM8 parses raw text (plan default) | One parser and one food database; typed and spoken logging behave the same; tested by the owner's phrase fixture; the assistant needs no food knowledge; any future client works. | The rule-based parser handles natural speech ("a couple of eggs on toast") worse than a language model; unresolved fragments come back as data and fall to `open_quick_log`. |
| Assistant extracts items first | A small model may cope better with messy phrasing. | A 0.6B slot-filling model on quantities and units is unproven (U); DietM8 must still match every name, so there are two sources of truth; the contract becomes coupled to one model. |
| Hybrid (later) | DietM8 first; the optional `items[]` hints are used once `FragmentResolver` is implemented. | Needs the on-device LLM decision (OD-4) and a measured fixture comparison. |

**Contract files.** `contracts/assistant-card.v1.schema.json` (shared by DietM8 `today_summary` and HealthM8 answers, so one speaking path serves both), `contracts/assistant-capabilities.v1.json` (the OACP manifest source) and fixtures in `docs/fixtures/assistant/`. HealthM8 copies the card schema the same way it copies the export schema.

**What the initial build can and cannot show.** It can show, with `adb shell content read` and `am broadcast` and a small test script, a food phrase previewed, committed once, committed twice without a duplicate, and refused when unsigned; a weight entry; and a `today_summary` that validates against the card schema. It cannot show a real voice interaction, because that needs the assistant's V1 and a Hark fork with the permission change. That is Phase F7.

---

## 6. Food data

### 6.1 Layers

| Layer | Use | Licence and facts | In Chompass? |
|---|---|---|---|
| Owner's own foods, meals and history | First in every search and parse. | Local. | Yes (saved foods, recents, frequents) (V) |
| CoFID 2021 (McCance and Widdowson) | Primary UK generic foods and micronutrients. | 2,887 foods plus 393 "old foods", OGL v3.0, one Excel workbook, last updated 19 Mar 2021. "Tr" means trace, "N" means present but unreliable ([GOV.UK](https://www.gov.uk/government/publications/composition-of-foods-integrated-dataset-cofid)). (V) | **No, add** |
| USDA Foundation, SR Legacy, FNDDS | Gap-filler for raw ingredients and nutrients CoFID lacks (choline, K fractions); FNDDS for mixed dishes. | CC0 ([FDC downloads](https://fdc.nal.usda.gov/download-datasets)). (V) | Foundation plus FNDDS with 26 nutrients (V). **Adapt:** add SR Legacy, widen to full panel. |
| OFF UK subset | Barcodes and branded macros, not micronutrients. | ODbL. About 197,118 UK-tagged products ([OFF](https://world.openfoodfacts.org/data)). (V) | Live API only (V). **Add** a bundled index. |
| Swiss Food Composition Database | Last-resort fallback. | Federal dataset; Chompass downloads it via the Food You project's mirror (`docs/ASSET_CREDITS.md`) (V). Terms not checked (U). | Yes. Keep off by default or drop. |

### 6.2 Build pipeline (reuse the Chompass pattern)

- Chompass ships each source as its own SQLite asset with a `foods` table (one column per nutrient), a `food_portions` table, an FTS5 index and a manifest with dataset version, food count, SHA-256, licence and attribution (V, `assets/usda/usda_foods.manifest.json`).
- DietM8 keeps one asset per source, as Chompass does, so sources stay separable for licence reasons. Build scripts move to `scripts/refdb/`: `build_usda_food_index.py` (adapted), `build_cofid_food_index.py` and `build_off_uk_food_index.py` (new). The manifest gains `nutrient_keys` and `built_from` (source file URLs and hashes) so builds are reproducible.
- FTS5 already works through Chompass's SQLite access on Android (V: the indexes ship with FTS5 tables). Whether it works on LineageOS's system SQLite is confirmed in Phase F0 by running search on the phone.
- **Normalisation rules (unchanged):** convert OFF `_100g` grams to mg or µg; never sum vitamin A forms (RE, RAE, IU), folate forms (total, DFE, folic acid) or niacin forms; vitamin D IU to µg; salt equals sodium times 2.5; CoFID "Tr" becomes 0 with `qual=trace`, "N" becomes unknown, blank stays unknown. Build the USDA crosswalk from `nutrient.csv` in the real download.
- **Completeness measurement first (Phase F2):** a one-off DuckDB query over the OFF UK dump and the CoFID workbook measures the share of foods with each nutrient. Gap-view decisions wait on those numbers.
- **Sizes (U):** Chompass's two assets are about 5.3 MB (V). CoFID about 2 to 5 MB, widened USDA about 10 to 25 MB, OFF UK trimmed to about 35 columns about 30 to 80 MB. The OFF UK index is the APK-size risk; options are fewer columns, products with energy only, or splitting it into a separate file copied once by hand.
- **Updates:** a script builds the assets; a release carries them. OFF every 1 to 3 months from a full dump; CoFID and USDA checked twice a year. Favourites relink by source plus source id or barcode.
- **Licence display:** About screen shows Open Food Facts (ODbL, attribution), CoFID (OGL v3), USDA (CC0), plus Chompass and Fud AI credits.

### 6.3 Barcode flow

Scan with Chompass's zxing-cpp plus CameraX scanner (V). Look up the bundled OFF UK index. If found, prefill the entry (macros, label quality). If not, open Chompass's manual/nutrition-label form prefilled with the barcode; the owner types label values and saves a custom food that ranks first next time. No network fallback.

---

## 7. Rules and algorithms

### 7.1 Free-text parser (rule-based, replaces Chompass's AI text entry)

1. Split on "and", commas, "+" and "with".
2. Per segment, parse quantity (digits, fractions, number words, "a" as 1) and unit (g, kg, oz, ml, tsp, tbsp, cup, slice, piece). Chompass's `ServingUnitHeuristics.kt` and the grounded `PortionResolver.kt` contain unit logic to adapt (V).
3. Strip preparation words.
4. Search candidates through Chompass's `FoodDatabaseSearch` and `FoodSuggestionRanker` (own foods first) (V).
5. Convert to grams through the food's portions, then a generic unit table.
6. Return `ParsedItem(raw, qty, unit, grams?, candidates[], confidence, unresolved[])`.

Show a preview card with one-tap confirm or edit. Test with a fixture file of the owner's real phrases. The LLM hand-off sits behind `FragmentResolver` with a no-op default; later it can call the on-device engine (section 4.5), returning the same `ParsedItem` schema.

### 7.2 Logging speed budget

| Case | Budget |
|---|---|
| Usual meal | 1 tap from Today (2 with a portion tweak) |
| Quick-add kcal and protein | 3 actions or fewer |
| Search and add | Under 10 actions |
| Recent or favourite | 2 actions |

Chompass's single search field with recents, frequents, favourites and recipes ranked together (CHANGELOG, "Adding food is one search field") (V) is close to this already. Measure with the owner's stopwatch in Phase F3.

### 7.3 Eating window and fasting

Derive the window per `local_date` from the first and last entry of **20 kcal or more** (owner decision); water is excluded. One-tap override. Store `fast_before_min` on the day the fast ends. No fasting streaks or badges; Chompass's fasting timer stays optional. Test: a coffee below the threshold, and a late snack across midnight.

### 7.4 Targets and the adaptive suggestion

- Manual calories and protein first. Protein presets in g/kg on **trend weight**: UK RNI floor 0.75, ISSN low 1.4, Morton breakpoint 1.6, commonly used 2.0, Morton upper 2.2 ([BNF](https://www.nutrition.org.uk/media/nmmewdug/nutrition-requirements.pdf), [ISSN](https://pubmed.ncbi.nlm.nih.gov/28642676/), [Morton et al.](https://bjsm.bmj.com/content/52/6/376)) (V).
- **The adaptive suggestion never overwrites the manual target.** Chompass's `AdaptiveGoalsService.refreshIfNeeded` currently saves the updated profile (V); this is changed to write an `adaptive_target_run` and show a card beside the manual target.
- Algorithm (every parameter is an unsourced starting point (U)): exponential trend with alpha 0.10, outlier gate 2.5 kg, 14-day window (21 to 28 if sparse), slope by OLS or Theil-Sen, rho 7,700 kcal/kg, Kalman filter over a Mifflin-St Jeor prior ([Mifflin et al. 1990](https://pubmed.ncbi.nlm.nih.gov/2305711/)) times an editable PAL; update only with at least 4 of 7 days logged and 6 weigh-ins in 14 days; freeze 14 days after an event flag; weekly updates capped at plus or minus 150 kcal; rounded to 25 kcal.
- Chompass already has an energy-balance and regression forecast (`WeightAnalysisService`, "No network, no LLM", V) and sparse-logging handling. Back-test both on synthetic data (known expenditure, noise, missing days, a creatine step) and the owner's history; keep the better one or combine. Report mean absolute error in kcal per day and days to reconverge.
- Remove Chompass's measured-TDEE input, which reads energy burn from Health Connect.

### 7.5 Nutrient-gap view and data-quality grading

- UK reference values are population figures, not personal requirements; the RNI "meets the needs of about 97.5% of a group" ([BNF](https://www.nutrition.org.uk/nutritional-information/nutrient-requirements/)) (V). LRNI and EAR were not retrievable, so **the gap view never shows red**.
- 14-day and 7-day means. Coverage per nutrient is the share of logged energy from entries with a value.
- **Green:** coverage 90% or more and at least 5 of 7 days logged, for a nutrient with a low estimated share. **Amber:** coverage 70 to 90%, or a high-estimated nutrient (vitamin E, carotenoids, K, choline, fibre fractions, omega-3 split). **Grey:** coverage under 70%, under 5 of 7 days logged, or a soil- or fortification-dependent nutrient without values (iodine, selenium, vitamin D, choline). Thresholds are mine (U).
- Wording: "logged intake is X% of the reference; data quality: amber". Never "deficient".
- Energy-linked references (saturated fat, free sugars) as a percentage of energy excluding ethanol; sodium only as a 6 g salt ceiling.
- Supplements shown as a separate source; NHS "unlikely to cause harm" figures are never shown as toxicity thresholds.
- **Watch list: none by default; all nutrients shown equally** (owner decision). The owner can pin nutrients.
- Chompass's `ProgressAdherence` and `NutrientGoals` (home top-nutrient tiles) are the UI starting points; the grading logic is new.

**UK default table** (men 19 to 64, from the [GDR 2016](https://assets.publishing.service.gov.uk/government/uploads/system/uploads/attachment_data/file/618167/government_dietary_recommendations.pdf); check every row against the official publication; CI fails on a row without a source URL or `verified_on`):

| Nutrient | Seed value | Note |
|---|---|---|
| Vitamin A | 700 µg RE | Form differs by database. |
| Thiamin | 1.0 mg | |
| Riboflavin | 1.3 mg | |
| Niacin | 16.5 mg NE | BNF gives 17 (19 to 50). |
| Vitamin B6 | 1.4 mg | |
| Vitamin B12 | 1.5 µg | |
| Folate | 200 µg | |
| Vitamin C | 40 mg | |
| Vitamin D | 10 µg | |
| Calcium | 700 mg | |
| Phosphorus | 550 mg | |
| Magnesium | 300 mg | |
| Potassium | 3,500 mg | |
| Iron | 8.7 mg | |
| Zinc | 9.5 mg | |
| Copper | 1.2 mg | |
| Iodine | 140 µg | |
| Selenium | 75 µg | |
| Chloride | 2,500 mg | |
| Fibre | 30 g | |
| Salt | below 6 g | A ceiling. |
| Not obtained: vitamin E, K, pantothenic acid, biotin, manganese, chromium, molybdenum, choline, EPA plus DHA, fluoride | | No default until sourced; user-set or hidden. |

### 7.6 Anti-obsession safeguards and the medical boundary

- No streaks, shaming colours, "earn your food" credits or fasting badges. Check Chompass's notifications, review prompts and progress screens for any of these and remove them (`services/ReviewPrompter.kt`, `InAppReview.kt` are removed outright).
- Hide calories and/or weight; "pause tracking" mode.
- No auto-reduced targets; adaptive suggestions are weekly, capped, and one tap from "ignore".
- Trends and 7-day averages first.

**Boundary text** (About screen, Insights screen, and every later LLM schema as a required field):

> This tool compares what you logged with general UK reference values for healthy adults. It is not medical advice and cannot diagnose a deficiency. Logged intake can be wrong where food data is incomplete or foods go unlogged. Reference values are population-level, not personal requirements, and NHS "unlikely to cause harm" figures are not safe-dose recommendations. Speak to a GP, pharmacist or registered dietitian before changing your diet or supplements, and about symptoms, abnormal blood results, or any supplement above NHS guidance.

---

## 8. Screens

| Screen | Chompass base | DietM8 changes | Phase |
|---|---|---|---|
| Today (home) | `ui/home` calorie ring, macros, meals | Remaining kcal and protein (hideable), eating-window chip, pinned meals, recents, timeline, "Mark day complete", copy yesterday. Remove AI and weather cards. | F0 (strip), F3 |
| Add food | One search field, photo and barcode buttons | Remove photo-AI and voice. Add the free-text parse preview. Barcode uses the offline index. | F0, F3 |
| Entry detail | Review/edit sheets | Edit this entry or update the template; show nutrient source and quality. | F3 |
| Meals | Saved meals, recipes, components | Versioning, pinning, exclusion tags. | F3 |
| Supplements | None | Products with per-dose panels, today's tick list. | F4 |
| Body | `ui/progress` weight, body fat, measurements, forecast | Trend line, outlier prompt, event flags. | F4 |
| Targets | Goals and macro plan screens | Manual kcal and protein, g/kg presets on trend weight, day types, adaptive suggestion card, micronutrient profile. | F4 |
| Insights | None (progress adherence is closest) | 14-day grades, contributing foods, data-problems list, boundary text. | F5 |
| Export and backup | Chompass diary and body exports | Export folder status (set or lost), export now, last export, restore with idempotent upsert. | F1 (backup), F6 (folder) |
| Onboarding | `ui/onboarding` | Section 9. | F4 |
| Settings and About | `ui/settings` | Remove AI, Health Connect, WebDAV, Mealie, weather, speech. Add "this app has no network permission", attributions, hide toggles, pause, boundary. | F0, F4 |
| Widgets | `widget/` | Keep calorie and protein widgets. | F0 |

---

## 9. Onboarding (adapting Chompass's flow)

Chompass's onboarding is a resumable step sequence persisted as a draft so it survives the process being killed (V, `OnboardingViewModel.kt`). DietM8 keeps that machinery and changes the steps. Every step is skippable, re-runnable from Settings, and writes only what it lists.

| DietM8 step | Chompass step reused | Writes |
|---|---|---|
| a. Welcome: no-network statement, where data lives, the medical boundary | WELCOME, DISCLAIMERS (merged) | Nothing |
| b. About you: height, weight, optional body fat (method picked for that reading, any available method), optional birth year | HEIGHT_WEIGHT, BODY_FAT, BIRTHDAY (made optional); GENDER kept optional | `user_profile`, first `body_measurement` |
| c. Targets: manual kcal and protein; adaptive suggestion explained and off until enough data | GOAL, GOAL_SPEED, BUILDING_PLAN, PLAN_READY (simplified) | First `target_set` (mode manual) |
| d. Eating window: usual window, midnight day boundary, 20 kcal threshold | New | `app_setting` |
| e. Food exclusion tags: pork, salmon, high-parasite foods, dairy except yoghurt and whey; notice toggle | New | Exclusion tag list |
| f. Supplements: name, dose, schedule, nutrients per dose (blank means unknown) | New | `supplement` rows |
| g. Watch list: none by default | New | `app_setting` |
| h. Export folder: pick the SAF folder or skip (manual share stays available) | New | Folder URI, state |
| i. Go-to meals: build or pin the usual meal | New (uses saved meals) | Saved meal, pinned |
| j. Summary | PLAN_READY (adapted) | Nothing |
| Removed | PROVIDER (AI key), HEALTH_CONNECT, ACTIVITY (kept only as PAL input inside Targets if wanted), DIET_MODE, GOAL_WEIGHT, HOME_LOOK, NOTIFICATIONS (moved to Settings) | |

**Done-when criteria (Phase F4):** the app is fully usable after skipping every step; re-running creates no duplicates; each step writes only what it lists; the "export folder lost" state shows; no step shows medical advice.

---

## 10. Phased build plan

Each phase is one branch and PR series with CI green before merge. Hours are estimates.

### Phase F0: fork, rename, strip the network, green build (10 to 16 hours)

Tasks:
- Fork at a recorded commit; write `docs/UPSTREAM.md`.
- Rename package, application id, app name, icons; new signing key.
- Remove the network subsystems in section 4.2, their settings, onboarding steps and tests. Remove the `okhttp` dependency and `RECORD_AUDIO`, Health Connect permissions and dependency, and the INTERNET permission. Set `allowBackup="false"`.
- Remove `web/`, `website/`, PWA parity scripts, F-Droid metadata.
- Add a GitHub Actions workflow: build, unit tests, and a manifest check.
- On the phone: confirm search over the bundled USDA index works (FTS5 on LineageOS), barcode decode on a real EAN-13, and the app survives a reboot with data intact.

**Done when:**
- `apkanalyzer manifest permissions` (or `aapt dump permissions`) on the release APK lists no `android.permission.INTERNET`, no `RECORD_AUDIO` and no `health.*` permission, and CI fails if one appears.
- `allowBackup` is false in the merged manifest.
- `grep` finds no `okhttp`, `java.net.URL` or `HttpURLConnection` use in `app/src/main`.
- All remaining tests pass; removed features' tests are deleted, not skipped.
- The owner can log food daily with manual entry, saved meals, USDA search and barcode-to-manual, and Chompass's diary JSON export works by manual share. **This is the daily-use milestone.**

### Phase F1: identity, nutrient map, registry, backup (12 to 18 hours)

Tasks:
- Add `nutrients` and `nutrientQual` maps, `localDate`, `utcOffsetMin`, `tzId`, `origin`, `recordingMethod`, `calcVersion` to `FoodEntry`, all with defaults. Backfill `localDate` for entries logged since F0.
- Make the revision ledger always on; add `rev` and `created_at`; generate `device_id` once.
- Seed `nutrient_def` from an asset with the typed-field mapping and UK values marked unverified.
- Draft `contracts/dietm8-export-1.0.schema.json`.
- Basic **Backup now** (`backup.json` in the envelope, manual share) and Restore with idempotent upsert, reusing `SyncMerge`.

**Done when:** decode tests read fixtures from every earlier version; a DST test covers 25 October 2026 (25-hour day) and 29 March 2027; restoring a backup onto a populated install creates no duplicates; the typed-field mapping is tested both ways and never duplicates a key.

### Phase F2: food databases (12 to 18 hours)

Tasks:
- Run the OFF UK and CoFID completeness measurement; commit the table.
- Write `build_cofid_food_index.py` and `build_off_uk_food_index.py`; widen and rebuild the USDA index with SR Legacy; reproducible manifests.
- Add `CofidFoodIndex` and `OffUkFoodIndex`; set source order; Swiss off by default (or removed).
- Barcode lookup against the OFF UK index; misses open the custom-food form.

**Done when:** builds are reproducible from scripts and recorded source versions; unit conversion tests pass (OFF grams to mg and µg, vitamin A and folate forms, "Tr" and "N"); logging a CoFID food snapshots the full vector with unknown left blank; APK size is recorded and accepted by the owner; search returns results in a time the owner accepts on the OnePlus 12.

### Phase F3: fast logging, DietM8-style (14 to 24 hours)

Tasks:
- Rule-based parser with preview card and the owner's phrase fixture; `FragmentResolver` no-op.
- Assistant groundwork (section 5.6): `PreviewEntry` and `CommitPreview` use cases with the preview cache, 5-minute expiry and idempotent commit keyed on `preview_id`; the in-app preview card calls the same use cases (about 2 to 4 hours).
- Pinned one-tap meals with time-of-day ranking; saved-meal versioning; quick-add; copy yesterday; undo snackbar.
- Exclusion tags on foods and meals.
- "Mark day complete" (`day_status`).

**Done when:** on the owner's phone the go-to meal takes 1 tap (2 with a tweak), quick-add 3 actions or fewer, search-and-add under 10, measured and recorded; the parser passes its fixture file with the pass rate recorded and unresolved fragments returned as data; a unit test shows preview then commit writes once, a repeated commit writes nothing new, and an expired preview is refused.

### Phase F4: targets, window, supplements, body, onboarding (18 to 27 hours)

Tasks:
- `TodaySummary` use case with the template sentences, coverage wording and `assistant-card.v1` output; `BodyPreview` validation (about 2 to 3 hours, section 5.6).
- Adaptive goals changed to suggest-only, with `adaptive_target_run`, event flags, and the back-test harness comparing Chompass's forecast with the DietM8 algorithm.
- `target_set` from the goal journal; g/kg presets on trend weight; day types.
- Derived eating window with override and `fast_before_min`.
- Supplements with per-dose panels and a tick list.
- Hide toggles and pause mode.
- Onboarding steps a to j (section 9).

**Done when:** the back-test reports kcal-per-day error and days to reconverge with parameters in `params_json`; gating freezes updates on missing data and event flags; the suggestion never replaces the manual target (tested); eating-window tests pass (coffee below 20 kcal, snack across midnight); onboarding criteria in section 9 pass.

### Phase F5: nutrient-gap view and grading (10 to 16 hours)

Tasks:
- Insights screen: 14-day grades, contributing foods, data-problems list (foods contributing the most energy with missing values, from `FoodEntryGaps`).
- `docs/rules.md` and golden vectors (shared later with HealthM8).
- Supplement-only versus NHS guidance shown separately.
- Source and verify the UK reference values; set `ref_verified_on`.

**Done when:** golden vectors pass for complete, sparse, supplement-heavy and low-coverage weeks; no screen says "deficient" or shows red; grey appears under 70% coverage; CI fails on a reference row without a source or verification date; the boundary text is on Insights.

### Phase F6: full export, SAF folder, WorkM8 PR (10 to 15 hours)

Tasks:
- Assistant contract files: `assistant-card.v1.schema.json`, `assistant-capabilities.v1.json`, fixtures in `docs/fixtures/assistant/`, schema validation in CI (about 2 to 3 hours).
- Full export zip (section 5.4) via WorkManager into a persisted SAF folder (`ACTION_OPEN_DOCUMENT_TREE` plus `takePersistableUriPermission`), one immutable file per export, per-month subfolders, the "folder lost" state.
- Syncthing-Fork (send-only) to Syncthing on the Umbrel (receive-only), tested end to end including an overnight idle run.
- Schema validation of sample exports in CI; publish sample exports to `docs/fixtures/` for HealthM8.
- The WorkM8 PR (section 5.5, about 3 to 4 hours of this phase).

**Done when:** an export validates against its schema and a round-trip import into a fresh install reproduces identical entries; CSV headers carry units and nothing is dropped; the WorkM8 sample export validates against the shared envelope; WorkM8 restores its own legacy backup unchanged; the sync path works with the real export, or a recorded fallback (manual share) is chosen; the assistant fixtures validate against their schemas.

### Phase F7: wire the assistant (optional, after the assistant's V1 is stable; 10 to 16 hours)

Starts only when the Sovereign Assistant's V1 runs on the phone and the Hark fork builds with the owner's signing key. Most of the work is on the DietM8 side; the Hark-side changes belong to the assistant plan.

Tasks:
- `oacp.json`, the `<applicationId>.oacp` provider and exported receivers and one activity, all behind `app.dietm8.permission.ASSISTANT` (signature); the OACP Android SDK if it is on Maven Central by then, otherwise a hand-written equivalent of the same three classes.
- Result broadcast addressed to the assistant package and protected by the permission; the `getSentFromPackage()` fallback if needed.
- Confirm that the Hark fork's dispatch can attach the permission and read the result; record the OACP version and the Hark commit used.
- Spoken-phrase fixture: 30 to 50 of the owner's real sentences run through the real voice path, with pass rate and the unresolved list recorded.
- CI check of the exported-component list and the unchanged permission list.

**Done when:** "log two eggs and toast" shows a confirm card, one OK writes one entry (checked in DietM8), a second OK or a replay writes none; "weight 82.4" does the same; "how much protein have I got left" returns the card sentences and none of the assistant's memory, log or synced files contain the figures (checked by searching them); a different app signed with another key cannot read or write through any of the receivers (tested with a throwaway app); an unresolved phrase opens DietM8's quick-log screen prefilled; the release APK still has no INTERNET permission.

### Totals (estimates)

| Phase | Hours |
|---|---|
| F0 Fork and strip (daily-use milestone) | 10 to 16 |
| F1 Identity, nutrient map, backup | 12 to 18 |
| F2 Food databases | 12 to 18 |
| F3 Fast logging (includes assistant groundwork) | 14 to 24 |
| F4 Targets, window, supplements, body, onboarding (includes `TodaySummary`) | 18 to 27 |
| F5 Gap view | 10 to 16 |
| F6 Export, SAF, WorkM8 PR (includes assistant contract files) | 10 to 15 |
| **Total F0 to F6** | **86 to 134** |
| F7 Assistant wiring (optional, later) | 10 to 16 |

The assistant groundwork adds about 6 to 10 hours to F0 to F6 (it was 80 to 124 before the assistant interface).

The from-scratch DietM8 plan estimated about 72 to 110 hours for the same scope. The fork does not save total hours by my estimate, because stripping and adapting a large codebase costs about what building the logging UI would. It does deliver a daily-usable app after F0 instead of after about 50 to 76 hours, and it inherits polished features the from-scratch plan deferred or omitted: camera barcode, widgets, recipes and meal components, body measurements, progress charts, water and caffeine trackers, draft recovery, and a large test suite.

---

## 11. Engineering standards

- **CI gates (added in F0):** ktlint (Chompass's `.editorconfig` scope) via spotless, detekt with a generated baseline for existing code (new code must pass with no new baseline entries), Android lint with a baseline, unit tests, the manifest permission check, JSON Schema validation of sample exports, and the decode-fixture test.
- **Branch and PR workflow** as in WorkM8. One concern per PR. Signed release APKs on GitHub Releases for Obtainium. Pin dependencies (including zxing-cpp and LiteRT-LM if kept).
- **Tests:** keep Chompass's Robolectric, Compose and screenshot tests for retained features; add parser fixtures, golden vectors, adaptive back-test, unit conversion tables, export round-trip and restore idempotency, DST tests.
- **Rules:** never destructive for user data (with `PersistedJsonGuard`); the read-only food-index exception; locale-safe decimals; snapshot, never recompute history from live indexes; no analytics, crash reporting or network code.
- **Docs in the repo:** `docs/UPSTREAM.md`, `docs/rules.md`, `contracts/` schemas, `docs/data_dictionary.json`, `docs/spike-results.md` (phone checks), this decision log, and updated `docs/ASSET_CREDITS.md` and `docs/NOTICE.md`. Write each phase's Claude Code prompt from its "Done when" list.

---

## 12. Decision log

IDs are DietM8-specific (DM-n); the master plan ID is in brackets where one exists.

| # | Decision | Rationale | Status |
|---|---|---|---|
| DM-1 [D1] | Native Android, no INTERNET permission | Owner decision | Fixed |
| DM-2 [D2] | Replace MyFitnessPal, start fresh | Owner decision | Fixed |
| DM-3 [D3] | Build order: logging, gaps, export | Owner decision | Fixed |
| DM-4 [D4] | Metric units only | Owner decision | Fixed |
| DM-5 [D5] | Data path v0 manual share; target SAF folder plus Syncthing; no HTTP push | Keeps the offline claim checkable | Recommended |
| DM-6 [D6] | Bundled layered food data, built by script, replaced not migrated | Offline, stable history | Recommended |
| DM-7 [D7] | OFF for macros and barcodes, not micronutrients | UK labels rarely list them | Recommended, rate unmeasured |
| DM-8 [D8] | Snapshot nutrients into each entry; unknown is no value | Stable history, honest gaps | Recommended (already true in Chompass) |
| DM-9 [D9, superseded] | Storage: Chompass's typed fields plus a registry-keyed `nutrients` map, in JSON buckets (replaces Room hot columns plus `log_nutrient` rows) | Owner decision, 26 Oct | Fixed |
| DM-10 [D10] | Ids, `rev`, tombstones, full exports with idempotent upsert; existing v4 ids kept, new ids v7 | Safe re-import | Recommended |
| DM-11 [D11] | DietM8 owns bodyweight and body fat; WorkM8 v1.1 log dropped | One series | Owner decision |
| DM-12 [D12] | Adaptive suggestion on the phone only, never overwrites manual targets | Owner control | Recommended; parameters unsourced |
| DM-13 [D13] | Gap view never shows red | LRNI and EAR not sourced | Recommended |
| DM-14 [D21] | No LLM in v1; `FragmentResolver` no-op | Owner decision | Fixed |
| DM-15 [D22] | Health Connect removed; `HealthSource` interface for later | Availability on LineageOS unverified | Recommended |
| DM-16 [D23, revised] | Camera barcode via Chompass's CameraX plus zxing-cpp from day one; no ML Kit | Already built | Recommended |
| DM-17 | Name DietM8, id `app.dietm8` | Owner decision | Fixed |
| DM-18 | Bundled food data with manual add for misses; no food API | Owner decision | Fixed |
| DM-19 | Day ends at midnight; eating window counts entries of 20 kcal or more | Owner decision | Fixed |
| DM-20 | Food exclusion tags from the start | Owner decision | Fixed |
| DM-21 | No default watch list; all nutrients shown equally | Owner decision | Fixed |
| DM-22 | Body fat method is chosen per reading, whatever method is available at the time (owner clarification, 26 Oct). Keep Chompass's `BodyFatMethod` (DEXA, 8-electrode BIA, 4-electrode BIA, tape; each with accuracy, repeatability and recency settings, V) and add an "other / unknown" value. The method is stored on every reading and exported. | Owner decision | Fixed |
| DM-23 [D38] | Skippable, re-runnable onboarding that collects supplements | Owner decision | Fixed |
| DM-24 | Build DietM8 as a hard fork of Chompass; cherry-pick upstream selectively | Owner decision, 26 Oct | Fixed |
| DM-25 [supersedes Room in the stack] | Keep Chompass's JSON bucket store and DataStore | Owner decision, 26 Oct | Fixed |
| DM-26 | Keep the 23 typed micronutrient fields and add a registry map | Owner decision, 26 Oct | Fixed |
| DM-27 | Remove all network subsystems in F0, including on-device model download | No-INTERNET rule | Recommended |
| DM-28 | Keep the on-device LLM engine switched off, loading a hand-copied model later | Reuse for "LLM-assisted later" | Recommended; cost unmeasured |
| DM-29 | Keep USDA; add CoFID and OFF UK; Swiss off by default or removed | UK-first | Recommended |
| DM-30 | Keep Chompass's diary/Markdown/CSV exports as extras; the envelope export is the only HealthM8 contract | One contract | Recommended |
| DM-31 | CI adopts WorkM8 standards using baselines | Green from day one | Recommended |
| DM-32 | Assistant interface: contracts, use cases and tests in the initial build; wiring in F7 after the assistant's V1 | Owner decision, 26 Oct | Fixed |
| DM-33 | Voice features in the first interface: log food, log weight (with optional body fat plus method), read today's numbers. No history, gap, supplement or exclusion queries | Owner decision, 26 Oct; keeps the medical-boundary surface small | Fixed |
| DM-34 | Every write is preview, assistant confirm card, then commit; commit is idempotent | Owner decision, 26 Oct | Fixed |
| DM-35 | Only the owner's own signed assistant build may call DietM8 (signature permission), reads included | Owner decision, 26 Oct; OACP trust model undocumented | Fixed |
| DM-36 | DietM8 parses the raw sentence; optional `items[]` hints reserved for later | Owner asked for trade-offs; plan default, reversible because the contract allows both | Recommended |
| DM-37 | Assistant payloads from DietM8 are tagged `sensitivity: health`; the assistant must not persist or forward them | Matches the assistant's own rule that health data is never stored automatically | Recommended (needs a matching line in the assistant plan) |

---

## 13. Open decisions and risks

### Open decisions

| # | Question | Needed by |
|---|---|---|
| OD-1 | Keep or remove the Swiss index (licence unchecked, low UK relevance). | F2 |
| OD-2 | Keep or remove nicotine, water and keto features (kept off by default otherwise). | F0 |
| OD-3 | OFF UK index size: accept the APK size, trim columns, or ship it as a separate file copied by hand. | F2 |
| OD-4 | Keep the on-device LLM engine in the repo, or remove it and reintroduce later. | F0 |
| OD-5 | Confirm DM-36 (DietM8 parses raw text) after seeing the parser's pass rate on the spoken-phrase fixture; switch to hybrid if it is poor. | F3, F7 |
| OD-6 | Whether the Hark fork can send OACP broadcasts with a permission and read protected results, or DietM8 must fall back to `getSentFromPackage()`. | F7 |
| OD-7 | Whether WorkM8 and DietM8 share one signing key (needed for one permission to cover both apps). | F0 |

### Risks

| Risk | Mark | Mitigation |
|---|---|---|
| Stripping 13,000+ lines of network code breaks hidden dependencies in a 107,000-line app. | (I) | Remove one subsystem per PR; keep tests green after each; Maestro smoke flow on the phone. |
| JSON store performance as history grows (no SQL, in-memory month caches). | (U) | Month buckets limit write cost (V); measure load time after a year of synthetic data in F1. |
| Revision ledger stored as one DataStore map grows with every entity. | (U) | Move to a bucket store if F1 measurement shows slow writes. |
| Vitamin A and folate form ambiguity in Chompass's typed fields. | (U) | Record form in the mapping; never sum different forms. |
| OFF UK micronutrient completeness unmeasured. | (U) | F2 measurement; OFF used for macros and barcodes. |
| UK reference values partly incomplete and model-extracted. | (U) | Verify against official publications; CI requires sources. |
| Adaptive parameters unsourced. | (U) | Back-test; suggestion only. |
| SAF plus Syncthing-Fork untested on Android 14/15 and OnePlus battery handling. | (U) | F6 end-to-end test; manual share fallback. |
| FTS5 on LineageOS system SQLite. | (U) | F0 phone check (Chompass already ships FTS5 indexes). |
| Upstream divergence makes later Chompass fixes hard to take. | (I) | Hard fork policy; record upstream commit per cherry-pick. |
| Single-maintainer upstream (11 stars at time of reading). | (V) | DietM8 does not depend on upstream after the fork. |
| Medical misuse of the gap view. | (U) | Boundary text, wording rules, no red. |
| OACP is a v0.3 preview, Hark had no releases when read, and the Android SDK was not yet on Maven Central. | (V) | Contracts and use cases do not depend on OACP; only F7 does, and F7 is optional and late. |
| OACP's trust model and result addressing are undocumented; an exported receiver can be called by any app. | (U) | Signature permission, explicit-package results, throwaway-app test in F7. |
| Signature permission needs the same signing key for DietM8 and the Hark fork; grant order after installs is quirky. | (U) | Use one key (OD-7); test install orders on the OnePlus 12. |
| The assistant stores or forwards a health payload (transcripts, synced memory, a context that also has web search). | (U) | `sensitivity` tag, F7 search of the assistant's files, requirements recorded for the assistant plan. |
| A mis-transcribed sentence logs the wrong food. | (I) | Preview card shows the parsed items before commit; commit is idempotent; entries stay editable in DietM8. |

---

## 14. Sources

- Chompass repository (GitHub mirror): https://github.com/fitguyfitguy/Chompass (commit `3cf301d`, 23 Oct 2026); upstream on Codeberg.
- CoFID: https://www.gov.uk/government/publications/composition-of-foods-integrated-dataset-cofid
- USDA FoodData Central downloads: https://fdc.nal.usda.gov/download-datasets
- Open Food Facts data: https://world.openfoodfacts.org/data
- UK Government Dietary Recommendations 2016: https://assets.publishing.service.gov.uk/government/uploads/system/uploads/attachment_data/file/618167/government_dietary_recommendations.pdf
- BNF nutrient requirements: https://www.nutrition.org.uk/nutritional-information/nutrient-requirements/
- ISSN protein position stand: https://pubmed.ncbi.nlm.nih.gov/28642676/
- Morton et al. 2018: https://bjsm.bmj.com/content/52/6/376
- Mifflin et al. 1990: https://pubmed.ncbi.nlm.nih.gov/2305711/
- Android SAF documents and files: https://developer.android.com/training/data-storage/shared/documents-files
- Hark (OACP reference assistant): https://github.com/OpenAppCapabilityProtocol/hark
- OACP protocol and README: https://github.com/OpenAppCapabilityProtocol/oacp
- OACP getting started: https://github.com/OpenAppCapabilityProtocol/oacp/blob/main/docs/getting-started.md
- OACP Android SDK: https://github.com/OpenAppCapabilityProtocol/oacp-android-sdk
- Sovereign Assistant V1 spec (owner's document): https://claude.ai/artifact/L52JmXrjJVkd723KzcuJmP
- Syncthing-Fork on F-Droid: https://f-droid.org/packages/com.github.catfriend1.syncthingfork/
