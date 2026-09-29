# Key Flows

Eight end-to-end traces. Each is a complete story you could tell in an interview. Later modules reuse these flows — different ones each time, deliberately.

---

## Flow 1: HTTP request → tenant → response

Why this flow matters: it *is* Orchard's multi-tenancy. Every other flow runs inside it.

Open these files first:
- [`src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs:27-69`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs) — the whole flow in 40 lines
- [`src/OrchardCore/OrchardCore/Shell/ShellHost.cs:80-148`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) — context/scope creation

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Kernel middleware | `ModularTenantContainerMiddleware.cs:30` | `IShellHost.InitializeAsync()` — idempotent; first request pre-creates all shells as placeholders | — | First-request latency |
| 2 | RunningShellTable | `:32` | `Match(httpContext)` maps host+path-prefix → `ShellSettings` | `ShellSettings` | Unmatched host → request silently unhandled (no `_next`) |
| 3 | Middleware | `:37-43` | Initializing tenant → `503` + `Retry-After: 10` | — | Setup races surface as 503s, by design |
| 4 | ShellHost | `ShellHost.cs:80-119` | Get-or-create `ShellContext`: semaphore per tenant, double-check, retry if `Released` | per-tenant DI container | Container build cost paid by first request |
| 5 | Middleware | `:48-56` | `GetScopeAsync` → store `ShellContextFeature` (original path/base preserved) | `ShellScope` | — |
| 6 | ShellScope | `ShellScope.cs:247-284` | `UsingAsync`: set `AsyncLocal` current, activate tenant events once (with distributed lock, `:302-350`), run `_next` | — | Activation lock timeout → `TimeoutException` |
| 7 | Router middleware | `ModularTenantRouterMiddleware.cs` | Tenant's own endpoint pipeline routes to a module controller | — | — |
| 8 | Scope disposal | `ShellScope.cs:444-509` | before-dispose callbacks (→ session commit), deferred signals, deferred tasks in fresh scopes | — | Deferred-task failures are logged, not retried |

Validation and authorization: none here — this flow is *pre-auth*; it decides which tenant's auth system will even see the request.

Persistence and side effects: none directly; step 8 is where every write in the request actually hits the database.

Tests that cover it: `test/OrchardCore.Tests/Shell/RunningShellTableTests.cs`, `ShellHostTests.cs`, `ShellScopeTests.cs` (existence verified by listing; contents not read).

What juniors usually miss: `RequestServices` is replaced — the "global" container never handles tenant requests.

What seniors notice: the placeholder-shell trick ([`ShellHost.cs:335-374`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)) makes startup O(#tenants) cheap while keeping the running-shell table complete; and `GetOrCreateShellContextAsync`'s `while` loop is a lock-free answer to "context released between my check and my use."

Interview angle: "How would you build multi-tenancy?" — describe container-per-tenant vs row-level tenancy, and when each wins (isolation & feature variance vs density & simplicity).

Drill: set a breakpoint (or add temp logging) in `Match` and in the session-commit callback; hit one admin page; write down every file the request touched in order.
Self-grade — Basic: name middleware → controller. Solid: include scope activation and commit-at-disposal. Strong: explain what happens to in-flight requests when the tenant is reloaded mid-request (existing scopes keep the old context alive via ref-counting — [`ShellScope.cs:28-37, 542-578`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)).

---

## Flow 2: Edit & publish a content item (admin UI)

Why this flow matters: the canonical CRUD++ write path; every layer of the architecture participates.

Open these files first:
- [`src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs:739-788`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) — `EditInternalAsync`
- [`src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) — `PublishAsync`

Trace (POST `submit.Publish` on an existing item):

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | MVC dispatch | `AdminController.cs:473-479` | `[FormValueRequired("submit.Publish")]` selects `EditAndPublishPOST` among 4 handlers of action `Edit` | form post | Button-name/handler drift |
| 2 | Authz | `:867-879` | `PublishContent` on the *item* (resource-based) | `ContentItem` | Owner-variant logic in handler, Flow 4 of 03-03 |
| 3 | Load | `DefaultContentManager.cs:102-186` | `GetAsync(id, DraftRequired)`: latest version loaded; **if published, a new draft version row is created** (`BuildNewVersionAsync:497-542`) | 2 version rows | Versionable=false types mutate in place (`:168-171`) |
| 4 | Bind+validate | `ContentItemDisplayManager.cs:146-191` | `UpdateEditorAsync` → every part/field driver's `UpdateAsync` binds and adds `ModelState` errors (e.g. [`TitlePartDisplayDriver.cs:49-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)) | shape tree + mutated item | Validation is distributed across drivers |
| 5 | Update event | `AdminController.cs:757-762` | `UpdateAsync` fires Updating/Updated handlers (e.g. Autoroute regenerates slug) | — | Handlers can enqueue further writes |
| 6 | Cancel path | `:764-768` | Invalid `ModelState` → `_session.CancelAsync()` → **nothing** from steps 3–5 persists | — | Forgetting cancel = persisting invalid drafts |
| 7 | Publish | `DefaultContentManager.cs:376-428` | Load previous published row; handlers may `Cancel`; demote previous (`Published=false`), promote current, save both `checkConcurrency:true` | 2 rows updated | Concurrency exception on parallel edits |
| 8 | Fan-out | e.g. `AutoroutePartHandler.cs:63-89` | `PublishedAsync` handlers: route entries update, tag-cache eviction, homepage flag | — | Handler ordering (reversed on \*ed events) |
| 9 | Commit | `OrchardCoreBuilderExtensions.cs:152-170` | Scope disposal commits the session once | SQL txn | Post-commit work must be a deferred task |

