---
name: android-build-cicd
description: Use when modifying Gradle build types/flavors, BuildConfig flags, version naming, lint/detekt config, keystore signing, Fastfile lanes, or .github/workflows on an Android/Kotlin app — debug/release + dev/staging/production flavors with SHOW_ADS gating and flavor titles, GitHub Actions workflows, Fastlane lanes.
---

## Build variants
This project uses `debug` and `release` build types, plus `dev`, `staging`, and `production` product flavors.

| Flavor | applicationIdSuffix | app_name | SHOW_ADS |
|---|---|---|---|
| `dev` | `.dev` | `Small Android (Dev)` | `false` |
| `staging` | `.staging` | `Small Android (Staging)` | `true` |
| `production` | *(none)* | `Small Android` | `true` |

Each flavor defines its app title via `resValue "string", "app_name", ...` and the ad-visibility flag via `buildConfigField "boolean", "SHOW_ADS", ...` — both set in the `productFlavors` block of `app/build.gradle.kts`. UI gates ad display on `BuildConfig.SHOW_ADS`. Ad SDK integration is handled by a separate dedicated skill; this skill only defines the flag.

### Version naming
`VERSION_CODE` and `VERSION_NAME` are read from environment variables (defaults `1` and `0.1-dev`). A `VERSION_DISPLAY` BuildConfig field is set in both `debug` and `release` build types from the resolved version name.

In CI, all workflows set these from git before building:

```yaml
- name: Set version from git
  run: |
    echo "VERSION_CODE=$(git rev-list --count HEAD)" >> $GITHUB_ENV
    TAG=$(git describe --tags --abbrev=0 --match='v*' 2>/dev/null || echo "")
    if [ -n "$TAG" ]; then
      echo "VERSION_NAME=$(echo $TAG | sed 's/^v//')" >> $GITHUB_ENV
    else
      echo "VERSION_NAME=0.1-dev" >> $GITHUB_ENV
    fi
```

`VERSION_CODE` is the git commit count; `VERSION_NAME` is the latest `v*` tag (stripped of the `v` prefix), falling back to `0.1-dev` when no tag exists.

### Signing
Release signing config is loaded from environment variables with `local.properties` fallback:
- `SIGNING_KEYSTORE_FILE`, `SIGNING_STORE_PASSWORD`, `SIGNING_KEY_ALIAS`, `SIGNING_KEY_PASSWORD`

Never commit `local.properties` or keystore files.

### Dependencies
Managed via a Gradle version catalog (`gradle/libs.versions.toml`). Plugins are referenced as `alias(libs.plugins.*)`.

### Gradle configuration
```kotlin
// app/build.gradle.kts
val localProperties = Properties().apply {
    val file = rootProject.file("local.properties")
    if (file.exists()) load(file.inputStream())
}

val versionCodeValue: Int = System.getenv("VERSION_CODE")?.toIntOrNull() ?: 1
val versionNameValue: String = System.getenv("VERSION_NAME") ?: "0.1-dev"

android {
    defaultConfig {
        versionCode = versionCodeValue
        versionName = versionNameValue
    }

    signingConfigs {
        create("release") {
            storeFile = (System.getenv("SIGNING_KEYSTORE_FILE")
                ?: localProperties.getProperty("signing.keystore.file"))?.let { rootProject.file(it) }
            storePassword = System.getenv("SIGNING_STORE_PASSWORD")
                ?: localProperties.getProperty("signing.store.password")
            keyAlias = System.getenv("SIGNING_KEY_ALIAS")
                ?: localProperties.getProperty("signing.key.alias")
            keyPassword = System.getenv("SIGNING_KEY_PASSWORD")
                ?: localProperties.getProperty("signing.key.password")
        }
    }

    buildTypes {
        debug {
            buildConfigField("String", "VERSION_DISPLAY", "\"$versionNameValue\"")
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            signingConfig = signingConfigs.getByName("release")
            buildConfigField("String", "VERSION_DISPLAY", "\"$versionNameValue\"")
        }
    }

    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            resValue("string", "app_name", "Small Android (Dev)")
            buildConfigField("boolean", "SHOW_ADS", "false")
        }
        create("staging") {
            dimension = "environment"
            applicationIdSuffix = ".staging"
            resValue("string", "app_name", "Small Android (Staging)")
            buildConfigField("boolean", "SHOW_ADS", "true")
        }
        create("production") {
            dimension = "environment"
            resValue("string", "app_name", "Small Android")
            buildConfigField("boolean", "SHOW_ADS", "true")
        }
    }

    buildFeatures {
        buildConfig = true
    }

    lint {
        abortOnError = true
        baseline = file("lint-baseline.xml")
    }
}

detekt {
    config.setFrom(rootProject.file("detekt.yml"))
    buildUponDefaultConfig = true
    autoCorrect = true
}
```

## CI/CD (GitHub Actions + Fastlane)
Three workflows handle automation:

| Trigger | Workflow | Flavor | Artifact |
|---|---|---|---|
| Pull request | `test-build.yml` | `dev` | Debug APK (7-day retention) |
| Push to `main` | `staging-build.yml` | `staging` | Signed AAB → Play Internal Testing |
| Tag `v*.*.*` | `release-build.yml` | `production` | Signed AAB → Play Production |

### GitHub Actions workflows
- **`test-build.yml`** (on PR): checkout → setup JDK → cache Gradle → `assembleDevDebug` → upload APK (7-day retention)
- **`staging-build.yml`** (on push to `main`): checkout → JDK → cache → decode signing keystore from secrets → `bundleStagingRelease` → upload AAB to Play Internal Testing
- **`release-build.yml`** (on tag `v*.*.*`): checkout → JDK → cache → decode signing keystore from secrets → `bundleProductionRelease` → upload AAB to Play Production

### Fastfile
```ruby
# Fastfile
lane :test do
  gradle(task: "test")
end

lane :build_dev_debug do
  gradle(task: "assembleDevDebug")
end

lane :build_staging_release do
  gradle(task: "assembleStagingRelease")
end

lane :build_production_release do
  gradle(task: "assembleProductionRelease")
end

lane :bundle_staging_release do
  gradle(task: "bundleStagingRelease")
end

lane :bundle_production_release do
  gradle(task: "bundleProductionRelease")
end

lane :deploy_staging do
  bundle_staging_release
  upload_to_play_store(track: "internal")
end

lane :deploy_production do
  bundle_production_release
  upload_to_play_store(track: "production", rollout: "0.1")
end
```

### CI secrets
- **Signing**: `SIGNING_KEYSTORE_FILE` (base64-encoded), `SIGNING_STORE_PASSWORD`, `SIGNING_KEY_ALIAS`, `SIGNING_KEY_PASSWORD`
- **Play upload**: `ANDROID_SERVICE_ACCOUNT_JSON_BASE64` (service account JSON for Play Console API)
- Store secrets in CI environment variables, never in the repo
- Trigger `deploy_staging` on merge to `main`
- Trigger `deploy_production` on version tag `v*.*.*` after internal validation