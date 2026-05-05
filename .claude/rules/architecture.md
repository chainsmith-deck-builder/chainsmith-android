# Architecture rules

This file describes the layered architecture, the UI state pattern, and the rules of engagement between layers. It is the Android-specific complement to `kotlin.md` (which covers language idioms) and `clean-code.md` (which covers function-level discipline).

Reference: developer.android.com's [Guide to app architecture](https://developer.android.com/topic/architecture). Where this file disagrees with that guide, this file wins, but those disagreements should be rare and called out explicitly.

## Layered architecture

Three layers, in order of dependency direction:

```
+----------------------+
|        UI layer      |  Compose screens, ViewModels, UiState
+----------------------+
              |
              v
+----------------------+
|  Domain layer (opt)  |  Use cases, pure models
+----------------------+
              |
              v
+----------------------+
|       Data layer     |  Repositories, remote (Retrofit) and local (Room, DataStore) sources
+----------------------+
```

Data flows up: repositories expose `Flow<T>`, ViewModels transform into `StateFlow<UiState>`, Composables render. Events flow down: Composables call lambdas, ViewModels call repository methods.

Layer rules:

- The UI layer never imports from `data/`. It depends on repository **interfaces** (declared in `domain/` or alongside the repository implementation) and Hilt provides the concrete implementation.
- The data layer never imports from `ui/`. Repositories don't know what a Composable is.
- The domain layer never imports anything Android-specific. No `Context`, no `Resources`, no Compose, no Room, no Retrofit, no `kotlinx-coroutines-android`. Just `kotlinx-coroutines-core` for `Flow` and pure Kotlin.

## When to add a domain layer

The domain layer is **optional**. Add a use case only when one of these is true:

1. The same business logic is invoked from two or more ViewModels.
2. The logic is genuinely complex — multi-source merge, non-trivial transformation, conditional fan-out — and putting it in the ViewModel would make the ViewModel hard to test.
3. The logic needs to be substituted in tests at a coarser granularity than "mock the repository."

If none of these apply, the ViewModel calls the repository directly. Don't introduce a use case to feel architectural.

Use case shape, when one is justified:

```kotlin
class GetLegalDecksForFormat @Inject constructor(
    private val decks: DeckRepository,
    @IoDispatcher private val io: CoroutineDispatcher,
) {
    operator fun invoke(format: Format): Flow<List<Deck>> = ...
}
```

One `operator fun invoke` per use case. Pure where possible. No mutable state.

## UI state pattern

Every screen has:

- A sealed `XxxUiState` type:

  ```kotlin
  sealed interface DeckListUiState {
      data object Loading : DeckListUiState
      data class Ready(val decks: List<DeckSummary>, val selectedFormat: Format) : DeckListUiState
      data class Error(val message: AppError) : DeckListUiState
  }
  ```

- A `XxxViewModel` exposing `val uiState: StateFlow<XxxUiState>`. The backing `MutableStateFlow` is private.
- A stateful screen Composable (e.g. `DeckListScreen(viewModel = hiltViewModel())`) that collects via `collectAsStateWithLifecycle()` and delegates to a stateless content Composable.
- A stateless content Composable (`DeckListContent(state, onSelectDeck, onChangeFormat)`) that takes state as a parameter and emits events via lambdas. **Stateless content composables are the unit of preview and screenshot testing.**

State hoisting rules:

- The lowest common parent that needs the state holds it. Don't hoist state above what actually uses it.
- A composable receives state and emits events; it does not reach into a ViewModel directly. The stateful wrapper at the screen root is the only place that touches the ViewModel.
- `rememberSaveable` is for transient UI state that should survive process death (scroll position, expanded sections). State the ViewModel already holds does not need `rememberSaveable`.

## ViewModel rules

- `@HiltViewModel`, dependencies via `@Inject constructor(...)`.
- No `Context`, `Resources`, `Activity`, or any Android Framework reference inside a ViewModel. UI strings are passed as resource IDs (or via a small `StringProvider` abstraction) and resolved at the Composable boundary.
- ViewModels survive configuration changes. Don't duplicate ViewModel state in `rememberSaveable` for the same data.
- A ViewModel exposes one `StateFlow<UiState>`. Multiple flows are a smell — combine into one state.
- One-shot events (navigation, snackbars) use a `SharedFlow` or a `Channel` consumed as a Flow, separate from `uiState`.
- ViewModels do not navigate. They emit a navigation event; the screen Composable consumes it and calls the `NavController`.

## Repository rules

