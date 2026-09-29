# Annotation Drills

For each excerpt, annotate **five things** before checking the answer key at the bottom of each drill:
**I**nputs (incl. ambient) · **O**utputs (incl. mutations) · **D**ependencies · **Inv**ariants protected · **F**ailure modes.

Self-grading rubric (per drill): **Basic** = I/O correct. **Solid** = + dependencies incl. ambient ones, and at least one real failure mode. **Strong** = + names the invariant precisely and one non-obvious edge (each answer key marks it ★).

---

## Drill 1 — `ShellHost.GetOrCreateShellContextAsync`
Open [`src/OrchardCore/OrchardCore/Shell/ShellHost.cs:80-119`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs). Annotate.

<details><summary>Answer key</summary>
I: `ShellSettings` (may lack loaded configuration). O: a non-released `ShellContext`; mutates `_shellContexts`, `_shellSemaphores`. D: settings manager, context factory (ambient: none). Inv: at most one *live* context per tenant name. F: factory throws inside semaphore (semaphore released by finally; no poisoned entry); settings reload fails. ★ the `while` handles a context released between creation and observation — deletion is done by the *observer* (`TryRemove` at 109-115), not the releaser.
</details>

## Drill 2 — `DefaultContentManager.GetAsync` with `DraftRequired`
Open [`DefaultContentManager.cs:102-186`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs).

<details><summary>Answer key</summary>
I: id + `VersionOptions`. O: content item — **and possibly two session writes** (previous version + new draft). D: session, definition manager, handlers via `LoadAsync`, `_contentManagerSession` identity cache. Inv: ≤1 Latest per item survives (new version created via `BuildNewVersionAsync` flips old `Latest=false`). F: concurrency exception at commit (`checkConcurrency:true`); non-versionable types mutate the published row in place (168-171). ★ a "Get" that writes — callers who "just read" with DraftRequired create drafts accidentally.
</details>

## Drill 3 — `AdminController.ListPOST` bulk publish
Open [`AdminController.cs:267-333`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs).

<details><summary>Answer key</summary>
I: `itemIds` = **DocumentIds** (long, not ContentItemIds), `options.BulkAction`. O: redirect; publishes/unpublishes/removes items. D: session query with `Latest` filter, per-item authorization, notifier. Inv: per-item authz (no bulk bypass). F: first failure → warning + `CancelAsync` + redirect — earlier successes in the loop are **rolled back too** (one session). ★ authorized-but-cancel-failed returns redirect while unauthorized returns `Forbid` — the `is var authorized` dance at 283-291 encodes two different failure meanings in one condition.
</details>

## Drill 4 — `AutoroutePartHandler.GenerateUniqueAbsolutePathAsync`
Open [`AutoroutePartHandler.cs:437-463`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs).

<details><summary>Answer key</summary>
I: candidate path, contentItemId. O: unique-at-check-time path. D: `AutoroutePartIndex` via session (uncommitted view!). Inv: tenant-wide path uniqueness (best-effort). F: TOCTOU across concurrent sessions; pathological loop if many collisions (each iteration = 1 query). ★ existing `-N` suffix is parsed and *continued* (443-446), and length is re-trimmed per iteration so `-10` doesn't overflow MaxPathLength (450-455).
</details>

## Drill 5 — `ShellScope.BeforeDisposeAsync`
Open [`ShellScope.cs:444-509`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs).

<details><summary>Answer key</summary>
I: accumulated callbacks/signals/tasks. O: session committed (via registered callback), signals sent, deferred tasks run. D: `ISignal`, `IShellHost` (new scopes per task). Inv: deferred tasks run **after** commit, each in a fresh scope. F: deferred task exception → logged + `HandleExceptionAsync`, *not* rethrown — silent partial completion. ★ fallback path (481-485): if the shell was released, a scope is built on the *dying* context so the task still runs.
</details>

## Drill 6 — `ContentItemIndexProvider.Describe`
Open [`ContentItemIndex.cs:28-73`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs).

<details><summary>Answer key</summary>
I: every saved `ContentItem`. O: one `ContentItemIndex` row per version row. D: none (pure mapping) — that's the design point. Inv: index mirrors document fields at save time. F: >255-char values silently truncated → index-based filters/sorts diverge from document truth. ★ `Latest=false` old versions still produce rows — "all versions" queries work, but every unfiltered query sees historical rows.
</details>

## Drill 7 — `LoginPOST` unknown-user branch
Open [`AccountController.cs:125-206`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs).

<details><summary>Answer key</summary>
I: form-bound `LoginForm` via display manager, `returnUrl`. O: cookie (success) / view with errors. D: site settings, sign-in manager, `ILoginFormEvent` collection, timing normalizer. Inv: indistinguishable responses for unknown-user vs wrong-password (message *and* timing). F: an `ILoginFormEvent` throwing mid-chain; lockout treated after 2FA branch. ★ password is verified twice (133, 149) — first without cookie issuance so `ValidatingLoginAsync` can still veto between them.
</details>

## Drill 8 — `ScheduledPublishingBackgroundTask.DoWorkAsync`
Open [`ScheduledPublishingBackgroundTask.cs:27-58`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs).

<details><summary>Answer key</summary>
I: scope `IServiceProvider`, cancellation token. O: published items; cleared `ScheduledPublishUtc`. D: session (index query), content manager. Inv: idempotency via `!Published && due` predicate. F: one bad item throws → whole batch aborts; token checked only in the query, not the loop. ★ `part.Apply()` before `PublishAsync` — clearing the schedule is part of the same commit as the publish, so a crash can't leave "published but still scheduled."
</details>

## Drill 9 — `CreateEndpoint.HandleAsync` (update branch)
Open [`CreateEndpoint.cs:95-133`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs).

<details><summary>Answer key</summary>
I: JSON `ContentItem`, `draft` query flag, Api-scheme principal. O: JSON of saved item / ValidationProblem / challenge. D: content manager, authz service, session. Inv: validation failure cancels session. F: merge semantics (arrays replaced) surprise clients; ★ `EditContent` authorized but `PublishAsync` reachable at 126 — the permission/operation mismatch (risk R1). If you caught ★ unprompted, you're reviewing at mid+ level.
</details>

## Drill 10 — `DefaultDynamicCacheService.SetCachedValueAsync` (private)
Open [`DefaultDynamicCacheService.cs:93-135`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs).

<details><summary>Answer key</summary>
I: key, rendered HTML string, `CacheContext` (expirations, tags). O: distributed-cache write + tag association; or nothing (failover). D: `IDynamicCache`, memory cache (flag), lazily-resolved `ITagCache`. Inv: cached value always has retrievable context (written in parallel by caller, 82-85). F: cache write throws → failover flag 30s, **return before tagging** — value absent but that's safe (miss). ★ default sliding expiration of 1 minute when nothing set (110-114) — "cached forever" never happens silently.
</details>
