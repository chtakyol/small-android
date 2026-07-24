---
name: android-koin-di
description: Use when adding ViewModels or repositories to Koin DI, configuring network/Room modules, or modifying di/ on an Android/Kotlin app — Koin module templates and registration patterns.
---

## Koin DI setup

```kotlin
// di/AppModule.kt — placeholder structure
val appModule = module {
    // ViewModels
    viewModel { [FeatureViewModel](get()) }
}

val dataModule = module {
    // Repositories
    single<[FeatureRepository]> { [FeatureRepositoryImpl](get(), get()) }
    // Data sources
    single { [FeatureLocalDataSource](get()) }
    single { [FeatureRemoteDataSource](get()) }
}

val networkModule = module {
    // HTTP client, API service
}

val databaseModule = module {
    // Room database, DAOs, entities — for domain/business data
    single { Room.databaseBuilder(get(), AppDatabase::class.java, "app.db").build() }
    single { get<AppDatabase>().[featureDao]() }
}

val preferencesModule = module {
    // DataStore (Preferences) — for user preferences and simple flags
    single { PreferencesDataStoreFactory.create(get()) }
}
```

## Notes
- ViewModel state exposure (`StateFlow`, `SharedFlow`) belongs to the UI layer, not DI. See the `android-compose-ui` skill.
- Repository interfaces live in `domain/repository`; their implementations are registered here in `dataModule`.
- Keep HTTP client and Room database configuration isolated in their own modules so they can be swapped or tested independently.
- Use `databaseModule` for Room (domain/business data with relationships); use `preferencesModule` for DataStore (user preferences, flags). Never mix the two in a single module.