# File Reading Order

28 files, in the order that builds a mental model fastest. Per file: why it matters, what to look for, what to ignore. Junior path = 1–14. Mid path = 1–22. Senior path = all, plus the "senior noticing" notes.

## Act I — the host and the tenant boundary (1–7)

1. [`src/OrchardCore.Cms.Web/Program.cs`](../../../src/OrchardCore.Cms.Web/Program.cs) — *Why*: proves the app is nothing but modules. *Look for*: `AddOrchardCms()`, `UseOrchardCore()`. *Ignore*: NLog.
2. [`src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs) — *Why*: the tenant identity record. *Look for*: `Name`, `RequestUrlHost`/`RequestUrlPrefix`, `State`, `VersionId` (lines 87–100 — note `TenantId` falls back to first `VersionId`). *Ignore*: disposal plumbing.
3. [`src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs) — *Why*: THE multi-tenancy trick. *Look for*: `_runningShellTable.Match(httpContext)` (line 32), the 503 for initializing tenants (37–43), `shellScope.UsingAsync` wrapping `_next` (58–67). *Senior noticing*: the exception handler feature is consulted *inside* the scope so scope-level exception handlers see request failures.
4. [`src/OrchardCore/OrchardCore/Shell/ShellHost.cs`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) — *Why*: tenant lifecycle. *Look for*: semaphore-per-tenant double-check (80–119), placeholder shells (335–374), reload retry loop with `ReloadShellMaxRetriesCount` (182–245). *Senior noticing*: `while (shell is null)` re-loops when a released shell is observed — a lock-free retry against a concurrent release.
5. [`src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs) — *Why*: the execution-context spine. *Look for*: `AsyncLocal` current scope (14, 227–230), `UsingAsync` (247–284), deferred tasks running in a *new* scope after disposal (468–508). *Ignore*: the static `Get/Set` item helpers.
6. [`src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) — *Why*: unit of work + provider matrix. *Look for*: session registration with `RegisterBeforeDispose(CommitAsync)` + `AddExceptionHandler(CancelAsync)` (137–173). *Senior noticing*: only the *shell scope's own* session auto-commits (`sp == shellScope?.ServiceProvider`, line 155) — child service scopes must commit explicitly.
7. [`src/OrchardCore/OrchardCore/Modules/ModularTenantRouterMiddleware.cs`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantRouterMiddleware.cs) — *Why*: each tenant builds its own request pipeline. *Look for*: pipeline built once per shell context.

## Act II — the content domain (8–15)

8. [`src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs) — *Why*: the queryable projection of JSON documents. *Look for*: `Published`, `Latest` flags; truncation to 255 chars in the provider (50–68). *Senior noticing*: silent truncation means index values can diverge from document values — filtering on long titles is lossy.
9. [`src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) — *Why*: the heart. *Look for*: `GetAsync` version options (102–186), `PublishAsync` previous-version demotion (376–428), `BuildNewVersionAsync` (497–542), `CreateAsync` (604–663). *Ignore on first pass*: `ImportAsync` batching.
10. [`src/OrchardCore/OrchardCore.ContentManagement/Handlers/`](../../../src/OrchardCore/OrchardCore.ContentManagement/Handlers) (skim folder) — *Why*: the event pipeline every mutation flows through. *Look for*: context types (Creating/Created, Publishing/Published…), forward vs `ReversedHandlers` invocation in `DefaultContentManager`.
11. [`src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) — *Why*: canonical CRUD controller. *Look for*: `EditInternalAsync` (739–788): update → `ModelState` check → `_session.CancelAsync()` on failure. `FormValueRequired` dispatching multiple POST handlers to one action name.
12. [`src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs) — *Why*: permission hierarchy with "Own" variants. *Look for*: implied-by chains (16–44), `OwnerPermissionsByName` map (46–56).
13. [`src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs) — *Why*: converts static permission checks into per-content-type dynamic ones; ownership check at line 88–96.
14. [`src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) — *Why*: smallest complete driver: Display (20–36), Edit (38–47), UpdateAsync with validation (49–64).
15. [`src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs) — *Why*: how a content item becomes a shape tree. *Look for*: shape type/alternates computation (45–99).

## Act III — module anatomy and cross-cutting flows (16–22)

16. [`src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) — features + dependencies as assembly attributes.
17. [`src/OrchardCore.Modules/OrchardCore.Contents/Startup.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Startup.cs) — a big `ConfigureServices`; note how *everything* is DI registration.
18. [`src/OrchardCore.Modules/OrchardCore.Contents/placement.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json) — shape → zone:position mapping.
19. [`src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs) — *Look for*: `LoginPOST` (102–206): rate-limit attribute, event hooks, lockout, and the timing-normalization branch (183–188).
20. [`src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) — *Look for*: `ValidatingAsync` uniqueness check (115–133), Liquid-pattern slug generation (378–423), `-2` suffix loop (437–463).
21. [`src/OrchardCore.Modules/OrchardCore.PublishLater/`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater) (whole module, it's small) — background task (Services/), index (Indexes/), migration with `UpdateFrom1/2` (Migrations.cs), driver — a full vertical slice in ~9 files. **Best single module to imitate.**
22. [`src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) — *Look for*: per-tenant scheduling loop (78–96), distributed lock acquisition (133–157), fake `HttpContext` for background flows (108–109).

## Act IV — senior extension (23–28)

23. [`src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs) — JSON steps + scripting substitution (198–264).
24. [`src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs) — dynamic where-clause building over index tables.
25. [`src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs) — failover circuit breaker (93–135).
26. [`src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs) — minimal API + validation + authz; compare its permission checks to AdminController's.
27. [`test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs) + [`test/OrchardCore.Tests.Functional/README.md`](../../../test/OrchardCore.Tests.Functional/README.md) — the two test styles.
28. [`AGENTS.md`](../../../AGENTS.md) — the repo's own onboarding for agents/contributors; compare against what you now know.

## Drill

After Act II, close everything and write the life of one blog post from `CreatePOST` to a committed row, naming each file it passes through. Check yourself against Flow 2 in [05-key-flows.md](05-key-flows.md).
