# Writing Tests Here — Six Recipes

Commands are __inferred__ from `AGENTS.md`/`global.json` (xUnit v3 on Microsoft.Testing.Platform); adjust if your local runner differs. Repo test style observed in [`ContentElementTests.cs`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs): plain `[Fact]`, arrange/act/assert with comments, `JConvert` helpers for serialization round-trips.

```bash
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj                      # all unit tests
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-class "*ContentElementTests*"
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-method "*.MyNewTest"
dotnet test test/OrchardCore.Tests.Functional/OrchardCore.Tests.Functional.csproj -c Release --filter-class "*Cms*"
```

## Recipe 1 — Happy path: content round-trip (unit)

Goal: a welded part survives serialize→deserialize with data intact.
Model: [`ContentElementTests.cs:9-38`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs).

```csharp
// Illustrative fake code: not from this repo (new test, repo style)
[Fact]
public void Weld_ThenRoundTrip_PreservesPartData()
{
    var item = new ContentItem { ContentType = "BlogPost" };
    item.Weld(new TitlePart { Title = "hello" });

    var restored = JConvert.DeserializeObject<ContentItem>(JConvert.SerializeObject(item));

    Assert.Equal("hello", restored.Get<TitlePart>(nameof(TitlePart)).Title);
}
```
Place in `test/OrchardCore.Tests/ContentManagement/`. Run with `--filter-method "*Weld_ThenRoundTrip*"`.

## Recipe 2 — Validation failure surfaces (unit, driver level)

Goal: `TitlePartDisplayDriver.UpdateAsync` adds a model error when settings demand a title and it's blank ([`TitlePartDisplayDriver.cs:49-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)).
Approach: construct the driver with a stub `IStringLocalizer`, build an `UpdatePartEditorContext` whose updater is a test `IUpdateModel` capturing ModelState (see `test/OrchardCore.Tests/Stubs/` for existing fakes before writing your own). Assert one error keyed by prefix.
The transferable part: test the driver *directly* — you don't need MVC to test binding logic that lives in your code.

## Recipe 3 — Permission failure (unit, handler level)

Goal: non-owner with only `EditOwnContent` grant is not authorized for someone else's item; owner is.
Model: the existing `test/OrchardCore.Tests/Security/PermissionHandlerTests.cs` + `PermissionHandlerHelper.cs` (listed; open them first and follow their fixture style).
Assert both directions of [`ContentTypeAuthorizationHandler.HasOwnership`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs) (`:88-96`): claims id equal / different from `content.Owner`.

## Recipe 4 — Cross-tenant isolation (functional)

Goal: content created on tenant A never appears on tenant B.
Model: the functional suite already covers multi-tenancy per its [README](../../../test/OrchardCore.Tests.Functional/README.md) — extend rather than duplicate: new fixture, create two tenants via setup recipes, POST content to A (Recipe: `api/content` with Api auth), GET on B, assert 404/absence.
This is the test that *proves* the container-per-tenant story instead of asserting it.

## Recipe 5 — Async side effect: scheduled publish (integration-style)

Goal: an item with `ScheduledPublishDateTimeUtc` in the past becomes published when the task runs.
Approach: resolve/construct `ScheduledPublishingBackgroundTask` with a fake `IClock` pinned *after* the schedule; invoke `DoWorkAsync(serviceProvider, ct)` directly against a test shell scope — do **not** wait for the real cron. Assert: item published *and* `ScheduledPublishUtc` cleared (the pairing at [`ScheduledPublishingBackgroundTask.cs:45-56`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)).
Then the **idempotency assertion** juniors skip: run `DoWorkAsync` again; assert nothing changes.

## Recipe 6 — Migration behavior (matrix)

Goal: `UpdateFrom2Async` ([`PublishLater/Migrations.cs:63-89`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)) upgrades a v2 schema.
Honest answer: locally, run the site on SQLite with a pre-upgrade `App_Data`, watch `AutomaticDataMigrations` apply it; in CI, trust `functional_all_db.yml`. Migrations are the one place where "test = run it on every provider" is the strategy, and saying so beats pretending a unit test covers it.

---

## The regression-test habit

Every bug fix in [03-systematic-debugging.md](03-systematic-debugging.md) ends with "add the test that would have caught it." That test goes at the **lowest layer that can express the bug** — e.g. the R1 authz asymmetry needs an endpoint-level test (permission checks are endpoint code), while a suffix-generation bug in Autoroute fits a pure unit test of `GenerateRelativeUniquePath` inputs.

Interview angle: "how would you test X?" — always answer in the shape *layer choice → seam used → the one non-obvious assertion* (idempotency, rollback, isolation), never just "I'd write a unit test."
