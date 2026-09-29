# Boundaries and Layers

A **boundary** is a line where the rules change: different owner, different lifetime, different failure semantics. A layer is only real if crossing it costs something (an interface, a serialization, a permission check). Orchard has unusually honest boundaries — and a couple of instructive leaks.

## The layer stack

| Layer | Owns | Must NOT own | Example |
| --- | --- | --- | --- |
| **Host** (`OrchardCore.Cms.Web`) | process config, logging sink, top middleware | any business logic | [`Program.cs`](../../../src/OrchardCore.Cms.Web/Program.cs) |
| **Kernel** (`src/OrchardCore/OrchardCore*`) | tenants, scopes, extensions, background scheduling, display engine, data wiring | knowledge of any specific feature | [`ShellHost.cs`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) |
| **Domain services** (`OrchardCore.ContentManagement`, etc.) | content invariants, versioning, events | HTTP, HTML, permissions* | [`DefaultContentManager.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) |
| **Modules** (`src/OrchardCore.Modules/*`) | HTTP endpoints, UI drivers, permissions, migrations for their feature | other modules' internals | [`OrchardCore.Contents/`](../../../src/OrchardCore.Modules/OrchardCore.Contents) |
| **Themes** | templates & assets | logic beyond display | `src/OrchardCore.Themes/` |

\*One asterisk expanded below.

## Boundary crossings done well

**1. Controller → domain: authorization stays on the web side.**
`AdminController.Publish` checks `PublishContent` *before* calling `_contentManager.PublishAsync` ([`AdminController.cs:616-631`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). `DefaultContentManager` contains zero permission checks — its contract is "caller has decided you may do this." That's a clean separation *and* a sharp knife: every new caller must remember the check (see the API asymmetry in [06-architecture-critique.md](06-architecture-critique.md)).

**2. Domain → persistence: invariants above, storage below.**
`PublishAsync` owns the Latest/Published invariant; YesSql owns durability. The domain never writes SQL; it manipulates documents + index-queries ([`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).

**3. Module → module: events, not references.**
Autoroute reacts to publishes via `ContentPartHandler<AutoroutePart>` ([`AutoroutePartHandler.cs:63-89`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) rather than Contents calling Autoroute. New behaviors attach without touching the publisher — the open/closed principle expressed as DI-resolved handler collections.

**4. Kernel → modules: manifests as the only coupling.**
The kernel loads features declared in `Manifest.cs` assembly attributes ([`Contents/Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs)); it never references module types.

## Boundary leaks worth studying (how leaks actually look)

**Leak A — domain service reaching into web-layer model state.**
`DefaultContentManager` (domain) injects `IUpdateModelAccessor` and writes MVC `ModelState` errors when a handler cancels a publish ([`DefaultContentManager.cs:398-414`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)). Convenient — cancellation messages surface in forms automatically — but now the *content manager* knows about MVC model binding, and non-MVC callers (background tasks, GraphQL) carry a null `ModelUpdater` path. This is what "pragmatic leak" looks like: intentional, guarded (`is not null`), and still a coupling you must know about before reusing the service.

**Leak B — UI strings decided in the controller.**
Success notifications ("Your {0} has been published.") are baked into `AdminController` per action ([`AdminController.cs:893-905`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) — repeated across six actions. Not a correctness problem; a duplication/ownership smell (the notifier message belongs with the operation outcome). Good first-refactor material — and a good "what would you clean up?" interview answer because it's *low blast radius*.

**Non-leak that looks like one:** the fake `HttpContext` in background service ([`ModularBackgroundService.cs:108-109`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). It *looks* like web bleeding into background; it's actually a compatibility bridge with a documented purpose (URL generation via site BaseUrl). Distinguish "leak" from "adapter" by asking: is the dependency direction acknowledged and contained?

## The vocabulary, defined by this code

- **Invariant**: "≤1 Published, exactly-1-or-0 Latest version per item" — maintained at [`DefaultContentManager.cs:416-423`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs).
- **Contract**: `IContentManager` + the handler context types — what modules may rely on across versions.
- **Ownership**: Autoroute owns path uniqueness; Contents owns publish workflow; the kernel owns tenant lifecycle. When a bug spans two owners (slug collides on publish), you now know which repo area each half belongs to.
- **Blast radius**: changing `IShellHost` = every Orchard site; changing `TitlePartViewModel` = one module's views.

## Drill

Take Flow 2 (edit & publish) and mark every boundary crossing with what *changes* at the line: identity (user → system), failure mode (ModelState → exception → cancelled session), data shape (form → shape tree → JSON doc → SQL rows). Self-grade — Strong: you can say for each crossing *who is responsible for rejecting bad input there*.

Interview angle: "Describe a well-layered system and where layering broke down" — Leak A is a precise, honest example that shows judgment rather than dogma. Cross-link: [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).
