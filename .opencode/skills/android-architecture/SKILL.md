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

## 3. Multi-module Gradle architecture

Follow Google's "Now in Android" pattern — split the app into isolated Gradle modules for faster builds, feature isolation, and testability.

### Module structure

```
small-android/
├── app/                          # :app — Android application module, wiring only
├── core/
│   ├── core-common/              # :core:common — extensions, utils, base classes
│   ├── core-domain/              # :core:domain — models, repository interfaces, use cases
│   ├── core-data/                # :core:data — repository impls, local/remote, infrastructure
│   ├── core-datastore/           # :core:datastore — DataStore preferences
│   ├── core-database/            # :core:database — Room database, DAOs, entities
│   ├── core-network/             # :core:network — Retrofit/OkHttp, API services, DTOs
│   ├── core-designsystem/        # :core:designsystem — theme, shared composables, design tokens
│   ├── core-ui/                  # :core:ui — shared UI components, navigation helpers
│   └── core-di/                  # :core:di — Koin modules shared across features
├── feature/
│   ├── feature-home/             # :feature:home — Screen, ViewModel, components
│   ├── feature-detail/           # :feature:detail
│   └── feature-settings/         # :feature:settings
└── gradle/                       # Convention plugins (build-logic)
```

### Dependency rules

- **Direction**: `app` → `feature:*` → `core:*`. Features depend on `core:domain` and `core:ui`, never on other features. `core:data` depends on `core:domain`, never the reverse.
- **Feature isolation**: Each feature module is self-contained (Screen + ViewModel + components). Features communicate via navigation, not direct imports.
- **Core layering**: `core:domain` has zero Android dependencies (pure Kotlin). `core:data` implements domain interfaces. `core:ui`/`core:designsystem` for shared Compose.
- **DI per module**: Each module exposes its own Koin module. `app` aggregates all modules.
- **Build convention plugins**: Shared Gradle config in `build-logic/` to avoid duplication across modules.

### When to add a feature module

- When a feature has 3+ screens or complex navigation
- When a feature has its own data sources (API, database) that shouldn't leak into other features
- When build times become a problem (feature modules compile in parallel)

### When to keep it in `app`

- Simple apps with 1-2 screens
- Prototypes and MVPs
- When the overhead of multi-module isn't justified yet