# Fast Track — One Weekend in Orchard Core

Goal: by Sunday night you can run the CMS, trace two end-to-end flows, make one safe change, run one test, and explain the architecture aloud like a mid-level candidate.

## Saturday morning — run it

Prerequisites (from `global.json` and `AGENTS.md`): .NET SDK 10.0.200+ (rolls forward), Node 22.x + Yarn 4 only if you touch frontend assets.

```bash
# From repo root — __inferred__ from AGENTS.md:50-57 (not run in this workspace)
cd src/OrchardCore.Cms.Web
dotnet run -f net10.0
# → http://localhost:5000
```

The entire host is ~20 lines: [`src/OrchardCore.Cms.Web/Program.cs:1-22`](../../src/OrchardCore.Cms.Web/Program.cs) — `AddOrchardCms()` plus `app.UseOrchardCore()`. Everything else is modules. On first launch you get the **setup screen**: pick the "Blog" recipe, SQLite, create the admin user. That setup run is itself a flow worth understanding later (a *recipe* executes JSON steps inside a shell scope — [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md), Flow 7).

```bash
# Unit tests — __inferred__ from AGENTS.md:59-75
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj
# One class only:
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-class "*ContentElementTests*"
```

## Saturday afternoon — the first 10 files, in order

Open these in order. Predictions before scrolling — that's the drill.

1. [`src/OrchardCore.Cms.Web/Program.cs`](../../src/OrchardCore.Cms.Web/Program.cs) — the whole host. *Predict: where does routing get wired?* (Answer: inside `UseOrchardCore()`, per tenant.)
2. [`src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs:27-69`](../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs) — every request is matched to a tenant, and `RequestServices` is swapped to that tenant's container.
3. [`src/OrchardCore/OrchardCore/Shell/ShellHost.cs:80-119`](../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) — how tenant containers are created lazily, one semaphore per tenant.
4. [`src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs:9-13`](../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs) — where tenant config lives (`App_Data/tenants.json`).
5. [`src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs:376-428`](../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) — `PublishAsync`: the Latest/Published version invariant in one method.
6. [`src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs`](../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs) — the YesSql index table: JSON documents made queryable.
7. [`src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs:739-788`](../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) — `EditInternalAsync`: authorize → update editor → validate → publish-or-cancel.
8. [`src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs`](../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) — the smallest complete display driver: Display/Edit/Update.
9. [`src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs:137-173`](../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) — the unit of work: the session auto-commits when the shell scope ends.
10. [`src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs`](../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) — how a module declares features and dependencies.

## Sunday morning — trace two flows

Do these with the debugger or by reading, filling in a trace table (template in [04-code-reading-gym/02-trace-tables.md](04-code-reading-gym/02-trace-tables.md)):

1. **Edit & publish a blog post** — Flow 2 in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md). Controller → display manager → drivers → content manager → handlers → session commit at scope disposal.
2. **Request routing to a tenant** — Flow 1. Middleware → `RunningShellTable.Match` → shell scope → tenant pipeline.

## Sunday afternoon — one safe change, one test

- **Safe change**: in your *local run only*, alter a template override or add a log line in `TitlePartDisplayDriver.UpdateAsync` ([`src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs:49-64`](../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)), rerun, and watch it hit when you save a post's title. Revert after.
- **One test**: run `dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-class "*ContentElementTests*"` and then *read* [`test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs:9-38`](../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs) — it tests a real serialization subtlety (getting a part as base type then concrete type returns different instances).

## Sunday evening — teach-back (interview simulation)

Say aloud, 3 minutes, no notes: *"Orchard Core is a modular multi-tenant CMS. A request comes in, middleware matches the host/prefix to a tenant, and swaps in that tenant's own DI container — tenants are isolated at the container level, not just by a column in the database. Content is stored as JSON documents in YesSql with index tables for querying. Every content item ID can have multiple version rows; exactly one may be `Published` and one is `Latest` — that invariant is maintained in `PublishAsync`. Writes go through a session that commits once, when the request's shell scope ends — a unit of work. Rendering goes through drivers that emit shapes, and placement files decide where shapes land."*

If any sentence in that paragraph feels like someone else's words, that's the part to re-trace.

## What the fast path skips

Deliberately skipped: GraphQL, recipes/deployment, background tasks, dynamic cache, Liquid templating, workflows, search/indexing, media. All are covered in the full tracks — start with [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).
