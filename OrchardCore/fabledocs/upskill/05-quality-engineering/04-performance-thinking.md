# Performance Thinking

Rule zero: **measure first**. This repo ships `test/OrchardCore.Benchmarks` (BenchmarkDotNet) and a MiniProfiler module (`src/OrchardCore.Modules/OrchardCore.MiniProfiler/`) — enable it on a dev tenant before theorizing. The sections below are *where to look*, ranked by likelihood, each with the measurement that confirms or kills the hypothesis.

## Performance domains in this system

| Domain | Where it lives | Likely hotspot | Confirming measurement |
| --- | --- | --- | --- |
| DB round-trips | YesSql queries | admin list page; per-item handler queries | MiniProfiler SQL panel; YesSql logger |
| Server CPU | shape building, Liquid | pages with many shapes/widgets | dotnet-trace / MiniProfiler steps |
| Cold start | shell container builds | first request per tenant after deploy/reload | request timing after `ReloadShellContextAsync` |
| Distributed cache | DynamicCache round-trips | cache-context discriminator fan-out | count `GetDiscriminatorsAsync` calls per page |
| Memory | container-per-tenant | high tenant counts | dotnet-counters, GC heap per tenant added |
| Frontend | per-module assets | admin pages loading many module bundles | browser devtools network |

## Concrete hotspots with anchors

**1. The admin list count** — `await query.CountAsync()` runs unless `MaxPagedCount` caps it ([`AdminController.cs:188-190`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)); on large tables the count can dominate the page. Also each listed item runs `BuildDisplayAsync` (`:199-204`) — O(page size) shape trees. Measurement: MiniProfiler; expect count + 1 query per additional index a driver touches.

**2. Sequential handler loops.** Every content operation awaits each handler serially ([`DefaultContentManager.cs:85-96`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)); handlers that query (Autoroute uniqueness check — [`AutoroutePartHandler.cs:465-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) add per-save round-trips. A save touching k handler queries = k sequential awaits. This is the "serial async" smell: correct (they may depend on each other) but worth knowing when saves feel slow.

**3. N+1 in disguise: `GetAsync` in loops.** `ScheduledPublishingBackgroundTask` loads items one by one inside its loop ([`ScheduledPublishingBackgroundTask.cs:41-43`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)) — fine at 5 items/tick, not at 5,000. The repo's own batch alternative: multi-id `GetAsync(IEnumerable<string>, ...)` ([`DefaultContentManager.cs:205-318`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) which recalls cached items and fetches the rest with one `IsIn` query — including a **read-cache** (`_contentManagerSession.RecallPublishedItemId`, `:250-258`) that turns repeated loads within a request into no-ops.

**4. Index write amplification.** Every content save re-maps every registered index for that item ([`ContentItemIndexProvider`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs) + per-part indexes). Ten indexed parts = ten table writes per save. Measure before adding an index to a hot type; consider `QueryIndex` reads to justify each.

**5. Missing DB indexes on custom YesSql tables.** The repo's own convention is a covering composite per background query ([`PublishLater/Migrations.cs:31-39`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)). A custom index table without a DB index = table scan per poll tick, forever, invisibly.

**6. Unbounded queries.** GraphQL caps results via `DefaultNumberOfResults` ([`ContentItemsFieldType.cs:203-222`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)); `GetAllVersionsAsync` does not cap ([`DefaultContentManager.cs:188-203`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) — an item with thousands of versions loads them all (version pruning exists as a feature for this reason: `OrchardCore.Contents.VersionPruning` in the [manifest](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs)).

**7. Cold starts.** First hit per tenant builds the container + pipeline (Flow 1). Mitigations already present: placeholders defer cost; `ShellWarmup` option for background service ([`ModularBackgroundService.cs:52-63`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). If p99 spikes after deploys, this is why.

## How to find the classics here

- **N+1**: YesSql logger on; load a list page; count identical-shaped queries. Fix with `IsIn` batching (pattern at [`DefaultContentManager.cs:260-265`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
- **Expensive renders**: MiniProfiler step per shape; suspect Liquid templates with queries inside loops.
- **Oversized bundles**: check per-module `Assets.json` outputs; admin pages aggregate many modules' scripts.
- **Missing index**: `EXPLAIN` the query the YesSql logger shows; compare against the migration's `CreateIndex` columns.

## The senior framing (use in interviews)

Performance work here is *architecture-aware*: you don't "optimize the ORM," you decide **document load vs index-only read** (`QueryIndex` vs `Query<T,TIndex>`), **per-request cache hits** (content manager session), **write amplification budgets** (indexes per type), and **cold-start amortization** (placeholders/warmup). Every fix names which axis it moves and what it costs on another.

## Drill

The admin list is slow on a site with 2M content items. Rank your first three measurements and predict the winner. Model answer: (1) MiniProfiler → is it `CountAsync`? (2) if yes, check `MaxPagedCount`/filtering; (3) per-item `BuildDisplayAsync` cost × page size. Strong answers mention: fix = cap counts or filtered indexes — *not* "cache the page" as step one.
