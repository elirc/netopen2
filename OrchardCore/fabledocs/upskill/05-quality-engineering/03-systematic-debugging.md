# Systematic Debugging

The method, always: **reproduce → narrow (bisect the flow) → hypothesize → test the hypothesis cheaply → fix the root cause → add regression coverage.** The five scenarios below practice it on this codebase. Tools you actually have here: the debugger against `dotnet run`, NLog output in `App_Data/logs/`, YesSql query logging (a `Logger` is wired into the store config — [`OrchardCoreBuilderExtensions.cs:190-200`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)), browser devtools, and `--filter-method` for tight test loops.

---

## Scenario 1: "I saved the item but my changes are gone"

Reproduction: edit a content item; a custom part's values revert after save. No error shown.
First question: is the bug in **binding** (driver never wrote the value) or **persistence** (value written, transaction cancelled)?
Narrowing path:
1. Breakpoint in your driver's `UpdateAsync` — does it fire? If not: driver registration or prefix mismatch (compare [`TitlePartDisplayDriver.cs:51`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) `TryUpdateModelAsync(model, Prefix, ...)`).
2. Fires but value missing → inspect form field names vs `Prefix`.
3. Value set but lost → check `ModelState.IsValid` at [`AdminController.cs:759-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs): **any** driver's error cancels the whole session — another part's validation may be reverting *your* part.
4. Confirm cheaply: watch for the error notification path / dump ModelState.
Likely root causes: prefix mismatch; a different part failing validation; mutating the part without `Apply()`.
Regression test: driver-level unit test (Recipe 2).
Senior lesson: in a unit-of-work system, *your* data loss is often *someone else's* validation failure — think in transactions, not fields.
Interview version: narrate as "I bisect at the boundary: does the mutation reach the model? does the model reach the session? does the session commit? — three breakpoints, one minute each."

## Scenario 2: "Two pages have the same URL / wrong page served"

Reproduction: two items claim `/about`; which renders is unpredictable.
First question: prevention failed (race) or data imported around validation?
Narrowing path:
1. Query `AutoroutePartIndex` for the path — how many rows `Published || Latest`? (This is the check the code itself runs: [`AutoroutePartHandler.cs:465-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs).)
2. Two rows → check audit trail / timestamps: near-simultaneous publishes ⇒ TOCTOU race (risk R2); far apart ⇒ import path or handler-skipping writer.
3. Confirm cheaply: try to reproduce with two parallel publish requests (curl ×2) on a scratch item.
Fix root cause: for the data — resave one item (regenerates a `-2` path). For the class of bug — unique constraint + retry (see critique item 3), not a bigger in-app check.
Regression: concurrency test firing N parallel publishes, assert 1 row.
Senior lesson: fixing the *instance* (data repair) and the *class* (constraint) are separate deliverables; say both.

## Scenario 3: "My template change doesn't render"

Reproduction: you edited a `.cshtml` in a module but output is unchanged.
First question: is your template even being **selected** (alternates), or selected-but-cached?
Narrowing path:
1. Is DynamicCache on? Disable/bypass (cached HTML strings would mask everything — [`DefaultDynamicCacheService.cs:49-68`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)).
2. Wrong template picked: list candidate names — shape type + alternates from [`ContentItemDisplayManager.cs:75-83`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs); a *theme* override may shadow your module template.
3. Placement moved/suppressed the shape: check module + theme `placement.json` ([example](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json)).
Useful probe: the shape-tracing feature of the Templates/Diagnostics modules if enabled; otherwise rename your shape temporarily — if output changes, selection was fine.
Senior lesson: in convention-resolved systems, debug the *resolution order* before the artifact.

## Scenario 4: "Scheduled publish didn't happen"

Reproduction: item scheduled for 10:00; at 10:05 still draft.
Narrowing path (follow Flow 5's chain, cheapest probe first):
1. Is the item even eligible? Query `PublishLaterPartIndex`: `Latest && !Published && due` ([`ScheduledPublishingBackgroundTask.cs:29-32`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)) — timezone bugs live here (stored UTC vs entered local).
2. Did the task run? logs: "Start processing background task" info lines ([`ModularBackgroundService.cs:209-215`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)).
3. Task ran, item skipped → lock timeouts ("Timeout to acquire a lock", `:143-146`) or an exception in the batch (one poison item aborts the commit — risk R4).
4. Tenant not running / no pipeline → `GetRunningShells` filter (`:360-364`).
Likely root causes ranked: timezone entry error > batch-poisoning by another item > lock contention on a farm > feature disabled.
Regression: Recipe 5's idempotency test + a poison-item test.
Senior lesson: for background failures, debug **eligibility → execution → outcome** as three separate questions; most "cron is broken" reports are eligibility.

## Scenario 5: "Login suddenly slow (2s) for some users"

Reproduction: intermittent slow POST `/Login`.
First question: which branch — successful logins, failures, or unknown users?
Narrowing path:
1. Segment by outcome in logs (`LoggedInAsync` vs `LoggingInFailedAsync` events fire on distinct paths — [`AccountController.cs:155, 193-202`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)).
2. Unknown users slow ⇒ that's timing normalization *working* (`:182-188`) — not a bug; the dummy hash costs a real hash.
3. Everyone slow ⇒ password work factor raised, or an `ILoginFormEvent` doing I/O (they run serially, `:123`).
4. Probe: time `CheckPasswordSignInAsync` alone vs the event chain.
Senior lesson: before "fixing" a slowdown, ask whether it's a **deliberate security property**. Deleting the normalizer would be the classic well-intentioned regression (Kata 3).

---

## Drill

Pick Scenario 2 or 4 and write the narration you'd give in a debugging interview: symptom → the two competing hypotheses → the single cheapest probe that discriminates them → expected evidence each way. 90 seconds aloud. Rubric — Strong: your probe is a *query or log line*, not "add logging everywhere"; you name the regression test before being asked. Timed versions: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
