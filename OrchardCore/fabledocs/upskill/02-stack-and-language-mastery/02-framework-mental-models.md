# ASP.NET Core Mental Models (as this repo stretches them)

## 1. DI lifetimes are the framework's core design axis

Three lifetimes: **singleton** (per container), **scoped** (per request/scope), **transient** (per resolve). Orchard multiplies this: there are *many containers* (one per tenant), so "singleton" means **per tenant** for anything registered in a module `Startup`, and truly global only for host-level registrations.

Real examples:
- Per-tenant singleton: the YesSql `IStore` — [`OrchardCoreBuilderExtensions.cs:56-118`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) (`AddSingleton` inside tenant container = one store per tenant).
- Scoped: `ISession` — one unit of work per request (`AddScoped`, `:137-173`).
- Host singleton: `ShellHost`, `ModularBackgroundService` (a `BackgroundService` hosted in the root container — [`ModularBackgroundService.cs:17`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)).

Captive-dependency rule: a singleton must not hold a scoped service. Watch how the repo threads this needle: `ContentTypeAuthorizationHandler` (registered in DI, potentially long-lived) lazily resolves `IAuthorizationService` from `IServiceProvider` to dodge a circular/captive graph — [`ContentTypeAuthorizationHandler.cs:69-70`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs), and `AutoroutePartHandler` does the same for `IContentManager` ([`AutoroutePartHandler.cs:39, 69`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)) with a comment-worthy reason: circular dependency between handler and manager.

## 2. Middleware = onion; Orchard adds a second onion per tenant

Host pipeline (Program.cs) → `ModularTenantContainerMiddleware` (picks tenant, swaps `RequestServices`) → `ModularTenantRouterMiddleware` (runs the *tenant's own* endpoint pipeline). Modules can contribute middleware/endpoints via their `Startup.Configure`. So "where do I add middleware?" always has a second question: *host level or tenant level?*

## 3. MVC model binding, but decentralized

Standard MVC: one action, one view model, `[FromBody]`/`[FromForm]`. Orchard's admin instead binds through **display management**: `UpdateEditorAsync` walks all drivers, each binding its own prefix-scoped slice — [`TitlePartDisplayDriver.cs:49-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) (`TryUpdateModelAsync(model, Prefix, t => t.Title)`).

Two MVC features you may not have met:
- **`[ActionName]` + `[FormValueRequired]`**: four POST handlers share the action name `Edit`, dispatched by which submit button posted — [`AdminController.cs:450-487`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs). `FormValueRequired` is a custom action-selection constraint in `OrchardCore.Routing`.
- **`[FromServices]` in actions** for per-action dependencies ([`AdminController.cs:67-75`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) — keeps constructor lean when only one action needs a service.

Minimal APIs coexist: `MapPost("api/content", ...)` with `[Authorize(AuthenticationSchemes = "Api")]` on the handler — [`CreateEndpoint.cs:24-33`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs). Note `.AllowAnonymous().DisableAntiforgery()` on the route *plus* attribute auth on the delegate: anonymous at routing level, challenged inside — an idiom worth pausing on (and questioning) in review.

## 4. Options pattern

Config binds to typed options: `services.Configure<YesSqlOptions>(configuration.GetSection("OrchardCore_YesSql"))` — [`OrchardCoreBuilderExtensions.cs:41`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs); consumed as `IOptions<T>` ([`AccountController.cs:50, 62`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)). Site-level *mutable* settings use a different mechanism (`ISiteService`, stored as a document) — config = deploy-time, site settings = runtime-editable. Knowing which bucket a setting belongs in is a real design decision.

## 5. Hosted services and the "no request" world

`BackgroundService.ExecuteAsync` runs for process lifetime. Orchard fabricates an `HttpContext` for background flows so components relying on `IHttpContextAccessor` still work — [`ModularBackgroundService.cs:108-109, 446-467`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs). That's a smell *and* a pragmatic bridge: lots of ecosystem code assumes a request. Know both sides for interviews.

## 6. AuthN vs AuthZ plumbing

- Authentication: ASP.NET Identity (`SignInManager`, `UserManager`) — [`AccountController.cs:27-28`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs).
- Authorization: policy/requirement handlers. Orchard's `PermissionRequirement` + handlers evaluate named permissions with implication chains; resource-based checks pass the content item as resource ([`AdminController.cs:850-851`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) → [`ContentTypeAuthorizationHandler.cs:20-76`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)).
- Handlers **succeed or abstain, never fail-close individually** (`context.HasSucceeded` early-return at `:22-26`) — aggregation semantics worth quoting in authz design questions.

## Pitfall checklist

- [ ] Right lifetime? (would this hold per-request state in a singleton?)
- [ ] Right container? (host vs tenant)
- [ ] Service resolved inside the scope that will use it (esp. background code)
- [ ] Form POST handlers: does the right `[FormValueRequired]` guard exist?
- [ ] New endpoint: which auth scheme, which permission, is antiforgery intentionally on/off?

## Drill

`services.AddScoped(sp => store.CreateSession())` returns `null` when `IStore` is null (pre-setup) — [`OrchardCoreBuilderExtensions.cs:137-144`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs). Predict: what happens if a controller injects `ISession` on an uninitialized tenant? Then explain why registering a null is preferable here to throwing at registration. Self-grade — Strong: connects it to the setup flow (services must be *constructible* before the DB exists).

## Interview angle

1. "Explain DI lifetimes and a bug from getting them wrong" → §1 captive dependency, → [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q8.
2. "Middleware order — why does it matter?" → §2 two-onion story.
3. "Resource-based authorization in ASP.NET Core" → §6, → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q4.
4. "How would you run per-tenant background jobs?" → §5 + Flow 5.
