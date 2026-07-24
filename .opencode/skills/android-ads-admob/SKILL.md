---
name: android-ads-admob
description: Use when integrating Google AdMob banner, interstitial, or rewarded ads on an Android/Kotlin Compose app — MobileAds init, BuildConfig per-flavor ad unit IDs, AndroidView banner composable, AdManager singleton caching Interstitial/Rewarded, and remember* controller composables for one-line ad display.
---

## Overview
Google Mobile Ads SDK (AdMob) provides banner (inline adaptive), interstitial, and rewarded ad surfaces. All ad surfaces must gate on `BuildConfig.SHOW_ADS` (the `dev` flavor is `false`). The SDK is initialized once in `Application.onCreate`; UMP (User Messaging Platform) consent flow is deferred and noted as a TODO.

This skill wires the SDK so ads are ready to use but **not placed anywhere**. Developers add a real impression with one line — a `AdmobBanner()` composable, a `controller.show(context)` call — near the relevant UI.

## Gradle / BuildConfig wiring

### Version catalog (`gradle/libs.versions.toml`)
```toml
[versions]
playServicesAds = "23.6.0"

[libraries]
play-services-ads = { group = "com.google.android.gms", name = "play-services-ads", version.ref = "playServicesAds" }
```

### `app/build.gradle.kts` — ad unit IDs per flavor
Add three `buildConfigField "String"` entries to each `productFlavors` block. Use Google's standard test ad unit IDs for `dev` and `staging`; real IDs for `production`. Keep `SHOW_ADS` as defined in the `android-build-cicd` skill.

```kotlin
// app/build.gradle.kts (additions to the existing productFlavors block)
productFlavors {
    create("dev") {
        // existing fields…
        buildConfigField("String", "ADMOB_APP_ID", "\"ca-app-pub-3940256099942544~3347511713\"")
        buildConfigField("String", "BANNER_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/6300978111\"")
        buildConfigField("String", "INTERSTITIAL_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/1031126224\"")
        buildConfigField("String", "REWARDED_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/5224354917\"")
    }
    create("staging") {
        // existing fields…
        buildConfigField("String", "ADMOB_APP_ID", "\"ca-app-pub-3940256099942544~3347511713\"")
        buildConfigField("String", "BANNER_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/6300978111\"")
        buildConfigField("String", "INTERSTITIAL_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/1031126224\"")
        buildConfigField("String", "REWARDED_AD_UNIT_ID", "\"ca-app-pub-3940256099942544/5224354917\"")
    }
    create("production") {
        // existing fields…
        buildConfigField("String", "ADMOB_APP_ID", "\"<production-AdMob-app-id>\"")
        buildConfigField("String", "BANNER_AD_UNIT_ID", "\"<production-banner-unit-id>\"")
        buildConfigField("String", "INTERSTITIAL_AD_UNIT_ID", "\"<production-interstitial-unit-id>\"")
        buildConfigField("String", "REWARDED_AD_UNIT_ID", "\"<production-rewarded-unit-id>\"")
    }
}
```

> Replace the `<production-*>` placeholders with real AdMob unit IDs before release. Never commit real production unit IDs outside the `production` flavor config.

## AndroidManifest (`app/src/main/AndroidManifest.xml`)
```xml
<application
    ...>

    <!-- AdMob app ID (flavor-specific via BuildConfig → resValue fallback to meta-data) -->
    <meta-data
        android:name="com.google.android.gms.ads.APPLICATION_ID"
        android:value="${ADMOB_APP_ID}" />
</application>
```

Inject the flavor value via manifest placeholder in `productFlavors`:
```kotlin
create("dev") {
    // …
    manifestPlaceholders["ADMOB_APP_ID"] = "ca-app-pub-3940256099942544~3347511713"
}
create("staging") {
    // …
    manifestPlaceholders["ADMOB_APP_ID"] = "ca-app-pub-3940256099942544~3347511713"
}
create("production") {
    // …
    manifestPlaceholders["ADMOB_APP_ID"] = "<production-AdMob-app-id>"
}
```

> No `<activity>` entry for `com.google.android.gms.ads.AdActivity` — the SDK declares it merged from the library manifest.

## SDK initialization (`Application.onCreate`)
```kotlin
// App.kt — extends Application
import com.google.android.gms.ads.MobileAds
import com.google.android.gms.ads.RequestConfiguration

class App : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.SHOW_ADS) {
            MobileAds.initialize(this)
            // Mark this device as a test device so test ads always serve in dev/staging.
            // On the `production` flavor, remove/replace these test IDs with real test device IDs
            // captured from Logcat the first time a real ad loads.
            MobileAds.setRequestConfiguration(
                RequestConfiguration.Builder()
                    .setTestDeviceIds(listOf("TEST_DEVICE_ID")) // replace per device
                    .build()
            )
        }
        // TODO: integrate Google UMP (User Messaging Platform) consent flow before show,
        //       and gate `MobileAds.initialize` behind `canRequestAds()`. Deferred — see
        //       https://developers.google.com/admob/android/privacy/java for the recipe.
    }
}
```

