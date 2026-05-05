# API contract rules

This Android client consumes the HTTP API exposed by the `chainsmith` Rust backend. The contract is the OpenAPI 3.x spec the backend publishes as `openapi.json` at a specific commit SHA. This client pins that SHA and regenerates a typed Kotlin client from it.

This file is the consumer-side complement to the backend's `.claude/rules/api-contract.md`. Read both before changing anything that touches the API.

## What this client consumes

- The pinned spec lives at `https://raw.githubusercontent.com/chainsmith-deck-builder/chainsmith/<sha>/openapi.json`.
- The SHA is stored in a single-line `.api-version` file at the repo root. That file is the only thing that needs to change to bump the API version.
- The generator runs as a Gradle task `:app:generateApi`, reading `.api-version`, fetching the spec, and writing Kotlin sources under `app/build/generated/openapi/`. Generated sources are rebuilt on every build and are not committed to the repo (kept out of PR diffs).

## Generator configuration

`openapi-generator-cli` with the Kotlin generator:

```
openapi-generator-cli generate \
    -g kotlin \
    -i $spec \
    -o app/build/generated/openapi \
    -p library=jvm-retrofit2 \
    -p serializationLibrary=kotlinx_serialization \
    -p dateLibrary=java8 \
    -p useCoroutines=true \
    -p enumPropertyNaming=UPPERCASE \
    -p packageName=com.chainsmith.android.api
```

Why these options:

- `library=jvm-retrofit2` matches our Retrofit + OkHttp + kotlinx-serialization stack.
- `serializationLibrary=kotlinx_serialization` keeps the wire layer on one JSON library; we don't pull in Moshi or Gson.
- `dateLibrary=java8` produces `java.time` types (`OffsetDateTime`, `LocalDate`) for RFC3339 strings. Backend emits RFC3339; this is the matching parse.
- `useCoroutines=true` makes service interfaces `suspend fun`, which is what our repositories call.
- `enumPropertyNaming=UPPERCASE` is the Kotlin convention; the wire enum stays `snake_case` as the backend defines it.

If the generator config changes, document the change here and in the relevant Gradle task definition.

## Wrapping the generated client

The generated DTOs and service interfaces are an internal detail of `data/api/`. They never escape that package.

- Repositories depend on a small adapter type (e.g. `DeckApi`, `CardApi`) that internally calls the generated service.
- Generated DTOs are mapped to domain models inside the repository. There is no `Deck` -> `DeckDto` -> `Deck` round-trip in business code; the wire model only exists at the seam.
- Errors from Retrofit (`HttpException`, `IOException`) are translated to `AppError` at the adapter boundary, then surfaced as `DataResult.Failure` (see `kotlin.md`).

Tag-based grouping: the backend's handlers carry tags (`Health`, `Catalog`, `Validation`, `Decks`, `Auth`, etc.). The Kotlin generator splits these into separate API interfaces. Maintain that split in the repository layer — one repository per tag domain — rather than collapsing into a single mega-repository.

## Error shape

All backend error responses use a stable JSON body:

```json
{
  "error": {
    "code": "string_enum_value",
    "message": "human-readable explanation",
    "details": {}
  }
}
```

Client handling:

- `code` is a stable string. Map every value to a member of the sealed `AppError` hierarchy. New codes are added to the sealed hierarchy in the same PR that bumps `.api-version`.
- `message` is for humans and may change without warning. Don't switch on it.
- `details` is structured context. Surface it in error UI when relevant (e.g. validation field-level errors).
- Exhaustiveness: a `when (error)` over `AppError` should require an `else` branch only for forward-compat unknown codes; all known codes must be handled explicitly.

When the deserializer encounters an `error.code` it does not recognize (because the backend has shipped a new code we haven't pulled yet), it produces an `AppError.Unknown(code, message)` value. Production builds render a generic error and log a warning; debug builds render the raw code so it surfaces immediately during development.

## JSON conventions to expect

- Field names on the wire are `camelCase`. Generator handles this; don't override.
- Dates and times are RFC3339 strings. They deserialize to `java.time.OffsetDateTime` (timestamps) or `LocalDate` (dates). Format at the Composable boundary; don't pass `OffsetDateTime` into the UI layer if the UI is going to format it anyway — pass the formatted string and a stable sort key.
- Optional fields may be omitted or `null`. Configure kotlinx-serialization with `ignoreUnknownKeys = true`, `coerceInputValues = true`, and `explicitNulls = false` to be permissive about additive backend changes.
- Enums are `snake_case` strings on the wire, `UPPERCASE` on the Kotlin side. Configure the deserializer to fall back to a known sentinel value (`UNKNOWN`) for new enum members so an additive backend change doesn't crash old clients.
- Numbers: integers are `Long`, decimals are explicit (`Double` or `BigDecimal` per the schema). Don't auto-convert.

## Auth

- Bearer JWT in the `Authorization` header. An OkHttp `Interceptor` reads the current access token from the encrypted DataStore (see `security.md`) and attaches it to every request that needs it.
- Token refresh is handled by an OkHttp `Authenticator` that runs when the server returns 401, swaps the access token, and retries the original request once. Never refresh in a manual loop or from a Composable.
- Public endpoints (the few that exist) are explicitly tagged in the generator-aware wrapper as not requiring auth, so the interceptor skips them.
- The client trusts the bearer token. Decoding the JWT for UX (e.g. computing time-until-expiry to schedule a refresh) is fine, but **never derive authorization decisions from client-decoded claims** — the backend is the authority.

## Pagination

Cursor-based, matching the backend. List endpoints take `limit` and `cursor`; responses include `nextCursor` (string or null).

- Repositories backing list UIs use AndroidX Paging 3 (`androidx.paging:paging-compose`) with a custom `PagingSource<String, Item>` keyed on the cursor.
- The cursor is opaque to the client. Don't decode, parse, or compare cursors as if they had structure.

## Phase-dependent rules

Current phase is set in `/CLAUDE.md`. Read it before bumping `.api-version`.

### Pre-launch (no real users, current phase)

- Backend may break shapes freely. The expected workflow when an `.api-version` bump breaks the client:
  1. Bump `.api-version`.
  2. Run `./gradlew :app:generateApi`.
  3. Fix every compile error in `data/api/` and the repositories that adapt to it.
  4. Update `AppError` sealed hierarchy if `error.code` set changed.
  5. Update tests.
  6. Land all of the above in **one PR**. Do not split.
- No deprecation cycles. Move fast and rewrite.

### Production (after real users have data on devices)

- Backend is additive-only. The client must absorb additive changes without crashes:
  - Unknown JSON fields ignored (`ignoreUnknownKeys = true`).
  - Unknown enum values fall back to a sentinel.
  - Unknown `error.code` values map to `AppError.Unknown`.
- The client itself ages backwards-compatibly: an older installed APK reading a newer backend response must not crash. The points above are how we get there; verify with a generated-code-against-old-spec test in CI.
- The minimum supported app version coexists with the backend's deprecation cycle. When the backend deprecates a field, schedule client work to remove the dependency before the field is deleted.

## Backwards-compatibility checklist (production phase)

Before merging any change that bumps `.api-version` in production, confirm:

- The deserializer config is still permissive about unknown fields and enum values.
- No `when` over an enum or sealed hierarchy of API origin lacks a sentinel/`else` branch.
- The `AppError` sealed hierarchy reflects the latest spec's `error.code` enum.
- Generated code compiles cleanly; no manual edits to generated files.
- A smoke test against the updated backend (staging or local) was run for at least one happy path and one auth-required path.
