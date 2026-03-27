# Android OS 17 (CinnamonBun) Preview Compatibility Report

## Current Configuration
| Setting | Value |
|---|---|
| compileSdk | 36 |
| compileSdkPreview | CinnamonBun |
| targetSdk | 36 |
| targetSdkPreview | CinnamonBun |
| minSdk | 24 |
| AGP | 9.1.0 |
| Gradle | 9.3.1 |
| Robolectric | 4.16.1 |
| Java | 21 |

## Summary

Robolectric (4.16.1 and 4.17-SNAPSHOT) cannot parse the Android manifest when `targetSdkPreview = "CinnamonBun"` is set. The `ShadowPackageParser` fails on the string codename regardless of the Robolectric emulated SDK level, `@Config` annotation, or `robolectric.properties` settings.

### Working Configurations

The only configurations where both build and Robolectric tests pass use `compileSdkPreview = "CinnamonBun"` **without** `targetSdkPreview`:

**Option A — Stable toolchain (recommended)**
```
compileSdkPreview = "CinnamonBun"
targetSdk = 36
AGP = 9.1.0
Gradle = 9.3.1
Robolectric = 4.16.1
```
Note: AGP emits a warning that CinnamonBun has not been tested with 9.1.0. Suppress with `android.suppressUnsupportedCompileSdk=CinnamonBun` in `gradle.properties`.

**Option B — Preview toolchain (no warnings)**
```
compileSdkPreview = "CinnamonBun"
targetSdk = 36
AGP = 9.2.0-alpha04
Gradle = 9.5.0-milestone-5
Robolectric = 4.16.1
```

### Key Findings

- Setting `targetSdkPreview` in any combination causes the "CinnamonBun" codename to appear in the merged manifest, which Robolectric's `PackageParser` cannot parse.
- Even setting both `targetSdk = 36` and `targetSdkPreview = "CinnamonBun"` together does not help — the preview value takes precedence.
- The workaround is to use `compileSdkPreview` (to compile against Android 17 APIs) while keeping `targetSdk = 36` (numeric, no preview codename in the manifest).
- AGP 9.2.0-alpha04 hard-requires Gradle >= 9.5.0-milestone-5.

## Test Results

| # | compileSdk | targetSdk | AGP | Gradle | Robolectric | Robolectric @Config(sdk) | Build | Tests | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 36 | 36 | 8.12.0-alpha05 | 8.14.3 | 4.14.1 | 35 | PASS | PASS | Baseline Android 16 |
| 2 | 36 | 36 | 9.1.0 | 9.3.1 | 4.16.1 | 35 | PASS | PASS | Upgraded toolchain |
| 3 | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | 35 (@Config) | PASS | FAIL | See error details below |
| 4 | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | 35 (robolectric.properties) | PASS | FAIL | Same PackageParser error; properties file doesn't help either |
| 5 | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.17-SNAPSHOT | 35 (robolectric.properties) | PASS | FAIL | Snapshot also fails; error now at ShadowPackageParser.java:68 |
| 6 | CinnamonBun | 36 | 9.1.0 | 9.3.1 | 4.16.1 | 35 (robolectric.properties) | PASS | **PASS** | Keeping targetSdk=36 avoids codename in manifest |
| 7 | CinnamonBun | 36 | 9.2.0-alpha04 | 9.5.0-M5 | 4.16.1 | 35 (robolectric.properties) | PASS | **PASS** | AGP alpha + milestone Gradle, no CinnamonBun warning |
| 8 | CinnamonBun | 36 | 9.2.0-alpha04 | 9.3.1 | 4.16.1 | 35 (robolectric.properties) | FAIL | — | AGP 9.2.0-alpha04 requires Gradle >= 9.5.0-milestone-5 |
| 9 | 36+CinnamonBun | 36+CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | 35 (robolectric.properties) | PASS | FAIL | Both compileSdk/Preview and targetSdk/Preview set; preview takes precedence, same PackageParser error |

## Error Details

### Run #3
- **Tests failed:** `activityLaunches`, `textViewDisplaysHelloMessage` (2/2)
- **Error:** `java.lang.RuntimeException` at `ShadowPackageParser.java:53`
  - **Caused by:** `android.content.pm.PackageParser$PackageParserException` at `PackageParser.java:1240`
- **AGP Warning:** compile SDK preview version "CinnamonBun" has not been tested with AGP 9.1.0 (tested up to compile SDK 36.1)
- **Analysis:** Robolectric's `ShadowPackageParser` cannot parse the manifest when `targetSdkPreview = "CinnamonBun"` is set. The preview codename is not recognized by the Robolectric internals.

### Run #4
- **Change:** Added `robolectric.properties` with `sdk=35` (instead of `@Config` annotation)
- **Result:** Same `ShadowPackageParser` / `PackageParserException` failure
- **Analysis:** The issue is not which SDK Robolectric emulates — it's that the merged AndroidManifest contains `targetSdkVersion="CinnamonBun"` which Robolectric's PackageParser cannot parse at all.

### Run #5
- **Change:** Upgraded Robolectric to `4.17-SNAPSHOT` (from Sonatype snapshots repo)
- **Result:** Same `ShadowPackageParser` / `PackageParserException` failure (now at line 68 instead of 53)
- **Analysis:** The 4.17 snapshot has code changes but still does not handle the "CinnamonBun" preview codename in the manifest's targetSdkVersion.

### Run #6
- **Change:** Reverted Robolectric to 4.16.1, set `compileSdkPreview = "CinnamonBun"` but `targetSdk = 36` (not preview)
- **Result:** BUILD SUCCESSFUL, 2/2 tests pass
- **Analysis:** The workaround is to compile against the CinnamonBun preview SDK while keeping `targetSdk = 36`. This avoids putting the codename string into the merged manifest, which is the root cause of the Robolectric PackageParser failure.

### Run #7
- **Change:** Upgraded AGP to 9.2.0-alpha04, Gradle to 9.5.0-milestone-5
- **Result:** BUILD SUCCESSFUL, 2/2 tests pass, no AGP compile SDK warning
- **Analysis:** AGP 9.2.0-alpha04 supports compileSdkPreview CinnamonBun without warnings (max API level 36.1 still listed but no warning emitted). Combined with targetSdk=36 this is a fully clean build.

### Run #9
- **Change:** Set all four: `compileSdk = 36`, `compileSdkPreview = "CinnamonBun"`, `targetSdk = 36`, `targetSdkPreview = "CinnamonBun"`
- **Result:** Build passes, tests fail with same PackageParser error
- **Analysis:** When both `targetSdk` and `targetSdkPreview` are set, the preview value takes precedence in the merged manifest. The numeric `targetSdk = 36` is ignored and "CinnamonBun" still appears in the manifest, triggering the Robolectric failure.
