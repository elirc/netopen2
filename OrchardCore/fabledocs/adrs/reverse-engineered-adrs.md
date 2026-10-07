# Orchard Core — Reverse-Engineered ADRs

Ten Architecture Decision Records inferred from the code of Orchard Core v3.0.1 (git `29649872e`). These were never written by the authors as ADRs; they are reconstructed from the shapes the code actually takes. Each is grounded in specific evidence. Status reflects what the v3.0.1 tree shows, not official project governance.

Anchors are `path:line` relative to the repo root and were taken from live reads of this working copy; line numbers drift as the repo moves.

---

## ADR-001: One process, one container per tenant (isolation as object-graph shape)

**Status:** Accepted — the defining architectural decision of the framework.

**Context.** Orchard Core is a multi-tenant CMS: many independent sites in one deployment. Tenants differ not only in *data* but in *enabled features* — tenant A may run e-commerce modules that tenant B never loads. A shared code path with a `tenant_id` column cannot express "this tenant has different services registered."

**Decision.** Give every tenant ("shell") its own dependency-injection container and request pipeline, built lazily on first use. A middleware resolves the tenant from host/URL-prefix and swaps `HttpContext.RequestServices` to that tenant's container for the rest of the request.

**Evidence.**
- Tenant resolution + container swap: `src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs:27-69` (`_runningShellTable.Match(httpContext)` → `GetScopeAsync` → `shellScope.UsingAsync(...)`).
- Per-tenant lazy container creation, one semaphore per tenant name with double-check + retry: `src/OrchardCore/OrchardCore/Shell/ShellHost.cs:80-119`.
- Placeholder shells so startup is O(#tenants)-cheap yet the running-shell table is complete: `ShellHost.cs:335-374`.
- Per-tenant store (own DB or shared DB with `TablePrefix`): `src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs:56-118`.

**Alternatives considered.**
- *Row-level tenancy* (single container, `tenant_id` filter): denser, simpler, but can't vary features per tenant and makes cross-tenant leakage a query-discipline problem forever.
- *Process/container per tenant* (OS-level): stronger isolation, far worse density and cold-start cost.

**Consequences.**
- Positive: isolation is structural — you cannot inject another tenant's services, so cross-tenant IDOR mostly can't happen through the ORM. Feature variance is just different registrations. Tenants reload independently (`ShellHost.ChangedAsync` releases one shell without touching others).
- Negative: memory per tenant; first request per tenant pays container + pipeline build; the lifecycle machinery (ref-counted `ShellContext`, reload retry loops, placeholders) is genuinely complex and is where new-contributor bugs cluster.

---

## ADR-002: Ship one binary as toggleable modules, coupled only through `*.Abstractions`

**Status:** Accepted.

**Context.** A CMS needs ~100 optional capabilities (blogging, media, search, workflows). Compiling them into a monolith prevents per-tenant selection; splitting them into independently versioned services adds distributed overhead nobody wants for an in-process CMS.

**Decision.** Structure the system as **modules** (`src/OrchardCore.Modules/*`) over a **framework** (`src/OrchardCore/*`), where each subsystem ships an `*.Abstractions` (and often `*.Core`) project holding interfaces/contracts, separate from its implementation. Modules declare **features** and dependencies as assembly attributes; enabling a feature registers its services in the tenant container. Modules never reference each other's implementation projects — only abstractions, resolved at runtime through DI.

**Evidence.**
- Feature/dependency declaration: `src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs` (six features in one assembly, each with `Dependencies`).
- The abstractions-only coupling stated in the code itself: `src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs:5-8` — permissions live in `.Core` "so they can be used in other modules without having to reference OrchardCore.Contents by itself."
- Host is nothing but composition: `src/OrchardCore.Cms.Web/Program.cs:1-22` (`AddOrchardCms().AddSetupFeatures(...)`, `app.UseOrchardCore()`).
- Central package management removes version drift across ~200 projects: `Directory.Packages.props` + cascading `Directory.Build.props`.

**Alternatives considered.** A plugin DLL model with reflection-only contracts (looser, less type-safe); microservices per capability (operationally heavy for a CMS); a single project with feature flags as `if` statements (disabled code still loaded and coupled).

**Consequences.**
- Positive: features are composition-level, not `if`-statements — disabled code is *not registered*, not merely skipped. New capability = new module, zero core edits (open/closed at the assembly level).
- Negative: dependency errors surface at runtime, not compile time; the feature matrix explodes testing combinations; newcomers must learn that "where do I add this?" always has a second question — abstractions or implementation, host-level or tenant-level.

---

## ADR-003: Persist domain data as JSON documents with projected relational index tables (YesSql)

**Status:** Accepted.

**Context.** Content types are defined by users at runtime, so the storage schema is not known at compile time. But the product still needs efficient relational queries ("all published BlogPosts, newest first") across four SQL engines (SQL Server, SQLite, MySQL, Postgres).

**Decision.** Use **YesSql**: store each entity as a serialized JSON document, and maintain **index tables** — ordinary SQL tables projected from documents by `IndexProvider`s — for querying. Queries target index columns; matching documents are rehydrated from JSON. Documents and their index rows are written in the same transaction.

**Evidence.**
- Index projection: `src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs:28-73` (`Describe(...).Map(contentItem => new ContentItemIndex {...})`).
- Two-type query binding document to index: `src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs:113-118` (`Query<ContentItem, ContentItemIndex>().Where(x => x.ContentItemId == id && x.Latest)`).
- Index-only scans for cheap existence/due checks: `src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs:29-32` (`QueryIndex<PublishLaterPartIndex>(...)`).
- Provider-specific store construction: `src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs:70-118`.

**Alternatives considered.** A document database (Mongo) — loses the four-SQL-backend requirement and SQL-shop operability; pure relational with EAV — notoriously slow and query-hostile; JSONB + GIN on Postgres only — would give constraint-backed invariants but abandons portability.

**Consequences.**
- Positive: schema-flexible writes with relational read performance; index and document commit atomically so an index can't reference a missing document; the same query surface works on all four providers.
- Negative: index *content* can be lossy — values are truncated (255 chars) on projection (`ContentItemIndex.cs:50-68`), so filtering/sorting can diverge from document truth; only indexed fields are queryable; adding an index to a hot type incurs write amplification (every save re-maps every registered index).

---

## ADR-004: A request is one unit of work that commits at scope disposal; rollback is an explicit cancel

**Status:** Accepted.

**Context.** Many services mutate state during a single request (controller, handlers, drivers). Requiring each to call `SaveChanges` invites "saved half the aggregate" bugs and scatters transaction control.

**Decision.** Register the YesSql `ISession` as scoped, and register a **before-dispose callback that commits** it (and an exception handler that cancels it) — but only for the shell scope's own session. A request is therefore transactional by default; opting out means explicitly calling `session.CancelAsync()`.

**Evidence.**
- Auto-commit / auto-cancel wiring: `src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs:152-170` (`RegisterBeforeDispose(... CommitAsync())` + `AddExceptionHandler(... CancelAsync())`), guarded to only the scope's own provider (`sp == shellScope?.ServiceProvider`).
- Explicit rollback on validation failure: `src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs:764-768` (`if (!ModelState.IsValid ...) { await _session.CancelAsync(); return View(...); }`); same idiom in bulk actions (`:283-303`) and the API (`CreateEndpoint.cs:83-111`).
- Optimistic concurrency opt-in per save: `DefaultContentManager.cs:423` (`SaveAsync(contentItem, checkConcurrency: true)`).

**Alternatives considered.** Explicit `SaveChanges` per repository call (forgettable, non-atomic); an ambient `TransactionScope` (heavier, provider-variable semantics); a separate command bus with per-command transactions (over-engineered for CRUD-shaped work).

**Consequences.**
- Positive: whole-request atomicity for free; forgetting to commit is impossible; concurrency conflicts surface as exceptions rather than lost updates.
- Negative: forgetting to *cancel* on a failure path silently persists partial work — a real footgun; long-running requests hold a transaction open; anything that must happen *after* commit needs a separate mechanism (see ADR-010).

---

## ADR-005: Model content as composable parts welded onto documents, with types defined as data and versions as flagged rows

**Status:** Accepted.

**Context.** Editors define content types at runtime; the same logical item needs drafts, a published version, and history; behavior (routing, titles) must be reusable across types.

**Decision.** A `ContentItem` is a JSON document; reusable **parts** are welded on and projected out on demand via `Get<T>()`/`Apply()`. Content **types are data** (compositions of parts/fields stored via `IContentDefinitionManager`), not C# classes. **Versioning** stores multiple rows per `ContentItemId` with boolean `Latest`/`Published` flags; the invariant "≤1 Published and ≤1 Latest per item" is maintained **procedurally** inside `IContentManager`, not by a DB constraint.

**Evidence.**
- Weld/project identity subtlety proven by test: `test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs:9-38` (getting a part as base then concrete type returns *different instances*).
- Types as data: `src/OrchardCore.Modules/OrchardCore.Contents/Migrations.cs:16-23` (`AlterPartDefinitionAsync("CommonPart", ...)`).
- Publish transition maintaining the invariant in one transaction: `DefaultContentManager.cs:376-428` (load previous published row, demote `Published=false`, promote current, save both with `checkConcurrency:true`).
- Draft-on-read: `GetAsync(..., DraftRequired)` creates a new version row (`DefaultContentManager.cs:159-183`), except for non-versionable types which mutate in place (`:168-171`).

**Alternatives considered.** A class-per-content-type ORM model (no runtime type creation); a separate `drafts` table instead of flags (two code paths, no uniform query); DB-enforced single-published constraint (portability cost across four engines, and flags make partial-unique indexes provider-dependent).

**Consequences.**
- Positive: uniform queries for "live site" (`Published`) and "admin" (`Latest`); history is free; new behaviors attach as parts without schema changes.
- Negative: the flag invariant holds only as strongly as the code that maintains it — any writer bypassing `IContentManager` can create two published rows; two `Get<T>()` calls aren't reference-equal, so change detection by identity is wrong; unbounded version growth needs a pruning feature.

---

## ADR-006: Extend behavior through lifecycle event handlers and display drivers, not inheritance

**Status:** Accepted.

**Context.** Many modules must react to the same domain events (an item is published) and contribute to the same UI (an edit form), without the core knowing they exist.

**Decision.** Two symmetric extension contracts. (1) **Lifecycle handlers**: `IContentHandler`/`ContentPartHandler<TPart>` are invoked in registration order for `*ing` (pre) events and in **reversed** order for `*ed` (post) events — onion semantics. Handlers carry business side effects and can veto operations via `context.Cancel`. (2) **Display drivers**: `IDisplayDriver` maps models to *shapes* (dynamic view fragments) with declared placement, and binds form posts back; themes override by supplying a template for a shape alternate.

**Evidence.**
- Forward/reverse invocation: `DefaultContentManager.cs:85-96` (`Handlers.InvokeAsync(...)` then `ReversedHandlers.InvokeAsync(...)`); reversed collection at `:60-65`.
- Veto: `DefaultContentManager.cs:396-414` (`PublishingAsync` handlers set `context.Cancel`, surfaced as a ModelState error).
- Generic-constrained handler routing: `src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs:26,63-89` (`ContentPartHandler<AutoroutePart>` fires only for items containing that part).
- Driver + placement: `src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs:20-64` (`.Location("Detail","Header")`); `src/OrchardCore.Modules/OrchardCore.Contents/placement.json`.

**Alternatives considered.** A base-class hierarchy (single inheritance can't compose orthogonal behaviors); a message bus (isolation/retry benefits, but async complexity and eventual consistency the CMS doesn't want inline); MVC controllers owning whole view models (defeats per-module UI contribution).

**Consequences.**
- Positive: open for extension with zero core edits; reversal gives clean pre/post nesting; drivers let themes restyle without forking code.
- Negative: hidden ordering dependencies between handlers; a slow handler taxes every operation; the shape/placement contract is stringly-typed, so renames break silently at render time.

---

## ADR-007: Publish stable interface contracts; let HTTP/GraphQL be thin adapters over the domain

**Status:** Accepted.

**Context.** Orchard ships as NuGet packages that thousands of downstream sites compile against, and exposes HTTP + GraphQL APIs. Both are contracts with a large, invisible blast radius.

**Decision.** Treat `*.Abstractions` interfaces as the **published contract**: evolve them additively, deprecate with `[Obsolete]` rather than delete. Keep HTTP endpoints as thin minimal-API adapters that authorize, bind, delegate to `IContentManager`, and translate results — including RFC-style validation payloads. GraphQL compiles a whitelisted `where` grammar to parameterized SQL over index tables with a default result cap.

**Evidence.**
- Deprecate-don't-delete on the domain contract: `DefaultContentManager.cs:461-466` (`#pragma warning disable CS0618` keeping an obsolete member alive).
- Thin HTTP adapter with explicit auth scheme and error translation: `src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs:24-48` (`[Authorize(AuthenticationSchemes="Api")]`, `ChallengeOrForbid("Api")`) and `:88-93` (`TypedResults.ValidationProblem(...)`); merge semantics fixed as arrays-replace at `:19-22`.
- Route declares `AllowAnonymous().DisableAntiforgery()` while the handler enforces token auth — antiforgery is off *because* cookies aren't the auth mechanism: `CreateEndpoint.cs:26-33`, `DeleteEndpoint.cs:14-46`.
- GraphQL parameterized SQL from a typed where-input: `src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs:190-198` (`query.WithParameter(...)`), default cap at `:203-222`.

**Alternatives considered.** Versioned API interfaces (`IContentManagerV2`) — churn; fat controllers owning business logic — duplicates rules across surfaces; ORM-direct GraphQL resolvers — SQL-injection and unbounded-result risk.

**Consequences.**
- Positive: downstream sites keep compiling across upgrades; the same domain rules back every surface; GraphQL input is injection-resistant by construction and bounded by default.
- Negative: obsolete members accumulate; **contract enforcement is per-endpoint, so it can drift** — the API's create/update path authorizes `EditContent` yet can call `PublishAsync`, while the admin UI demands `PublishContent` (`CreateEndpoint.cs:95-127` vs `AdminController.cs:876-879`) — an authorization asymmetry to investigate.

---

## ADR-008: Exceptions are the failure channel; contain them at unit-of-work boundaries, never mid-flow

**Status:** Accepted.

**Context.** A modular system runs untrusted-ish extension code (handlers, recipe steps, background tasks) inside shared execution. One failing extension must not corrupt a transaction or take down a tenant, but failures must also not vanish.

**Decision.** Use exceptions as the dominant error style, and catch **only at unit-of-work boundaries** — per request, per background task, per recipe step — with `when (!ex.IsFatal())` filters that let truly fatal exceptions escape. At those boundaries: log with tenant/task context, cancel or record the unit of work, and continue or rethrow. Use `context.Cancel` + ModelState for *expected* domain rejections rather than exceptions.

**Evidence.**
- Boundary catch with non-fatal filter, log-and-continue: `src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs:89-92` and per-task `:222-230` (`_logger.LogError(ex, "... '{TaskName}' on tenant '{TenantName}'", ...); context.Exception = ex;`).
- Capture-and-rethrow-with-stack across an async boundary: `src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs:95-123` (`ExceptionDispatchInfo.Capture(...)` then `.Throw()`).
- Expected rejection as data, not exception: publish veto → ModelState (`DefaultContentManager.cs:398-414`); validation failure → cancel session (`AdminController.cs:764-768`).
- Scope-level exception handlers see request errors too: `ModularTenantContainerMiddleware.cs:62-66`.

**Alternatives considered.** Result/Either types throughout (verbose in a C# codebase built before that idiom was idiomatic; the repo uses results only for *validation*, e.g. `ContentValidateResult`); bare `catch {}` (erases fatal exceptions and hides failures); no boundary catch (one bad handler kills the tenant loop).

**Consequences.**
- Positive: one failing tenant/task/step doesn't sink the others; stack traces are preserved; expected vs exceptional failures are cleanly separated.
- Negative: log-and-continue means failures are **only** visible in logs — no retry, no dead-letter, no metric emission in the kernel; a task that fails every tick looks identical to a quiet one until someone reads the logs.

---

## ADR-009: Test correctness at three layers — pure units, in-process functional against all DBs, and injected seams for time/IO

**Status:** Accepted.

**Context.** Correctness spans pure logic (flag transitions, permission implication), integration behavior (does a publish actually demote the previous row in the database?), and cross-provider portability (migrations on four SQL engines).

**Decision.** Three layers. **Unit** (`test/OrchardCore.Tests`, xUnit v3) for logic provable without HTTP/DB. **Functional** (`test/OrchardCore.Tests.Functional`, Playwright) that boots the real CMS **in-process** with per-fixture data directory, per-fixture database, dynamic port, and embedded recipes — fully hermetic, parallel-safe. A **CI matrix** runs functional tests against SQLite/SQL Server/MySQL/Postgres. Testability is designed in via injected seams (`IClock`, `ISession`, `IContentManager`).

**Evidence.**
- Fixture isolation described by the suite itself: `test/OrchardCore.Tests.Functional/README.md` (isolated `App_Data_Tests_{Fixture}`, per-fixture DB drop/recreate, embedded-resource recipes, parallel fixtures).
- Injected clock making time-dependent logic testable: `ScheduledPublishingBackgroundTask.cs:21-25` (`IClock`), used in the due-predicate at `:29-32`.
- Serialization-round-trip unit style: `ContentElementTests.cs:9-38`.
- All-DB matrix + auto-migrations enforce migration portability rather than review vigilance: CI `functional_all_db.yml`; migrations note that drop/alter "don't work on all providers" — `PublishLater/Migrations.cs:44-48`.
- Runner config: `global.json` (`"test": { "runner": "Microsoft.Testing.Platform" }`).

**Alternatives considered.** Heavy mocking of `ISession` (tests your mocks, not YesSql semantics); a single external test DB (shared mutable state, flaky parallelism); testing migrations only on SQLite (misses provider-specific breakage the matrix catches).

**Consequences.**
- Positive: portability regressions caught mechanically; hermetic fixtures give safe parallelism; the session-boundary and stringly-typed-rendering rules tell contributors which layer a test belongs in.
- Negative: the all-DB functional matrix is slow and infrastructure-heavy; rendering/shape correctness can only be caught at the functional layer (no compile-time check), so template regressions are expensive to guard.

---

## ADR-010: Handle farm-scale concurrency with distributed locks, optimistic concurrency, at-most-once deferred effects, and fail-open caching

**Status:** Accepted, with known gaps.

**Context.** Orchard runs on multi-instance web farms sharing a database and (optionally) a distributed cache. Scheduled work must run once across the farm; concurrent edits must not lose data; side effects must respect transaction outcome; a dead cache must not take rendering down.

**Decision.** Compose four primitives. **Distributed locks** gate background tasks and shell activation so exactly one instance acts. **Optimistic concurrency** (version check at flush) guards content writes. **Deferred post-commit tasks** run side effects *after* the transaction commits, in a fresh scope — accepting **at-most-once** delivery. A **failover circuit breaker** disables the distributed cache for a cool-down window on I/O failure (fail-open).

**Evidence.**
- Distributed lock around background tasks: `ModularBackgroundService.cs:133-157` (`TryAcquireBackgroundTaskLockAsync`); around tenant activation: `src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs:314-329`.
- Idempotent cron work via predicate (`Latest && !Published && due`): `ScheduledPublishingBackgroundTask.cs:29-32`; guard-idempotent publish: `DefaultContentManager.cs:380-383`.
- Deferred post-commit effect, failures logged and dropped: `ShellScope.cs:387-392,468-508`; consumer refreshes route entries "after the session is committed": `AutoroutePartHandler.cs:74-76`.
- Cache failover flag with retry latency: `src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs:93-135`.
- Optimistic concurrency: `DefaultContentManager.cs:418-423`.

**Alternatives considered.** A transactional **outbox** for at-least-once effects (more infrastructure; Orchard bets that its effects are idempotent recomputations — a dropped route refresh heals on the next publish); pessimistic DB locks for uniqueness (chosen *against* — slug uniqueness is an app-level check-then-write, `AutoroutePartHandler.cs:465-477`, with a TOCTOU window); fail-closed caching (would let a cache outage break pages).

**Consequences.**
- Positive: correct single-execution across a farm; no lost updates on version rows; side effects never fire for a rolled-back transaction; a Redis outage degrades to uncached rendering, not errors.
- Negative: at-most-once means a crash between commit and deferred task loses the effect — fine for self-healing recomputations, wrong for emails/webhooks (no outbox exists yet); app-level uniqueness (slugs) can race under concurrency; deferred/background failures are invisible beyond logs (ties back to ADR-008).

---

# Interview translation

For each ADR: a ~90-second spoken answer (how *you* would describe the decision and its tradeoffs), then one follow-up an interviewer is likely to ask. Deliver each as **example → tradeoff → failure mode**, not a definition.

### ADR-001 — Container-per-tenant
> "Orchard Core is multi-tenant, and the interesting decision is that isolation is structural, not a query filter. Each tenant gets its own DI container and request pipeline. A middleware matches the incoming host or URL prefix to a tenant and swaps `RequestServices` to that tenant's container, so everything downstream — the session, the content manager — is that tenant's. The payoff is that you literally cannot inject another tenant's services, so cross-tenant data leakage mostly can't happen through the ORM, and tenants can run different feature sets, not just different rows. The cost is real: memory per tenant and a cold start on the first request while the container builds. They soften the cold start with placeholder shells so startup stays cheap and the routing table is still complete. I'd contrast it with row-level tenancy — a `tenant_id` column — which is denser and simpler but can't vary features and turns isolation into a discipline you can forget on every query."

Follow-up: *"How would you handle 10,000 tenants with this model?"* (Watch for: tenant eviction of idle shells, the single `tenants.json`, background polling per tenant.)

### ADR-002 — Modular features over `*.Abstractions`
> "It ships as one binary made of ~100 modules over a framework core, but the coupling rule is what matters: modules depend only on `*.Abstractions` projects — interfaces — never on each other's implementations, and they integrate through DI at runtime. Features and their dependencies are declared as assembly attributes, and enabling a feature registers its services in the tenant container. So a disabled feature isn't dead code that's skipped — it's never registered. The upside is genuine open/closed at the assembly level: new capability, new module, no core changes. The downside is that dependency problems show up at runtime instead of compile time, and the feature-combination matrix is huge to test. The tell that it's deliberate is a comment in the code — permissions live in a `.Core` project specifically so other modules can reference them without pulling in the whole implementation."

Follow-up: *"What breaks at runtime that a monolith would catch at compile time, and how would you defend against it?"*

### ADR-003 — Documents + index tables (YesSql)
> "Content types are user-defined at runtime, so the schema isn't known at compile time — but they still need relational queries across four SQL engines. The answer is YesSql: store each entity as a JSON document, and project index tables — real SQL tables — from those documents for querying. Queries hit index columns; the document is rehydrated from JSON on match; and crucially the document and its index rows commit in the same transaction, so an index can never point at a missing document. The tradeoff is that the index content can be lossy — they truncate indexed strings to 255 characters — so a filter on a long title can diverge from the actual document, and only indexed fields are queryable. It's basically 'document store for write flexibility, relational projections for read performance,' and the failure mode to watch is treating the index as the source of truth."

Follow-up: *"If you were Postgres-only, what would you do differently?"* (JSONB + GIN, partial unique indexes for constraint-backed invariants.)

### ADR-004 — Request-scoped unit of work
> "The session is scoped per request, and they register a before-dispose callback that commits it and an exception handler that cancels it. So a request is transactional by default — everything it writes commits together at the end of the scope — and you never call SaveChanges yourself. The elegant part is that 'don't save' becomes an explicit action: on a validation failure the controller calls `session.CancelAsync()`. That kills the entire class of 'saved half the aggregate' bugs. The footgun is the mirror image — if you forget to cancel on a failure path, the partial work still commits at scope end. And because the commit is at the very end, anything that has to happen strictly after commit — cache invalidation, a webhook — needs a separate post-commit mechanism, which is its own design decision."

Follow-up: *"Show me a bug this pattern makes easy to introduce."* (Forgetting `CancelAsync`; long-held transactions.)

### ADR-005 — Composable, versioned content model
> "A content item is a JSON document with reusable 'parts' welded on, content types are defined as data rather than classes, and versioning is done with multiple rows per item carrying `Latest` and `Published` boolean flags. Publishing is a transaction that demotes the previously-published row and promotes the new one, with an optimistic-concurrency check. What I find instructive is that the invariant — at most one published, one latest per item — is enforced in code, not by a database constraint, which is a deliberate portability choice across four engines. The upside is uniform queries: the live site filters on Published, the admin on Latest, and history is free. The risk is that the invariant is only as strong as the code path that maintains it — anything writing around the content manager could create two published rows — and there's a subtle gotcha where getting the same part twice returns different object instances, so you can't detect changes by reference identity."

Follow-up: *"Would you add a DB constraint to enforce single-published, and what would it cost?"*

### ADR-006 — Handlers and drivers for extensibility
> "Extension happens two ways, both avoiding inheritance. Lifecycle handlers subscribe to content events — creating, published, removed — and they're invoked in registration order for the 'pre' events and reversed order for the 'post' events, which gives you clean onion nesting; handlers can also veto an operation. And display drivers map models to 'shapes' — dynamic view fragments — with declarative placement, so themes restyle by supplying a template rather than forking code. It's the open/closed principle expressed as DI-resolved collections: a new behavior attaches without touching the thing it reacts to. The cost is hidden coupling — handlers can have ordering dependencies on each other — and the shape/placement layer is stringly-typed, so renaming a shape breaks rendering silently at runtime instead of at compile time."

Follow-up: *"Why reverse the handler order for post-events?"* (First-in-last-out nesting; symmetry with middleware.)

### ADR-007 — Published contracts, thin API adapters
> "There are really three contracts here: the `.Abstractions` interfaces that downstream sites compile against, and the HTTP and GraphQL surfaces. For the interfaces they deprecate rather than delete — you'll find obsolete members kept alive under pragmas — because changing a shipped interface breaks every Orchard site. The HTTP endpoints are thin minimal-API adapters: authorize, bind, delegate to the content manager, translate the result, including RFC-style validation problem responses. GraphQL compiles a whitelisted filter grammar to parameterized SQL with a default result cap, so it's injection-resistant and bounded by default. The tradeoff — and this is a real finding — is that because authorization is enforced per endpoint, it can drift: the API's update path authorizes EditContent but can still publish, while the admin UI requires a separate PublishContent permission. Same operation, different guard on different surfaces. That's the classic argument for centralizing the check at the operation level."

Follow-up: *"How would you stop that authorization drift between surfaces?"* (One operation-level check both surfaces call.)

### ADR-008 — Exceptions contained at unit-of-work boundaries
> "The dominant error style is exceptions, but the discipline is where they're caught: only at unit-of-work boundaries — per request, per background task, per recipe step — using exception filters that let genuinely fatal exceptions escape. At those boundaries they log with tenant and task context and either cancel the unit of work or continue. Expected domain rejections, like a publish being vetoed, aren't exceptions at all — they're a cancel flag plus a model-state error. So exceptional and expected failures are cleanly separated. The upside is that one bad handler can't sink the whole tenant loop and stack traces survive. The honest downside is that background failures are log-and-continue — no retry, no dead-letter, no metric — so a task failing every minute looks exactly like a task doing nothing until someone reads the logs. That's the first thing I'd improve for operability."

Follow-up: *"How would you make those silent background failures observable and recoverable?"*

### ADR-009 — Three-layer testing with an all-DB matrix
> "Testing is layered by what each layer can honestly prove. Pure logic — flag transitions, permission implication — is unit-tested. Integration behavior is covered by functional tests that boot the real CMS in-process with Playwright, and the key design move is total fixture isolation: each fixture gets its own data directory, its own database it drops and recreates, a dynamic port, and recipes served from embedded resources, so fixtures run in parallel with no shared state. Then CI runs those functional tests against all four databases, which is how migration portability is actually enforced rather than trusted to reviewers. Testability is designed in — time and the session are injected, so you never reach for `DateTime.UtcNow`. The tradeoff is that the all-DB matrix is slow and infrastructure-heavy, and rendering correctness has no compile-time check, so template regressions can only be caught at the functional layer."

Follow-up: *"How do you decide whether a given behavior belongs in a unit or a functional test?"* (The session/commit boundary rule.)

### ADR-010 — Farm-scale concurrency primitives
> "On a multi-instance farm they compose four primitives. Distributed locks make scheduled tasks and tenant activation run exactly once across instances. Optimistic concurrency — a version check at flush — protects content writes from lost updates. Post-commit side effects run as deferred tasks in a fresh scope after the transaction commits, which is at-most-once delivery. And the distributed cache has a failover circuit breaker: on an I/O error it disables itself for a cool-down and renders uncached, so a dead Redis degrades latency instead of throwing. The interesting judgment call is at-most-once instead of a transactional outbox — they get away with it because their post-commit effects are idempotent recomputations, like rebuilding a route entry that heals on the next publish. That's fine for recomputations and wrong for emails or webhooks, where you'd need an outbox. The other soft spot is that slug uniqueness is an app-level check-then-write, so it has a TOCTOU race under concurrency with no DB constraint behind it."

Follow-up: *"Walk me through adding guaranteed-delivery webhooks on publish."* (Transactional outbox: write intent in the same transaction, drain with a locked background task, idempotent consumer.)

---

*Grounding and caveats: anchors were read from this working copy; no build or test was executed, so runtime-dependent claims (notably the ADR-007 authorization asymmetry and the ADR-010 slug race) are labeled as findings to verify, consistent with `../upskill/09-reference/risk-register.md`.*
