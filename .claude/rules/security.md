# Security rules

Hygiene, not paranoia. Threat model: an Android app on a potentially-compromised device, talking to a public API, holding a user's auth tokens and locally cached data.

Reference: OWASP MASVS / MASTG. Where this file is silent on a topic, MASVS L1 is the floor we aim for, with L2 controls applied where the cost is low.

## Token storage

- Access tokens, refresh tokens, and any session-derived secrets live in Tink-backed encrypted DataStore (or `EncryptedSharedPreferences` if you must), with the master key in the Android Keystore. See `persistence.md` for the storage details.
- Never in plain `SharedPreferences`. Never on the filesystem in plaintext. Never in a debug log.
- Where the platform supports it, the master key is created with `setUnlockedDeviceRequired(true)` (API 28+). This prevents the key from being usable while the device is locked.
- The token store is the only code that reads tokens. Repositories and interceptors call into it; nothing else touches the bytes.

## Token semantics on the client

- The **backend** verifies signatures and authorization. The client trusts the bearer token.
- Decoding the unsigned claims for UX is fine — e.g. reading `exp` to schedule a proactive refresh, or `sub` to display the user id.
- **Never derive authorization decisions from client-decoded claims.** If the UI needs to know whether the user can perform an action, ask the backend (an endpoint or a `permissions` claim the backend issues for the UX, not for security).
- A failed refresh clears the session and routes the user to sign-in. Don't keep retrying with a stale refresh token.

## Network

- HTTPS only. Cleartext disabled in `network_security_config.xml` for production builds. Debug builds may allow cleartext to `localhost` and configurable staging hosts; the config has separate `<debug-overrides>` for that.
- Certificate pinning via OkHttp `CertificatePinner` for the production backend host. Pin the SPKI hashes of the leaf certificate and at least one backup. Document the rotation procedure in this file when the production host is finalized; until then, leave a TODO with a clear marker.
- Pinning is **not** enabled in debug builds against staging or localhost. The pinning configuration reads the build type at OkHttp configuration time and only attaches the `CertificatePinner` for `release`.
- TLS version and cipher suite policy: rely on OkHttp defaults plus the platform. Don't hand-roll SSLContext.

## Secrets in the repo

- No production keystore, signing key, or upload key in this repo. They live in the CI/CD secret store and are decrypted at release time.
- A debug keystore *may* live in-tree if it is clearly marked as debug-only; if it does, it is committed solely so contributors can produce reproducible debug APKs. **No release builds may be signed with a committed key.**
- `.env`-style values for local development (API base URL overrides, dev token issuer) go in a `local.properties` file (which is `.gitignore`d) and are read into `BuildConfig` via the Gradle build script.
- Gitleaks runs in the pre-commit hook (see `.githooks/pre-commit`). Treat any gitleaks finding as a hard stop — fix the value and rewrite history if necessary, do not bypass the hook.

## Dependency hygiene

- The Gradle `libs.versions.toml` version catalog is the only place dependency coordinates live. Versions are pinned to specific releases; no major-version floats.
- The lock file (`gradle.lockfile`) is committed once Gradle dependency locking is enabled.
- `./gradlew dependencyUpdates` (Versions plugin) and `./gradlew dependencyCheckAnalyze` (OWASP Dependency-Check) run in CI on every PR. A new advisory blocks merge until addressed:
  1. Bump the dependency, or
  2. Apply a documented mitigation (e.g. exclude a transitive that's exploitable), or
  3. Suppress with justification in `dependency-check-suppressions.xml`. The justification names the CVE and explains why we are not exploitable.
- Renovate or Dependabot opens version bump PRs. Triage at least weekly during active development.
- Before adding a new dependency: check its maintenance signal (recent activity, issue count, ownership), license, and how much code it really brings in. Prefer libraries from established orgs (Google, Square, JetBrains, AndroidX) over single-maintainer hobby projects for anything load-bearing.

## Build-time safeguards

Release build settings:

- `isMinifyEnabled = true` (R8 enabled).
- `isShrinkResources = true`.
- `isDebuggable = false`.
- `usesCleartextTraffic = false` in the manifest's release variant.
- ProGuard/R8 rules colocated with the source they protect. Library modules ship `consumer-rules.pro` so consumers don't have to know.
- An R8 mapping file is uploaded with each release for crash deobfuscation.

Debug build settings:

- `isMinifyEnabled = false`, `isDebuggable = true`, cleartext to `localhost` allowed.
- A `debug` applicationIdSuffix so debug and release can coexist on a device.

## Input handling

- Anything from an `Intent` (deep links, share targets, extras) is untrusted.
- Validate URIs from deep links against an allowlist of hosts and schemes. Reject anything else with a no-op or a generic error.
- Use SafeArgs or kotlinx-serialization route types (Navigation Compose) to enforce parameter shapes. Don't `intent.action.toString()` directly into business logic.
- Anything pasted by the user (deck import) is untrusted: validate against the schema before sending to the backend.
- WebViews, if introduced later, are off by default; enable JavaScript only when justified, and apply MASVS-WEBVIEW guidance (file access disabled, JS-to-native bridges minimized).

## PII and logging hygiene

- Never log tokens, refresh tokens, emails, full deck contents, or full collections.
- If a correlation handle is needed, log a short stable hash prefix derived from the value.
- In release, plant a no-op Timber tree. The default `DebugTree` is debug-only.
- Crash reporting (when added) must scrub PII before upload. Configure the SDK's PII filters and verify with a test crash.
- Don't log full request or response bodies in release. The OkHttp `HttpLoggingInterceptor` is wired with `Level.NONE` in release and `Level.HEADERS` (not `Level.BODY`) in debug by default; opt into `BODY` only locally and never commit a configuration that ships it.

## Card images

Card images are served by LSS's CDN. The backend surfaces the per-printing CDN URL on each `Printing`, and this client loads images directly from LSS via Coil. We do not proxy or cache imagery on our infrastructure.

Reasoning (mirrors the backend's stance):

- LSS owns the card art. Hot-linking is the same posture FaBrary, FaBDB, and Talishar take and is currently tolerated. Routing images through our infrastructure — even as a thin pass-through — would have us redistributing LSS imagery, a stronger infringement claim than direct linking.
- The URL is upstream-controlled. When LSS rotates URLs, the backend's next sync picks them up; the client always re-requests the URL from the backend rather than caching the URL string.

Engineering rules that follow:

- Coil loads from `imageUrl` directly. No backend proxy URL is constructed client-side.
- A null `imageUrl` renders a placeholder; we never invent a URL.
- Surface attribution where appropriate (a credits/legal screen is sufficient).

## Disclosure

This project does not yet have a published security policy. When it does, link it from this file. Until then, the response to any security report is to treat it seriously and fix it quickly, even informally. Cross-repo security issues are coordinated with the `chainsmith` backend team via the `tracker` repo.
