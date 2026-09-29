# Review Katas

Eight fake PRs against this codebase. For each: read the intent and diff summary, write your review (findings graded **Blocking / Important / Optional**), then compare. Diffs are described, not shown — like reviewing from a PR summary + the real files, which you should open.

Review like this repo's maintainers would: kind, specific, anchored ("In `AutoroutePartHandler.cs:465` the check…"), and always distinguishing *must fix* from *could improve*.

---

## Kata 1: "Add UnpublishLater feature"
Author intent: mirror PublishLater for scheduled unpublishing.
Fake diff summary: copies `ScheduledPublishingBackgroundTask` → `ScheduledUnpublishingBackgroundTask` querying `index.Published && due`, calls `UnpublishAsync`; adds part+driver; **reuses `PublishLaterPartIndex`** by adding a `ScheduledUnpublishDateTimeUtc` column via `UpdateFrom3Async` that also renames the table.
Files this resembles: [`PublishLater/*`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater), [`Migrations.cs:44-48`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs).
Expected findings — **Blocking**: table rename in a migration violates the repo's own cross-provider constraint (the file's comment says drop/alter don't work on all providers); reusing another feature's index couples two features' lifecycles (disabling PublishLater breaks Unpublish). **Important**: no DB index covering the new query; missing `[BackgroundTask]` schedule attribute; unpublish loop needs the same `part.Apply()` clear-schedule-then-act ordering. **Optional**: naming symmetry, docs.
Good review comment example:
> Reusing `PublishLaterPartIndex` ties this feature's queries to PublishLater's migrations — if that feature is disabled or its schema evolves, Unpublish breaks silently. Suggest a separate `UnpublishLaterPartIndex` following the same three-column pattern (Migrations.cs:24-39). Also: the rename in UpdateFrom3 won't run on all providers per the comment at Migrations.cs:47-48.

## Kata 2: "Faster admin list"
Author intent: speed up content list by skipping the display manager.
Fake diff summary: replaces per-item `BuildDisplayAsync(..., "SummaryAdmin")` ([`AdminController.cs:199-204`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) with a hand-built table reading `contentItem.DisplayText` directly; removes the per-type `ListContent` authorization filtering from `GetListableContentTypeOptionsAsync`.
Expected findings — **Blocking**: removing per-type authorization exposes types the user may not list (`AdminController.cs:828-837`); bypassing SummaryAdmin breaks every module contributing admin actions via drivers/placement (Contents' own `placement.json` places `Parts_Contents_Publish_SummaryAdmin` in Actions). **Important**: perf claim unmeasured — the expensive part is likely the count query (`:188-190`), propose measuring first. **Optional**: pagination unchanged.

## Kata 3: "Login: friendlier errors"
Author intent: tell users whether the username or the password was wrong.
Fake diff summary: in `LoginPOST`, returns "Unknown username" when `GetUserAsync` is null; removes the `_timingNormalization.NormalizeResponseTime()` call as "dead code."
Files: [`AccountController.cs:125-206`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs).
Expected findings — **Blocking**: username enumeration via message *and* via timing (the "dead code" is the timing defense, `:182-188`). **Important**: UX request can be met safely with rate-limited "forgot username" flows. **Optional**: none. This kata tests whether you review *intent*, not just diff mechanics.

## Kata 4: "API delete: move the NotFound up"
Author intent: "cleaner flow" — in `DeleteEndpoint`, move the null check **above** the `AccessContentApi` check so missing ids short-circuit before any authorization.
Files: [`Endpoints/Api/DeleteEndpoint.cs:31-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/DeleteEndpoint.cs) — read the real ordering first: `AccessContentApi` → load → `NotFound` → per-item `DeleteContent`.
Expected findings — **Blocking**: with the change, *anonymous* callers (the route is `AllowAnonymous`, auth happens in the handler) gain an existence oracle over content ids — 404 vs 401 reveals which ids exist with zero credentials. **Important**: note the *current* code already lets any `AccessContentApi` holder probe existence before the per-item `DeleteContent` check (`:36-46`) — worth an issue asking whether uniform 404-after-authz is the intended contract. **Optional**: idempotent delete debate (200 with body vs 204 vs 404 for already-gone).

## Kata 5: "Deferred email on publish"
Author intent: send notification email after publishing.
Fake diff summary: new handler `PublishedAsync` → `ShellScope.AddDeferredTask(scope => emailService.SendAsync(...))`; no retry; email address read from the *unpublished* draft's data inside the deferred lambda via captured `contentItem` reference.
Expected findings — **Blocking**: captured `contentItem` crosses scopes — deferred task runs in a *fresh scope/session* ([`ShellScope.cs:474-506`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)); reload by id inside the task. **Important**: at-most-once delivery acceptable? propose outbox if the email is contractual ([03-architecture-and-patterns/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)); failure only logged. **Optional**: localization of the email body.

## Kata 6: "Slug hardening"
Author intent: fix the slug race (risk R2).
Fake diff summary: wraps `IsAbsolutePathUniqueAsync` + save in `lock(_slugLock)` (a static object) inside `AutoroutePartHandler`.
Expected findings — **Blocking**: `lock` around `await` doesn't compile / if made `SemaphoreSlim`, it's per-process only — multi-instance farms still race; and it serializes *all* saves across all tenants (head-of-line blocking). **Important**: correct fix is a DB unique constraint + violation retry, or `IDistributedLock` keyed by path; needs a dedupe migration first ([06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) item 3). **Optional**: add a collision metric regardless.

## Kata 7: "Remove double password check"
Author intent: "we check the password twice in LoginPOST; delete the first `CheckPasswordSignInAsync`."
Expected findings — **Blocking**: the first check exists so `ValidatingLoginAsync` events run *between* password validation and cookie issuance ([`AccountController.cs:133-149`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)) — removing it either signs users in before validators can veto, or skips lockout accounting. **Important**: if perf is the concern, measure; hashing dominates and happens once per attempt either way (`PasswordSignInAsync` re-verifies — acknowledge the real duplication and why it's accepted). This kata trains "the weird code is load-bearing until proven otherwise."

## Kata 8: "Truncate less in ContentItemIndex"
Author intent: raise `MaxDisplayTextSize` from 255 to 4000 for long titles.
Fake diff summary: changes constants in [`ContentItemIndex.cs:7-12`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs); no migration.
Expected findings — **Blocking**: the column width lives in existing databases; changing the constant without an `AlterColumn` migration (which the repo avoids for portability!) means new values still truncate at the DB or fail on insert depending on provider. **Important**: index-size/perf cost of wide indexed columns on four providers; existing rows remain truncated — is the inconsistency acceptable? **Optional**: document the truncation behavior at the field.

---

Self-grade per kata — Basic: found the blocking issue. Solid: graded findings correctly (didn't block on style). Strong: your comments name file:line, state the failure scenario, and offer a concrete alternative. [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) runs katas 3, 4, and 6 as timed rounds.
