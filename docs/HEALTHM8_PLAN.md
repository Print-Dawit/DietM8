# HealthM8 plan: a local-first health dashboard on the Umbrel

**HealthM8 is a small Python web app packaged for the Umbrel. It reads the exports that DietM8, WorkM8 and Gadgetbridge write on the phone, joins them by local date, and shows weight, training, nutrition and sleep together.** The phone sends files to the Umbrel with Syncthing (Option 1). If a one-afternoon test shows that route does not work, the plan switches to a receiver inside HealthM8 fed by Termux (Option 2). The Umbrel stays the always-on hub: it computes everything, serves the dashboard, runs the read-only MCP server with sovereignty tiers, and produces the weekly report. A small snapshot of results is copied back to the phone, so a native viewer can be added later for widgets and offline use without calculating anything twice. The laptop model only adds language to the weekly report when the laptop is awake. Per the owner's decision, no summary derived from personal data is ever sent to an API.

**Evidence legend.** **(V)** a fetched source states it (links given; most were read through a model-written extraction, so spot-check any number before hard-coding it). **(I)** my design inference or recommendation. **(U)** unverified or an unsourced starting point. Hour figures are my estimates. Nothing here is medical advice (see the boundary text).

**Scope.** This plan covers the dashboard, importer, analytics layer, MCP server, transport and weekly summary, plus a small "assistant answers" interface (section 9.5) that lets the owner's Sovereign Assistant ask "how was my sleep this week" and get a spoken-ready answer, live over Tailscale or from a snapshot on the phone. DietM8 (the Android app) has its own plan and owns the export schema; this plan only consumes it.

---

## 1. Key recommendations

| # | Recommendation | Why | Status |
|---|---|---|---|
| 1 | Transport: Option 1 first (Syncthing-Fork Send Only on the phone, Syncthing on the Umbrel Receive Only, with an umbrelOS 2.0 custom folder shared read-only into HealthM8). If the H0 afternoon test fails, switch to Option 2 (receiver inside HealthM8 plus Termux on the phone). | No new apps for Option 1. Option 2 keeps working if Umbrel changes folder sharing. | Owner decision (26 Oct) |
| 2 | Garmin Forerunner 165 data comes from **Gadgetbridge only**. The Garmin Connect bulk archive importer is dropped from v1. | Daily, passive, no Garmin cloud. Owner chose to switch fully, accepting that automatic Strava upload via Garmin Connect ends. | Owner decision |
| 3 | The Umbrel is the hub: importer, analytics, dashboard, MCP and the weekly report all run there. A snapshot of derived daily results syncs back to the phone. A native phone viewer comes last and only if wanted. | Agents and the laptop need an always-on source. The phone is not always reachable. Calculations stay in one place. | (I) |
| 4 | Build the dashboard on the laptop against sample files first, then package it. | Debugging on the shared Umbrel (Home Assistant and CCTV also planned) is slower. | (I) |
| 5 | One Python app: FastAPI plus SQLite, one container behind `app_proxy`. | One language for importer, analytics, dashboard and MCP. | (I) |
| 6 | Shared analytics module `healthcore` with golden vectors shared with DietM8. | Phone and Umbrel must grade nutrients identically. | (I) |
| 7 | Read-only MCP on the Umbrel with three sovereignty tiers and an audit log. No cloud consumer is built. | The server, not the client, must enforce sovereignty. | (I) |
| 8 | The weekly deterministic report is always produced on the Umbrel. The laptop model adds wording only; the owner moves the summary file by hand. | The laptop sleeps, and the owner prefers not to install more on it. | Owner decision |
| 9 | Show every import: an Inbox page with file name, size, SHA-256, arrival time, and a warning if no Gadgetbridge export arrives for 26 hours. | Turns "did it arrive?" into something visible. | (I) |
| 10 | Assistant answers: HealthM8 produces fixed-template `assistant-card.v1` answers (sleep, HRV and resting heart rate, weight trend, intake against weight, weekly headline; 7 and 14 days). The same cards are served live on a separate token-protected Tailscale port and written to a snapshot file on the phone. The initial build delivers and tests both with `curl` and a script; the assistant is wired to them later. | Owner decisions (26 Oct): ask HealthM8 insights by voice; answer from the phone's last snapshot when away; contracts now, wiring later. No language model touches the numbers in the initial build, which satisfies either model-boundary option the owner is leaning towards. | Owner decision |

---

## 2. Context and constraints

| Area | Constraint or decision | Status |
|---|---|---|
| Home server | Umbrel Pro, umbrelOS 2.0 (released 22 Sep 2026), i3-N300, 16GB RAM, no GPU. Tailscale and Syncthing already in use. Home Assistant and CCTV also planned on the same box. | Owner |
| Phone | OnePlus 12, degoogled LineageOS, apps via Obtainium and F-Droid. | Owner |
| Laptop | ASUS Vivobook Pro 15: AMD Ryzen 9, 16GB RAM, NVIDIA RTX 3050 with 4GB VRAM plus AMD Radeon integrated graphics, Windows, Tailscale installed. Prefers not to install more software. Not always on. | Owner |
| Dashboard | A web app on the Umbrel named HealthM8. Phone use through an installable page first; a native viewer later if needed. | Owner |
| Inputs | DietM8 exports, WorkM8 exports (joined by local date; apps stay separate), Gadgetbridge export from a Garmin Forerunner 165. Blood tests later. | Owner |
| Strava | The owner switches fully to Gadgetbridge; automatic Strava upload through Garmin Connect ends. | Owner |
| v0 pages | Weight trend vs calories; training volume vs intake; nutrient gaps with data quality; sleep, HRV and resting heart rate. Plus additions in section 8. | Owner |
| Weekly summary | A page on the dashboard. Summary file moved to the Umbrel by hand. | Owner |
| LLM | Local model on the laptop (Ollama or LM Studio). Rule-based and deterministic first. | Owner |
| Sovereignty | Personal-data-derived summaries never go to an API. An API is only for generic tasks with no personal data. | Owner |
| MCP role | Health-data access for AI agents, not food lookup. | Owner |
| Voice assistant | The owner's Sovereign Assistant (Hark fork, Flutter, on the phone; Ollama on the Umbrel; Tailscale only) should be able to answer HealthM8 questions by voice. Away from home it answers from the phone's last snapshot and says how old it is. Health data may reach on-device or Umbrel models at most, never a cloud API, never assistant memory (owner leaning towards the Umbrel-plus-on-device or on-device-only option). DietM8 first, assistant later. | Owner (26 Oct) |
| Build | Claude Code on the laptop. No deadlines. | Owner |

---

## 3. The export contract as HealthM8 consumes it

DietM8 owns the schema; HealthM8 restates only what it relies on. The source of truth is DietM8's `contracts/dietm8-export-1.0.schema.json` and its `docs/rules.md`.

- **Envelope:** `envelope_version`, `export_id`, `app` (id, name, version), `schema` (name, version), `exported_at`, `timezone_at_export`, `device_id`, `export_kind` (full), `date_range`, `day_boundary`, `units`, `counts`, `files` (path, entity, rows, sha256). Data sits under `data`.
- **Entity rules:** every record has `id`, `rev`, `created_at`, `updated_at`, `deleted_at` (tombstone) and `origin_device_id`. Chompass-derived ids may be UUIDv4 and new ones UUIDv7, so treat ids as opaque strings. A missing nutrient key means unknown, not zero. `kcal` equals `nutrients.energy_kcal`. Readers ignore unknown fields and enum values, refuse a higher MAJOR schema version and warn on a higher MINOR.
- **Nutrient keys carry units** (`vitamin_d_ug`). The export includes `nutrient_def` with UK reference values and the `ref_verified_on` date.
- **Packaging:** one immutable zip per export (`manifest.json`, `backup.json`, per-entity CSVs, `daily_totals.csv`, `daily_nutrients_long.csv`, `data_dictionary.json`).
- **WorkM8:** will adopt the same envelope through a separate PR in the WorkM8 repo (DietM8 plan, section 5.5). Until then HealthM8 reads WorkM8's existing JSON backup and CSV, treating the unwrapped shape as version 0.
- **Grading rules restated from DietM8** (source of truth stays in DietM8): per-nutrient coverage is the share of logged energy from entries that have a value; green at 90% or more coverage and at least 5 of 7 days logged for a low-estimate nutrient; amber for 70 to 90% or a high-estimate nutrient; grey under 70%. Never red, never "deficient".
- **Boundary text (required in every LLM schema and on the dashboard footer):**