Koin is started in `App.onCreate` as described by the `android-koin-di` skill; add `adsModule` to the modules list.

## AdManager (singleton, Koin)
A `single`-scoped cache that pre-loads one interstitial and one rewarded ad, surfaces `show(context)` calls, and auto-reloads after dismissal. No-ops when `BuildConfig.SHOW_AADS` is false.

```kotlin
// data/infrastructure/ads/AdManager.kt
import android.app.Activity
import android.content.Context
import com.google.android.gms.ads.LoadAdError
import com.google.android.gms.ads.interstitial.InterstitialAd
import com.google.android.gms.ads.interstitial.InterstitialAdLoadCallback
import com.google.android.gms.ads.rewarded.RewardedAd
import com.google.android.gms.ads.rewarded.RewardedAdLoadCallback

class AdManager(context: Context) {

    private val appContext = context.applicationContext

    private var interstitialAd: InterstitialAd? = null
    private var rewardedAd: RewardedAd? = null

    init {
        // Preload at construction so the first user-facing call is near-instant.
        // Update the test device list in Application.onCreate to get demo creatives.
        if (BuildConfig.SHOW_ADS) {
            preloadInterstitial()
            preloadRewarded()
        }
    }

    // ---------- Interstitial ----------
    fun preloadInterstitial() {
        if (!BuildConfig.SHOW_ADS) return
        InterstitialAd.load(
            /* context = */ appContext,
            /* adUnitId = */ BuildConfig.INTERSTITIAL_AD_UNIT_ID,
            /* adRequest = */ com.google.android.gms.ads.AdRequest.Builder().build(),
            /* loadCallback = */ object : InterstitialAdLoadCallback() {
                override fun onAdLoaded(ad: InterstitialAd) {
                    interstitialAd = ad
                }

                override fun onAdFailedToLoad(error: LoadAdError) {
                    interstitialAd = null
                }
            }
        )
    }

    fun showInterstitial(activity: Activity, onDismissed: () -> Unit = {}) {
        val ad = interstitialAd
        if (ad != null) {
            ad.fullScreenContentCallback =
                object : com.google.android.gms.ads.FullScreenContentCallback() {
                    override fun onAdDismissedFullScreenContent() {
                        interstitialAd = null
                        preloadInterstitial()
                        onDismissed()
                    }
                }
            ad.show(activity)
        } else {
            preloadInterstitial()
            onDismissed() // graceful no-op when not yet loaded
        }
    }

    // ---------- Rewarded ----------
    fun preloadRewarded() {
        if (!BuildConfig.SHOW_ADS) return
        RewardedAd.load(
            /* context = */ appContext,
            /* adUnitId = */ BuildConfig.REWARDED_AD_UNIT_ID,
            /* adRequest = */ com.google.android.gms.ads.AdRequest.Builder().build(),
            /* loadCallback = */ object : RewardedAdLoadCallback() {
                override fun onAdLoaded(ad: RewardedAd) {
                    rewardedAd = ad
                }

                override fun onAdFailedToLoad(error: LoadAdError) {
                    rewardedAd = null
                }
            }
        )
    }

    /**
     * @param onReward Invoked on the main thread after the user earns the reward.
     */
    fun showRewarded(activity: Activity, onReward: () -> Unit, onDismissed: () -> Unit = {}) {
        val ad = rewardedAd
        if (ad != null) {
            ad.fullScreenContentCallback =
                object : com.google.android.gms.ads.FullScreenContentCallback() {
                    override fun onAdDismissedFullScreenContent() {
                        rewardedAd = null
                        preloadRewarded()
                        onDismissed()
                    }
                }
            ad.show(activity) { onReward() }
        } else {
            preloadRewarded()
            onDismissed() // graceful no-op when not yet loaded
        }
    }
}
```

### Koin `adsModule`
```kotlin
// di/AdsModule.kt
import org.koin.dsl.module

val adsModule = module {
    single { AdManager(get()) }
}
```

Add `adsModule` to the `modules(...)` list in `startKoin { }` (inside `App.onCreate`).

## Banner composable — inline adaptive
One composable, `AndroidView`-backed, auto-hides when `SHOW_ADS` is false or when the ad fails to load. Adaptive size is computed against the current orientation anchored width.

```kotlin
// ui/ads/AdmobBanner.kt
import android.view.ViewGroup
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.viewinterop.AndroidView
import com.google.android.gms.ads.AdSize
import com.google.android.gms.ads.AdView

@Composable
fun AdmobBanner(modifier: Modifier = Modifier) {
    if (!BuildConfig.SHOW_ADS) return

    val context = LocalContext.current
    val adView = remember {
        AdView(context).apply {
            setAdSize(
                AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize(
                    context,
                    /* anchored width in px: full screen width works for inline banners */
                    context.resources.displayMetrics.widthPixels,
                )
            )
            adUnitId = BuildConfig.BANNER_AD_UNIT_ID
            layoutParams = ViewGroup.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.WRAP_CONTENT,
            )
        }
    }

    AndroidView(
        factory = {
            adView.apply { loadAd(com.google.android.gms.ads.AdRequest.Builder().build()) }
        },
        modifier = modifier,
        onRelease = { adView.destroy() },
    )
}
```

