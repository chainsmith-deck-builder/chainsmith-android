# Persistence rules

Local persistence on the Android client. The backend (`chainsmith`) owns the durable database; this file is about caching and on-device state. Replaces the backend repo's `database.md`.

Three concerns, three tools:

| Concern | Tool |
|---|---|
| Structured cached data (cards, decks, listings) | Room |
| Small key/value preferences (theme, last-used format) | DataStore Preferences |
| Sensitive material (auth tokens, refresh tokens) | Tink-backed encrypted DataStore + Android Keystore |

No `SharedPreferences` for new code. DataStore replaces it.

## Room

- One `@Database` class (`ChainsmithDatabase`) under `data/local/`. Bumped via the `version` parameter; migrations are written for every bump.
- Entities live in `data/local/entity/`, end in `Entity` (`DeckEntity`, `CardEntity`, `DeckCardCrossRef`).
- DAOs live in `data/local/dao/`, end in `Dao`. Reads return `Flow<T>` for queries the UI observes; one-shot reads return `suspend fun`. Writes return `suspend fun` (or `suspend fun` returning the inserted id when relevant).
- Multi-step operations use `@Transaction` on a DAO method. Don't compose transactions outside the DAO.
- KSP for the Room compiler, not KAPT. KSP is faster and is the default Google recommends.
- Never expose `LiveData` from a DAO. We're Compose-first; `Flow` is the right shape.
- Never expose Room entities past the repository boundary. Map to domain models in `data/repository/`.

Schema conventions:

- Table names plural snake_case (`cards`, `decks`, `deck_cards`).
- Primary keys: `id: String`, holding the UUIDs the backend issues. We do not auto-increment local IDs for entities that have a server-side identity.
- Foreign keys: `<other>_id`, declared with `@ForeignKey` and indexed.
- Timestamps: `created_at`, `updated_at`, `deleted_at` for soft delete. Stored as ISO-8601 strings or epoch millis — pick one per database and stay consistent (we recommend epoch millis as `Long` for index-friendliness).
- Booleans named with positive sense (`is_published`, not `is_unpublished`).

## Migrations

### Pre-launch phase

- Migrations may be skipped during dev: enable `fallbackToDestructiveMigration()` only in the `debug` build type. Production builds must always go through real migrations.
- Before transitioning to production, finalize the schema and write the baseline migrations the production launch will use.

### Production phase

- Append-only. Every schema change ships with a `Migration(from, to)` written and tested.
- Migrations are tested with Room's `MigrationTestHelper` against a representative dataset.
- Destructive changes go through expand-and-contract: add new column, dual-write, backfill (if needed), switch reads, drop old in a later release.
- Never drop a column in the same migration that adds its replacement.
- A single migration runs in a single transaction (Room enforces this) unless explicitly disabled. If you need to disable it, document why in the migration's KDoc.

## DataStore Preferences

- One DataStore per concern, not a global mega-store. Examples: `uiPrefsDataStore`, `lastSeenDataStore`, `featureFlagsDataStore`.
- Created via `preferencesDataStore(name = "...")` at the top level of a file in `data/prefs/`.
- Provided via Hilt as a singleton.
- Keys are `Preferences.Key<T>` constants in the same file as the wrapper.
- The wrapper exposes domain-shaped APIs (`fun observeTheme(): Flow<Theme>`, `suspend fun setTheme(theme: Theme)`), not raw `Preferences` reads.
- Migrations from older formats use the `produceMigrations` parameter on the builder.

## Encrypted storage (tokens, refresh tokens)

- Tink-backed encrypted DataStore is the default for sensitive material. It uses an AEAD primitive (AES-GCM) keyed by an Android Keystore master key.
- Master key creation: `MasterKey.Builder(context).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).setUserAuthenticationRequired(false)` for tokens; flip user-auth-required on for material that should require device unlock to access.
- Where the platform supports it, set `setUnlockedDeviceRequired(true)` on the master key (Android 9+, API 28). This prevents key use while the device is locked.
- Never store tokens in `SharedPreferences` (encrypted or otherwise legacy) for new code.
- Never log token values. If you need a correlation handle, log a short fixed-length hash prefix.
- The token store exposes domain APIs (`suspend fun saveSession(Session)`, `fun observeSession(): Flow<Session?>`, `suspend fun clearSession()`), not raw token getters.

## Repository conventions

- A repository is the single entry point for one domain area's persistence. It merges remote (Retrofit) and local (Room/DataStore) sources into a single `Flow<T>` exposed to the UI.
- Mapping happens at the repository boundary. No Room or Retrofit types leak.
- Reads are stale-while-revalidate by default for read-heavy data: emit cached value immediately, refresh in the background, emit again. The TTL is a constant in the repository file (`private val CARD_CACHE_TTL = 1.hours`).
- Writes are write-through where the entity has a server-side identity: send to the API first, then update the local cache from the response. Optimistic local updates with reconciliation are acceptable for UX-critical flows; document them with a comment and have a test for the reject-and-rollback path.
- Single-flight: concurrent requests for the same key collapse into one network call. A `Mutex` keyed by request, or a `MutableMap<Key, Deferred<T>>`, are both fine.
- Errors are translated at the boundary: `IOException`, `HttpException`, `SQLiteException` become typed `AppError`s inside `DataResult.Failure`.

## Cache freshness rules

- Card data: 24h TTL by default. Cards rarely change, and the backend's nightly sync sets the upstream rhythm.
- Banned/restricted lists: 1h TTL. The data is small but timely (LSS announcements).
- User-owned data (decks, collections): no TTL; revalidate on screen entry and after writes.
- Format catalog: 24h TTL.

Document any TTL different from these in the repository's KDoc with the reason.

## Card images

Load images via Coil from the `imageUrl` the backend surfaces on each `Printing`. Coil handles disk caching and memory pressure. We do not proxy or re-host LSS imagery (see `security.md` for the legal reasoning, mirroring the backend's stance).

- A null `imageUrl` renders a placeholder. Never invent a URL.
- Coil's disk cache size is set in the application's `ImageLoader` configuration; default to 100 MB. Memory cache uses Coil's defaults (a fraction of the runtime's max memory).
- Don't pre-warm the disk cache speculatively. Cache fill should follow user behavior, not anticipate it.
