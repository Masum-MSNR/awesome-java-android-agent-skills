---
name: java-build-optimization
description: Expert guidance on optimizing a Java Android Gradle build — configuration cache, R8 optimization, resource shrinking, parallel execution, and remote build cache. Use this when build times regress or APK/AAB size bloats.
---

# Java Android Build Optimization

## Instructions

Slow builds compound: every developer pays the cost every day. Make optimization continuous, not a one-time sprint.

### 1. Gradle JVM Settings

`gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4g -XX:+UseG1GC -XX:MaxMetaspaceSize=1g -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=warn

android.useAndroidX=true
android.nonTransitiveRClass=true
android.nonFinalResIds=true
```

- `parallel` executes modules concurrently.
- `caching` enables the local build cache.
- `configuration-cache` stores the configured task graph — huge wins on incremental builds.
- `nonTransitiveRClass` shrinks each module's `R.java` and speeds up compilation.

### 2. R8 (Code Shrinking)

Enable for release only — debug builds must stay fast.

```groovy
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                          'proguard-rules.pro'
        }
    }
}
```

- `proguard-android-optimize.txt` enables full optimization passes (not just obfuscation).
- Add `-keep` rules for reflection users (Gson, Moshi, Room schemas referenced by migrations).
- Verify release builds in CI by running instrumentation tests against `assembleReleaseAndroidTest` so R8-stripped reflection bugs surface before Play.

Example `proguard-rules.pro`:

```proguard
# Keep data classes read by Gson
-keep class com.example.app.data.dto.** { *; }

# Keep Retrofit service interfaces and parameter annotations
-keep,allowobfuscation interface retrofit2.** { *; }
-keepattributes Signature, *Annotation*, RuntimeVisibleAnnotations

# Room
-keep class * extends androidx.room.RoomDatabase { *; }
-keep @androidx.room.Entity class *
```

### 3. Resource Shrinking

`shrinkResources true` removes resources that R8-stripped code no longer references. Write a `res/raw/keep.xml` if you load resources by name:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@drawable/dynamic_ic_*"
    tools:discard="@layout/debug_only_overlay" />
```

Enable `android.enableResourceOptimizations=true` (default on AGP 8) to dedupe strings across configurations.

### 4. Split APKs / App Bundles

Ship `.aab`, not `.apk`:

```groovy
android {
    bundle {
        language { enableSplit = true }
        density  { enableSplit = true }
        abi      { enableSplit = true }
    }
}
```

Typical savings: 30–60% reduction in on-device size.

### 5. Dependency Hygiene

- Run `./gradlew :app:dependencies` on every merge; reject new dependencies over 200kb without justification.
- Prefer the `-android` variant of Guava (`com.google.guava:guava:X.Y.Z-android`).
- Remove `implementation` dependencies that are no longer used — `ruler` (Spotify) reports the top contributors to APK size.
- Turn on `android.defaults.buildfeatures.buildconfig=false` globally and re-enable per module only where needed.

### 6. Incremental Annotation Processing

Most common processors (Hilt, Room, Glide) are incremental by default. Verify:

```
./gradlew assembleDebug --info | grep "non-incremental"
```

If you see non-incremental warnings, upgrade the processor or replace it. Prefer **KSP-compatible** alternatives for processors that offer it (some Room tooling).

### 7. Remote Build Cache

For teams > 3 developers, stand up a `gradle-build-cache-node` (Gradle Enterprise) or a simple HTTP cache:

```groovy
// settings.gradle
buildCache {
    local { enabled = true }
    remote(HttpBuildCache) {
        url = uri("https://cache.example.com/cache/")
        push = System.getenv('CI') == 'true'
    }
}
```

Only CI pushes — developer machines read.

### 8. Measure

- `./gradlew assembleDebug --scan` to generate a build scan URL (gradle.com).
- `--profile` produces a local HTML report under `build/reports/profile/`.
- Track `clean + assemble` and `incremental assemble` times per release in CI.

## Checklist

- [ ] `gradle.properties` has parallel, caching, and configuration-cache enabled.
- [ ] Release builds enable `minifyEnabled` and `shrinkResources`.
- [ ] `proguard-rules.pro` keeps only what reflection needs; nothing is over-kept.
- [ ] App is shipped as `.aab` with language/density/abi splits.
- [ ] `android.nonTransitiveRClass=true` is set and each module's `R` is shrunk.
- [ ] No annotation processor reports "non-incremental".
- [ ] A build scan is generated in CI and used to diagnose regressions.
