# Type System and Contracts

Orchard Core lives on an unusual tension: a **strongly-typed framework** hosting a **schema-less, JSON-backed content model**. Understanding how it keeps (and where it loses) type safety is the most transferable lesson in the repo.

## 1. The JSON-backed model: `ContentItem.Get<T>()`

A `ContentItem` is a JSON document. Parts are *projections*: `contentItem.Get<TitlePart>("TitlePart")` deserializes a JSON subtree into a typed class on demand; `part.Apply()` writes mutations back into the JSON.

The unit test that proves the subtlety: getting the same part as `ContentPart` (base) and then as `TitlePart` (concrete) returns **different instances** — [`test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs:9-38`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs). Consequence: mutate the instance you got, call `Apply()`, and never assume reference identity between two `Get` calls.

You see the write-back discipline in real flows: `part.Path = ...; part.Apply();` — [`AutoroutePartHandler.cs:154-156, 409-421`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs).

Transferable: this is the same "hydrated view over a document" pattern as Mongo ODMs or Prisma JSON columns. The failure mode is always identity/staleness confusion, and the mitigation is always an explicit apply/save step.

## 2. Generics as behavior routers

- `ContentPartHandler<TPart>` (e.g. `AutoroutePartHandler : ContentPartHandler<AutoroutePart>` — [`AutoroutePartHandler.cs:26`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) dispatches lifecycle events *only* for items containing that part — generic constraint as routing.
- `ContentPartDisplayDriver<TPart>` same idea for UI ([`TitlePartDisplayDriver.cs:11`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)).
- `IDisplayManager<T>` renders arbitrary models through the shape system, not just content — the login form uses `IDisplayManager<LoginForm>` ([`AccountController.cs:33, 93`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)).
- YesSql queries: `session.Query<ContentItem, ContentItemIndex>().Where(x => x.Published)` — two type parameters bind document type to index table; the lambda compiles to SQL against the index's columns ([`DefaultContentManager.cs:113-118`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).

## 3. Interfaces-as-contracts and the Abstractions split

Every subsystem ships an `*.Abstractions` project (interfaces, contexts, models) separate from implementation. That split *is* the compatibility contract: downstream modules reference abstractions; implementations can change. When you see `IShellHost` with 10+ members ([`src/OrchardCore/OrchardCore.Abstractions/Shell/IShellHost.cs`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/IShellHost.cs)), that's a *published* API — changing it is a breaking change for every Orchard site, which is why you'll find `[Obsolete]` members kept alive (e.g. the pragma-suppressed `PublishingItem` in [`DefaultContentManager.cs:461-466`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).

Transferable rule: **deprecate, don't delete, on public surfaces**; and keep your abstractions package free of implementation dependencies so consumers' graphs stay small.

## 4. Where typing is deliberately loose — and the guardrails

- Shapes are dynamic (`IShape` with `Properties` dictionary; `List<dynamic>` of summaries — [`AdminController.cs:200-204`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). The guardrail is convention: shape type names + alternates are a *stringly-typed contract* with templates. Renaming a shape is a breaking change you won't get a compile error for — grep templates before renaming.
- Settings blobs: `contentTypePartDefinition.GetSettings<AutoroutePartSettings>()` deserializes a JSON settings bag ([`AutoroutePartHandler.cs:165-167`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) — typo in a settings class property silently reads defaults.
- `ShellSettings["DatabaseProvider"]` string indexer over configuration ([`OrchardCoreBuilderExtensions.cs:60-70`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)) — with a `switch` + `throw` as the runtime validator (line 100-101).

Interview framing: "static typing where the compiler can help; conventions + tests where flexibility pays; and *know which regime each line is in*."

## 5. System.Text.Json specifics worth knowing

- `JsonMergeSettings` with `MergeArrayHandling.Replace` for content updates ([`DefaultContentManager.cs:23-26`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs), [`CreateEndpoint.cs:19-22`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)) — merge semantics for PATCH-like updates: arrays replace, objects merge. Wrong choice here = duplicated list entries; it's a classic API-design interview probe.
- `[JsonIgnore]` marking runtime-only members on `ShellSettings` ([`ShellSettings.cs:75-82, 114-117`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs)); enum-as-string with `JsonStringEnumConverter` (`:148-149`) so `tenants.json` stays human-readable.

## Pitfall checklist

- [ ] After mutating a part: did you `Apply()`?
- [ ] Two `Get<T>()` calls are not the same object — don't diff them for change detection
- [ ] Renamed a shape/template/placement key? grep `Views/` and `placement.json` across modules and themes
- [ ] Query lambda only touches index columns (anything else can't translate)
- [ ] Changing an Abstractions interface = ecosystem break; prefer additive + obsolete

## Drill

Write (on paper) the sequence for "rename `TitlePart.Title` to `Heading`" — every artifact that must change: C# model, driver binding expression, view model, `.cshtml` templates, existing JSON documents (migration or tolerant read?), and the index if any. Self-grade — Strong: notices stored JSON doesn't migrate itself and proposes a lazy-upgrade handler or a one-shot migration, with the tradeoff.

## Interview angle

1. "How do you store flexible/user-defined schemas in a typed language?" → §1/§4, → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q6.
2. "Generics beyond collections — a real use?" → §2 handler routing.
3. "PATCH semantics: merge vs replace arrays" → §5.
4. "How do you evolve a public API without breaking consumers?" → §3.
