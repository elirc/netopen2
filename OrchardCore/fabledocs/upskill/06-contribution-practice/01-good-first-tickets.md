# Good First Tickets (Junior)

16 tickets, spread across subsystems. T1 is written in full template as the model; the rest are condensed but complete. All are **practice-realistic**: small diff, existing pattern to follow, clear test story. For genuine upstream contributions, check the issue tracker first — some may already exist or be intentional.

---

## Ticket T1: Add debug logging to Autoroute slug collision suffixing

Difficulty: Easy — Estimated time: 2–4 h
Skills practiced: handlers, logging idioms, reading concurrency-sensitive code.
Story: As a site operator, I want a log entry whenever a slug gets `-N` suffixed, so that I can detect collision hot-spots (and evidence for risk R2).
Why this is a good contribution: zero behavior change; adds a missing operational signal ([05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md), Flow 4 gap).
Acceptance criteria:
- [ ] `GenerateUniqueAbsolutePathAsync` logs (Debug) original path, final path, contentItemId when a suffix was applied
- [ ] Log-level guard used, matching house style
- [ ] No log when no collision
Read these anchors first: [`AutoroutePartHandler.cs:437-463`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) — the loop; [`ShellHost.cs:337-340`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) — the `IsEnabled` guard idiom.
Files likely touched: `AutoroutePartHandler.cs` (constructor gets `ILogger<AutoroutePartHandler>`, loop logs).
Implementation plan: 1. inject logger; 2. capture original path before loop; 3. log after successful uniqueness.
Illustrative fake-code shape (labeled):
```csharp
// Illustrative fake code: not from this repo
if (_logger.IsEnabled(LogLevel.Debug) && versionedPath != originalPath)
    _logger.LogDebug("Autoroute path '{Original}' collided; using '{Versioned}' for item {ContentItemId}.", originalPath, versionedPath, contentItemId);
```
What could go wrong: logging PII in paths (paths are public URLs — fine); noisy logs if a site has systematic collisions (that's the point).
Suggested checks: unit-test the suffix helper if extractable; otherwise manual repro with two same-titled posts.
Review questions: right log level? does the message carry enough context to act on?
Interview story potential: "I added the observability that turned a suspected race into a measurable event."

---

## T2 — Notification message duplication in `AdminController`
Easy, 3–6 h. Extract the six near-identical "Your {0} has been published/unpublished/…" blocks ([`AdminController.cs:365-367, 399-401, 465-467, 574-576, 638-645, 893-905`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) into one private helper. Pattern: existing private helpers in the same file (`:844-865`). Reject-risk: localization keys must remain extractable by the string-localizer tooling — keep `H[...]` literal strings intact (this constrains the refactor — discover why!). Test: behavior-neutral; existing functional tests. Interview story: "refactor constrained by an i18n toolchain."

## T3 — XML-doc the `VersionOptions` semantics
Easy, 2 h. Document that `GetAsync(..., DraftRequired)` creates a version row (surprising write-on-read — [`DefaultContentManager.cs:159-183`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) on the `VersionOptions` members / `IContentManager` XML docs. Docs-only, zero risk. Reject-risk: wording must match maintainer intent — quote behavior, not judgment. Interview story: "found a surprising API contract and made it explicit."

## T4 — Cancellation token in scheduled publishing loop
Easy-Medium, 3–5 h. The per-item loop at [`ScheduledPublishingBackgroundTask.cs:41-57`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) ignores `cancellationToken`; add a `cancellationToken.IsCancellationRequested` break (mirror [`ModularBackgroundService.cs:114-117`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). Consider: partial batch then cancel = partial commit at scope end — is that acceptable? (Yes: predicate idempotency; *write that reasoning in the PR*.) Interview story: "graceful shutdown reasoning in a batch job."

## T5 — Functional-test README: add SQLite quick-start
Easy, 1–2 h. `test/OrchardCore.Tests.Functional/README.md` documents shared-DB setups; add the zero-config SQLite path explicitly for newcomers. Docs-only. Verify commands before writing them (run the suite once).

## T6 — Admin list: empty-state message when no creatable types
Easy-Medium, 4–8 h. When `options.CreatableTypes` is empty ([`AdminController.cs:128-136`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) the "New" UI renders empty; show a helpful message in the ContentsAdminList view. Touches: view + possibly view model. Pattern: existing empty-list handling in the same views. Test: functional. Reject-risk: theme overrides — change the *module* template minimally.

## T7 — `PublishLaterPartIndexProvider`: verify null-mapping for items without the part
Easy (investigation ticket), 2–4 h. Read [`Indexes/PublishLaterPartIndexProvider.cs`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Indexes/PublishLaterPartIndexProvider.cs); document (in a short note/PR comment) whether items lacking the part produce index rows. Outcome: a doc comment or a test, not a code change. Trains: reading index providers precisely.

## T8 — Uniform `ArgumentException` guards in `DefaultContentManager`
Easy, 2–3 h. Some entry points use `ArgumentException.ThrowIfNullOrEmpty` ([`DefaultContentManager.cs:69, 190, 344`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)), others null-return (`:104-107`). Pick ONE inconsistent private/internal seam and align it *after* asking (issue first — the null-return on `GetAsync` is a public contract; changing it is a breaking change. Learning the difference **is the ticket**).

## T9 — Login failure metrics event handler (new small module feature)
Medium-Easy, 1 day. Implement an `ILoginFormEvent` that logs structured warnings on `IsLockedOutAsync`/`LoggingInFailedAsync` with tenant name — the observability seam identified in [05-06](../05-quality-engineering/06-observability-and-operations.md). Pattern: any existing `ILoginFormEvent` implementer (search the Users module). Blast radius: additive registration. Interview story: "closed a security-observability gap via an existing extension seam."

## T10 — Document the deferred-task at-most-once semantics
Easy, 2 h. `ShellScope.AddDeferredTask` XML docs ([`ShellScope.cs:414-417`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)) don't state failure semantics (logged, dropped, no retry). Add doc comments. High-value docs: prevents the Kata-5 bug class.

## T11 — `ContentItemIndex` truncation: add doc comment + constant reference
Easy, 1–2 h. Document the 255-char silent truncation ([`ContentItemIndex.cs:50-68`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs)) at the class level: "index values may be truncated; documents are canonical."

## T12 — Add a `placement.json` example to module docs
Easy, 2–3 h. `src/docs` has reference docs (mkdocs); verify placement docs cover the `Actions:5` SummaryAdmin idiom used by [`Contents/placement.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json); extend with one worked example if missing. Check docs build (`mkdocs`) before PR.

## T13 — Test: idempotency of `PublishAsync` on already-published item
Easy-Medium, 3–5 h. Unit/integration test asserting the early-return contract ([`DefaultContentManager.cs:380-383`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)): second publish returns true, fires no handlers. The "fires no handlers" assertion is the interesting part (capture with a fake `IContentHandler`).

## T14 — Consistent `Url.IsLocalUrl` usage audit
Medium-Easy, 4–6 h. Audit `returnUrl` handling across `AdminController` (compare `:509-516` vs `:770-787`); produce a table (issue comment) of guarded/unguarded paths; fix any unguarded one found. Trains: systematic security sweeps. (All paths seen in this review were guarded — the audit's value is *completeness*.)

## T15 — `GetAllVersionsAsync` doc note on unbounded result
Easy, 1 h. Add doc comment referencing version pruning ([`DefaultContentManager.cs:188-203`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs), perf hotspot #6 in [05-04](../05-quality-engineering/04-performance-thinking.md)).

## T16 — First-run experience: log the setup URL
Easy, 2–3 h. On uninitialized default tenant, log an Information line "Navigate to ... to complete setup" at host start (investigate `AutoSetup` module first — it may make this moot; the investigation is part of the ticket). Trains: checking for existing features before adding one — the #1 maintainer-rejection cause.

---

Rejection checklist that applies to all: duplicate of existing issue/PR; behavior change smuggled into a "refactor"; touching generated/`wwwroot` files; missing localization for user-facing strings; tests absent where the repo has an obvious place for them.