Validation and authorization: authz at step 2 (+ per-type dynamic permission via [`ContentTypeAuthorizationHandler.cs:20-76`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)); validation at steps 4 and 7 (`PublishingAsync` handlers can veto — `context.Cancel`, `DefaultContentManager.cs:398-414`).

Persistence and side effects: all writes buffered in the YesSql session; visible to *this* request's queries, invisible to others until step 9.

Tests that cover it: unit tests around content elements exist ([`ContentElementTests.cs`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs)); full controller flow is exercised by Playwright functional tests (`test/OrchardCore.Tests.Functional/`, per its README). No direct unit test of `EditInternalAsync` was located — worth verifying before refactoring it.

What juniors usually miss: `GetAsync(..., DraftRequired)` *writes* (creates a version row) inside a "get."

What seniors notice: the Latest/Published pair is an **invariant maintained procedurally** — nothing in the schema enforces "one Published row per item"; a crash between demote and promote is only safe because both happen in one session/transaction.

Interview angle: "Design document versioning with drafts" — this flow is a full worked answer including the concurrency story (`checkConcurrency: true` → optimistic concurrency on the version row).

Drill: fill a trace table for the `submit.Save` (draft) variant instead. What steps disappear? (7–8; `SaveDraftAsync` at `DefaultContentManager.cs:358-374` fires DraftSaving/DraftSaved instead.)
Self-grade — Basic: correct handler chosen by `FormValueRequired`. Solid: cancel-path semantics. Strong: explains why publish saves the *previous* version first and what `checkConcurrency` protects against.

---

## Flow 3: Login (password sign-in)

Why this flow matters: a security boundary with three defenses juniors rarely name: lockout, rate limiting, timing normalization.

