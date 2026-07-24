---
name: android-compose-ui
description: Use when writing @Composable functions, UI state, events, or collecting Flow in Jetpack Compose on Android — Compose patterns, StateFlow exposure, naming conventions for UiState/Event, and collectAsStateWithLifecycle.
---

## Compose
- One `Screen` per route, split into a stateful `FeatureScreen` (collects state + events) and a stateless `FeatureContent` (renders data, emits lambdas). See Screen structure section.
- The `Screen` composable holds the scaffold and top-level layout; `Content` holds the actual UI tree.
- Extract reusable pieces into `components/` within the feature package.
- Pass lambdas down for events, never pass ViewModel references into child composables.

## State Ownership
- All application state lives in the ViewModel's `StateFlow` and is surfaced via `collectAsStateWithLifecycle()`.
- Do **not** use `remember` or `rememberSaveable` for application state — that belongs in the ViewModel.
- **Exception**: Compose-internal state the framework requires you to hold in composition. Use `remember*` for these:
  - `LazyListState`, `LazyGridState`, `ScrollState`
  - `PagerState`
  - `SnackbarHostState`
  - `TextFieldState` (foundation)
- Use `derivedStateOf` **only** in rare cases where Compose-internal state (LazyListState, ScrollState, etc.) drives a derived UI value — e.g. `val showFAB by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }`. If the derivation can happen in the ViewModel, it should.

## Flow
- Expose state as `StateFlow` from ViewModel (`_uiState` private, `uiState` public)
- Use `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initialValue)`
- Prefer `combine` / `map` / `filter` over imperative collection logic
- Collect in UI with `collectAsStateWithLifecycle()`
- One-shot side effects (navigation, toasts, snackbar triggers) flow on a `SharedFlow<FeatureEvent>` backed by a `Channel` — never on `StateFlow`.

## Screen structure
Every `[Feature]Screen.kt` follows the stateful/stateless split:

> **UDF loop**: Actions UI → VM via `onAction()` ; State VM → UI via `StateFlow` ; Events VM → UI via `SharedFlow`.

- **`FeatureScreen` composable** (stateful):
  - Takes `viewModel: FeatureViewModel` plus navigation/event callbacks as parameters.
  - Collects state: `val uiState by viewModel.uiState.collectAsStateWithLifecycle()`.
  - Collects one-shot events: `LaunchedEffect` keyed on the `SharedFlow`, wrapped in `lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) { viewModel.events.collect { handleEvent(it) } }`.
  - Delegates rendering to `FeatureContent(state = uiState, onAction = viewModel::onAction)` (complex) or `FeatureContent(state = uiState, onBack = …)` (simple).
- **`FeatureContent` composable** (stateless):
  - Accepts **plain data + lambdas** derived from the UI state — never the ViewModel, never `LocalLifecycleOwner`.
  - Contains the actual UI tree and `components/` calls.
  - Bears `@Preview` annotations (see @Preview section).
- No `@Preview` on `FeatureScreen` (it needs a ViewModel + lifecycle — not renderable in Preview).

### Actions — individual lambdas vs. `onAction`

- When `FeatureContent` has **3+ distinct user actions**, collapse them into a single `onAction: (FeatureAction) -> Unit` lambda. The ViewModel declares one entry point:
  ```kotlin
  fun onAction(action: FeatureAction) {
      when (action) {
          is FeatureAction.ItemClicked -> …
          FeatureAction.BackPressed -> …
          FeatureAction.Refresh -> …
      }
  }
  ```
  And `FeatureContent` takes one callback:
  ```kotlin
  @Composable
  fun FeatureContent(state: FeatureUiState, onAction: (FeatureAction) -> Unit) { … }
  ```
- Below 3 actions, individual lambdas are fine — no ceremony for trivial screens:
  ```kotlin
  @Composable
  fun FeatureContent(state: FeatureUiState, onBack: () -> Unit) { … }
  ```

```kotlin
@Composable
fun FeatureScreen(viewModel: FeatureViewModel, onBack: () -> Unit, onOpenDetail: (String) -> Unit) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(viewModel.events, lifecycleOwner) {
        lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.events.collect { event ->
                when (event) {
                    is FeatureEvent.NavigateBack -> onBack()
                    is FeatureEvent.OpenDetail -> onOpenDetail(event.id)
                }
            }
        }
    }
    FeatureContent(state = uiState, onAction = viewModel::onAction)
}

@Composable
fun FeatureContent(state: FeatureUiState, onAction: (FeatureAction) -> Unit) {
    val listState = rememberLazyListState()
    Scaffold { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            // ... render state
        }
    }
}
```

## Navigation
- Library: **Jetpack Navigation Compose** (`androidx.navigation:navigation-compose`).
- Declare a single `AppNavHost` composable hosting one `NavHost`, consumed from `MainActivity`'s `setContent { AppNavHost() }`.
- Each destination block is the **only** place a ViewModel is resolved and `NavController` is wired:
  ```kotlin
  composable(route = FEATURE_ROUTE) {
      FeatureScreen(
          viewModel = koinViewModel(),
          onBack = { navController.popBackStack() },
          onOpenDetail = { id -> navController.navigate("detail/$id") },
      )
  }
  ```
- Navigation actions flow via **callbacks passed in** from the NavHost — never pass `NavController` into a Screen or Content composable.
- One-shot navigation events are emitted by the VM on its `SharedFlow`, collected by the Screen, which then invokes the appropriate callback.
- Group related destinations with nested `navigation(startDestination = …)` graphs when a feature has multiple screens.

## @Preview
- Every `FeatureContent` and every `components/` composable ships a single `@PreviewLightDark` preview — auto-renders both light and dark variants:
  ```kotlin
  @PreviewLightDark
  @Composable
  private fun FeatureContentPreview() {
      AppTheme {
          FeatureContent(state = FeatureUiState.Preview, onAction = {})
      }
  }
  ```
- Previews live in the **same file** as the composable, as `private` functions — they must not leak into the public API.
- Supply representative sample state via a `companion object { val Preview = FeatureUiState(...) }` on the `UiState` class — never real data sources or ViewModels in previews.
- No `@Preview` on stateful `*Screen` composables; preview the stateless `*Content` instead. Use `AppTheme` wrapper in previews so light/dark styling renders correctly.

## Naming
- ViewModel state: `[Feature]UiState` (sealed class or data class), with a `companion object { val Preview = ... }` for previews.
- ViewModel events: `[Feature]Event` (sealed class)
- ViewModel actions (UI → VM): `[Feature]Action` (sealed class)
- Screen route constants: `[FEATURE]_ROUTE` (top-level `const val`)
- Stateful composable: `FeatureScreen`; stateless composable: `FeatureContent`