- A repository is the single source of truth for one domain area (decks, cards, auth, user prefs).
- Repositories expose **interfaces** that the UI/domain layer depends on. The implementation is bound via a Hilt module.
- Reads that the UI observes return `Flow<T>`. Writes are `suspend fun`.
- Never expose Retrofit response types, Room entities, or generated DTOs. Map at the repository boundary.
- Errors are translated at the boundary: `IOException`, `HttpException`, `SQLiteException` become typed `AppError` values inside a `DataResult.Failure` (see `kotlin.md` for the result shape).
- Single-flight: concurrent requests for the same key collapse into one network call. `Mutex` keyed by request, or a `MutableMap<Key, Deferred<T>>` are both fine.
- Offline-first reads where the data has a local cache: emit cached value immediately, refresh in the background, emit again. Document the TTL constant per repository.

## Hilt rules

- `@HiltAndroidApp` on `ChainsmithApplication`.
- `@AndroidEntryPoint` on `MainActivity` and on any other Android entry point that uses `@Inject`.
- Modules under `di/`. One file per scope; one `@Module` per concern (`NetworkModule`, `DatabaseModule`, `RepositoryModule`, `DispatcherModule`).
- Default scope: `@InstallIn(SingletonComponent::class)` for app-wide singletons (Retrofit, OkHttpClient, Room database, DataStore instances). Use `ViewModelComponent` for per-ViewModel scoping when state must be tied to a ViewModel's lifecycle.
- Constructor injection wherever possible; `@Provides` only for types you don't own (Retrofit, OkHttp, Room).
- Qualifiers required when more than one binding for the same type exists (e.g. `@AuthOkHttpClient` for the authenticated client, `@DefaultOkHttpClient` for the unauthenticated one). Once a qualifier exists for a type, **all bindings of that type must use a qualifier**.

## Compose conventions

- Material 3 only. The theme is defined once in `ui/theme/` and applied at `MainActivity`'s root.
- Composables are pure. Side effects go in `LaunchedEffect`, `DisposableEffect`, `produceState`, or `rememberCoroutineScope`. **Don't launch coroutines inside a Composable's body** outside of these primitives.
- Stable parameters where possible. Apply `@Stable` or `@Immutable` only when measurably necessary; do not sprinkle them defensively.
- The first non-state parameter of any reusable Composable is `modifier: Modifier = Modifier`. State parameters come first, callbacks last.
- Test tags via `Modifier.testTag(...)` on every interactive element a UI test will look up. The tag is a constant in a sibling `*Tags.kt` file.
- Previews go on the stateless content Composable, not the stateful wrapper. One preview per significant state (`Loading`, `Ready`, `Error`).
- Avoid passing `MutableState` down. Pass the value plus an event lambda.

## Navigation

- Navigation Compose with type-safe routes. Define routes as `@Serializable` Kotlin classes / objects:

  ```kotlin
  @Serializable data object DeckList
  @Serializable data class DeckDetail(val deckId: String)
  ```

- A single `NavHost` lives at the root of `MainActivity`. Nested graphs are fine when a feature has its own internal navigation.
- Deep links are declared on the destination they target via `navDeepLink { uriPattern = ... }`. Validate parameters from deep links — anything from outside the app is untrusted (see `security.md`).
- ViewModels never hold a `NavController`. They emit a navigation event; the screen consumes it.

## Lifecycle and background work

- Short async work owned by a screen: `viewModelScope` (canceled when the ViewModel is cleared).
- Process-wide async work that must complete: WorkManager. Expedited only when the user is actively waiting and the rules apply.
- Foreground services for user-visible long-running tasks (active sync, large download). Never as a way to dodge background-execution limits.
- Lifecycle-aware collection: `repeatOnLifecycle(Lifecycle.State.STARTED)` or `collectAsStateWithLifecycle()`. Don't `launchWhenStarted` (deprecated patterns).

## Module strategy

Start with a single `:app` module. The starter package layout in `kotlin.md` is enough until at least one of these is true:

- Build time exceeds ~30s for an incremental change you make often.
- Three or more features share a non-UI utility that has grown beyond a single file.
- A feature needs to be feature-flagged off the main APK or built into a separate dynamic feature module.
- Two engineers are stepping on each other's `app/build.gradle.kts` weekly.

When the time comes, move toward [Now in Android](https://github.com/android/nowinandroid)'s layout: `:app`, `:core:*`, `:data:*`, `:domain`, `:feature:*`, with convention plugins under `build-logic/composite-build`. Don't half-modularize: pick a feature, extract it cleanly, learn from that, then extract the rest. The version catalog (`gradle/libs.versions.toml`) is the single source of dependency coordinates either way.

Until that day, resist the urge to modularize prematurely — the cost of a wrong split is higher than the cost of a single-module project that grows.
