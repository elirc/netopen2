# Pattern Catalog

16 cards. The goal is **recognition** — seeing the shape in any codebase and knowing its failure modes — not name-dropping. Each card: problem → shape → two real anchors → why it works → failure modes → when to use/avoid → interview angle → drill.

---

## Pattern 1: Container-per-Tenant

Problem it solves: tenants need different features, config, and data stores in one process, with hard isolation.
General shape: resolve tenant from request → swap the DI container → everything downstream is tenant-scoped for free.
Real example: [`ModularTenantContainerMiddleware.cs:27-69`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs)
Second example: background variant — [`ModularBackgroundService.cs:100-124`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) (scope per tenant per tick).
Why this implementation works: isolation is structural (you *can't* inject another tenant's services), and feature variance is just different registrations.
Failure modes: memory per tenant; slow cold start per container; host-level singletons become accidental cross-tenant channels.
Use it when: tenants vary in behavior/features. Avoid when: thousands of identical tenants → row-level tenancy is cheaper.
Interview angle: "compare multi-tenancy strategies" — you can now name a real implementation of the heavyweight end.
Drill: find where a tenant's container is *discarded* (feature toggle → `ShellHost.ChangedAsync` → release — [`ShellHost.cs:173-174`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)).

---

## Pattern 2: Lazy Init with Per-Key Locking (double-check + semaphore map)

