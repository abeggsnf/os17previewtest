# Android OS 17 (CinnamonBun) Preview — Robolectric Compatibility Test

A minimal Android app used to test Robolectric compatibility with the Android 17 "CinnamonBun" preview SDK. Each branch demonstrates a specific configuration scenario.

## Quick Start

```bash
# Pick a scenario branch
git checkout scenario/05-compile-preview-target-numeric

# Run tests
./gradlew app:testDebugUnitTest
```

Requires Java 21.

## Scenario Branches

| | Branch | compileSdk | targetSdk | AGP | Gradle | Robolectric | Build | Tests |
|--|--------|-----------|-----------|-----|--------|-------------|-------|-------|
| 🟢 | [`scenario/01-baseline-android16`](https://github.com/abeggsnf/os17previewtest/tree/scenario/01-baseline-android16?tab=readme-ov-file) | 36 | 36 | 9.1.0 | 9.3.1 | 4.16.1 | PASS | PASS |
| 🔴 | [`scenario/02-cinnamonbun-full-preview`](https://github.com/abeggsnf/os17previewtest/tree/scenario/02-cinnamonbun-full-preview?tab=readme-ov-file) | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | PASS | FAIL |
| 🔴 | [`scenario/03-cinnamonbun-robolectric-properties`](https://github.com/abeggsnf/os17previewtest/tree/scenario/03-cinnamonbun-robolectric-properties?tab=readme-ov-file) | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | PASS | FAIL |
| 🔴 | [`scenario/04-cinnamonbun-snapshot-robolectric`](https://github.com/abeggsnf/os17previewtest/tree/scenario/04-cinnamonbun-snapshot-robolectric?tab=readme-ov-file) | CinnamonBun | CinnamonBun | 9.1.0 | 9.3.1 | 4.17-SNAPSHOT | PASS | FAIL |
| 🟢 | [`scenario/05-compile-preview-target-numeric`](https://github.com/abeggsnf/os17previewtest/tree/scenario/05-compile-preview-target-numeric?tab=readme-ov-file) | CinnamonBun | 36 | 9.1.0 | 9.3.1 | 4.16.1 | PASS | **PASS** |
| 🟢 | [`scenario/06-agp-alpha-preview`](https://github.com/abeggsnf/os17previewtest/tree/scenario/06-agp-alpha-preview?tab=readme-ov-file) | CinnamonBun | 36 | 9.2.0-alpha04 | 9.5.0-M5 | 4.16.1 | PASS | **PASS** |
| 🔴 | [`scenario/07-both-sdk-and-preview`](https://github.com/abeggsnf/os17previewtest/tree/scenario/07-both-sdk-and-preview?tab=readme-ov-file) | 36+CinnamonBun | 36+CinnamonBun | 9.1.0 | 9.3.1 | 4.16.1 | PASS | FAIL |

---

### Scenario 1: Baseline Android 16
**Branch:** `scenario/01-baseline-android16`
**Result:** PASS

Baseline configuration with `compileSdk = 36` and `targetSdk = 36`. No preview SDK involved. Confirms the app and Robolectric work correctly before introducing the CinnamonBun preview.

```bash
git checkout scenario/01-baseline-android16
./gradlew app:testDebugUnitTest   # PASS
```

---

### Scenario 2: Full CinnamonBun Preview
**Branch:** `scenario/02-cinnamonbun-full-preview`
**Result:** FAIL

Sets both `compileSdkPreview = "CinnamonBun"` and `targetSdkPreview = "CinnamonBun"`. Uses `@Config(sdk = 35)` on the test class to pin the Robolectric emulated SDK.

**Error:**
```
java.lang.RuntimeException at ShadowPackageParser.java:53
  Caused by: android.content.pm.PackageParser$PackageParserException at PackageParser.java:1240
```

**Why it fails:** The merged AndroidManifest contains `android:targetSdkVersion="CinnamonBun"` (a string codename). Robolectric's `ShadowPackageParser` delegates to the framework `PackageParser` which cannot parse a non-numeric `targetSdkVersion`.

```bash
git checkout scenario/02-cinnamonbun-full-preview
./gradlew app:testDebugUnitTest   # FAIL
```

---

### Scenario 3: CinnamonBun + robolectric.properties
**Branch:** `scenario/03-cinnamonbun-robolectric-properties`
**Result:** FAIL

Same as Scenario 2, but uses `robolectric.properties` with `sdk=35` instead of the `@Config` annotation. Tests whether configuring the Robolectric SDK via properties avoids the issue.

**Why it fails:** `robolectric.properties` controls which SDK Robolectric *emulates*, not how it parses the manifest. The `targetSdkVersion="CinnamonBun"` string in the merged manifest still causes `PackageParser` to fail.

```bash
git checkout scenario/03-cinnamonbun-robolectric-properties
./gradlew app:testDebugUnitTest   # FAIL
```

---

### Scenario 4: CinnamonBun + Robolectric 4.17-SNAPSHOT
**Branch:** `scenario/04-cinnamonbun-snapshot-robolectric`
**Result:** FAIL

Uses Robolectric `4.17-SNAPSHOT` from the Sonatype snapshots repository to test whether the latest development version has CinnamonBun support.

**Why it fails:** The 4.17 snapshot has code changes (error now at `ShadowPackageParser.java:68` instead of `:53`) but still does not handle preview codenames in the manifest's `targetSdkVersion`.

```bash
git checkout scenario/04-cinnamonbun-snapshot-robolectric
./gradlew app:testDebugUnitTest   # FAIL
```

---

### Scenario 5: Compile Preview + Numeric Target (Recommended)
**Branch:** `scenario/05-compile-preview-target-numeric`
**Result:** PASS

Sets `compileSdkPreview = "CinnamonBun"` to compile against Android 17 APIs, but keeps `targetSdk = 36` (numeric). This is the **recommended workaround**.

**Why it works:** The numeric `targetSdk = 36` means the merged manifest contains `android:targetSdkVersion="36"` which `PackageParser` can parse. You still get access to all CinnamonBun APIs via `compileSdkPreview`.

**Note:** AGP 9.1.0 emits a warning about CinnamonBun not being tested. Suppressed with `android.suppressUnsupportedCompileSdk=CinnamonBun` in `gradle.properties`.

```bash
git checkout scenario/05-compile-preview-target-numeric
./gradlew app:testDebugUnitTest   # PASS
```

---

### Scenario 6: AGP Alpha + Preview
**Branch:** `scenario/06-agp-alpha-preview`
**Result:** PASS

Same as Scenario 5 but with AGP `9.2.0-alpha04` and Gradle `9.5.0-milestone-5`. No CinnamonBun compile SDK warning.

**Note:** AGP 9.2.0-alpha04 hard-requires Gradle >= 9.5.0-milestone-5.

```bash
git checkout scenario/06-agp-alpha-preview
./gradlew app:testDebugUnitTest   # PASS
```

---

### Scenario 7: Both compileSdk and Preview Set
**Branch:** `scenario/07-both-sdk-and-preview`
**Result:** FAIL

Sets all four properties: `compileSdk = 36`, `compileSdkPreview = "CinnamonBun"`, `targetSdk = 36`, `targetSdkPreview = "CinnamonBun"`. Tests whether having both numeric and preview values allows Robolectric to use the numeric one.

**Why it fails:** When both `targetSdk` and `targetSdkPreview` are set, AGP uses the preview value in the merged manifest. The numeric `targetSdk = 36` is ignored and "CinnamonBun" still appears, causing the same `PackageParser` failure.

```bash
git checkout scenario/07-both-sdk-and-preview
./gradlew app:testDebugUnitTest   # FAIL
```

---

## Key Takeaway

To use the Android 17 CinnamonBun preview SDK with Robolectric, set `compileSdkPreview = "CinnamonBun"` but keep `targetSdk` as a numeric value (e.g. `36`). Do **not** set `targetSdkPreview` — it puts the string codename into the manifest which Robolectric cannot parse.

See [COMPATIBILITY_REPORT.md](COMPATIBILITY_REPORT.md) for the full test matrix and error details.
