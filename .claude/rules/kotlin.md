# Kotlin rules

Project-specific Kotlin rules for this Android client. Architecture-level rules (UI layer, ViewModel, repositories, navigation) live in `architecture.md`. Generic clean-code preferences live in `clean-code.md`. Where they overlap, the more specific wins (this file > `clean-code.md` for Kotlin idioms; `architecture.md` > this file for layer-shape questions).

## Library choices

The default library set is fixed for this project. Do not introduce a competing library without flagging it for review.

- UI: `androidx.compose.*` (BOM-pinned), Material 3 (`androidx.compose.material3`), accompanist only when a gap is real and current
- DI: Hilt (`com.google.dagger:hilt-android`, KSP processor)
- Async: `kotlinx-coroutines-core`, `kotlinx-coroutines-android`
- HTTP: Retrofit, OkHttp (with logging interceptor in debug only)
- Serialization: `kotlinx-serialization-json`, with the Retrofit converter
- Generated API client: `openapi-generator-cli --generator-name kotlin -p library=jvm-retrofit2,serializationLibrary=kotlinx_serialization`
- Persistence: Room (KSP), DataStore Preferences, Tink for AEAD on the encrypted DataStore
- Image loading: Coil (`io.coil-kt.coil3:coil-compose`)
- Navigation: Navigation Compose with type-safe serializable routes
- Logging: Timber
- Testing: JUnit 4, MockK, Turbine, `kotlinx-coroutines-test`, AssertK or Truth, MockWebServer
- Static analysis: ktlint (Spotless or the standalone Gradle plugin), detekt, Android Lint

If a need arises for a library not on this list, propose it with a one-line justification, a link to its docs, and the maintenance signal (recent releases, issue count, ownership).

## Error handling

- Expected failures use a sealed result type, not exceptions. The default shape:

  ```kotlin
  sealed interface DataResult<out T> {
      data class Success<T>(val value: T) : DataResult<T>
      data class Failure(val error: AppError) : DataResult<Nothing>
  }
  ```

  `AppError` is a sealed hierarchy mirroring the backend's `error.code` enum (see `api-contract.md`) plus client-only variants (`Offline`, `Unauthenticated`, `RateLimited`).
- Throw only for programmer errors (failed invariants, illegal states). Wrap library throwables at the data-layer boundary; never let an `IOException` or `HttpException` leak past a repository.
- No `!!` outside genuinely unreachable paths. If you must use `!!`, leave a comment explaining why.
- Don't swallow throwables. Log via Timber with structured context, then return a `Failure` or rethrow as a wrapped programmer error.
- UI surfaces errors via the screen's `UiState.Error` variant. Composables never display raw exception messages; they display strings derived from `AppError`.

## Naming

- Classes and types: `PascalCase`
- Functions, properties, parameters, local variables: `camelCase`
- Compile-time constants: `SCREAMING_SNAKE_CASE`
- Packages: `lower.case` with no underscores
- Composables: PascalCase nouns or imperative verbs (`DeckListScreen`, `ConfirmDialog`, `SubmitButton`)
- Wire DTOs: end in `Dto`, `Request`, or `Response` (`CreateDeckRequest`, `DeckDto`)
- Room entities: end in `Entity` (`DeckEntity`, `CardEntity`)
- Domain models: no suffix (`Deck`, `Card`, `Hero`)
- DAOs: end in `Dao` (`DeckDao`)
- Hilt qualifiers: PascalCase annotation classes (`@AuthOkHttpClient`, `@IoDispatcher`)
- Test functions: backtick-quoted English sentences are encouraged (`` `emits Loading then Success when query succeeds`() ``)

## Coroutines and Flow

- Launch only inside a defined scope: `viewModelScope` from `androidx.lifecycle.viewModelScope`, `lifecycleScope` from `androidx.lifecycle.lifecycleScope`. **Never `GlobalScope`.** A coroutine that outlives its conceptual owner is a bug.
- Suspend functions are main-safe by contract: callers can call them from `Dispatchers.Main` without freezing the UI. Switch with `withContext(Dispatchers.IO)` for blocking IO and `withContext(Dispatchers.Default)` for CPU-bound work inside the suspend function itself.
- Dispatchers are injected, never hard-coded. Use named qualifiers:

  ```kotlin
  @Qualifier annotation class IoDispatcher
  @Qualifier annotation class DefaultDispatcher
  @Qualifier annotation class MainDispatcher
  ```

  Tests substitute a `StandardTestDispatcher` from `kotlinx-coroutines-test`.
