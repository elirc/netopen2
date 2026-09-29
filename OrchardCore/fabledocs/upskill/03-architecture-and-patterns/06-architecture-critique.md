# Architecture Critique

An honest assessment, written as if you owned this codebase for the next quarter. This file doubles as system-design interview material — [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) turns it into a whiteboard exercise.

## Strongest design choices (steal these)

1. **Structural tenant isolation.** Container-per-tenant + per-tenant store means isolation isn't a WHERE clause you can forget — it's the shape of the object graph ([`ModularTenantContainerMiddleware.cs:27-69`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs)). Cost (memory, cold starts) is paid deliberately and mitigated with placeholders ([`ShellHost.cs:335-374`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)).
2. **Transactional-by-default requests.** Commit-at-scope-end ([`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)) eliminates the entire class of "saved half the aggregate" bugs. Few frameworks give you this.
3. **Documents + indexes.** Schema-flexible content with relational query performance, atomic across both ([02-data-model-and-persistence.md](02-data-model-and-persistence.md)). The right call for a CMS whose types are user-defined at runtime.
4. **Resource-based authorization with owner variants.** Ownership policy centralized in one handler instead of scattered if-statements ([`ContentTypeAuthorizationHandler.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)).
5. **Concurrency honesty.** The kernel doesn't pretend races don't exist: semaphore maps, retry loops with "Atomicity:"/"Consistency:" comments ([`ShellHost.cs:208-244`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)), ref-counted contexts, distributed locks. Rare to see this much explicit reasoning in source.
6. **Security details done properly**: login timing normalization, rate-limit groups, lockout, separate Api auth scheme ([Flow 3](../01-codebase-cartography/05-key-flows.md)).

## Risks and tradeoffs (ranked, with evidence)

**R1 — API publish-permission asymmetry (possible security gap — investigate).**
`POST api/content` update path authorizes `EditContent` only, then calls `PublishAsync` when `draft=false` ([`CreateEndpoint.cs:95-127`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)); the admin UI requires `PublishContent` to publish ([`AdminController.cs:876-879`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). If no upstream check exists, an editor-without-publish-rights can publish via API. *Not runtime-verified here*; even if mitigated, the inconsistency between surfaces is itself a defect class: **authorization decided per-endpoint instead of per-operation**. Fix direction: move the publish permission check inside a shared application-service operation, or add the explicit check + test in the endpoint.

**R2 — Slug uniqueness TOCTOU (possible correctness gap).**
Check-then-write with no DB constraint ([`AutoroutePartHandler.cs:465-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)). Concurrent publishes can produce duplicate paths; routing then picks one nondeterministically. Likelihood low per-site, nonzero at scale. Fix: unique index on (tenant-scoped) path + violation-retry mapping into the existing "-2" suffix logic; migration must dedupe first.

**R3 — Failure invisibility in async machinery.**
Background task and deferred-task failures are logged and dropped ([`ModularBackgroundService.cs:222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs), [`ShellScope.cs:497-504`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)). No retries, no dead-letter, no metrics. A failing scheduled-publish tick looks identical to a quiet one unless you read logs.

**R4 — Unbatched scheduled work.**
`ScheduledPublishingBackgroundTask` publishes *all* due items in one scope/one commit ([`ScheduledPublishingBackgroundTask.cs:29-57`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)); one poison item can fail the batch's commit, and the batch retries wholesale next tick. Import shows the house style for batching (500 — [`DefaultContentManager.cs:21`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)); apply it here.

**R5 — Stringly-typed rendering contract.**
Shape names, alternates, placement keys, Liquid — none compiler-checked. Theme/template breakage is discovered at render time. Mitigation is convention + functional tests; cost is real for newcomers (see [05-quality-engineering/03-systematic-debugging.md](../05-quality-engineering/03-systematic-debugging.md), Scenario 3).

**R6 — Draft exposure via GraphQL `status` argument (investigate).**
`status: DRAFT` is a first-class query argument ([`ContentItemsFieldType.cs:69-76`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)); the permission gating path was not fully traced in this review. Verify before exposing GraphQL publicly.

**R7 — Complexity tax.** The kernel's flexibility (placeholders, ref counts, reload retries) is genuinely hard. New-contributor bugs cluster where two lifecycle mechanisms meet (scope vs context vs settings disposal). This is the price of choices #1/#5 — acknowledge it rather than "fix" it.

## What I'd change owning this for 3 months (prioritized)

1. **Close R1**: add the `PublishContent` check + regression test to `CreateEndpoint`; audit the other minimal-API endpoints (`GetEndpoint`, `DeleteEndpoint`) for the same class. Small diff, high value. *Test strategy*: integration test with a user granted EditContent only; assert 403 on `draft=false`. *Migration*: none. *Blast radius*: API clients doing edit-then-autopublish with underprivileged tokens break — that's the point; release-note it.
2. **Background-task observability (R3)**: implement an `IBackgroundTaskEventHandler` that records last-run/last-error/duration per task per tenant + surface in the admin Health/Diagnostics UI. Additive, no kernel change. Then add opt-in retry policy in `BackgroundTaskSettings`.
3. **Slug constraint (R2)**: two-phase — release 1 ships a dedupe migration + telemetry logging collisions; release 2 adds the unique index + retry. Cross-provider index semantics (case sensitivity!) are the hidden work.
4. **Batch scheduled publishing (R4)**: page the due-item query (500), child scope per page, per-item try/catch with counter. Follows the import precedent.
5. **Leave alone**: the shape system (R5) and kernel complexity (R7). Wrong quarter, wrong owner, enormous blast radius, working as designed. Saying *what you would not touch* is the seniority signal.

## Confirmed vs hypothesis

- Confirmed by reading: the *mechanics* cited above (what code does on the happy path).
- Hypotheses needing runtime/upstream verification: R1 exploitability, R2 frequency, R6 gating. All carried in [09-reference/risk-register.md](../09-reference/risk-register.md) with confidence levels. Do not present these as CVEs; present them as review findings pending verification — that phrasing matters in interviews too.

Interview angle: this file *is* the answer to "critique a system you know well" / "what would you improve at your last job." Practice delivering risks R1–R4 in 90 seconds each: claim → evidence (file:line) → impact → fix → cost.