Open these files first:
- [`src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs:102-206`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Rate limit | `:105` | `[RateLimitGroup(PasswordAuthentication)]` throttles the endpoint | Fail-open if policy misconfigured — verify per deployment |
| 2 | Settings gate | `:108-115` | `DisableLocalLogin` site setting → redirect | — |
| 3 | Form bind | `:119-123` | `LoginForm` bound via display manager (even the login form is shapes/drivers); `ILoginFormEvent.LoggingInAsync` hooks can add model errors | Extensibility = order-dependent validation |
| 4 | User lookup | `:129-133` | `GetUserAsync` then `CheckPasswordSignInAsync(lockoutOnFailure: true)` | Lockout counts wrong passwords |
| 5 | Unknown user | `:182-188` | **Dummy hash verification** so response time matches a real check — anti user-enumeration | Remove it and timing leaks usernames |
| 6 | 2FA / lockout branches | `:162-180` | `RequiresTwoFactor` → redirect; `IsLockedOut` → error + event | — |
| 7 | Sign-in | `:147-157` | `PasswordSignInAsync` issues the auth cookie; `LoggedInAsync` events; redirect via `LoggedInActionResultAsync` | Open-redirect guarded upstream by local-URL checks |

What juniors usually miss: two password checks (`CheckPasswordSignInAsync` then `PasswordSignInAsync`) — the first validates without issuing a cookie so `ValidatingLoginAsync` events can still veto.

What seniors notice: every failure path fires a distinct event (`LoggingInFailedAsync` overloads for unknown-user vs known-user, `:193-202`) — that's audit/observability design, not ceremony.

Interview angle: "How do you prevent username enumeration?" — cite the timing-normalization branch as a concrete answer; most candidates only know "same error message."

Drill: list the three places a wrong-password attempt is recorded/limited (rate-limit group, identity lockout, `LoggingInFailedAsync` events).
Self-grade — Strong includes: why the dummy hash must cost the same as a real verification.

---

## Flow 4: URL → content item (Autoroute) and slug creation

Why this flow matters: bidirectional mapping between URLs and documents, with uniqueness as a contended invariant.

Open these files first:
- [`src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs:115-133, 378-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)

Trace (slug creation on save):

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Handler | `:135-145` | `CreatedAsync`/`UpdatedAsync` → generate path if empty | — |
| 2 | Liquid | `:386-409` | Render the type's pattern (e.g. `{{ ContentItem | display_text | slugify }}`) under the item's culture | Pattern is admin-editable code |
| 3 | Truncate | `:411-414` | Clamp to `MaxPathLength` | Truncation can collide |
| 4 | Uniqueness | `:465-477` | Query `AutoroutePartIndex` for path variants (`/x`, `x/`, …) | **Check-then-write race** — two concurrent saves can both pass (labeled *possible risk*, see risk register R2) |
| 5 | Suffix loop | `:437-463` | `-1`, `-2`, … until unique | Unbounded loop bounded in practice |
| 6 | Validation | `:115-133` | `ValidatingAsync` fails publish with "permalink already in use" | Same TOCTOU window |
| 7 | Publish fan-out | `:63-89` | `PublishedAsync`: refresh route entries **after commit** (`_entries.UpdateEntriesAsync`), evict `slug:{path}` tag from cache, set homepage if flagged | Route table staleness window |

What juniors usually miss: slugs are *data with an invariant* (global uniqueness per tenant), not cosmetic strings.

What seniors notice: uniqueness is enforced in application code against an index table — no DB unique constraint — so the invariant holds only as strongly as the check's isolation level. This is a textbook TOCTOU discussion.

Interview angle: "How do you guarantee unique slugs under concurrency?" — options ladder: unique constraint + retry on violation (strongest), serializable transaction, advisory lock, or accept-and-repair. Say which the repo chose and its tradeoff.

Drill: write the SQL unique constraint that would harden step 4, and describe what code change you'd need to translate the constraint violation into the friendly "permalink in use" error.

---

## Flow 5: Scheduled publishing (background/async)

Why this flow matters: the repo's full async story in one slice: cron scheduling, per-tenant scopes, distributed locking, idempotency.

Open these files first:
- [`src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) — the task (60 lines)
- [`src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs:98-237`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) — the scheduler

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Host service | `ModularBackgroundService.cs:78-96` | Poll loop over running shells | Polling latency (`PollingTime`) |
| 2 | Scheduler | `:111-124` | Per (tenant × task) `BackgroundTaskScheduler`; cron `Schedule = "* * * * *"` from the `[BackgroundTask]` attribute (`ScheduledPublishingBackgroundTask.cs:12-15`) | — |
| 3 | Lock | `:133-157` | `IDistributedLock.TryAcquireBackgroundTaskLockAsync` — one runner across a web farm | Lock timeout → skipped tick (logged Information) |
| 4 | Scope | `:159-231` | `shellScope.UsingAsync`: resolve `IBackgroundTask`, optionally build tenant pipeline, invoke `DoWorkAsync` | Fake HttpContext; URLs need site BaseUrl (`:169-181`) |
| 5 | Work | `ScheduledPublishingBackgroundTask.cs:29-57` | Query `PublishLaterPartIndex` for due items (`Latest && !Published && due`), clear `ScheduledPublishUtc`, `PublishAsync` each | Batch failure mid-loop: earlier publishes still commit at scope end |
| 6 | Commit | scope disposal | Session commits all publishes at once | One poison item fails the whole batch's commit — *investigate* |

Idempotency analysis (say this in interviews): the query predicate `!Published && ScheduledPublishDateTimeUtc < now` makes a re-run *mostly* idempotent — an item published by a previous run no longer matches. The index columns exist precisely for this ([`PublishLater/Migrations.cs:24-39`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)).

What juniors usually miss: `DoWorkAsync` receives the *scope's* `IServiceProvider` — resolving from the outer container would cross the tenant boundary.

What seniors notice: error handling is per-task, log-and-continue ([`ModularBackgroundService.cs:222-228`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)) — no retry, no dead-letter. Failure visibility depends entirely on logs; there's an `IBackgroundTaskEventHandler` seam (`:202-230`) you could use for metrics.

Interview angle: "Design scheduled publishing" — cover: where state lives (index table), who runs it (locked cron), idempotency (predicate), failure modes (batch commit).

Drill: what happens if two app instances tick simultaneously? Walk the lock path. Strong answer: second instance times out on the lock, terminates its scope, skips — publishes happen exactly once per tick *per lock scope*, but `PublishAsync` itself is also guarded by the `!Published` predicate.

---

## Flow 6: GraphQL content query (API read path)

Why this flow matters: a public API surface that compiles user input into SQL — validation, authorization, and injection concerns all live here.

Open these files first:
- [`src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs:87-124`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Schema | `:28-85` | Per content type, a field with `where`, `orderBy`, `first`, `skip`, `status` args | Draft data exposed via `status` — see below |
| 2 | Filters | `:100-105` | `IGraphQLFilter<ContentItem>.PreQueryAsync` — extension seam (authz filters plug in here) | If no filter registered, no row-level authz — *investigate* |
| 3 | Predicate build | `:126-201` | `where` JSON → alias resolution over index tables → SQL string + parameters (`query.WithParameter`, `:194-198`) | Parameterized — injection-resistant by construction |
| 4 | Paging | `:203-222` | `first` defaults to `DefaultNumberOfResults`; `Take/Skip` | Default cap prevents unbounded reads |
| 5 | Version filter | `:232-240` | `Published` vs `Latest && !Published` | — |
| 6 | Post filters | `:118-121` | `PostQueryAsync` — last chance to drop rows | O(n) filtering after fetch |

What juniors usually miss: the `where` argument is *compiled to SQL against index tables*, not evaluated in memory — that's why only indexed fields are filterable.

What seniors notice: the resolver takes `first` before authorization-style post-filters run (`:114-121`), so a post-filter that drops rows under-fills pages — a classic paging×authz interaction.

Interview angle: "How do you let API clients filter safely?" — whitelist of filterable fields (the where-input type), parameterization, and a hard result cap; this file demonstrates all three.

Drill: find where `status: DRAFT` should be permission-gated, starting from `PublicationStatusGraphType` — record what you find in the risk register (we did not fully trace it; see verification log).

---

## Flow 7: Recipe execution (setup / import)

Why this flow matters: it's how a tenant goes from empty DB to configured site, and how content/config imports work; also a scripting-in-JSON design worth critiquing.

Open these files first:
- [`src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs:40-158`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Events | `:42` | `RecipeExecutingAsync` handlers | — |
| 2 | Parse | `:54-60` | Stream-parse recipe JSON; root must be object | Large recipes stay streamed |
| 3 | Steps loop | `:71-139` | For each step: build `RecipeExecutionContext`, execute, record `RecipeStepResult` | A failing step throws `RecipeExecutionException` — no rollback of prior steps |
| 4 | Scripting | `:198-264` | Every string node scanned; `[js:...]`/`[file:...]` evaluated recursively until stable | Recipes are code — treat imports as privileged |
| 5 | Scoped execution | `:160-193` | Each step runs in `ShellScope.Current` or a fresh scope (`RequireNewScope`), invoking all `IRecipeStepHandler`s | Handlers are matched by step name across modules |
| 6 | Inner recipes | `:130-138` | Steps can enqueue nested recipes | Recursion depth unbounded |

What seniors notice: recipes are **not transactional across steps** — step 3's error handling captures and rethrows, but completed steps' scopes have already committed. Import design lesson: idempotent steps (content import keyed by `ContentItemVersionId` — see `DefaultContentManager.ImportAsync` batching at [`DefaultContentManager.cs:673-700`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) are what make retry-after-failure safe.

Interview angle: "Design a data import system" — batching (500/batch), version-id dedupe, per-step results, and the transactionality tradeoff.

Drill: trace what `"[js: parameters('AdminUsername')]"` in a setup recipe resolves through (ParametersMethodProvider, `:49`).

---

## Flow 8: Cached shape rendering (DynamicCache)

Why this flow matters: cache correctness (keys, contexts, invalidation, failover) in real code.

Open these files first:
- [`src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs:49-135`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Key | `:137-150` | Cache key = `CacheId` + hash of *discriminators* (route, role, culture… via `ICacheContextManager`) | Missing context in key = cross-user bleed |
| 2 | Read | `:49-68` | Context descriptor fetched first; unknown context = miss | Two round-trips per hit |
| 3 | Write | `:70-86` | Value + serialized context written (`Task.WhenAll`); local per-request dictionary too | — |
| 4 | Expiry | `:103-114` | Absolute/sliding from context; default sliding 1 min | — |
| 5 | Failover | `:95-99, 116-130` | Distributed-cache write/read failure → set `FailoverKey` in memory for 30s → *skip caching entirely* | Fail-open on availability: correct choice for a cache |
| 6 | Invalidation | `:88-91` + `AutoroutePartHandler.cs:199-202` | Tag-based: publishing content removes `slug:{path}` tag → all tagged keys deleted | Tag registry drift |

What seniors notice: the failover flag is a **circuit breaker** — protecting page latency from a dead Redis at the cost of temporarily uncached rendering. And the cache stores *strings of rendered HTML*, so anything user-specific must be a cache context or it leaks between users.

Interview angle: "How do you invalidate cache when content changes?" — tag-based invalidation from the publish handler is the concrete answer; contrast with TTL-only.

Drill: list the discriminators you'd need to add to safely cache a "Hello {username}" widget. What's the cost? (Key cardinality explodes per user — maybe don't cache it.)
