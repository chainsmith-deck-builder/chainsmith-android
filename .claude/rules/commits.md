# Commit message rules

This project uses [Conventional Commits 1.0.0](https://www.conventionalcommits.org/) (a.k.a. Commitizen style) for every commit on every branch. The format gives the log a machine-readable shape — future tooling can generate changelogs, infer semver bumps, and surface breaking changes — without asking much of the author beyond a brief discipline.

Enforcement is mechanical: `.githooks/commit-msg` validates the subject line against the Conventional Commits format on every commit. The hook is pure bash — no Node, no commitlint dependency. Body and footer formatting (72-col wrap, `BREAKING CHANGE:` shape) are still social conventions; only the subject is mechanically checked. If subject-line regex stops being enough later, the natural escalation is `commitlint` or `cocogitto`.

## Format

```
<type>[(scope)][!]: <subject>

[body]

[footer(s)]
```

The subject line is the only required part. Type is required; scope is optional but encouraged when the change is localized.

## Types

Use one of:

- `feat` — a new user-visible feature
- `fix` — a bug fix
- `perf` — a performance improvement that is neither a feature nor a fix
- `refactor` — a code change that neither fixes a bug nor adds a feature
- `docs` — documentation only
- `style` — formatting, whitespace, no logic change
- `test` — adding or correcting tests
- `build` — changes to the build system, dependencies, packaging
- `ci` — changes to CI configuration or scripts
- `chore` — housekeeping, tooling, repo metadata
- `revert` — reverts a previous commit

If a change doesn't fit any of these, the commit is probably doing too many things. Split it.

## Scope

Scope is a noun in parentheses naming the area touched: package or feature name. Examples we expect to use: `ui`, `data`, `domain`, `di`, `auth`, `deck`, `card`, `nav`, `theme`, `build`, `claude`, `githooks`. Don't invent new vocabulary per commit; reuse existing scope names where they fit.

```
feat(ui): add deck list screen
fix(data): retry once on cold openapi 502
refactor(di): split NetworkModule from RepositoryModule
```

## Subject line

- **Imperative mood.** "add," not "added" or "adds." Read it as completing "If applied, this commit will…"
- **Lowercase** after the colon.
- **No trailing period.**
- **Under 72 characters.** GitHub truncates around 70 in some views.
- **Concrete.** "fix bug" is not enough. "fix off-by-one in pitch counter" is.

## Body (optional)

Add a body when the change benefits from explanation:

- Blank line after the subject.
- Wrap at 72 columns.
- Explain *why* and the relevant context — not *what*, the diff shows that.
- Reference issues in the footer (`Refs: #123`, `Closes: #123`), not the body prose.

## Breaking changes

Mark a breaking change with `!` in the subject **and** a `BREAKING CHANGE:` footer:

```
feat(data)!: rename DeckRepository.findById to get

BREAKING CHANGE: callers must migrate from findById(id) to get(id).
```

Use `!` only for genuinely breaking changes. What counts as breaking in this project depends on the phase model — see `api-contract.md` and `persistence.md`.

## Reverts

```
revert: feat(ui): add deck list screen

Refs: <sha-of-reverted-commit>
```

Use the `revert` type plus the original subject verbatim. Put the SHA of the reverted commit in a footer.

## Authorship trailers

This project does **not** use `Co-Authored-By:` trailers by default. The git log credits the human committer only; AI-assisted authorship (Claude, Copilot, Cursor, etc.) is implicit and does not appear in commit metadata.

Add a `Co-Authored-By:` trailer only when crediting a real human collaborator who pair-programmed on the change. When committing on behalf of the user, do not append a Claude attribution trailer even if a default behavior would otherwise add one — the rule in this file overrides that default.

## One change per commit

A commit should be one logical change. If you need the word "and" to describe what a commit does, it's two commits.

This applies even mid-branch: if you've made messy WIP commits, **squash them before merging** so what lands on `main` has a clean Conventional Commits log. The branch's pre-squash history is allowed to be messy; the merged history is not.

## Examples

Good:

```
feat(ui): add deck list screen
fix(data): retry once on cold openapi 502
refactor(di): split NetworkModule from RepositoryModule
docs: link gitleaks install instructions in README
chore(deps): bump compose-bom to 2026.05.00
ci: add detekt to the check workflow
test(deck): cover empty-state in DeckListViewModel
build: bump kotlin to 2.1.20
perf(card): cache CardThumbnail measurement across recompositions
```

Bad — and why:

- `Updated stuff` — no type, vague, wrong tense
- `feat: added new screen.` — past tense, trailing period
- `fix(ui): Fix the thing` — capital F after colon, placeholder subject
- `feat: add deck screen and fix unrelated nav crash` — two changes in one commit; split
- `WIP` — never lands on `main`; squash it out before merge
