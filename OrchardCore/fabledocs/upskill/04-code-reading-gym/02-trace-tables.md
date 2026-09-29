# Trace Tables

Fill each table yourself first (columns: Step · File/line · Value shape · Owner · Transformation · Risk). Model answers provided. These map one-to-one onto debugging-interview narration: each row is a sentence.

---

## Trace 1 (UI → API → domain): Saving a new BlogPost title

Scenario: admin fills "My First Post" and clicks **Publish** on `Contents/ContentTypes/BlogPost/Create`.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | browser form | `TitlePart.Title=My First Post`, `submit.Publish` | UI | — | prefix must match driver's `Prefix` |
| 2 | [`AdminController.cs:376-408`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) | route id=`BlogPost` | Contents module | `FormValueRequired` picks CreateAndPublishPOST; authz `PublishContent` on dummy item | wrong-permission path (403) |
| 3 | [`DefaultContentManager.cs:67-100`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) | empty `ContentItem` + welded parts | domain | `NewAsync`: Activating handlers build item from type definition | type definition missing → built ad hoc (73-75) |
| 4 | [`ContentItemDisplayManager.cs:146-191`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs) | shape tree + form values | display layer | `UpdateEditorAsync` walks drivers | driver not registered → field silently unbound |
| 5 | [`TitlePartDisplayDriver.cs:49-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) | `TitlePart.Title` string | Title module | `TryUpdateModelAsync` prefix-bound; sets `DisplayText` | required-title rule only if settings say so |
| 6 | [`AdminController.cs:713-716`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) | draft item | Contents | `CreateAsync(Draft)` → Creating/Created handlers | ModelState invalid → `CancelAsync` (720) |
| 7 | [`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) | `Published=true` | domain | publish + handler fan-out | handler veto (`context.Cancel`) |
| 8 | [`OrchardCoreBuilderExtensions.cs:157-163`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) | SQL: Document insert + index rows | YesSql | JSON serialize + map indexes | commit-time concurrency exception |

## Trace 2 (persistence): What one publish writes

Starting state: item `4f...` has one row (v1, Published+Latest). User edits & publishes.

| Step | File/line | Rows touched | Transformation |
| --- | --- | --- | --- |
| 1 | `GetAsync(DraftRequired)` [`:159-183`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) | v1 saved (concurrency-checked); v2 created `Latest=true, Published=false`, v1.Latest=false | 1 row → 2 rows |
| 2 | drivers/handlers mutate v2 JSON | v2 `Data` | in-memory |
| 3 | `PublishAsync` [`:388-423`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) | query previous published (v1); v1.Published=false; v2.Published=true | flag flips |
| 4 | commit | Document ×2 updated/inserted; `ContentItemIndex` ×2 remapped; `AutoroutePartIndex` etc. remapped | one transaction |

Invariant check row: after commit — exactly one `Published` row (v2), one `Latest` row (v2), v1 = history. If you crashed before step 4: **nothing changed** (single transaction). That answer wins debugging rounds.

## Trace 3 (auth): `AuthorizeAsync(User, EditContent, item)` for a non-owner editor

| Step | File/line | Decision state |
| --- | --- | --- |
| 1 | [`AdminController.cs:850-851`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) | resource-based check enters policy pipeline |
| 2 | [`ContentTypeAuthorizationHandler.cs:22-26`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs) | prior handler already succeeded? → abstain |
| 3 | `:38-46` | owner check: `user.NameIdentifier == content.Owner`? No → keep `EditContent` (no Own downgrade) |
| 4 | `:48-62` | convert to dynamic `EditContent_{BlogPost}` |
| 5 | `:70-75` | recursive `AuthorizeAsync` on the dynamic permission → role grants decide; implication chain (`PublishContent` ⇒ `EditContent`) evaluated by the permission handler |
| 6 | aggregate | any handler succeeded → allow; none → forbid (handlers never *fail*, they abstain) |

Risk row: step 3 depends on the caller passing the item as resource. An endpoint checking `AuthorizeAsync(User, EditContent)` (no resource) skips ownership entirely — the exact bug class to hunt in review.

## Trace 4 (error path): Slug conflict on publish

| Step | File/line | What the user sees |
| --- | --- | --- |
| 1 | `ValidatingAsync` [`AutoroutePartHandler.cs:115-133`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) | — (`context.Fail("Your permalink is already in use.")`) |
| 2 | `ValidateAsync` results → ModelState (API: [`CreateEndpoint.cs:136-152`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)) | 400 ValidationProblem with member name `Path` |
| 3 | UI path: ModelState invalid → [`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) `CancelAsync` + re-render | form with inline error, nothing persisted |
| 4 | If instead both requests validated concurrently (race) | both succeed; duplicate path; symptom appears later as "wrong page served" — see [05-quality-engineering/03-systematic-debugging.md](../05-quality-engineering/03-systematic-debugging.md) Scenario 2 |

## Trace 5 (background): One tick of scheduled publishing on tenant "agency"

| Step | File/line | Concurrency guard |
| --- | --- | --- |
| 1 | poll loop [`ModularBackgroundService.cs:78-96`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) | single hosted service per process |
| 2 | scheduler due? `:415-418` | per (tenant×task) scheduler state |
| 3 | distributed lock `:133-157` | cross-instance exactly-one-runner |
| 4 | scope + task resolve `:159-215` | tenant container only |
| 5 | due-item query [`ScheduledPublishingBackgroundTask.cs:29-32`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) | predicate idempotency |
| 6 | publish loop + commit at scope end | one transaction per tick |

Fill-in challenge: add the failure column yourself — what happens at each step if it throws? (Answers are all in [`ModularBackgroundService.cs`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)'s catch blocks; every one logs and continues/skips.)

---

Self-grade across traces — Basic: right files in right order. Solid: value shapes and owners correct. Strong: every risk cell filled with a *mechanism*, not "might fail."
