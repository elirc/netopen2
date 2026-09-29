# Fake-Code Contrasts

Ten bad-vs-better pairs. **All snippets are fake** (labeled) — the "better" shape mirrors what this repo actually does, with the real anchor cited. Read the bad one, articulate the smell aloud, then compare.

---

## 1. Validation without rollback

```csharp
// Illustrative fake code: not from this repo — BAD
var shape = await _displayManager.UpdateEditorAsync(item, this, false);
if (!ModelState.IsValid) return View(shape);   // mutations still commit at scope end!
```
```csharp
// Illustrative fake code: not from this repo — BETTER
if (!ModelState.IsValid) { await _session.CancelAsync(); return View(shape); }
```
Real shape: [`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs). In commit-at-scope-end systems, *not saving* requires an explicit act.

## 2. Ownership check in the controller

```csharp
// Illustrative fake code: not from this repo — BAD
if (item.Owner != CurrentUserId() && !User.IsInRole("Admin")) return Forbid();
```
```csharp
// Illustrative fake code: not from this repo — BETTER
if (!await _authorizationService.AuthorizeAsync(User, CommonPermissions.EditContent, item)) return Forbid();
```
Real shape: resource-based check + owner-variant handler ([`ContentTypeAuthorizationHandler.cs:38-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)). Inline ownership logic forks policy per endpoint — the IDOR factory.

## 3. Coupling UI shape to storage shape

```csharp
// Illustrative fake code: not from this repo — BAD
public IActionResult Edit(string id) {
    var doc = _session.GetRawDocument(id);
    return View(doc); // view renders the JSON document directly
}
```
Better: bind through drivers into view models ([`TitlePartDisplayDriver.cs:38-47`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)); the document schema can evolve without breaking every template. Coupling view ⇄ storage means every storage rename is a UI incident.

## 4. Missing permission filter in a list

```csharp
// Illustrative fake code: not from this repo — BAD
var types = await _contentDefinitionManager.ListTypeDefinitionsAsync();
model.CreatableTypes = types.Select(t => new SelectListItem(t.DisplayName, t.Name)).ToList();
```
Better: filter each option through authorization, as the real list page does per type ([`AdminController.cs:790-812`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). Menus are authorization surfaces too; "hidden by CSS" is not authz.

## 5. N+1 by loading documents to count

```csharp
// Illustrative fake code: not from this repo — BAD
var items = await _session.Query<ContentItem, PublishLaterPartIndex>().ListAsync();
var due = items.Where(i => i.As<PublishLaterPart>().ScheduledPublishUtc < now);
```
Better: query the **index only** with the predicate in SQL — [`ScheduledPublishingBackgroundTask.cs:29-32`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) (`QueryIndex<PublishLaterPartIndex>(index => ... && index.ScheduledPublishDateTimeUtc < now)`). Loading documents to filter in memory is the document-store N+1.

## 6. Side effect before the write commits

```csharp
// Illustrative fake code: not from this repo — BAD
public override async Task PublishedAsync(PublishContentContext ctx, AutoroutePart part) {
    await _routeTable.RebuildNowAsync();   // reads DB — the publish isn't committed yet!
}
```
Better: defer to post-commit — the real handler schedules entry updates "after the session is committed" ([`AutoroutePartHandler.cs:74-76`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) via the deferred mechanism ([`ShellScope.cs:468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)). Rebuilding from uncommitted state produces a route table that's wrong either way the transaction goes.

## 7. Swallowed exception in a loop

```csharp
// Illustrative fake code: not from this repo — BAD
foreach (var item in itemsToPublish) {
    try { await contentManager.PublishAsync(item); } catch { /* keep going */ }
}
```
Better: catch with filter, log with identifiers, record for handlers — the scheduler's model: `catch (Exception ex) when (!ex.IsFatal()) { _logger.LogError(ex, "... '{TaskName}' on tenant '{TenantName}'", ...); context.Exception = ex; ... }` ([`ModularBackgroundService.cs:222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). Bare `catch {}` erases the failure *and* fatal exceptions.

## 8. Cache key missing a context

```csharp
// Illustrative fake code: not from this repo — BAD
var html = await cache.GetOrSetAsync($"widget:{widgetId}", RenderAsync); // same HTML for every user/culture
```
Better: key = id + hash of declared contexts (role, culture, route…) — [`DefaultDynamicCacheService.cs:137-150`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs). Under-keyed caches are cross-user data leaks, not just staleness bugs.

## 9. `any`-equivalent: dynamic where typed is available

```csharp
// Illustrative fake code: not from this repo — BAD
dynamic part = contentItem.Content.TitlePart;
string title = part.Title;   // typo'd property = runtime null, no compiler help
```
Better: `contentItem.TryGet<TitlePart>(out var part)` / `Get<TitlePart>()` then typed access — the style used everywhere ([`AutoroutePartHandler.cs:294`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)). Reserve dynamic/JSON access for genuinely unknown schemas.

## 10. Casually changing a public contract

```csharp
// Illustrative fake code: not from this repo — BAD (in an *.Abstractions project)
- Task<bool> PublishAsync(ContentItem contentItem);
+ Task<PublishResult> PublishAsync(ContentItem contentItem, PublishOptions options);
```
Better: additive overload + `[Obsolete]` on the old member, deleted a major version later — the repo keeps even awkward members alive under pragmas ([`DefaultContentManager.cs:461-466`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)). An Abstractions signature is a promise to every site that compiled against it.

---

Self-grade: for each pair, can you name (a) the smell in one sentence, (b) the failure it causes in production terms, (c) the real file that does it right? 8+/10 = ready for the review katas.
