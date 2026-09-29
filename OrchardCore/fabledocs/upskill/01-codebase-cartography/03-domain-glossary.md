# Domain Glossary

The vocabulary this codebase assumes. Each term: what it means here, where it lives, and near-synonyms that will confuse you.

| Term | Meaning in Orchard Core | Code home |
| --- | --- | --- |
| **Tenant** | One logical site (own DB/table-prefix, features, users) inside one process. Identified by `ShellSettings.Name`; matched by host and/or URL prefix. | [`ShellSettings.cs:9-13`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs) |
| **Shell** | The *runtime materialization* of a tenant: its DI container + pipeline. "Shell" ≈ tenant-at-runtime. | [`ShellHost.cs:11-16`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) |
| **ShellContext** | The per-tenant container object; can be a lightweight `PlaceHolder` before first request. | [`ShellHost.cs:367`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) |
| **ShellScope** | A DI scope over a ShellContext plus lifecycle machinery: before-dispose callbacks, deferred tasks, exception handlers. One per request; also created for background work. | [`ShellScope.cs:12-23`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs) |
| **Default tenant** | The special tenant named `Default` that manages others; cannot be removed. | [`ShellSettings.cs:19`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs), [`ShellHost.cs:513-526`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) |
| **Feature** | The unit of enable/disable. A module assembly can declare several features; each maps to DI registrations. Declared in `Manifest.cs`. | [`Contents/Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) |
| **Module** | A project under `src/OrchardCore.Modules/` — controllers, drivers, views, migrations, manifest. | e.g. [`OrchardCore.PublishLater/`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater) |
| **Content item** | A JSON document with identity (`ContentItemId`) and versions (`ContentItemVersionId`). The universal noun of the CMS. | [`DefaultContentManager.cs:67-100`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) |
| **Content type** | A named composition of parts/fields, defined *as data* (not classes) via `IContentDefinitionManager`. | [`Contents/Migrations.cs:16-23`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Migrations.cs) |
| **Content part** | A reusable behavior+data chunk welded onto items (TitlePart, AutoroutePart). C# class + driver + handler. | [`TitlePartDisplayDriver.cs`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) |
| **Content field** | A smaller unit attached to parts (TextField, etc.) under `OrchardCore.ContentFields`. Part:field ≈ table:column, loosely. | `src/OrchardCore.Modules/OrchardCore.ContentFields/` |
| **Latest / Published** | Version flags. Invariant: per `ContentItemId`, at most one row `Latest == true` and at most one `Published == true` (may be the same row). | [`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) |
| **Handler** (`IContentHandler`) | Lifecycle event subscriber (Creating/Published/Removed…). Business side effects live here. | [`AutoroutePartHandler.cs:63-113`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) |
| **Driver** (`IDisplayDriver`) | Maps a model to shapes for display/edit and binds form posts back. UI concerns only. | [`TitlePartDisplayDriver.cs:20-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) |
| **Shape** | A dynamic view-model node with a type name and *alternates*; resolved to a template at render time. Themes override by supplying a template for an alternate. | [`ContentItemDisplayManager.cs:45-99`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs) |
| **Placement** | Declarative rule putting a shape in a zone at a position (`"Content:5"`), per display type. | [`Contents/placement.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json) |
| **Display type** | Rendering context: `Detail`, `Summary`, `SummaryAdmin`, `Edit`. Same item, different shape trees. | [`AdminController.cs:203`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) |
| **Aspect** | A computed projection over an item (e.g. `ContentItemMetadata`, `RouteHandlerAspect`) populated by handlers on demand — not stored. | [`AutoroutePartHandler.cs:161-175`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) |
| **YesSql / Session** | Document store over relational DBs. `ISession` = unit of work; documents serialize to JSON; **indexes** are SQL tables you query. | [`OrchardCoreBuilderExtensions.cs:33-181`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) |
| **Index (YesSql)** | A projection table maintained from documents by an `IndexProvider`. `MapIndex` = one row per doc. Not a SQL index (though migrations add those too — naming collision, watch out). | [`ContentItemIndex.cs:5-73`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs) |
| **Migration** | Per-module schema/definition evolution: `CreateAsync` + `UpdateFromNAsync`, tracked per tenant. | [`PublishLater/Migrations.cs:18-89`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs) |
| **Recipe** | JSON script of setup/config steps, executed at setup or import; supports `[js:...]` scripting substitution. | [`RecipeExecutor.cs:40-158`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs) |
| **Deployment plan** | Export counterpart of recipes (content/config → JSON package). | `src/OrchardCore.Modules/OrchardCore.Deployment/` |
| **Autoroute** | Slug system: Liquid pattern → unique path → route entries mapping URL→item. | [`AutoroutePartHandler.cs`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) |
| **Deferred task** | Work registered during a request that runs *after* the scope ends (post-commit) in a fresh scope. Orchard's answer to "side effects after save". | [`ShellScope.cs:387-392, 468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs) |
| **Permission** | Named grant with implication chains (`PublishContent` implies `EditContent`…). Modules contribute their own; content types get dynamic variants. | [`CommonPermissions.cs:16-56`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs) |
| **Stereotype** | A content-type tag ("Widget", "MenuItem") that changes rendering/listing behavior. | [`ContentItemDisplayManager.cs:53-60`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs) |

## Confusables

- **Shell vs Scope vs Context**: `ShellContext` = the tenant's container (long-lived); `ShellScope` = one unit of execution over it (short-lived); "shell" informally = both. If you say "the shell commits the session," a maintainer will hear "the *scope's* before-dispose callback commits."
- **YesSql Index vs SQL index**: `PublishLaterPartIndex` is a *table*; `CreateIndex("IDX_...")` in the same migration is a *database index on that table* ([`PublishLater/Migrations.cs:24-39`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)).
- **Publish (content) vs Publish (event)**: `PublishAsync` flips version flags; `PublishedAsync` handlers are the fan-out.
- **ContentItemId vs ContentItemVersionId vs Id**: logical identity vs version identity vs YesSql document row id (`long`). Bulk actions use document ids ([`AdminController.cs:267-274`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)).
- **Feature vs Module**: one module assembly may host several toggleable features ([`Contents/Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) declares six).

Interview angle: "define multi-tenancy isolation levels" — answer with the Shell/ShellContext/table-prefix story; it's more concrete than the textbook "shared schema vs separate DB" list, and this repo supports *both* (per-tenant connection string or shared DB with `TablePrefix`, [`OrchardCoreBuilderExtensions.cs:104-109`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)).
