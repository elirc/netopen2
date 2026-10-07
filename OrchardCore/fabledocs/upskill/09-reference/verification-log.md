# Verification Log

Running log of what was actually inspected while building this curriculum. Date: **2026-07-16**. Working copy: `C:\Users\Owner\Desktop\.netopen2\OrchardCore`, git `main` @ `29649872e` ("Fix theme extensions not appearing when manifest includes additional …").

## Commands run (all __verified__)

| Command | Result |
| --- | --- |
| `git log --oneline -5` | HEAD `29649872e`; active repo with recent commits |
| `ls` on repo root, `src/`, `src/OrchardCore/`, `src/OrchardCore.Modules/`, `test/` | 97 entries under `OrchardCore.Modules`, 104 under `src/OrchardCore` (core libs), 9 test projects |
| `cat package.json`, `global.json` | Yarn 4.17 workspaces for assets; .NET SDK 10.0.200, Microsoft.Testing.Platform runner |
| `cat AGENTS.md` (build/test sections) | Documented commands recorded in command-cheatsheet.md as source-verified |
| `wc -l` on ~20 source files | Sizes recorded before reading; all reads below are full-file unless noted |

**Not run**: `dotnet build`, `dotnet test`, `dotnet run`, `yarn` — no build was executed in this workspace. Every build/run/test command in these docs is therefore marked __inferred__ (sourced from `AGENTS.md`, `test/OrchardCore.Tests.Functional/README.md`, `package.json`, CI workflows in `.github/workflows/`), not demonstrated here.

## Files read in full (line anchors written from these reads)

| File | Focus |
| --- | --- |
| `src/OrchardCore.Cms.Web/Program.cs` | Host bootstrap |
| `src/OrchardCore/OrchardCore/Shell/ShellHost.cs` | Tenant lifecycle, semaphore-per-tenant, reload retry loop |
| `src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs` | Tenant settings model, state machine |
| `src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs` | Scope lifecycle, deferred tasks/signals, exception handlers |
| `src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs` | Tenant resolution middleware |
| `src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs` | Background scheduler, distributed locks |
| `src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs` (lines 1–700 of 1264) | New/Get/Publish/Unpublish/versioning/Create/Import |
| `src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs` | Map index + provider |
| `src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs` | BuildDisplay/BuildEditor/UpdateEditor |
| `src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs` | Store config per provider, session registration, auto-commit |
| `src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs` | Full admin CRUD controller |
| `src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs` | Minimal-API content create/update |
| `src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs` | Dynamic permissions, owner variants |
| `src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs` | Permission hierarchy |
| `src/OrchardCore.Modules/OrchardCore.Contents/Migrations.cs`, `Manifest.cs`, `placement.json` (head), `Startup.cs` (lines 1–80 of large file) | Module anatomy |
| `src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs` | Login flow, timing normalization, rate-limit group |
| `src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs` | Slug generation/uniqueness, publish side effects |
| `src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs` | Display driver example |
| `src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs` | Background task example |
| `src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs` | Schema migration evolution (UpdateFrom1/2) |
| `src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs` | Cache, failover circuit breaker |
| `src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs` (lines 1–240 of 477) | GraphQL query resolution |
| `src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs` | Recipe execution |
| `test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs` (lines 1–60) | Unit test style |
| `test/OrchardCore.Tests.Functional/README.md` (head) | Playwright in-process E2E architecture |

## Uncertainties and open questions (also in risk-register.md)

- **`CreateEndpoint` publish authorization asymmetry** — update path checks only `EditContent` yet may call `PublishAsync` when `draft=false` (`CreateEndpoint.cs:97-127`). Labeled *investigate*, not a confirmed vulnerability: an outer dynamic-permission handler or upstream fix may mitigate; not exercised at runtime here.
- **Autoroute path uniqueness race** — `IsAbsolutePathUniqueAsync` is check-then-write within a session (`AutoroutePartHandler.cs:465-477`); two concurrent publishes could both pass. Labeled *possible risk*; no DB-level unique constraint was located, but I did not exhaustively search migrations.
- **GraphQL draft exposure** — `status: DRAFT` argument exists (`ContentItemsFieldType.cs:69-76`); which permission gates it was not fully traced (GraphQL execution options in `OrchardCore.Apis.GraphQL` were not read). Labeled *investigate*.
- Line numbers for files listed as partial reads are only cited within the ranges actually read.
- The ~95 modules not listed above (Workflows, Media, Lucene/Elasticsearch, Layers, Localization, etc.) were surveyed by directory listing only; claims about them in these docs are structural (existence, naming) not behavioral.

## 2026-10-06 re-check

Static re-check against `elirc/netopen2` (single snapshot commit `88abe97`; the `29649872e` history is not included). Nothing was built or run.

- Risk-register anchors R1–R9 still point at the described code. R1: `CreateEndpoint.cs` checks only `EditContent` (L97) and publishes when `!draft` (L124–127), while `AdminController.cs:876` requires `PublishContent`. R2: `IsAbsolutePathUniqueAsync` is at `AutoroutePartHandler.cs:465`, and `OrchardCore.Autoroute/Migrations.cs` creates `AutoroutePartIndex` with no unique index. R3: both catch blocks (`ModularBackgroundService.cs:222-228`, `ShellScope.cs:497-504`) log and call `HandleExceptionAsync`, with no retry.
- The counts above still hold: 97 entries under `OrchardCore.Modules`, 104 under `src/OrchardCore`, 9 test projects, SDK `10.0.200`.
- `Program.cs` has 22 lines; three `:1-23` anchors were corrected. `package.json` lists a `src/Frontend/` workspace that is not in this snapshot.

## Method note

Anchors were written immediately after reading each file in this working copy, not from memory of upstream OrchardCore. If the repo is updated, anchors may drift; prefer searching for the quoted symbol names.
