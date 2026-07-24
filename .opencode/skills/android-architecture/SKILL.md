---
name: android-architecture
description: Use when scaffolding a new feature or making cross-layer architecture decisions on an Android/Kotlin app — Clean Architecture Data/Domain/UI/Infrastructure layering, UDF, SSOT, domain UseCase rules, and package structure.
---

## 1. Architecture rules
- **Pattern**: Clean Architecture — Data / Domain / UI / Infrastructure layers
- **State**: Unidirectional data flow (UDF). UI observes state; it never mutates it directly.
- **SSOT**: Single source of truth lives in the Repository. ViewModel never holds its own copy of remote data.
- **Domain layer**: Only introduce `UseCase` classes when business logic is non-trivial or shared across multiple ViewModels. Skip for simple pass-throughs.
- **Side effects**: Handle via `Channel` + `SharedFlow` (one-shot events), not `StateFlow`.
- **Infrastructure layer**: Concrete framework adapters (location, sensors, permissions, Bluetooth) implement interfaces declared in `domain`. No business logic — only platform API calls and permission gating. e.g. `FusedLocationProviderImpl` wraps `FusedLocationProviderClient`; `domain` owns the `LocationProvider` interface. Lives under `data/infrastructure` alongside local/remote sources.
- **Persistence split**: Use Room for domain/business data — structured entities with relationships, queryable. Use DataStore (Preferences) for user preferences and simple flags — theme, locale, onboarding state. Never store domain models in DataStore; never store preferences in Room.
- **No business logic in Composables.** Composables are dumb — they render state and emit events.

```
com.[package].[appname]
├── data
│   ├── infrastructure  # Framework adapters (location, sensors, permissions)
│   │   ├── location    # FusedLocationProviderImpl, permission gating
│   │   ├── sensors     # Sensor adapters
│   │   └── permissions # Permission request orchestration
│   ├── local
│   │   ├── database    # Room database, DAOs, entities
│   │   └── preferences # DataStore (Preferences), settings keys
│   ├── remote          # API service, DTOs
│   └── repository      # Repository implementations
├── domain
│   ├── model           # Domain models (not DTOs)
│   ├── repository      # Repository interfaces
│   ├── provider        # Port interfaces (LocationProvider, SensorProvider)
│   └── usecase         # UseCases (only when justified)
├── ui
│   ├── [feature]
│   │   ├── [Feature]Screen.kt
│   │   ├── [Feature]ViewModel.kt
│   │   └── components
│   └── theme           # Color, Type, Shape, Theme
├── di                  # Koin modules
└── util                # Extensions, helpers
```

> Dependencies always point inward: `data/infrastructure` implements interfaces declared in `domain` (ports). Repositories may depend on infrastructure providers, never the reverse.

## 2. Kotlin conventions
- Prefer `data class` for models, `sealed class` for UI state and events
- Use `object` for singletons and companion factories
- Scope coroutines to `viewModelScope` in ViewModel, `lifecycleScope` in Activity/Fragment
- Favour `extension functions` over utility classes