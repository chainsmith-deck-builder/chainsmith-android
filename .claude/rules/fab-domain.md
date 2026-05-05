# Flesh and Blood domain rules

This file captures the FaB-specific knowledge the Android client needs. The backend (`chainsmith`) owns the validation engine and the canonical interpretation of every rule — this client renders results, surfaces violations, and lets users build decks within the constraints the backend enforces. When the backend changes, this client follows.

Keep this file short and current. Defer to the official Comprehensive Rules and the Living Legend / B&R announcements at fabtcg.com for anything not stated here.

## Terminology

- **Hero**: A card that defines the deck. Decks are built around exactly one hero.
- **Class**: The faction the hero belongs to (Guardian, Warrior, Wizard, Mechanologist, Brute, Ninja, Runeblade, Ranger, Illusionist, and so on). Most non-hero cards are restricted to one class plus Generic.
- **Talent**: A subtype that further restricts deck inclusion (Light, Shadow, Elemental, Earth, Ice, Lightning, etc.). Some heroes have talents, some do not.
- **Pitch**: A card's pitch value, color-coded as Red (1 resource), Yellow (2), Blue (3). UI colorings come from this.
- **Equipment**: Cards that go in equipment slots (head, chest, arms, legs, weapons, off-hand). Slot rules vary by format.
- **Combat chain**: The sequence of attacks and defenses played in a turn. Not modeled by the deck builder; this is play-time, not deck-build-time.
- **Living Legend**: A rotation status that retires powerful heroes from Classic Constructed once their LL points threshold is hit. Cards from those heroes' sets remain legal where the format allows.
- **Young hero**: A reduced-statline version of a hero used in Blitz and certain other formats.

Use this vocabulary in code, in tests, and in user-facing strings. Don't invent generic synonyms (`item`, `category`, `value`) for things that already have a name in the domain.

## Formats supported

The backend's validation engine supports:

- **Classic Constructed (CC)**: the flagship constructed format.
- **Blitz**: short-game format, Young heroes only, restricted equipment loadout.
- **Commoner**: Common-rarity-only, Young heroes only.

Limited formats (Draft, Sealed) are out of scope for the deck builder — not modeled on either side.

UX implications for this client:

- The format picker exposes the formats the backend supports. The list comes from the API (a `formats` endpoint or equivalent), not a hard-coded enum, so a new backend format does not require a client release.
- Deck-size hints and equipment-slot affordances are driven by per-format metadata returned by the backend. Don't hard-code a deck size of 60 in the UI; read it from the format response.
- Legality cutoffs (the date a card or hero stops being legal) are backend-controlled and surfaced through the violation messages and the catalog metadata.

## Card data shape

Cards come from the backend's `/catalog` (or equivalent) endpoints, originally synced from `the-fab-cube/flesh-and-blood-cards`. Fields the UI typically reads:

- `uniqueId`, `name`, `pitch`, `cost`, `power`, `defense`
- `types`, `class`, `talents`
- `keywords`
- `printings` (set, edition, rarity, art variant, `imageUrl`)
- `legalityByFormat` (derived backend-side)

Image URLs come from `Printing.imageUrl`. When it is null, the UI shows a placeholder — we never invent a URL. See `security.md` and `persistence.md` for the rationale (LSS owns the imagery; we hot-link directly).

## Banned and restricted lists

Banned and restricted cards are backend-controlled. They are surfaced via:

- `Card.legalityByFormat` (per-card legality state)
- The validation result returned by the backend when a deck contains a banned/restricted card (a `Violation` with a stable `code`)

When LSS announces a B&R update:

1. The backend ships the update first.
2. The client gets the new state on the next data refresh — no client code change is required for the data itself.
3. A client code change *is* required if the new violation introduces a `code` that doesn't yet have a UI mapping (see `api-contract.md`).

Never hard-code a banned-card list in this client. The backend is the source of truth.

## Validation results in the UI

The backend validation engine returns either Legal or a list of `Violation`s. Each violation has a stable `code`, a human `message`, and optional `details`. UI rules:

- The violation `code` is matched in an exhaustive `when` over a sealed Kotlin hierarchy mapped from the backend's enum (see `api-contract.md`). Adding a new `code` server-side requires a corresponding client mapping.
- The human `message` is for fallback display. Prefer rendering localized strings keyed by `code`; show `message` only when a `code`-keyed string isn't available.
- `details` (e.g. "3 copies of X, max 2") are surfaced inline near the offending card or in a per-violation explanation row.
- Violations are not warnings to dismiss — the deck is genuinely illegal until the violation is resolved. UI affordances ("fix", "remove", "replace") link the violation to the offending card.

## What the deck builder does not do

Explicitly out of scope, on every client:

- Match playing or simulation
- Combat chain resolution
- Goldfishing or playtest mode
- Tournament or event management
- Local re-implementation of legality rules (the backend is the authority; this client renders)

If a feature request implies any of the above, push back and confirm before building. "It would be nice to test this deck against a goldfish" is a legitimate desire but not a feature in this product.
