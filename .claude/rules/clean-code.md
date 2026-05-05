# Clean code rules

Project-specific clean-code preferences for Kotlin code in this repo. Curated, not orthodox — some of Uncle Bob's *Clean Code* rules hold up, others don't, and modern thinking (Ousterhout's *A Philosophy of Software Design*) plus direct experience with LLM-generated code adds rules of its own.

This file overlaps with `kotlin.md`, `architecture.md`, and the global `~/.claude/CLAUDE.md`. Where they differ, the more specific wins (`architecture.md` > `kotlin.md` > this file > global).

## Functions

- **One job per function.** A function is a verb. If the name needs an "and," split it. *Validate, then save* is two functions, not one.
- **Don't extract for the sake of extraction.** Single-use helpers that exist only to make a parent function shorter usually hurt readability — the reader jumps around, and the name lies about how reusable the code is. Inline beats extract when the helper would be called once and is not independently testable. This is doubly true for Composables: a five-line `@Composable` that's used once is not a component.
- **Depth over surface.** A function with a small interface and a substantial body is usually better than four wrappers around a one-liner. Ousterhout's "deep module" idea applied to functions.
- **Parameter count is a smell, not a hard rule.** Three is fine, four is fine, six is a smell — but the fix is a `data class`, not a field on the ViewModel. Don't move parameters into shared state to dodge a parameter count.
- **Pure where possible, side-effecting where necessary, never both.** A function either computes a value or performs IO. Mixing them makes testing painful. The `domain/` package is kept pure for this reason; respect that boundary. Composables are pure too — effects belong in `LaunchedEffect` and friends, not inline in the body.

## Naming

- **Names reveal intent, not implementation.** `fetchCardById` describes implementation; `card` describes intent. Prefer the shorter form when context makes the implementation obvious.
- **Use domain language.** When the FaB rules say *hero*, *pitch*, *talent*, the code says hero, pitch, talent. Don't invent generic synonyms (*item*, *value*, *category*) for things that already have a name in the domain. See `fab-domain.md` for the canonical vocabulary.
- **Length scales with scope.** A lambda parameter can be `c`. A function-level variable used across 30 lines should be `legalCards`. A property referenced from many call sites should be unambiguous on its own (`legalCardsByFormat`).
- **Verbs for functions, nouns for types and values.** `validateDeck` not `deckValidation`. `Card` not `CardData`. `isLegal` not `legalityCheck`.
- **Prefer concrete over generic.** `decks` beats `items`. `parseDeckExport` beats `processInput`.
- **Composables are nouns most of the time** (`DeckCard`, `PitchBadge`) and imperative verbs sometimes (`SubmitButton`, `ConfirmDialog`).

## Comments

- **Explain *why*, not *what*.** The code shows what. Comments exist for context the reader can't deduce: a workaround, a non-obvious tradeoff, a link to a bug, a domain rule that justifies an otherwise odd-looking branch.
- **Comments are not a failure.** Uncle Bob argued they are; that hasn't aged well. A short comment explaining intent is cheaper than a heroic rename, and some context (LSS rulings, GHSA references, recomposition tradeoffs) genuinely cannot live in a name.
- **Update or delete drifting comments.** A wrong comment is worse than no comment. If you change behavior, scan for comments above and below the change.
- **Module-level KDoc earns its keep.** A short doc comment at the top of `data/repository/DeckRepository.kt` explaining the cache strategy and the freshness guarantee saves a future reader twenty minutes.
- **No commit-message comments in code.** "Added to fix #1234" belongs in the commit, not the source.

## Avoiding over-engineering

The most common failure mode in LLM-generated code is solving problems that don't exist yet. Rules to push back:

- **YAGNI.** Don't add a parameter, an interface, a generic, or a config flag unless something in this repo uses it now. "We might want this later" is an entry in the issue tracker, not code.
- **No speculative abstractions.** One impl, no interface. One value, no config option. Concrete code is cheaper to generalize later than a wrong abstraction is to undo. The exception is the repository pattern — we keep an interface there because Hilt and tests depend on the seam.
- **No premature builders or `Default` parameter explosions.** Add a parameter when a third caller needs it, not before. Don't accept ten optional parameters when two callers pass two.
- **No wrapper types without a reason.** A `value class` exists to enforce an invariant or distinguish two semantically different `String`s (a `DeckId` from a `UserId`). It does not exist to be tidy.
- **No defensive nullables.** If a function cannot fail and the value is always present, the return type should not lie. `T` not `T?`. `List<T>` not `List<T>?`. The empty list is the empty list.
- **No `internal` "just in case".** Visibility is the smallest scope that compiles. `private` first, escalate only when forced.
- **No `Flow<T>` where a single `suspend fun` is honest.** A one-shot read should be a suspend function. A reactive stream of values over time is a Flow.
- **Don't anticipate threading.** Don't add `Mutex`, `Channel`, or extra dispatchers until profiling or a concrete requirement says you need them.
- **Don't `@Stable` or `@Immutable` defensively.** Apply only when a measured recomposition problem warrants it.

## When to break a rule

Every rule above has exceptions. The bar for breaking one is:

1. Name the rule you are breaking, in the code comment or the PR description.
2. Explain why the alternative is worse for this specific case.
3. Be willing to defend it in review.

If you can't do all three, follow the rule.
