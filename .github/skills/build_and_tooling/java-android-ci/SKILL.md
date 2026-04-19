---
name: java-android-ci
description: Expert guidance on setting up GitHub Actions CI for a Java Android project — JDK 17, Gradle caching, lint, unit + instrumented tests, signing, and Play Publisher. Use this when creating or tuning CI pipelines for a Java-based Android app.
---

# Java Android CI with GitHub Actions

## Instructions

CI must reproduce the local build deterministically, run fast enough to gate every PR, and produce signed, shippable artifacts on tag.

### 1. Minimal CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}

      - name: Spotless check
        run: ./gradlew spotlessCheck

      - name: Lint
        run: ./gradlew lintDebug

      - name: Unit tests
        run: ./gradlew testDebugUnitTest

      - name: Assemble debug
        run: ./gradlew assembleDebug

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: build-reports
          path: |
            **/build/reports/
            **/build/test-results/
```

### 2. Instrumented Tests on Emulator

Instrumented tests are slow — split into their own job and gate on `pull_request`:

```yaml
  instrumented:
    runs-on: macos-latest   # hardware-accelerated KVM
    timeout-minutes: 45
    strategy:
      matrix:
        api-level: [26, 33]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - uses: gradle/actions/setup-gradle@v4

      - name: AVD cache
        uses: actions/cache@v4
        id: avd-cache
        with:
          path: |
            ~/.android/avd/*
            ~/.android/adb*
          key: avd-${{ matrix.api-level }}

      - name: Create AVD
        if: steps.avd-cache.outputs.cache-hit != 'true'
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: ${{ matrix.api-level }}
          force-avd-creation: false
          emulator-options: -no-window -gpu swiftshader_indirect -noaudio -no-boot-anim -camera-back none
          script: echo "Generated AVD snapshot."

      - name: Run connectedCheck
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: ${{ matrix.api-level }}
          script: ./gradlew connectedDebugAndroidTest
```

### 3. Signing & Release Build

Keep keystores and passwords in GitHub Actions secrets. Decode at runtime.

```yaml
  release:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - uses: gradle/actions/setup-gradle@v4

      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > release.keystore
          echo "KEYSTORE_PATH=$PWD/release.keystore" >> $GITHUB_ENV

      - name: Assemble + bundle release
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS:         ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD:      ${{ secrets.KEY_PASSWORD }}
        run: ./gradlew bundleRelease assembleRelease

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: release-artifacts
          path: |
            app/build/outputs/bundle/release/*.aab
            app/build/outputs/apk/release/*.apk
```

### 4. Play Publisher

For automated distribution to internal / open testing tracks:

```yaml
      - name: Publish to Play internal track
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app
          releaseFiles: app/build/outputs/bundle/release/app-release.aab
          track: internal
          status: completed
          mappingFile: app/build/outputs/mapping/release/mapping.txt
          debugSymbols: app/build/outputs/native-debug-symbols/release/native-debug-symbols.zip
```

Upload `mapping.txt` so Play's crash reporter can deobfuscate stack traces.

### 5. Gradle Cache Strategy

- `gradle/actions/setup-gradle@v4` caches `~/.gradle/caches` and `~/.gradle/wrapper` automatically.
- Pin the wrapper: `./gradlew wrapper --gradle-version=8.9 --distribution-type=bin` and commit the files.
- Set `org.gradle.configuration-cache=true` so task graphs are reused.

### 6. Matrix: JDK and API Level

Build with exactly the JDK you ship; for Java projects that is **JDK 17**. Do not run CI on JDK 21 for a project that will be released on 17 — byte-code differences can surface via R8.

### 7. Security

- Require status checks to pass before merge (Settings → Branches).
- Enable Dependabot for Gradle (`.github/dependabot.yml`).
- Enable CodeQL on pull requests.
- Scan keystore leaks with `gitleaks` pre-push locally.

### 8. Failure Triage

Every failing job uploads:
- `**/build/test-results/` — JUnit XML for GitHub's test annotations.
- `**/build/reports/lint-results-debug.html` — lint report.
- `**/build/reports/androidTests/connected/` — Espresso reports with screenshots.

Use `actions/upload-artifact@v4` with `if: always()` so failed jobs still publish reports.

## Checklist

- [ ] CI builds with JDK 17 Temurin; no mixed JDK versions across jobs.
- [ ] Gradle cache is configured via `gradle/actions/setup-gradle@v4`.
- [ ] Unit tests, lint, and Spotless run on every PR; instrumented tests run at least nightly.
- [ ] Keystore is base64-decoded from a secret; no keystore in git.
- [ ] Release job uploads `.aab` and `mapping.txt`.
- [ ] `concurrency: cancel-in-progress: true` prevents stale runs.
- [ ] Play Publisher uploads to the internal track automatically on tags.