**Usage (any screen, one line):**
```kotlin
AdmobBanner(Modifier.fillMaxWidth())
```
Common placement is at the bottom of a `Scaffold` via the `bottomBar` slot, or inline below a list. The composable is stateless and safe to recompose; the `AdView` is `remember`-ed so it does not reload on every recomposition.

## Interstitial controller — one-line `show`
A `remember`-backed controller exposing `show(activity)`. Preload is automatic on first composition of any screen that calls `rememberInterstitialAdController()`; `show` is a no-op + `preload` if the cache is empty.

```kotlin
// ui/ads/InterstitialAdController.kt
import android.app.Activity
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
import org.koin.compose.koinInject

class InterstitialAdController(
    private val adManager: AdManager,
    private val activity: Activity,
) {
    fun show(onDismissed: () -> Unit = {}) = adManager.showInterstitial(activity, onDismissed)
}

@Composable
fun rememberInterstitialAdController(): InterstitialAdController {
    val adManager: AdManager = koinInject()
    val context = LocalContext.current
    // LocalContext in a Compose Activity is the Activity itself.
    val activity = remember(context) { context as Activity }
    return remember(adManager, activity) {
        InterstitialAdController(adManager, activity)
    }
}
```

**Usage:**
```kotlin
val interstitial = rememberInterstitialAdController()
// Preload already happened at AdManager init — just call show at the trigger:
Button(onClick = { interstitial.show { /* continue */ } }) { Text("Continue") }
```

## Rewarded controller — one-line `show` with reward callback
Same shape as interstitial; the `onReward` lambda is supplied at call site.

```kotlin
// ui/ads/RewardedAdController.kt
import android.app.Activity
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
import org.koin.compose.koinInject

class RewardedAdController(
    private val adManager: AdManager,
    private val activity: Activity,
) {
    fun show(onReward: () -> Unit, onDismissed: () -> Unit = {}) =
        adManager.showRewarded(activity, onReward, onDismissed)
}

@Composable
fun rememberRewardedAdController(): RewardedAdController {
    val adManager: AdManager = koinInject()
    val context = LocalContext.current
    val activity = remember(context) { context as Activity }
    return remember(adManager, activity) {
        RewardedAdController(adManager, activity)
    }
}
```

**Usage:**
```kotlin
val rewarded = rememberRewardedAdController()
Button(onClick = {
    rewarded.show(
        onReward = { viewModel.onAction(GameAction.GrantCoins) },
        onDismissed = { /* resume game */ }
    )
}) { Text("Watch ad for coins") }
```

## One-line usage summary

| Ad surface | One line to display | Notes |
|---|---|---|
| Banner | `AdmobBanner(Modifier.fillMaxWidth())` | Self-contained composable; place anywhere in the tree. `dev` flavor renders nothing. |
| Interstitial | `rememberInterstitialAdController().show { }` | Pre-cached by `AdManager`; `show` no-ops if not loaded. Pass an Activity-scoped composable scope. |
| Rewarded | `rememberRewardedAdController().show(onReward = { }, onDismissed = { })` | Reward callback fires after the user completes the ad. |

## Conventions
- **Gating**: every surface (`AdmobBanner`, `AdManager.preload*`, `AdManager.show*`) checks `BuildConfig.SHOW_ADS` internally. Never duplicate the check at call sites.
- **Lifecycle**: the `AdView` is `remember`-ed in `AdmobBanner` and released via `AndroidView.onRelease` to avoid leaks. Every `show` callback triggers a background reload so the next impression is ready.
- No business logic in ad composables — use the controllers/`AdManager` for state. Composables only bind UI to ad lifecycle.
- Never pass `AdView`, `InterstitialAd`, or `RewardedAd` instances across recompositions; obtain them from a remembered controller or `AdManager`.
- `AdManager` is `single` scoped (app-wide cache); the `remember*Controller` composables return an Activity-scoped wrapper so `show()` has a valid `Activity`.
- ProGuard/R8 rules ship inside `play-services-ads` consumer rules; nothing extra needed in `proguard-rules.pro`.
- Replace the `TEST_DEVICE_ID` in `App.onCreate` (and add real production unit IDs to `production` flavor) before release. Discover the test device ID from the Logcat line emitted the first time an ad loads.
- UMP consent flow is **not** wired here; before public release, gate `MobileAds.initialize` behind `ConsentInformation.canRequestAds()` per the recipe at https://developers.google.com/admob/android/privacy/java.

## Package layout (under the base package)
```
com.[package].[appname]
├── data
│   └── infrastructure
│       └── ads
│           └── AdManager.kt
└── ui
    └── ads
        ├── AdmobBanner.kt
        ├── InterstitialAdController.kt
        └── RewardedAdController.kt
```
`di/AdsModule.kt` sits next to the other Koin modules. `Application.onCreate` init lives in the existing `App` class.