- Hot UI state uses `StateFlow` exposed as a read-only `StateFlow<UiState>` from the ViewModel. The backing `MutableStateFlow` is private.
- One-shot events (navigation, snackbars) use `SharedFlow` with `replay = 0` and `extraBufferCapacity = 1`, or a `Channel(Channel.BUFFERED)` consumed as `receiveAsFlow()`. Never put one-shot events on `StateFlow`.
- `combine`, `flatMapLatest`, and `stateIn` over manual collection. `stateIn(scope, SharingStarted.WhileSubscribed(5_000), initial)` is the default for derived UI state — the 5s timeout matches the lifecycle's stop delay so a config change does not retrigger upstream collection.
- Cold flows for one-shot reads, hot flows for shared state. A `suspend fun` is the right shape when there is exactly one value and one read. Don't return `Flow<T>` to be tidy.
- Cancellation is cooperative. Honor it: avoid `runBlocking` and `NonCancellable` except for cleanup that must complete.

## Logging

- Use Timber. Tag is the file's class by default (Timber's debug tree handles this). Never `println` or `Log.d` directly in committed code.
- In release, plant a no-op tree. The release `Application` looks roughly like:

  ```kotlin
  if (BuildConfig.DEBUG) Timber.plant(Timber.DebugTree())
  ```

- Never log PII, tokens, refresh tokens, full deck contents, or full collections. If you need a correlation handle, log a short stable hash prefix derived from the value.
- Log at the right level: `v` for noisy traces, `d` for development, `i` for state transitions worth keeping, `w` for recoverable issues, `e` for errors. Don't `e()` something that's expected.

## Compiler and language settings

- Kotlin 2.x with the K2 compiler.
- `kotlinOptions { allWarningsAsErrors = true; jvmTarget = "17" }` (or the Kotlin 2.x equivalent on the `kotlin { compilerOptions { ... } }` DSL).
- Java toolchain: 17.
- Explicit API mode: enable for any library module that exists later. The starter `:app` module does not need it.
- `-Xjsr305=strict` for stricter nullness on Java interop.
- Treat ktlint and detekt findings as errors in CI; warnings during local development are fine.

## Package layout (single-module starter)

```
app/src/main/kotlin/com/chainsmith/android/
    ChainsmithApplication.kt        # @HiltAndroidApp, Timber init
    MainActivity.kt                 # @AndroidEntryPoint, single Activity hosting NavHost
    ui/
        theme/                      # Material 3 theme, typography, color
        component/                  # reusable Composables (CardThumbnail, PitchBadge, ...)
        deck/                       # feature: deck list, deck detail, deck editor
            DeckListScreen.kt
            DeckListViewModel.kt
            DeckListUiState.kt
        card/                       # feature: card catalog, card detail
        auth/                       # feature: sign-in flow
        nav/                        # NavHost setup, route definitions
    domain/
        model/                      # pure domain types (Deck, Card, Hero, Format, Violation)
        usecase/                    # use cases when justified (see architecture.md)
    data/
        api/                        # generated client wrapper, error mapping
        local/
            entity/                 # Room entities
            dao/                    # Room DAOs
            ChainsmithDatabase.kt
        prefs/                      # DataStore wrappers (UI prefs, encrypted auth)
        repository/                 # repositories, one per tag/domain area
    di/                             # Hilt modules
    util/                           # small, reluctant
```

Rules about this layout:

- `domain/` is pure: no `android.*` imports, no Compose, no Room, no Retrofit. It can be unit-tested without Robolectric or an emulator.
- The validation engine **lives in the backend** (`chainsmith`'s `domain/`). This client renders the backend's `Violation` results; it does not re-implement format legality. If you find yourself writing legality checks here, stop and call the API.
- `data/` types do not leak past the repository boundary. Room entities and generated API DTOs are converted to domain models inside the repository.
- `ui/` may depend on `domain/` and on a repository interface, but never directly on `data/` implementations. Wire concrete implementations via Hilt.
- `util/` is a smell. Prefer growing a sibling package over a generic utilities bin.

## Style nits

- Prefer `when` over chained `if/else if` for three or more branches.
- Prefer sequence and collection chains for transformations of three or fewer steps; reach for `Sequence` only when the data is large or the chain is long.
- Avoid one-letter names except in lambdas over short-lived values.
- Trailing commas in multi-line argument lists. ktlint enforces this.
- Top-level declarations only when the function is genuinely free-standing (parsers, extension functions). Otherwise put it on the class that owns the data.
- Composables' first parameter that is not state-related is `modifier: Modifier = Modifier`. State arguments come before callbacks; callbacks are last.
- Don't use `@Composable` extension functions on `Modifier` to read state — it breaks composition tracking. Use a plain composable.
- Comments explain why, not what. The code shows what.
