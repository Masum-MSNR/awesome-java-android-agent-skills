---
name: gradle-groovy
description: Expert guidance on writing and maintaining Android Gradle build files in Groovy DSL for a Java project — product flavors, signing configs, build types, and dependency management. Use this when configuring build.gradle for new modules or flavors.
---

# Android Gradle (Groovy DSL)

## Instructions

Groovy DSL (`build.gradle`) is acceptable; Kotlin DSL (`build.gradle.kts`) is preferred for new modules because of IDE autocomplete and type safety. This skill targets Groovy because many Java projects inherited it from older AGP versions.

### 1. Root `build.gradle`

```groovy
buildscript {
    ext {
        agpVersion     = '8.5.2'
        javaVersion    = JavaVersion.VERSION_17
        minSdkVersion  = 21
        targetSdkVersion = 35
        compileSdkVersion = 35
    }
}

plugins {
    id 'com.android.application' version "$agpVersion" apply false
    id 'com.android.library'     version "$agpVersion" apply false
    id 'com.google.dagger.hilt.android' version '2.52'  apply false
    id 'androidx.navigation.safeargs'    version '2.8.0' apply false
}

tasks.register('clean', Delete) {
    delete rootProject.layout.buildDirectory
}
```

### 2. Module `build.gradle`

```groovy
plugins {
    id 'com.android.application'
    id 'com.google.dagger.hilt.android'
    id 'androidx.navigation.safeargs'
}

android {
    namespace 'com.example.app'
    compileSdk rootProject.compileSdkVersion

    defaultConfig {
        applicationId "com.example.app"
        minSdk rootProject.minSdkVersion
        targetSdk rootProject.targetSdkVersion
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "com.example.app.HiltTestRunner"
        resourceConfigurations += ['en', 'es', 'fr']
    }

    compileOptions {
        sourceCompatibility rootProject.javaVersion
        targetCompatibility rootProject.javaVersion
    }

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(17)
        }
    }

    buildFeatures {
        viewBinding true
        buildConfig true
    }
}
```

`namespace` is required on AGP 8. Do not duplicate it in `AndroidManifest.xml`.

### 3. Build Types

```groovy
android {
    buildTypes {
        debug {
            applicationIdSuffix '.debug'
            versionNameSuffix '-DEBUG'
            buildConfigField 'String', 'API_BASE_URL', '"https://api-staging.example.com/"'
        }
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                          'proguard-rules.pro'
            signingConfig signingConfigs.release
            buildConfigField 'String', 'API_BASE_URL', '"https://api.example.com/"'
        }
    }
}
```

### 4. Product Flavors

```groovy
android {
    flavorDimensions 'tier', 'env'

    productFlavors {
        free {
            dimension 'tier'
            applicationIdSuffix '.free'
            buildConfigField 'boolean', 'IS_PREMIUM', 'false'
        }
        premium {
            dimension 'tier'
            applicationIdSuffix '.premium'
            buildConfigField 'boolean', 'IS_PREMIUM', 'true'
        }
        staging {
            dimension 'env'
            buildConfigField 'String', 'ANALYTICS_KEY', '"stg-xxx"'
        }
        production {
            dimension 'env'
            buildConfigField 'String', 'ANALYTICS_KEY', '"prod-xxx"'
        }
    }
}
```

Variants: `freeStagingDebug`, `premiumProductionRelease`, etc.

Per-flavor resources:

```
src/premium/res/drawable/ic_app_logo.xml
src/staging/res/values/strings.xml
```

### 5. Signing

Never commit keystores or passwords. Read from Gradle properties / env vars.

```groovy
android {
    signingConfigs {
        release {
            storeFile     file(System.getenv('KEYSTORE_PATH') ?: 'release.keystore')
            storePassword System.getenv('KEYSTORE_PASSWORD')
            keyAlias      System.getenv('KEY_ALIAS')
            keyPassword   System.getenv('KEY_PASSWORD')
        }
    }
}
```

In CI, inject via secrets. Local developers set values in `~/.gradle/gradle.properties`.

### 6. Dependency Declarations

Prefer `libs.versions.toml` version catalog even with Groovy:

```toml
# gradle/libs.versions.toml
[versions]
retrofit = "2.11.0"
okhttp   = "4.12.0"
moshi    = "1.15.1"
hilt     = "2.52"

[libraries]
retrofit     = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-moshi = { module = "com.squareup.retrofit2:converter-moshi", version.ref = "retrofit" }
okhttp       = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
moshi        = { module = "com.squareup.moshi:moshi", version.ref = "moshi" }
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
```

```groovy
dependencies {
    implementation libs.retrofit
    implementation libs.retrofit.moshi
    implementation libs.okhttp
    implementation libs.moshi
    implementation libs.hilt.android
    annotationProcessor libs.hilt.compiler
}
```

### 7. `buildSrc` or Convention Plugins

Extract repeated module configuration into `buildSrc/src/main/groovy/android-app.gradle` so each module is a one-liner:

```groovy
// app/build.gradle
plugins {
    id 'android-app'
}
```

### 8. Common Pitfalls

- Do not upgrade AGP without reading the release notes; compatibility with Gradle, JDK, and Kotlin is strict.
- `packagingOptions { resources.excludes += ['META-INF/*.md'] }` — keep this explicit, not copy-pasted.
- Test dependencies must not leak into `implementation`; use `testImplementation` / `androidTestImplementation`.
- When multiple flavors share a large block, use `productFlavors.all { ... }`.

## Checklist

- [ ] AGP, Gradle, JDK versions are aligned with a known-good matrix.
- [ ] `namespace` is set in every module's `build.gradle`.
- [ ] Java source/target compatibility and toolchain are both set to 17.
- [ ] Keystore and passwords are not in source control.
- [ ] A version catalog (`libs.versions.toml`) is the single source of truth for versions.
- [ ] Flavor-specific code lives in `src/<flavor>/` folders, not behind `if (BuildConfig.FLAVOR)` branches.
- [ ] Release build runs `minifyEnabled true` and `shrinkResources true`.
