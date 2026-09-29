# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path, stated intent achieved.
2. **Is it correct?** — edge cases, concurrency, failure paths. *In this repo*: session cancelled on validation failure? invariants (Latest/Published) preserved? token/scheme right?
3. **Will it stay correct?** — tests at the right layer, migration portability, invariants documented.
4. **Does it fit?** — follows the pattern the neighboring code uses (driver? handler? deferred task?), right project (Abstractions vs impl), right lifetime.
5. **Is it kind to future maintainers?** — names, doc comments where behavior surprises (write-on-read `GetAsync`!), localized strings, no drive-by refactors.

Block on 1–2, push firmly on 3–4, suggest on 5. Grading findings is the skill: a review that blocks on naming while missing a missing-`CancelAsync` teaches the team to ignore you.

## Repo-specific review checklist

- [ ] New mutation path: `ModelState`/validation failure → `_session.CancelAsync()` ([`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) is the reference shape)
- [ ] Authorization: resource passed to `AuthorizeAsync`? same permission demanded on every surface for the same operation? (R1 class)
- [ ] Post-commit effects deferred, never inline in `*edAsync` handlers doing DB reads of their own writes
- [ ] Migrations: additive; `CreateAsync` returns latest version; provider portability respected ([`PublishLater/Migrations.cs:44-48`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs) precedent)
- [ ] Background/deferred code resolves services from the *scope's* provider
- [ ] User-facing strings through `S[...]`/`H[...]` with literal format strings
- [ ] Shape/placement renames: grep across modules **and** themes
- [ ] `wwwroot/` untouched; `Assets/` sources changed instead
- [ ] Public Abstractions surface: additive + `[Obsolete]`, never in-place signature changes

## Example comments (tone calibration)

Blocking, correctness:
> This handler queries `AutoroutePartIndex` inside `PublishedAsync` before the session commits, so it reads its own uncommitted write on some providers and stale data on others. Can we move it to a deferred task like the entries update at AutoroutePartHandler.cs:74-76?

Important, design:
> This adds a second place where publish permission is decided. Could we route both the controller and the endpoint through one operation-level check so they can't drift again? Happy to pair on where that seam should live.

Optional, kindness:
> Nit: `GetAsync(id, DraftRequired)` creates a version row — a one-line comment here would save the next reader the surprise it gave me.

Requesting context (instead of assuming):
> Is the 404-before-authz ordering here intentional as an API contract? If yes, a comment; if not, I think swapping the checks avoids an existence oracle.

## Anti-patterns to unlearn

- Reviewing the diff without opening the surrounding file (the `FormValueRequired` dispatch means the diff's action is one of four siblings — [`AdminController.cs:450-487`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)).
- "LGTM" on migrations (the only unreviewable-later artifact — they run once, everywhere).
- Restating preferences as blockers. Tie every blocker to a failure scenario you can narrate.
- Deleting "weird" code without archaeology (Kata 7: the second password check is load-bearing).

Interview angle: "walk me through how you review a PR" — the five layers + one concrete example comment from above is a complete, memorable answer. Practice it against [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md).
