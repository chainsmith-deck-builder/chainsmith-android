# Testing rules

Strict testing discipline applies regardless of phase. This is not negotiable.

## What "tested" means

A function, ViewModel reducer, repository, or screen is tested when it has:

1. At least one happy-path test confirming it does what its name says.
2. At least one negative-path test confirming it fails the way it should when given bad input.
3. Variant coverage for every meaningfully distinct input class. "It works for one input" is not coverage.

Snapshot and screenshot tests do not count as coverage on their own. They confirm the output has not changed, not that it is correct. A snapshot or Paparazzi screenshot is acceptable as a regression guard alongside real assertions, never instead of them.

## When a test fails

A failing test is a finding, not a chore. Investigate before rewriting it.

The order of operations:

1. Read what the test is asserting and what actually happened.
2. Decide whether the production code is wrong or the test is wrong.
3. If the production code is wrong, fix the production code. Do not weaken the assertion to make the test pass.
4. If the test is wrong, fix the test, and add a comment explaining what the test was originally trying to assert and why the new version is correct.
5. If you cannot tell which is wrong, stop and ask before changing either.

Never delete a failing test to "clean up" without an explicit, documented reason.

## Test layout

- **Unit tests** live under `app/src/test/kotlin/` and run on the JVM with no Android dependencies. Mappers, use cases, ViewModel state reducers, repository logic with fakes, kotlinx-serialization parsing, and Retrofit-via-MockWebServer plumbing all go here.
- **Instrumented tests** live under `app/src/androidTest/kotlin/` and run on a connected device or emulator. Compose UI tests, Room DAO tests with in-memory Room, Hilt-wired integration tests go here.
- **Domain logic** in `domain/` is tested with pure unit tests. No Robolectric, no emulator. If a domain test needs Android, the domain has leaked.
- **Repositories** are tested in `app/src/test/` with fake data sources. Tests of cache strategies should drive the test dispatcher's clock to assert TTL behavior.
- **DAOs and migrations** are tested in `app/src/androidTest/` with `Room.inMemoryDatabaseBuilder` (DAO tests) and `MigrationTestHelper` (migration tests).
- **Composables** are tested in `app/src/androidTest/` via `createComposeRule()` (or `createAndroidComposeRule()` when Activity setup is required). Stateless content composables are the unit; previews drive the screenshot/regression budget.

## Test naming

- Unit: backtick-quoted English sentences are encouraged. `` fun `emits Loading then Success when query succeeds`() ``. The name should read as a sentence describing what is asserted.
- Instrumented: `<screen-or-feature>_<scenario>_<outcome>`. `deckList_emptyState_showsCta`, `deckDetail_addCard_disablesIfFormatIllegal`.
- Avoid `testFoo` style names. They do not describe what is being asserted.

## Tooling

- **JUnit 4.** Industry default; integrates with AGP without extra config. Don't reach for JUnit 5 unless there's a concrete reason.
- **MockK** for mocks (Kotlin-native, supports `final` classes, coroutines, and relaxed mocks where needed). Don't introduce Mockito.
- **Turbine** for `Flow` assertions. The default pattern:

  ```kotlin
  viewModel.uiState.test {
      assertThat(awaitItem()).isEqualTo(UiState.Loading)
      assertThat(awaitItem()).isInstanceOf(UiState.Ready::class.java)
      cancelAndIgnoreRemainingEvents()
  }
  ```

- **`kotlinx-coroutines-test`** for any test touching coroutines. Use `runTest` and `StandardTestDispatcher`. Never `runBlocking` for tests of suspend code.
- **MockWebServer** for the HTTP layer in unit tests where the seam is too narrow for a fake.
- **AssertK or Truth** for fluent assertions. Hamcrest is acceptable but not required. Pick one and stay with it within a file.
- **Compose UI tests** via `ComposeTestRule`. Use `Modifier.testTag(...)` on every interactive element a test will look up. Tags are constants in a sibling `*Tags.kt` file.
- **Hilt tests** via `@HiltAndroidTest` with a custom test runner. Replace bindings with `@TestInstallIn` modules.

## Fakes vs mocks

Fakes are preferred for repositories, data sources, and dispatchers. Mocks are reserved for narrow seams that don't deserve a fake.

Reasoning: fakes are easier to debug (real Kotlin code, breakpoints work), easier to read (the test reads like a scenario), and survive refactors that would break a `every { repo.get(any()) } returns ...` chain. The price is a few lines of fake-state code; that price is worth paying.

Build fakes that match the repository interface, hold their data in a `MutableMap` or `MutableStateFlow`, and expose helpers like `addDeck(deck)` for arrange steps.

## What to test for the FaB display logic

Even though the validation engine lives in the backend, the Android client renders its results and needs tests that:

- Every backend `error.code` value is handled by the violation-rendering UI — an exhaustive `when` over the sealed `AppError` hierarchy with a compile-time guarantee. Add a test asserting that any new `code` either has a UI mapping or is intentionally a fallback.
- Format pickers, deck-size hints, and legality affordances handle each canonical legal/illegal fixture parsed correctly.
- Any specific bug fixed in the rendering layer has a regression test, with a comment linking to the issue or report.
- Optimistic local actions (e.g. dragging a card into a deck) reconcile correctly with the backend's eventual response, including the rejection path.

See `fab-domain.md` for the format and rule definitions the engine implements server-side.

## What not to test

- Generated code (the openapi-generator-kotlin output, generated Hilt components, generated Room DAOs). Their generators are tested upstream.
- Pure data classes with no behavior.
- Trivial wrappers that do nothing but forward to a tested function.
- Third-party library behavior. If you find yourself testing that `kotlinx.serialization.json.Json.encodeToString` works, stop.
- Composable previews. Previews are documentation, not verification — add a real test if the rendering matters.

## Test data

- JSON fixtures live under `app/src/test/resources/fixtures/` (and `app/src/androidTest/assets/fixtures/` when an instrumented test needs them).
- Card and deck fixtures should be a small representative subset, not the full upstream dataset.
- Each fixture has a top-of-file comment describing the scenarios it is designed to cover.
- Where a test needs a deterministic clock, inject one. Tests must not depend on `System.currentTimeMillis()` or `Clock.systemDefaultZone()` directly.
