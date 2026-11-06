# SmartClean – Duplicate & Junk Finder

An offline, ad-free Android cleaner written in Kotlin + Jetpack Compose (Material 3).
It finds exact duplicates, similar photos, near-duplicate videos and songs, junk files,
WhatsApp media, large files and old files – and never deletes anything without you tapping
**Move to Trash**. Everything goes to a Trash you can restore from for 30 days.

- **No ads, no analytics, no tracking.** The app has **no INTERNET permission at all**.
- minSdk 26 (Android 8.0) · targetSdk 35 · Kotlin 2.0 · Compose · Room · WorkManager · Coil

---

## Get the APK

### Option A – GitHub Actions (free cloud build, no Android Studio needed)
1. Create a free account at github.com and make a new **private** repository.
2. Upload the contents of this folder, keeping the folder structure (easiest from a computer browser:
   drag the unzipped folder's contents onto the "uploading an existing file" page). If the hidden
   `.github` folder doesn't upload, use **Actions → set up a workflow yourself** and paste in
   `.github/workflows/build.yml`.
3. Open the **Actions** tab. The *Build APK* workflow runs automatically (about 5–8 minutes).
4. Open the finished run → **Artifacts** → download **SmartClean-debug-apk** → unzip → install `app-debug.apk`.
   (Android will ask you to allow installing from your browser/file manager.)

### Option B – Android Studio
1. Open this folder in Android Studio (Ladybug or newer). Let Gradle sync.
2. **Build → Build App Bundle(s) / APK(s) → Build APK(s)**.
3. The APK is at `app/build/outputs/apk/debug/app-debug.apk`.

### Option C – Command line
```
./gradlew assembleDebug        # debug APK
./gradlew testDebugUnitTest    # unit tests
./gradlew bundleRelease        # release AAB for Play Store
```

### Release AAB for Google Play
Create a keystore once:
```
keytool -genkey -v -keystore smartclean.jks -keyalg RSA -keysize 2048 -validity 10000 -alias smartclean
```
Then create `keystore.properties` in the project root (it is git-ignored – never commit it):
```
storeFile=smartclean.jks
storePassword=YOUR_PASSWORD
keyAlias=smartclean
keyPassword=YOUR_PASSWORD
```
`./gradlew bundleRelease` then produces a signed `app/build/outputs/bundle/release/app-release.aab`.
Without the file, the AAB is built unsigned (Play App Signing can still be used with an upload key).

---

## Features

| Area | What it does |
|---|---|
| Exact duplicates | Size → partial MD5 (first + last 64 KB) → full MD5 only on collisions. Images, videos, audio, documents, APKs, archives, anything. |
| Similar photos | 64-bit pHash (DCT) + dHash, BK-tree search, Hamming distance ≤ 6 by default (≈ 90 %). Burst-shot detection. Filter by Camera / Screenshots / WhatsApp / Downloads / Telegram. Sensitivity: Strict / Normal / Loose. |
| Best copy | Auto-keeps 1 per group: Best quality (resolution, sharpness, size, Camera over WhatsApp) / Oldest / Newest / Prefer Camera. "KEEP" badge. |
| Bad photos | Possibly blurry (Laplacian variance), very dark, overexposed, almost blank. |
| Videos | Exact hash, plus duration ±1 s + frame hashes at 10/50/90 % (catches re-compressed WhatsApp forwards). |
| Audio | Exact hash, plus same title/artist and duration ±1 s. |
| Junk | Temp files (.tmp .log .bak .old …), thumbnail caches, empty folders, APK installers ("Installed / Not installed"), screenshots, screen recordings, Downloads review. |
| App cache | Per-app cache sizes (Usage Access) and the system "Clear all cache" dialog (Android 11+). |
| WhatsApp | Images, Videos, Audio, Voice notes, Documents, Stickers, GIFs, Statuses, Sent media, Duplicates. Databases and backups are never touched. |
| Large & old | > 100 MB / 500 MB / 1 GB, not modified in 6 / 12 months. |
| Safe delete | Moves to `/SmartClean_Trash` (with `.nomedia`) or Android's system trash in media-only mode. Restore, Undo snackbar, 30-day auto-purge (7/30/60), permanent delete needs two confirmations, deletion log with CSV export. |
| Smart | Weekly auto-scan while charging (scan only, never deletes), protected folders, ignore small files / file types, sort, search. |
| Performance | Hash cache in Room (rescans only touch new/changed files), 4 parallel workers, pause / resume / cancel, slows down when the phone gets warm. |
| UI | Material 3, dynamic color, light/dark/system theme, bottom nav (Home, Duplicates, Junk, Large Files, Trash), grid/list, side-by-side compare with synced zoom, in-app image / video / audio / PDF preview. |

## Safety rules (enforced in code)
- Nothing is removed unless you tap **Move to Trash** and confirm the "You will free up …" dialog.
- `KeepGuard` keeps at least one copy of every duplicate group, checked again in the delete path – not just the UI.
- Before deleting, each file is re-checked: still exists, same size and date, not changed in the last minute (treated as "in use"), not in a protected folder, and the kept copy still exists and isn't the same physical file.
- Never touched: `Android/data`, `Android/obb`, WhatsApp `Databases` and `Backups`.
- Extra warning when deleting originals from `DCIM/Camera`.
- If a move fails half-way, the original stays and the partial copy is removed.

## Permissions
| Permission | Why |
|---|---|
| READ_MEDIA_IMAGES / VIDEO / AUDIO (13+), READ_EXTERNAL_STORAGE (≤ 12) | Find duplicate media |
| MANAGE_EXTERNAL_STORAGE (optional) | Documents, APKs, temp files, empty folders, WhatsApp statuses, restorable trash folder |
| PACKAGE_USAGE_STATS (optional) | Show per-app cache size |
| POST_NOTIFICATIONS (optional) | Weekly scan summary |

**Play Store note:** All Files Access requires a declaration form in Play Console. File managers / cleaners
are an accepted use case; describe the duplicate- and junk-cleaning features. Data Safety form: no data
collected or shared.

## Project layout
```
app/src/main/java/com/smartclean/app/
  scanner/   FileHashUtil, ImageSimilarityUtil, MediaStoreScanner, FileSystemWalker,
             SimilarityGrouper (BK-tree), MediaMatcher, KeepSelector + KeepGuard, AppInfoUtil, ScanEngine
  delete/    TrashManager (trash, restore, purge, log, CSV export)
  data/      Models, Room database, settings, saved scan results
  work/      Weekly scan + trash purge workers, notifications
  ui/        ViewModel, theme, components, screens
app/src/test/  Unit tests (keep guard, hashing, pHash, grouping, safety rules)
```

## Known limitations / next steps
- Built without a compiler in the authoring environment – the first Gradle build may surface small compile fixes.
- Text is in English and mostly inline in Kotlin; moving it all into `strings.xml` is needed before translating.
- No ML Kit labeling/face detection (keeps the app small and fully offline); grouping uses perceptual hashes.
- Long-press-and-drag multi-select and a home-screen widget are not included yet.
- A manual scan runs while the app is open; if Android stops the app mid-scan, the next scan reuses all cached work.
- Blur thresholds are heuristics – the list is labelled "Possibly blurry" and never pre-selected.