> This tool compares what you logged with general UK reference values for healthy adults. It is not medical advice and cannot diagnose a deficiency. Logged intake can be wrong where food data is incomplete or foods go unlogged. Reference values are population-level, not personal requirements, and NHS "unlikely to cause harm" figures are not safe-dose recommendations. Speak to a GP, pharmacist or registered dietitian before changing your diet or supplements, and about symptoms, abnormal blood results, or any supplement above NHS guidance.

---

## 4. Gadgetbridge and the Forerunner 165

### 4.1 What the sources say

- Gadgetbridge is an Android app (AGPLv3) that uses Bluetooth devices without the vendor app, with no account and no vendor servers ([Codeberg](https://codeberg.org/Freeyourgadget/Gadgetbridge)). (V)
- Its Garmin page lists the Forerunner 165 and 165 Music as supported. Pairing: put the watch in pairing mode, choose "Pair phone", ignore the QR code, and pair from Gadgetbridge ([Garmin devices](https://gadgetbridge.org/basics/topics/garmin)). (V)
- Garmin data it syncs: heart rate, resting heart rate, SpO2, stress, respiratory rate, HRV values and summary, "body energy" (Body Battery), steps, distance, calories, intensity minutes, sleep times and stages with score (on devices that provide them), workout activity files, training load and race-time prediction, daily hydration. Skin temperature and the changelog are not parsed. (V)
- Downloaded FIT files are saved in the phone's internal storage. The documentation does not describe exporting them. (V)
- The database exports as one SQLite file named `Gadgetbridge` (no extension). Manual "Export zip" produces a full backup; GPX activities and preferences export as separate files ([data management](https://gadgetbridge.org/internals/development/data-management/)). (V)
- Auto export (Settings, Automations, Auto export) can schedule database and ZIP exports to a chosen folder and be triggered by the broadcast `nodomain.freeyourgadget.gadgetbridge.command.TRIGGER_EXPORT`, announcing `DATABASE_EXPORT_SUCCESS` or `DATABASE_EXPORT_FAIL` ([auto export](https://gadgetbridge.org/internals/automations/auto-export/), [intents](https://gadgetbridge.org/internals/automations/intents/)). The database export overwrites one file with the same name. (V, from the research run)
- **Not found:** real-world reports of the Forerunner 165 working with Gadgetbridge (a GrapheneOS forum thread asked the question and has no replies). Support is listed but unproven. (U)
- Pairing caveats from the docs: some devices fail to pair unless they were connected to the official app once; "Send fake OAuth responses" can invalidate the watch's credentials and may need a factory reset if you later return to Garmin Connect; AGPS and weather need authentication, with weather credentials refreshed every 90 days. (V)

### 4.2 What this means for the plan

- **Garmin Connect and Strava.** Garmin Connect cannot stay the main phone link, so automatic Strava upload ends. Activities still arrive from the Gadgetbridge database. If Strava history matters, upload FIT files by hand or skip it. A Strava importer is dropped from v1 (the Strava archive holds activities only, no sleep or heart rate, per [Strava support](https://support.strava.com/hc/en-us/articles/216918437-Exporting-your-Data-and-Bulk-Export)).
- **History before pairing** is not in Gadgetbridge. A one-off Garmin archive importer is an optional later phase (H7), not v1.
- **Exporter folder.** Gadgetbridge's default export location is under `Android/data`, which Android 11 and later blocks even with "All files access" (V, anexplorer and the Syncthing-Fork storage notes). Set the auto-export folder to a normal shared folder such as `Documents/HealthExport/Gadgetbridge`.
- **The export is a mutable file.** The database overwrites in place. The importer never opens it directly: it copies the file to a temporary name, opens the copy read-only, and records the SHA-256, so a file replaced mid-read does no harm. Because Syncthing writes to a temporary file and renames, a half-written file should not appear (I).
- **Schema.** The Gadgetbridge database layout was not retrieved. The importer reads table names at run time, parses defensively, keys rows by source plus device plus timestamp, and stores unknown columns untouched. Field names are unverified until the H0 spike reads a real export (U).
- **Privacy note.** Gadgetbridge's maintainers warn that an export folder in shared storage is readable by any app with storage access, so keep the number of apps holding "All files access" on the phone small ([issue](https://codeberg.org/Freeyourgadget/Gadgetbridge/issues/2159)). (V)
- **Sleep belongs to the wake-up date.** Join on `local_date` using the phone's offset at the time.

---

## 5. Architecture

### 5.1 Approach: Umbrel hub, phone snapshot, native viewer later

| Need | Approach |
|---|---|
| Quick glance on the phone | The dashboard is responsive and installable as a PWA from the Umbrel URL. |
| Works when away from the tailnet or offline | A daily snapshot file (derived daily totals, grades, weekly report) syncs back to the phone. A native viewer reads it (H6). Until then, the last page loaded is all that's cached (service worker behaviour on LineageOS unverified (U)). |
| Widgets and notifications | Native viewer only (Glance). Web push on a degoogled phone is unverified. |
| Agents and the laptop work while the phone is off | They talk to the Umbrel. |
| Code to maintain | One Python codebase; the viewer displays results only. |

The owner's answer of "phone has its own copy plus the Umbrel" is met by the snapshot, which copies derived results rather than duplicating analytics.

### 5.2 Overall diagram

```
 PHONE (OnePlus 12, LineageOS)                        UMBREL (CPU-only, 16GB, shared)
 ┌───────────────────────────────┐                    ┌──────────────────────────────────────────────┐
 │ DietM8 (no INTERNET)          │  Syncthing-Fork    │ Syncthing (Receive Only) -> Home/HealthInbox  │
 │ WorkM8 (no INTERNET)          │  Send Only         │            │ (custom folder, read-only into   │
 │ Gadgetbridge -> Forerunner 165│ ────────────────>  │            ▼  HealthM8)                       │
 │   auto-export to              │  over Tailscale    │ HealthM8 importer (quarantine, idempotent)    │
 │   Documents/HealthExport/...  │                    │            │                                  │
 │                               │                    │   SQLite (WAL, nightly .backup)               │
 │ Native viewer (later)         │ <────────────────  │            │ read-only role / views           │
 │   reads snapshot folder       │  Snapshot folder   │   healthcore -> dashboard, weekly report,     │
 └───────────────────────────────┘  Send Only         │                 snapshot writer, MCP server   │
                                                      └──────┬──────────────┬──────────────┬─────────┘
                                         Tailscale only, no   │              │              │
                                         Funnel               ▼              ▼              ▼
                                                      browser/phone   LAPTOP (when awake): weekly-summary
                                                                      job, MCP client, local model
```

Option 2 replaces the Syncthing box and arrow with an SFTP or WebDAV receiver inside HealthM8 and a Termux job on the phone (section 6.4).

---

## 6. Transport: Option 1, with Option 2 as the tested fallback

### 6.1 Phone-side requirement (applies to both options)

- Every exporter writes to a normal shared-storage folder such as `Documents/HealthExport/<app>/`, not to `Android/data`. Since Android 11 sync apps cannot read `Android/data`, and Android 13 and later also blocks SAF access to it ([storage permissions](https://anexplorer.io/solve/storage-permissions-android), [Syncthing-Fork storage notes](https://deepwiki.com/researchxxl/syncthing-android/9.1-storage-and-sd-card-issues)). (V)
- DietM8 writes through its SAF folder picker: pick `Documents/HealthExport/DietM8`. Gadgetbridge auto export: pick `Documents/HealthExport/Gadgetbridge`.
- WorkM8 exports only through manual share. A share-sheet target that saves into `Documents/HealthExport/WorkM8` is needed. I could not verify one on LineageOS (U). Until one is found, WorkM8 stays a manual step; the WorkM8 PR can add a SAF folder export later.

### 6.2 Option 1: Syncthing-Fork to Syncthing on the Umbrel (primary)

**How it works (V for the Umbrel feature, from Umbrel's own documentation; see unverified points).**
1. Exporters write into `Documents/HealthExport/...`.
2. Syncthing-Fork shares that folder as **Send Only**.
3. In umbrelOS 2.0, an app's settings page has Storage, "Access to your folders", Advanced, "Add custom folder". You choose the app service, a folder from Home, a USB drive or a network device, a path inside the app, and **Read-only** or **Read and write**. Files are not copied. A folder shared by two apps shows "Also shared with" ([Umbrel app settings](https://umbrel.com/support/apps/app-settings)).
4. Map `Home/HealthInbox` into the Syncthing app at `/var/syncthing/HealthInbox` as read and write, set the Umbrel side of the Syncthing folder to **Receive Only**, then map the same Home folder into HealthM8 as **read-only** at `/inbox`.
5. The importer reads `/inbox`.

**Settings that matter (I).**
- Phone folder Send Only, Umbrel folder Receive Only, no versioning, files never edited in place (except the Gadgetbridge database, handled in section 4.2).
- Switch off global discovery and relays on both ends and use static `tcp://100.x.y.z:22000` Tailscale addresses, so traffic never leaves the tailnet.
- Battery setting for Syncthing-Fork: Unrestricted. LineageOS lacks OnePlus's OxygenOS task killer, but I found no OnePlus-specific reports either way (U).
- Syncthing-Fork on the phone needs All files access.

**Trust point.** Syncthing-Fork changed hands in December 2025. The original developer later confirmed handing over his projects, apps and signing material to the new account, and F-Droid and the Syncthing developers reported no sign of malice, with builds reproducible. Releases have continued, the latest I saw being 2.1.6.0 on 12 Oct 2026 ([F-Droid](https://f-droid.org/packages/com.github.catfriend1.syncthingfork/), [Syncthing forum](https://forum.syncthing.net/t/does-anyone-know-why-syncthing-fork-is-no-longer-available-on-github/25661/251)). Pin the F-Droid build and keep the folders Send Only. (V)

**Unverified points (the afternoon test checks them).**
- Whether the Syncthing app's custom-folder option works for this folder, and whether its container (run as UID 1000 in the official package, per the compose file) can write to a Home folder without permission changes. (U)
- Whether a community app such as HealthM8 can be given the same Home folder read-only. (U)
- Reliability in the background on this phone. (U)

### 6.3 The H0 afternoon test (decides Option 1 or Option 2)

Run this on the Umbrel and phone before any other HealthM8 work.

1. **Phone:** create `Documents/HealthExport/`. Point Gadgetbridge auto export (database) at it and press "Run AutoExport now". Drop in a test DietM8 or other small file.
2. **Umbrel:** in Files create `Home/HealthInbox`. Open Settings, App settings, Syncthing, Storage, Advanced, Add custom folder: service Syncthing, folder `HealthInbox`, path `/var/syncthing/HealthInbox`, Read and write. Apply and restart.
3. **Syncthing:** in the Umbrel web UI add the phone's folder as **Receive Only** at that path. On the phone set it **Send Only**.
4. **HealthM8 stand-in:** install the "hello" test app from a private community store (section 7), add a custom folder `HealthInbox` mapped to `/inbox`, **Read-only**. From its terminal (Settings, Advanced, Terminal, App) run `ls -la /inbox && sha256sum /inbox/*` and compare with the phone.
5. **Edge cases:** switch the phone to mobile data with only Tailscale connected and export again; lock the phone for 2 to 3 hours with battery Unrestricted and confirm the next scheduled export arrives without opening any app; confirm `touch /inbox/x` fails inside the test app (proves the mount is read-only).

**Pass criteria for Option 1:** every file arrives with a matching SHA-256; the mount is read-only; the scheduled export arrives unattended; and the test app can read the folder after an Umbrel restart. **If step 2 fails, if no custom-folder option exists for Syncthing, if the test app cannot get the folder read-only, or if the scheduled sync does not arrive, switch to Option 2.** Write the result and the Android and umbrelOS versions into `docs/spike-results.md`.

### 6.4 Option 2: receiver inside HealthM8 plus Termux (fallback)

**Umbrel side (I).** Add a receiver service to HealthM8's own docker-compose. Choose one:
- an SFTP container with key-only authentication and a chroot into the inbox; or
- a small authenticated WebDAV or PUT endpoint written in Python.

It writes to `${APP_DATA_DIR}/inbox`, and the HealthM8 web service mounts that directory read-only. Publish the receiver on its own port (official apps publish ports directly where a protocol is not HTTP, for example Transmission's 51413 (V)). Keep machine uploads off the `app_proxy` path: it enforces the Umbrel login, and the only alternative is switching login off for the whole app. If a path is whitelisted in `PROXY_AUTH_WHITELIST`, treat it as public and add the app's own bearer token. Pin the image digest as official packaging asks. Do not use the `umbrel` SSH account from the phone: that gives the phone the admin password or a key with full shell rights, and I could not verify that an `authorized_keys` file persists across Umbrel restarts.

**Phone side.** Termux and Termux:API from F-Droid; `termux-setup-storage` for shared storage. A script runs `rsync -av --ignore-existing` over SSH, or `rclone copy` to WebDAV (never deletes on the remote), scheduled with `termux-job-scheduler --period-ms 3600000 --persisted true --network any`. This uses Android's JobScheduler. Termux's own cron may be killed by power management, so wake locks are advised for cron (a June 2025 community guide; URL not kept, re-check before relying on it). The script calls an `/ingest/ack` endpoint (or writes a log) so HealthM8 can show "last upload". rsync's `rrsync` can restrict the key to the inbox. rsync writes to a temporary name and renames, so the app never reads a half-written file. With immutable, uniquely named files and `--ignore-existing` there are no duplicates. Licences: Termux GPLv3, rsync GPLv3, rclone MIT.

**Costs.** Two new phone apps and a script; Termux's fit with modern Android is described as rough in places, and its background execution is not guaranteed.

**GUI alternative to Termux.** Round Sync (rclone for Android, GPL-3.0, F-Droid) offers scheduled tasks and SAF support, but its last official release was 2.5.6 in September 2024 and a forum post says it has not been updated in about 18 months. An unofficial fork ships newer rclone builds. Usable, not something to rely on for years. (V)

**Snapshot return path in Option 2.** The phone pulls a read-only snapshot directory from the receiver with `rsync` in the same Termux job, instead of Syncthing.

### 6.5 Other options considered and set aside

| Option | Reason |
|---|---|
| FolderSync, Resilio Sync, Autosync | Proprietary; FolderSync's own docs tie the Play version to a Google account and the licence costs money. |
| LocalSend, Taildrop, KDE Connect | Manual per-file transfer; LocalSend cannot receive unattended in the background; the Umbrel Tailscale app has no Taildrop landing folder. |
| Nextcloud on the Umbrel | Works without Google services, but is a heavy server for a few hundred KB a day, and its Android auto-upload had a 2026 reliability bug. |
| SFTP into umbrelOS with the `umbrel` account | Admin credential on the phone; unverified persistence. |
| Umbrel "Download from cloud" pulling from a phone WebDAV server | A phone is not an always-on server. |
| RSAF (rclone as an Android document provider, GPL-3.0, v4.15 on 20 Oct 2026) | Interesting for DietM8 writing straight to the Umbrel, but writes would fail when the Umbrel is unreachable. Untested; not recommended. |

---

## 7. Packaging HealthM8 for Umbrel

### 7.1 What a package is (V, from the official apps repo's skills)

- One folder named after the app id, holding `umbrel-app.yml` and `docker-compose.yml`. `exports.sh` and `hooks/` only when needed.
- `umbrel-app.yml`: `manifestVersion` (app-framework compatibility, default 1; `requiresHttps` needs 2.0 or higher), `id` (lowercase kebab-case, must match the folder, never changes), `name`, `version`, `port`, `category`, `storage: dataRoot: data`, listing fields.
- `docker-compose.yml`: one service per process; an `app_proxy` service configured by environment only (`APP_HOST` set to `<app-id>_<service>_1`, `APP_PORT`); Umbrel login stays on by default; durable state bind-mounted from `${APP_DATA_DIR}/data/...` with a committed `.gitkeep` for each empty source directory; no `build:`; no Docker socket, no privileged mode.
- Images: public, tagged, pinned by tag plus digest; official packages need `linux/amd64` and `linux/arm64`. The Umbrel Pro is amd64, so amd64 is the minimum for a personal store (I).
- Runtime: bind to `0.0.0.0` in the container, configurable host and port, non-root where possible, idempotent startup, a lightweight health endpoint, a browser first-run page, secrets generated into persistent storage at first run, nothing sensitive in logs.
- A community store is a GitHub repo made from Umbrel's template: `umbrel-app-store.yml` holds the store `id` (a prefix every app id must start with) and `name`; each app folder is named `<store-id>-<app-name>`. You add the repository URL in App Store, the three-dot menu, Community App Stores ([template](https://github.com/getumbrel/umbrel-community-app-store), [how to add](https://umbrel.com/support/apps/how-to-add-a-community-app-store)). Umbrel warns that community stores are unvetted.
- Official testing guidance: install on a real device, open the app through the Umbrel route (not a raw container port), create real state, restart, confirm persistence, then test an update.
- umbrelOS 2.0 adds a built-in MCP server for managing the Umbrel (Settings, AI agents) with revocable tokens. Scope any such token narrowly and revoke unused ones, because the health data sits in Umbrel storage. (V, I)

### 7.2 What was not verified

- The exact `manifestVersion` and field set for umbrelOS 2.0 community apps; read an existing app's `umbrel-app.yml` first. (U)
- Whether the Umbrel can add a **private** GitHub repo as a community store. Assume a public repo, so keep real data and secrets out of it. (U)
- Whether a unique `port` is enforced across community stores. Pick a high, unusual port. (U)
- The Python container details; no worked example of a Python Umbrel app was found. (U)
- Whether the official test container for umbrelOS runs on Windows (the docs mention WSL may not support it); the plan tests on the real Umbrel instead.

### 7.3 Build and release (I)

- Build the image with GitHub Actions, push to the GitHub container registry as a public package, tag by version, and record the digest in `docker-compose.yml`.
- Keep the app runnable outside the Umbrel (plain `docker compose` on the laptop) for development, with the host and port configurable.
- Packaging overhead on top of the app itself: about 6 to 10 hours (estimate).

---

## 8. Importer, storage and the dashboard

### 8.1 Importer (I unless marked)

- Polls `/inbox` every 60 seconds. Processes only complete `*.zip`, `*.json`, `*.csv` and Gadgetbridge database files whose size has been stable for a few seconds, and ignores `.sync-conflict-*`, `.stfolder`, `.stversions` and `~syncthing~*.tmp`.
- Validates DietM8 and WorkM8 files against the JSON Schema for the stated `schema.version`. On failure moves the file to `quarantine/` with an error sidecar and never deletes the original.
- Idempotent upsert keyed by (`app_id`, `entity`, `id`). A higher `rev` replaces; an equal `rev` with a later `updated_at` wins, with a warning; a tombstone wins over the same or a lower `rev`. Records the file SHA-256 in `import_log` and skips a file already seen (the same idea garmin-grafana uses, [garmin-grafana](https://github.com/arpanghosh8453/garmin-grafana)). (V)
- **Gadgetbridge importer:** copy-then-open-read-only (section 4.2); extract daily metrics (sleep, resting heart rate, HRV, steps, stress, Body Battery, intensity minutes) and activities; key by source, device and timestamp; assign sleep to the wake-up date; keep unknown columns raw.
- Joins the diet and workout daily tables on `local_date` (a left join from diet so rest days stay). Refuses to join exports whose `day_boundary` differs without a warning.
- Copies each processed file to an `archive/` folder outside the inbox. Never edits or deletes files in the inbox. The archive can rebuild the database.
- Backups: nightly `sqlite3 .backup` to a second location; the archive is the source of truth.
- Exposure: Tailscale and LAN only. No port forwarding, no Tailscale Funnel. Keep `app_proxy` login. Encryption at rest on the Umbrel was not researched (U): check whether umbrelOS 2.0 on the Umbrel Pro supports drive encryption before real data goes on it.

### 8.2 Inbox page (I)

Lists every file: name, source (DietM8, WorkM8, Gadgetbridge), size, SHA-256, arrival time, import result. Shows "last seen" per source and warns after 26 hours with no Gadgetbridge export. Doubles as the Option 1 and Option 2 health check.

### 8.3 Dashboard pages

**The owner's four v0 pages (all wanted):**

| Page | Contents |
|---|---|
| Weight trend vs calories | Faded raw weigh-ins over a smoothed trend, 7-day average intake, target and the adaptive suggestion records exported by DietM8 (display only, never recomputed). |
| Training volume vs intake | WorkM8 daily summary (sessions, sets, reps, volume in kg, duration) beside calories and protein, left-joined from diet so rest days remain. |
| Nutrient gaps with data quality | 14-day and 7-day grades (green, amber, grey), coverage, contributing foods, data-problems list. Never red. |
| Sleep, HRV and resting heart rate | From the Gadgetbridge import, with 7-day averages. |

**My suggested additions (I):**

| Page | Why |
|---|---|
| Weekly summary | The deterministic report, plus the laptop model's wording when the file has been moved in. This is the owner's chosen delivery route. |
| Sleep and eating window | Last meal time and fast length beside sleep and resting heart rate. Association only; no causal claims (the evidence for alcohol and HRV effects found was thin and vendor-run). |
| Data health | Inbox, quarantine, unlogged days, which streams are stale. |

Rules (I): lead with trends and 7-day averages rather than daily numbers; "days in the last 14 meeting target" instead of daily failures; no streaks or red colours; one vendored charting library and no CDN; layout works at phone width; installable as a PWA (manifest and service worker). Whether PWA installation and offline caching work well on the owner's browser on LineageOS is unverified (U); Chromium-based browsers install most smoothly.

### 8.4 Snapshot to the phone (H3)

The Umbrel writes `healthm8_snapshot_<date>.json` (envelope-wrapped: daily totals, nutrient grades, trends, weekly report text) into `Home/HealthSnapshot`. A second Syncthing folder (Umbrel Send Only, phone Receive Only) carries it to the phone. Contains tier 1 data on a device that already holds the raw data, so no new exposure is added. In Option 2 the snapshot is pulled by Termux. The same folder carries `assistant_cards.json` for the voice assistant (section 9.5).

---

## 9. Analytics layer, MCP and the three sovereignty tiers

### 9.1 Layering

`healthcore` is one Python module and the source of truth for aggregates, gap grading and coverage. The dashboard, a CLI (`health summary --week 2026-10-12 --json`), the weekly-summary job and the MCP server all call it. MCP is a thin adapter. No free-form SQL tool, because arbitrary SQL defeats redaction, scopes and rate limits. The weekly job does not go through MCP; it reads the internal read-only endpoint directly, so the pipeline controls the prompt and tool output cannot inject into it. Shared golden vectors (JSON inputs and expected statuses in `docs/rules.md`) must pass in both the Kotlin tests (DietM8) and the Python tests. (V)+(I)

### 9.2 MCP facts the plan relies on (V, from the spec pages and SDK READMEs; re-check versions when building)

- Latest spec revision 2026-07-28: stateless (no `initialize` handshake, no session id); Streamable HTTP is the remote transport; legacy HTTP+SSE deprecated ([MCP blog](https://blog.modelcontextprotocol.io/posts/2026-07-28/), [transports](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)).
- Authorization is optional; servers must validate `Origin` and bind to 127.0.0.1 locally; tools must validate inputs, enforce access control, rate limit and sanitise outputs; annotations are untrusted hints, so enforce read-only server-side ([tools spec](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)).
- The Python SDK v2 supports the revision ([python-sdk](https://github.com/modelcontextprotocol/python-sdk)). Pin exact versions. Client support for the stateless headers was not tested (U).

### 9.3 v1 tool surface (read-only, five tools, `health_` prefix, units beside values)

| Tool | Returns | Tier |
|---|---|---|
| `health_get_data_coverage` | Days with data and complete days per domain | 2 |
| `health_get_daily_nutrition` | Per-day energy, macros and requested nutrients with coverage | 1 |
| `health_get_weekly_nutrition_summary` | 7-day means, % of reference, status enum, coverage and caveats | 2 |
| `health_get_nutrient_gaps` | Gap objects | 2 |
| `health_get_body_composition_trend` | Weight and body fat trend with slope per week | 1 (2 if rounded) |

Later: `health_get_training_summary` and `health_get_sleep_summary` (tier 1) once ingestion is proven. A resource `health://methodology` documents the status enums, coverage and boundary text. Statuses: `below_reference_logged`, `near_reference`, `meets_reference`, `above_reference`, `insufficient_data`. `deficient` is not a permitted value. Aggregate tools return no free text. Free-text fields (food, supplement, exercise names) are an injection route, so raw tools, when added, strip control characters, truncate to about 60 characters and wrap the field as `untrusted_text`. Keep health tools out of agent sessions that also browse or send messages unless egress is blocked ("lethal trifecta", [Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)). (V)+(I)

### 9.4 Tiers

| Tier | Data classes | Policy | Who may receive it |
|---|---|---|---|
| **0: never exposed** | Direct identifiers, location, raw free-text notes, raw import files, credentials, second-level timestamps | In no view; reachable by no token; dashboard UI only. | Nobody via MCP |
| **1: local-only** | Blood results, sleep and heart-rate series, row-level logs, supplement brands and doses, eating-window timestamps, exact weight series | Token tier `local`; endpoint reachable only from tagged local hosts; free text stripped or tagged; sizes capped. | The owner's laptop model over Tailscale |
| **2: aggregate-only** | Weekly percent-of-reference with status and coverage, rounded weight slope, training totals | Rounded, relative dates, no names or brands, no blood results, no free text. | Reserved for any non-local consumer. **No cloud consumer is built in v1.** |

**Owner decision applied:** personal-data-derived summaries never go to an API. The earlier idea of a minted "cloud aggregate" session token is therefore not built. Tier 2 stays as defence in depth and as the payload shape for a possible future exception that would need an explicit new decision. The only permitted API use is for generic tasks that contain no personal data (section 10.5).

Controls (I): separate code paths and ports for tier 1 and tier 2, not a filter on one set of tools; 256-bit bearer tokens per client and tier, hashed at rest with expiry and rotation; `tailscale serve` only (never Funnel), the Umbrel tagged `tag:health-server` and the laptop `tag:health-client-local` with grants allowing only the MCP and dashboard ports ([Tailscale grants](https://tailscale.com/kb/1324/acl-grants); the example policy is untested (U)); rate limit and response-size cap per token (placeholders (U)); containers non-root, read-only filesystem, egress blocked, isolated Docker network, database role limited to SELECT on views (the views are the redaction boundary).

**Audit log:** append-only JSONL with timestamp, token id, tier, tool, arguments, status, bytes returned and payload SHA-256; tier-1 calls log the digest and field list only. A monthly script reports sessions, bytes and tools. The log is health data: restrict its permissions and include it in encrypted backups. Kill switches: per-tool disable and a token revocation list. Write a one-page data-flow register (what data, where, retention, who).

**UK GDPR note.** The ICO describes purely personal or household processing as outside UK GDPR, though its guidance is under review and does not address special-category data ([ICO](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/exemptions/a-guide-to-the-data-protection-exemptions/)). Giving access to others, building it for clients or commercialising it would change that, so take proper advice first. Use synthetic data in any demo. Whether the tool counts as a regulated medical device was not researched; personal use with no diagnostic claims is very likely out of scope (U).

### 9.5 Assistant answers (contracts in the initial build, wiring later)

**Why not give the assistant the MCP tools.** The MCP server is built for agent clients, with tiers 1 and 2, tool schemas and a client that supports the stateless headers (untested, section 9.2). The assistant on the phone is a Flutter app whose tool layer is its own; asking it to carry a full MCP client for five spoken questions adds risk and a second path to health data. So HealthM8 serves the assistant a much smaller thing: answer cards. (I)

**Answer cards.** `healthcore` builds an `assistant-card.v1` object (schema defined in the DietM8 repo and copied here, section 5.6 of that plan): `topic`, `period`, `as_of`, `sentences[]` from fixed templates, `figures[]` with ids, rounded values and units, `data_quality` (coverage and which stream is stale), `caveat`, `sensitivity: "health"`. No free text from the data (no food, supplement or exercise names). Status wording uses the existing enums; "deficient" and red wording are not permitted. A topic with too little data returns `insufficient_data` with a sentence saying so.

| Topic | Source | Periods | Note |
|---|---|---|---|
| `sleep` | Gadgetbridge | 7d, 14d | Depends on the H0 pairing test; otherwise `insufficient_data`. |
| `hrv_rhr` | Gadgetbridge | 7d, 14d | As above. |
| `weight_trend` | DietM8 body entries | 7d, 14d | Rounded slope per week, never a single noisy reading. |
| `intake_vs_weight` | DietM8 plus body entries | 7d, 14d | Average energy logged against weight trend, with coverage. |
| `weekly_headline` | The deterministic weekly report | 7d | The first sentences of the report. |

Not in v1: nutrient gaps, training volume, blood results, any row-level data. Each needs its own wording review first.

**Two delivery paths, one schema.**

1. **Live.** A separate listener, served only with `tailscale serve` (never Funnel), with its own port and its own `assistant` bearer-token class. It is not behind `app_proxy`, so the assistant does not need an Umbrel login session (the Umbrel proxy has an unauthenticated-path option that was not verified (U), and a separate port keeps the tier boundary visible). The token class can call the card endpoint and nothing else; MCP tokens cannot call it and it cannot call MCP (tested). Payload shape is tier 2 (rounded, relative dates, templated, no free text), delivered only to the phone, tagged `tag:health-client-phone` in the Tailscale grants. Every call goes to the audit log with the card digest.
2. **Snapshot.** The snapshot writer (section 8.4) also writes `assistant_cards.json`: every topic and period above, rebuilt after each import and nightly, wrapped in the envelope with `as_of`. It reaches the phone through the second Syncthing folder. The assistant needs a persisted read-only folder grant to it (U: whether the Hark fork can hold one is an assistant-plan item). It says the age of the data ("as of Tuesday evening"), and switches to live data when the Umbrel answers.

**Freshness.** The cards are only as fresh as the last Gadgetbridge export and the last DietM8 export. The Inbox page already warns after 26 hours of silence; a card inherits that warning in `data_quality`.

**Model boundary (owner leaning; not final).** The policy recorded here is: card text and figures may reach a model running on the phone or on the Umbrel, never a cloud API, never assistant memory, transcripts or synced files. The initial build goes further by default: the assistant speaks the template sentences as they are, so no model sees the numbers. Rephrasing by a local model is an option to switch on later. If the owner prefers the stricter on-device-only option, the Umbrel part of the policy is dropped and nothing else changes. The assistant must not hold a health card in a context that also contains web-search results or can send anything out (the lethal trifecta, section 9.3).

**Voice stack facts that affect this plan (V, from the Ollama blog post dated 24 Oct 2026 via a model-written extraction).** Ollama can now synthesise speech (`phonon`, an English Apache 2.0 model and four noncommercial multilingual variants) through an OpenAI-compatible `/v1/audio/speech` endpoint. It runs only on Apple Silicon Macs today, Windows and Linux are listed as planned, the post says nothing about phones, and it covers no speech-to-text. The Umbrel (Linux, no GPU) and the phone cannot use it now, so the assistant's own whisper.cpp and Piper plan stands. If it later runs on the Umbrel, the text to be spoken, health figures included, would travel to the Umbrel for synthesis, which the policy above allows only under the Umbrel-plus-on-device option. Re-check at the assistant's Phase 3.

**What the initial build can and cannot show.** It can show a card for each topic fetched with `curl` over Tailscale and read from the snapshot file with no network, both validating against the schema, a tier-2 token refused by MCP and an MCP token refused by the card endpoint, the injection fixture producing no changed text, and the stale-data wording. It cannot show a spoken answer, because that needs the assistant's V1 with a card-reading tool and, for the offline path, folder access. Those are assistant-plan tasks and are not counted in these hours.

---

## 10. The laptop weekly summary

### 10.1 Laptop facts

ASUS Vivobook Pro 15, Ryzen 9, 16GB RAM, NVIDIA RTX 3050 with 4GB VRAM plus AMD Radeon integrated graphics, Windows, Tailscale installed, prefers no more installs. The runner (Ollama or LM Studio) should use the NVIDIA GPU; the Radeon shares system RAM and is not expected to help. Check Task Manager and the runner's logs during the eval to see which GPU is used. (U)

### 10.2 Job design (I)

1. **The deterministic weekly report is always produced on the Umbrel** by `healthcore`, stored with `week`, `data_hash` and `ruleset_version`, and shown on the dashboard whether or not the laptop is on. The model summary is an optional layer.
2. **The laptop job** fetches the closed week's report JSON from the internal read-only endpoint over Tailscale, builds one small prompt, calls the local runner once with a JSON-schema-constrained output at temperature 0, validates, and writes a file. No tool loop. Ollama documents the `format` JSON-schema parameter and advises also putting the schema in the prompt and validating ([Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)). (V)
3. **Triggers:** a manual command, plus a catch-up at login and wake via Windows Task Scheduler. It generates a summary only if none exists for the latest closed week with the current `data_hash` and model digest. If the Umbrel is unreachable it exits quietly and retries later. To avoid installing more on the laptop, package the job as a single script run by a runtime already present; if none is, a single-file executable (U).
4. **Return path (owner decision): the owner moves the file by hand** into the Umbrel inbox (or uploads it through the dashboard's Inbox page). The summary file is wrapped in the shared envelope and imported like any other file.
5. **Validation before storing:** every number, unit and nutrient name must appear in the input within rounding; forbidden phrases ("deficient", "deficiency", "you should take", "diagnose", dosage advice) fail; low-confidence nutrients must carry their caveat; any failure falls back to a deterministic template. The boundary text is a required field.
6. **Food suggestions:** code builds a candidate list from the owner's own past logs first, filtered by the exclusion tags exported by DietM8 (pork, salmon, high-parasite foods, dairy except yoghurt and whey), ranked by nutrient density per portion. The model selects from the list and phrases it; it never invents foods, quantities or numbers.

### 10.3 Interactive use

For ad hoc questions the laptop runs an MCP-capable client (LM Studio from v0.3.17, mcphost, Open WebUI, or Goose) against the Umbrel MCP (tier 1 and 2 tools). Ollama itself offers tool calling but not MCP. Whether any client supports the 2026-07-28 stateless headers was not tested (U). Keep any session to eight tools or fewer with outputs under about 2K tokens, because small-model tool use degrades with longer chains and catalogues ([TinyLLM](https://arxiv.org/pdf/2511.22138), [LongFuncEval](https://arxiv.org/pdf/2505.10570)). (V)+(U)

### 10.4 Model size for this laptop (all uncertain)

Third-party roundups, not the research sources, name the candidates; do not pick from this table, run the eval.

| Resource | Expectation (U) | Candidates named by roundups (unverified) | Plan |
|---|---|---|---|
| 4GB VRAM | About 3B to 4B dense, quantised, fits on the GPU | Qwen 3.5 4B, Gemma 4 E2B, Llama 3.2 3B | Fast; weaker prose; rely on the schema, validator and fallback. |
| 16GB RAM | 7B to 9B dense with partial GPU offload, slower | Qwen 3.5 9B | Likely sweet spot for a weekly job; measure speed, since a weekly job tolerates slow tokens. |

**Eval before choosing (H5):** 20 to 30 cases: food phrases for the parser fallback (about 10), sample weekly summaries from fixture weeks (about 8), tool-call tests against the MCP (about 6), injection fixtures such as a food named with instructions (about 4). Gates: schema validity, number faithfulness, forbidden phrases, caveat presence when confidence is low, a "not enough data" response on empty input, stability across 5 runs at temperature 0, unchanged output under injection. Keep the cheapest model that passes all gates; pin the model digest; re-run on model or tool upgrades.

### 10.5 API use (only for generic tasks)

Under the owner's rule, an API model never receives data derived from personal data. A permitted example is generic, no-personal-data work such as asking for recipe or food-category ideas from a fixed prompt that mentions no logs, weights or dates. Provider terms (no training by default, about 30-day retention, zero-data-retention only through sales approval for Anthropic and OpenAI APIs) were gathered from third-party summaries in October 2026 and must be confirmed on each provider's own pages ([Anthropic retention summary](https://getvoibe.com/resources/claude-api-data-retention/), [OpenAI enterprise privacy](https://openai.com/enterprise-privacy)). The local model remains the default. This helper is optional (open decision OD-3); if built it takes about 4 to 8 hours and is never wired into the scheduled job.

---

## 11. Phased build plan

Hours are estimates. Each phase is one branch and PR series with CI green before merge.

### Phase H0: spikes on the real devices (4 to 8 hours)

Tasks:
- Run the transport afternoon test (section 6.3) and record the result.
- Pair the Forerunner 165 with Gadgetbridge; confirm sleep, heart rate, steps and activities appear; run an auto export to `Documents/HealthExport/Gadgetbridge` and read the SQLite file on the laptop to record real table names.
- Install a "hello" app from a private test community store on the Umbrel and confirm: the Umbrel login works, data survives a restart, and a read-only custom folder mount works.
- Record whether the Umbrel supports drive encryption.

**Done when:** `docs/spike-results.md` records pass or fail for each with the umbrelOS and Android versions, the transport is chosen (Option 1 or Option 2), the Gadgetbridge table names are written down, and any failed check has a recorded fallback. If the Forerunner pairing fails, the fallback is a one-off Garmin archive import (H7 brought forward).

### Phase H1: importer and dashboard on the laptop (14 to 22 hours)

Tasks: repo with Python tooling and tests; SQLite schema; importer with quarantine, idempotency and `import_log`; DietM8 and WorkM8 importers using hand-written fixtures first, then DietM8's sample exports; Gadgetbridge importer; the four v0 pages and the Inbox page; one vendored charting library.

**Done when:**
- A DietM8 export and a WorkM8 export both parse through one envelope reader and join by `local_date`, and the dashboard shows weekly volume against calories and protein, and weight trend against calories.
- Re-importing the same file changes nothing; a corrupt file lands in `quarantine/` with a sidecar and the original stays intact.
- A Gadgetbridge export imports sleep, resting heart rate, HRV and steps, and a file replaced mid-import causes no corruption.
- Runs with `docker compose` on the laptop with host and port configurable.

### Phase H2: package for the Umbrel (8 to 14 hours)

Tasks: container image built in GitHub Actions and published to the container registry with a pinned digest; community store repo from the template; `umbrel-app.yml` and `docker-compose.yml` from an existing app's pattern; the chosen transport wired in (custom folder mount for Option 1, or the receiver service for Option 2); nightly backup; archive folder.

**Done when:** the app installs from the community store, opens through the Umbrel route, imports a real file delivered by the phone, survives a restart and an update with data intact, and a database restore from the archived exports is tested. Access works only over the tailnet and the Umbrel login.

### Phase H3: snapshot to the phone and the weekly report page (9 to 15 hours)

Tasks: snapshot writer and the second sync folder (or the Option 2 pull); the deterministic weekly report and the Weekly summary page; the Sleep and eating window page and the Data health page; the answer-card builder for the five topics and two periods, and `assistant_cards.json` written beside the snapshot, validated against `assistant-card.v1` (about 3 to 5 hours, section 9.5).

**Done when:** a snapshot arrives on the phone without any manual step; the weekly report renders without the laptop; the boundary text appears on every page footer; `assistant_cards.json` validates, carries `as_of`, and a topic with no data returns `insufficient_data` with its sentence.

### Phase H4: analytics layer and read-only MCP (16 to 24 hours)

Tasks: extract `healthcore` and the shared golden vectors; read-only database role on views; the five MCP tools on the Python SDK v2 (pinned), stdio first and then Streamable HTTP on loopback via `tailscale serve`, with bearer tokens, Origin check, rate limits, size caps and the JSONL audit log; Tailscale tags and grants; the data-flow register; the live assistant-card listener on its own port with the `assistant` token class, `tag:health-client-phone` grant and audit entries (about 4 to 6 hours).

**Done when:** gap calculator tests pass; the injection fixture (a food named with instructions) alters no output, including in the cards; the laptop client can call all five tools over Tailscale and receives units, coverage and the boundary text; a tier-2 token cannot call a tier-1 tool (tested); an `assistant` token cannot call MCP and an MCP token cannot call the card endpoint (tested); the phone fetches every card over Tailscale and each validates against the schema; every call appears in the audit log and the monthly report script works; Funnel is off.

### Phase H5: laptop weekly summary with the model eval (14 to 22 hours)

Tasks: record the laptop runner and which GPU it uses; build the eval set and harness; the weekly job with catch-up and idempotency, constrained output, validator, deterministic fallback and candidate-food list with the exclusion filter; the Inbox hand-off for the summary file.

**Done when:** the eval (20 to 30 cases) passes every gate for the chosen model with the digest pinned and results committed; running the job twice for the same week produces one summary; the job exits cleanly when the Umbrel is unreachable; every summary contains the boundary text and a failed validation falls back to the template; the dashboard's weekly report is unaffected when the laptop is off.

### Phase H6: native phone viewer (optional, 20 to 35 hours)

Only if widgets or offline use are wanted after living with the PWA. A thin Kotlin app on the WorkM8 stack that reads the snapshot folder and displays results; no calculations, no network permission, a Glance widget. **Done when:** it displays the latest snapshot offline, shows a "snapshot is X days old" state, and the manifest has no INTERNET permission.

### Phase H7: later importers (as time allows)

Blood tests entered on a dashboard form (tier 1, `lab_result` table with LOINC code, value, unit as reported, lab range and date); an optional one-off Garmin archive import for history before pairing; check-ins and counters if DietM8 exports them. Garmin archive file names disagree between two third-party sources, so that importer globs, sniffs and parses defensively ([Takeout Day](https://takeoutday.org/guides/how-to-export-garmin-connect-data), [intervals.icu thread](https://forum.intervals.icu/t/which-files-are-needed-for-importing-all-garmin-data/97571)).

### Totals (estimates)

| Phase | Hours |
|---|---|
| H0 Spikes | 4 to 8 |
| H1 Importer and dashboard | 14 to 22 |
| H2 Package for Umbrel | 8 to 14 |
| H3 Snapshot, weekly report, assistant cards | 9 to 15 |
| H4 Analytics, MCP, live card listener | 16 to 24 |
| H5 Laptop summary and eval | 14 to 22 |
| **H0 to H5** | **65 to 105** |
| Assistant-side wiring (card tool, folder grant), in the assistant plan, not counted | not estimated |
| H6 Native viewer (optional) | 20 to 35 |
| Optional API helper | 4 to 8 |

---

## 12. Engineering standards

- **CI:** Python formatting, type checks and tests; JSON Schema validation of sample exports; golden-vector tests shared with DietM8; container build in GitHub Actions; lint of the Umbrel package where the official linter is usable (`lint:apps`, if it can be run against a community store (U)).
- **Rules:** never delete or edit files in the inbox; archive is the source of truth; no analytics or telemetry; secrets and token hashes outside images and outside any synced folder; pin images by digest, the MCP SDK by exact version and model digests; no real health data in the community store repo.
- **Tests:** importer idempotency and quarantine, join and DST tests (clocks go back on Sunday 25 October 2026, giving a 25-hour day), Gadgetbridge replace-during-read, MCP injection and tier tests, validator tests for the summary job.
- **Docs in the repo:** `docs/spike-results.md`, `docs/rules.md` (shared with DietM8), the data-flow register and this decision log.

---

## 13. Decision log

Per-plan IDs are HM-n with the master ID in brackets where one exists.

| # | Decision | Rationale | Status |
|---|---|---|---|
| HM-1 [D14] | Dashboard is a Python app (FastAPI plus SQLite), not Grafana | Date joins, shared code | Recommended |
| HM-2 | App name HealthM8 | Owner decision | Fixed |
| HM-3 [D5, revised] | Transport Option 1 (Syncthing-Fork to Syncthing, Home folder shared read-only into HealthM8); switch to Option 2 if the H0 test fails | Owner decision, 26 Oct | Fixed |
| HM-4 | Option 2 is a receiver inside HealthM8 plus Termux, not the `umbrel` SSH account | Limits blast radius | Recommended |
| HM-5 | Garmin data from Gadgetbridge only; Garmin archive importer deferred to H7 | Owner decision, 26 Oct | Fixed |
| HM-6 | Strava automatic upload ends; no Strava importer in v1 | Owner decision | Fixed |
| HM-7 [D15] | Analytics layer and MCP live beside the dashboard data on the Umbrel, over Tailscale | Single source of truth | Recommended |
| HM-8 | Phone access: PWA first; Umbrel writes a snapshot back to the phone; native viewer last | Offline and widgets without duplicate analytics | Recommended |
| HM-9 | Develop on the laptop first, then package | Avoids debugging on the shared Umbrel | Recommended |
| HM-10 [D16] | Local model on the laptop; weekly summary is a catch-up job; deterministic report always produced on the Umbrel | Laptop not always on | Owner decision |
| HM-11 [D17, revised] | The owner moves the summary file by hand | Owner decision | Fixed |
| HM-12 [D18] | The weekly job bypasses MCP | No injection channel | Recommended |
| HM-13 [D19] | Three tiers with separate endpoints and an audit log | Server-side enforcement | Recommended |
| HM-14 [D20, revised] | No personal-data-derived payload goes to an API; the cloud tier is not built | Owner decision | Fixed |
| HM-15 [D13] | The gap view never shows red; grades are green, amber, grey | LRNI and EAR not sourced | Recommended |
| HM-16 [D11] | DietM8 owns bodyweight; HealthM8 shows it | One series | Owner decision |
| HM-17 [D12] | Adaptive suggestions are computed on the phone and only displayed | One implementation | Recommended |
| HM-18 [D21] | No LLM-dependent feature in v1 | Owner decision | Fixed |
| HM-19 | Inbox page with hashes, arrival times and a 26-hour stale warning | Visible transport health | Recommended |
| HM-20 | Wearable: Garmin Forerunner 165 | Owner statement | Fixed |
| HM-21 | Assistant gets answer cards, not MCP tools; one `assistant-card.v1` schema shared with DietM8 | One small surface, no MCP client on the phone | Recommended |
| HM-22 | Cards are built from fixed templates with no free text from the data; the initial build uses no language model on the numbers | Owner leaning (26 Oct) towards on-device or Umbrel-only models; templates satisfy both | Recommended |
| HM-23 | Two delivery paths: live on a separate Tailscale port with an `assistant` token class, and `assistant_cards.json` in the phone snapshot; the assistant says the data's age | Owner decision, 26 Oct (answer from the phone's last snapshot when away) | Fixed |
| HM-24 | Contracts and tests in the initial build; assistant wiring later; DietM8 first | Owner decision, 26 Oct | Fixed |
| HM-25 | Health cards must not be stored by the assistant or held in a context with web search or any outbound tool | Lethal-trifecta rule applied to the assistant | Recommended (needs a matching line in the assistant plan) |

---

## 14. Open decisions and risks

### Open decisions

| # | Question | Needed by |
|---|---|---|
| OD-1 | Outcome of the H0 transport test (Option 1 or Option 2). | H0 |
| OD-2 | Outcome of the Forerunner 165 pairing test. | H0 |
| OD-3 | Whether to build the optional generic-tasks API helper at all, and which provider. | After H5 |
| OD-4 | Whether to build the native phone viewer after living with the PWA. | After H3 |
| OD-5 | How to get WorkM8 exports into the folder (share-sheet target, or a SAF folder export in the WorkM8 PR). | H1 |
| OD-6 | Which laptop runner (Ollama or LM Studio) is already installed, and how to run the summary job without adding software. | H5 |
| OD-7 | Final model boundary for health cards: Umbrel plus on-device models allowed (policy as written), or on-device only. Initial build is unaffected. | Before any rephrasing feature |
| OD-8 | Whether Umbrel's proxy allows an unauthenticated path for the card endpoint, or the separate-port design stays. | H4 |

### Risks

| Risk | Mark | Mitigation |
|---|---|---|
| umbrelOS 2.0 custom folders may not work for Syncthing or a community app. | (U) | H0 test; Option 2 fallback. |
| Syncthing-Fork maintainer handover. | (V) | Pin the F-Droid build; Send Only; no discovery or relays. |
| Background sync reliability on LineageOS, and any OnePlus battery handling. | (U) | H0 overnight test; stale-file warning; Option 2. |
| Forerunner 165 support is listed, with no real-world reports found. | (U) | H0 pairing test; Garmin archive importer as fallback. |
| Gadgetbridge database schema and table names not retrieved. | (U) | Read a real export in H0; defensive parsing. |
| Gadgetbridge export is overwritten in place and readable by other apps with storage access. | (V) | Copy before reading; keep few apps with All files access. |
| Garmin Connect link replaced; switching back may need a watch factory reset. | (V) | Owner accepted; avoid "fake OAuth responses" unless intended. |
| WorkM8 exports only by manual share and the WorkM8 export format has not been seen. | (U) | OD-5; check the WorkM8 PR against the repo. |
| `umbrel-app.yml` fields and `manifestVersion` for umbrelOS 2.0 not verified; private community stores may not work. | (U) | Read an existing app first; use a public repo with no real data. |
| Encryption at rest on the Umbrel and the laptop not checked. | (U) | Check before real data lands (H0, H5). |
| MCP spec and SDK details come from spec pages and READMEs; client support for stateless headers untested. | (V)/(U) | Pin versions; test with the real laptop client. |
| Model names, sizes and speeds on the RTX 3050 laptop are unverified. | (U) | Run the eval. |
| Laptop is not always on. | (I) | Catch-up job; deterministic report on the Umbrel. |
| The umbrelOS MCP management token gives agents file and app control. | (V) | Scope narrowly; revoke unused tokens. |
| UK GDPR and device-regulation position unreviewed for sharing or commercialising. | (U) | Personal use only; take advice before sharing. |
| The assistant (web search, memory, transcripts) becomes a route for health data or an injection route into it. | (U) | Templated cards with no free text; `sensitivity` tag; requirements on the assistant plan (HM-25); `assistant` token class cannot reach MCP. |
| Phone cannot reach the Umbrel when away, and the offline folder grant may not be possible in the Hark fork. | (U) | Snapshot path with an age statement; if the grant fails, the assistant answers live only and says it cannot reach the Umbrel. |
| Cards are only as fresh as the last export, so a confident spoken sentence may describe stale data. | (I) | `as_of` and stale-stream wording in every card. |
| Ollama speech support is Apple Silicon only (24 Oct 2026); an assumption that it will arrive on the Umbrel could break the voice plan. | (V) | Keep whisper.cpp and Piper on the phone; re-check later. |
| Scope and time. | (I) | Strict phase gates; H0 decides the shape before building. |

---

## 15. Sources

- Umbrel app settings and custom folders: https://umbrel.com/support/apps/app-settings
- umbrelOS 2.0: https://community.umbrel.com/t/umbrelos-2-0-is-out-now/25501 and https://freedom.tech/posts/2026-09-22-umbrelos-2-0/
- Umbrel apps repo and packaging skills: https://github.com/getumbrel/umbrel-apps
- Community app store template: https://github.com/getumbrel/umbrel-community-app-store
- How to add a community app store: https://umbrel.com/support/apps/how-to-add-a-community-app-store
- Syncthing app on Umbrel (folder access limit): https://community.umbrel.com/t/syncthing-on-umbrel-unable-to-create-sync-directory/23874
- Syncthing-Fork: https://f-droid.org/packages/com.github.catfriend1.syncthingfork/
- Syncthing-Fork handover thread: https://forum.syncthing.net/t/does-anyone-know-why-syncthing-fork-is-no-longer-available-on-github/25661/251
- Gadgetbridge: https://codeberg.org/Freeyourgadget/Gadgetbridge
- Gadgetbridge Garmin: https://gadgetbridge.org/basics/topics/garmin
- Gadgetbridge data management and auto export: https://gadgetbridge.org/internals/development/data-management/ and https://gadgetbridge.org/internals/automations/auto-export/
- Android storage permissions: https://anexplorer.io/solve/storage-permissions-android
- Termux job scheduling and rsync notes: https://b.tuxes.uk/termux-rsync-photos.html
- MCP specification: https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- Ollama structured outputs: https://docs.ollama.com/capabilities/structured-outputs
- Tailscale grants: https://tailscale.com/kb/1324/acl-grants
- Ollama speech models (Phonon, 24 Oct 2026): https://ollama.com/blog/speech-models
- Hark and OACP (assistant side): https://github.com/OpenAppCapabilityProtocol/hark and https://github.com/OpenAppCapabilityProtocol/oacp
- Sovereign Assistant V1 spec (owner's document): https://claude.ai/artifact/L52JmXrjJVkd723KzcuJmP
- The lethal trifecta: https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- ICO exemptions guide: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/exemptions/a-guide-to-the-data-protection-exemptions/