Problem: build an expensive resource at most once per key, under concurrency, without a global lock.
Shape: check → per-key `SemaphoreSlim` → re-check → build → publish; retry loop for the "released while I looked" race.
Real: [`ShellHost.cs:80-119`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs). Second: shell activation lock — [`ShellScope.cs:302-350`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs) (distributed flavor).
Why it works: contention isolated per tenant; the outer `while` covers the check-to-use gap.
Failure modes: semaphore map grows unbounded (here: keys = tenants, bounded); forgetting the inner re-check; deadlock if build path re-enters the same key.
Use when: expensive per-key resources (connections, compiled templates). Avoid when: `Lazy<T>`/`GetOrAdd` suffices (cheap or non-async factory).
Interview angle: "implement a thread-safe cache of expensive objects" — this is the model answer with an async twist.
Drill: explain why `ConcurrentDictionary.GetOrAdd` alone is insufficient for *async* factories (factory may run multiple times; can't await inside).

---

## Pattern 3: Unit of Work with Commit-at-Scope-End

Problem: many services mutate state per request; you want one atomic commit and easy rollback.
Shape: scoped session; register commit as a scope-disposal callback and cancel as the exception handler.
Real: [`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs). Second: explicit cancel on validation failure — [`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs).
Why it works: controllers never call `SaveChanges`; forgetting to commit is impossible; requests are transactional by default.
Failure modes: *forgetting to cancel* persists partial work; long scopes hold transactions; post-commit effects need a second mechanism (Pattern 4).
Use when: request-scoped writes dominate. Avoid when: you need multiple independent transactions per request.
Interview angle: "where do you put SaveChanges?" — argue for boundary-owned commit vs sprinkled saves.
Drill: list every `CancelAsync` call site in `AdminController` and what each protects (there are four).

---

## Pattern 4: Deferred Post-Commit Tasks (poor-man's outbox)

Problem: side effects that must not run unless the transaction commits.
Shape: collect callbacks during the scope; after disposal, run each in a fresh scope; log-and-drop failures.
Real: [`ShellScope.cs:387-392, 468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs). Second: consumer — Autoroute's "update entries after the session is committed" ([`AutoroutePartHandler.cs:74-76`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)).
Why it works: ordering guarantee (after commit) without infrastructure; fresh scope means a clean session.
Failure modes: **at-most-once** — crash between commit and task = lost effect; no retry/dead-letter; invisible failures.
Use when: effects are idempotent recomputations or tolerable losses. Avoid when: money, emails, external APIs → outbox.
Interview angle: "how do you avoid publishing an event for a rolled-back transaction?" — deferred tasks vs outbox vs CDC ladder.
Drill: write the outbox-table schema that would upgrade this to at-least-once.

---

## Pattern 5: Lifecycle Event Pipeline (forward/reverse handler chains)

Problem: many modules need to react to entity lifecycle without coupling to the core.
Shape: `IEnumerable<IContentHandler>` invoked in order for `*ing` (pre) events and **reversed** for `*ed` (post) events — symmetric nesting like middleware.
Real: [`DefaultContentManager.cs:60-65, 85-96`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs). Second: tenant events, same reversal — [`ShellScope.cs:336-345`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs).
Why it works: registration order = dependency order; reversal gives onion semantics (first-in, last-out).
Failure modes: hidden ordering dependencies; a slow handler taxes every operation; exceptions in one handler abort the chain (see `InvokeAsync` extension's logging).
Use when: cross-cutting reactions to domain events in-process. Avoid when: consumers need isolation/retry → real event bus.
Interview angle: "how would you implement domain events?" + the *why reversed* follow-up (few candidates can answer it).
Drill: predict handler order for Publishing vs Published given registrations [A, B, C].

---

## Pattern 6: Driver/Shape/Placement (decentralized UI composition)

Problem: a page composed by many modules, restylable by themes, without anyone owning the whole view model.
Shape: drivers emit typed shape fragments + declared zones; placement files position them; templates resolve by name/alternates.
Real: [`TitlePartDisplayDriver.cs:20-36`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) (`.Location("Detail", "Header")`). Second: [`Contents/placement.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json) + alternates computation [`ContentItemDisplayManager.cs:75-83`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs).
Why it works: open for extension (new part = new driver, zero core changes); themes override by *supplying a template*, not forking code.
Failure modes: stringly-typed contract — renames break silently; debugging "where did this HTML come from" requires knowing alternate precedence.
Use when: plugin UIs. Avoid when: single-team app — the indirection tax outweighs extensibility.
Interview angle: "how do plugins contribute UI safely?" — compare to React slots/portals; same problem, server-side answer.
Drill: given shape `Content_SummaryAdmin__BlogPost`, list the template names that could render it, most-specific first.

---

## Pattern 7: Versioned Rows with Flag Invariants

Problem: drafts + published versions + history for the same logical entity.
Shape: N rows per `ContentItemId`; boolean flags `Latest`/`Published`; all transitions flip flags in one transaction.
Real: [`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) (publish = demote previous + promote current). Second: draft creation clones `Data` with new version id — [`:497-542`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs).
Why it works: single-table queries for "live site" (`Published`) and "admin" (`Latest`); history is free.
Failure modes: invariant is code-enforced only — any writer bypassing `IContentManager` can create two Published rows; queries forgetting the flag return duplicates (the #1 novice YesSql bug).
Use when: draft/publish workflows. Avoid when: append-only audit is enough (event log) or no versioning needed.
Interview angle: classic schema-design question; you can also discuss the alternative (separate draft table) and why flags won here (uniform queries).
Drill: write the query for "items with a draft newer than the published version" (`Latest && !Published` exists ⇒ item has unpublished changes).

---

## Pattern 8: Permission Implication Hierarchy + Owner Variants

Problem: fine-grained grants without exploding role definitions; "own content" rules without controller if-statements.
Shape: permissions declare implied-by chains; resource-based handler swaps in Own-variant on ownership match; per-type dynamic permissions generated from templates.
Real: [`CommonPermissions.cs:16-56`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs). Second: [`ContentTypeAuthorizationHandler.cs:20-76`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs).
Why it works: policy lives in one handler; every endpoint that passes the resource gets ownership logic for free.
Failure modes: endpoints that *don't* pass the resource silently skip owner logic; implication chains are hard to audit ("who can effectively publish?").
Use when: role-based systems with ownership semantics. Avoid when: relationship-heavy authz (teams/hierarchies) → policy engine.
Interview angle: "RBAC vs ABAC vs ReBAC" grounded in real code.
Drill: compute the full set of users who pass `ViewContent` for a post they don't own (walk the implied-by graph).

---

## Pattern 9: Check-Then-Suffix Uniqueness (and its race)

Problem: human-friendly unique identifiers (slugs).
Shape: generate → query for conflict → append `-2`, `-3`… until free.
Real: [`AutoroutePartHandler.cs:437-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs). Second: relative variant for contained items — [`:341-376`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs).
Why it works: good UX, no schema support needed, works on all four DB providers.
Failure modes: TOCTOU under concurrency (two writers both see "free"); truncation interplay (`MaxPathLength` trimming, `:361-365`).
Use when: low write contention + human-visible ids. Avoid when: strict uniqueness matters → DB unique constraint + violation-retry.
Interview angle: the best small concurrency question in the repo — see Flow 4 drill.
Drill: implement constraint-plus-retry pseudocode preserving the friendly `-2` behavior.

---

## Pattern 10: Circuit-Breaker Failover for Caching

Problem: a dead distributed cache must not take page rendering down with it.
Shape: on cache I/O failure, set an in-memory "failover" flag with TTL; while set, skip cache entirely; retry after TTL.
Real: [`DefaultDynamicCacheService.cs:93-135, 157-191`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs).
Second example: No second example found in files read.
Why it works: converts repeated timeouts (slow) into a fast local check; self-heals after 30s.
Failure modes: fail-open means a cache outage = full render load on origin — capacity must tolerate it; flag is per-instance (thundering retry across a farm).
Use when: cache is an optimization. Avoid when: cache is load-bearing for correctness (then it's not a cache).
Interview angle: "what happens when Redis dies?" — concrete answer with the tradeoff named.
Drill: add jitter to the retry latency — one line; why does it matter at fleet scale?

---

## Pattern 11: Tag-Based Cache Invalidation

Problem: invalidate *all* cached fragments affected by a content change, without knowing their keys.
Shape: at write time, associate cache entries with tags; at change time, remove-by-tag.
Real: tagging — [`DefaultDynamicCacheService.cs:132-134`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs); eviction — [`AutoroutePartHandler.cs:199-202`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) (`slug:{path}`).
Why it works: producers of change don't enumerate consumers; tags are the contract.
Failure modes: missing tag = permanent staleness; over-broad tags = cache stampede on eviction.
Use when: fragment caching with content dependencies. Avoid when: TTL tolerance is fine (simpler).
Interview angle: pairs with "hardest problem in CS" joke — then show you actually have a mechanism.
Drill: what tag(s) should a "recent posts" widget carry?

---

## Pattern 12: Data Migrations with Version Ladder

Problem: evolve schema per tenant, per module, across four SQL dialects.
Shape: `CreateAsync` (fresh = final shape, returns latest N) + `UpdateFromNAsync` ladder for upgrades.
Real: [`PublishLater/Migrations.cs:18-89`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs). Second: definition-only migration — [`Contents/Migrations.cs:16-23`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Migrations.cs).
Why it works: fresh installs skip history; upgrades replay exactly what they're missing; per-tenant tracking.
Failure modes: provider portability limits (can't drop/alter everywhere — the file says so); migrations that read app services can break when code outpaces schema.
Interview angle: "how do you ship a breaking schema change?" — expand/contract with this ladder as mechanism.
Drill: sketch `UpdateFrom3Async` adding a `TimeZoneId` column + updated composite index.

---

## Pattern 13: Manifest-Declared Modularity (features as units)

Problem: ship one binary, let each tenant enable different capabilities.
Shape: assembly attributes declare features + dependencies; enabling a feature = registering its services in the tenant container.
Real: [`Contents/Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) (six features in one module). Second: dependency resolution driving shell descriptor — `IShellFeaturesManager` ([`src/OrchardCore/OrchardCore.Abstractions/Shell/IShellFeaturesManager.cs`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/IShellFeaturesManager.cs), surveyed).
Why it works: feature flags at the composition level, not `if` statements — disabled code isn't just skipped, it's *not registered*.
Failure modes: runtime dependency errors instead of compile-time; feature matrix testing explodes.
Interview angle: "feature flags vs modularity" — this is the far end of the spectrum.
Drill: what happens to a tenant's container when a feature is toggled? (Pattern 1 drill answer.)

---

## Pattern 14: Recipes — Declarative Setup with Embedded Scripting

Problem: reproducible site provisioning and content import.
Shape: JSON steps dispatched to named handlers; string values may be `[js: ...]` expressions evaluated recursively.
Real: [`RecipeExecutor.cs:40-158, 198-264`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs).
Second example: recipe files under `src/OrchardCore.Themes/*/Recipes/` and module `Recipes/` folders (surveyed).
Why it works: setup = data; same engine powers setup, import, deployment.
Failure modes: not transactional across steps; scripting makes recipes a privileged execution surface (treat imports like code review).
Interview angle: "infrastructure as data" tradeoffs; also a security-review talking point.
Drill: Flow 7 drill (trace a `[js: parameters(...)]`).

---

## Pattern 15: Anti-Enumeration Timing Normalization

Problem: login timing reveals whether a username exists.
Shape: on unknown user, burn the same CPU as a real password hash verification.
Real: [`AccountController.cs:182-188`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs) (`PasswordTimingNormalizationService`).
Second example: No second example found in files read.
Why it works: response-time distribution becomes indistinguishable; combined with uniform error messages (`:190`).
Failure modes: normalization must track hash cost (work-factor upgrades); other channels (registration "email taken") still leak.
Interview angle: instantly upgrades your security-round answer beyond "same error message."
Drill: list three other endpoints that leak account existence in typical apps (password reset, registration, invite) — check how this repo's `ResetPasswordController` handles it (not read here — go verify).

---

## Pattern 16: Placeholder Pre-Registration (lazy heavy objects behind a complete registry)

Problem: the registry (running-shell table) must know *all* tenants at startup; building all containers is too expensive.
Shape: register lightweight placeholders that carry settings; upgrade to the real object on first use.
Real: [`ShellHost.cs:335-374`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) (`ShellContext.PlaceHolder`, `PreCreated = true`). Second: placeholder re-added during reload to keep settings resolvable — [`:218-225`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs).
Why it works: O(n) cheap startup, complete routing table, pay-per-use container builds.
Failure modes: two-state objects invite "forgot it might be a placeholder" bugs (`HasPipeline()` checks scattered around).
Interview angle: lazy loading at architectural scale; compare to virtual proxies.
Drill: find the code that must *check* for a placeholder before using a pipeline ([`ModularBackgroundService.cs:127-131`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)).

---

## How to use this catalog

Flashcard mode: read a card's *problem* line, recall shape + failure modes + one anchor from memory. When you can do all 16, do the same with your current work codebase — the recognition transfers; that's the point.
