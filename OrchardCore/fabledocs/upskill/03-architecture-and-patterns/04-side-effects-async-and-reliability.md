# Side Effects, Async, and Reliability

A **side effect** is any write beyond the primary transaction: cache eviction, search index, route table, email, webhook. The reliability question is always the same: *what happens if the transaction commits but the side effect fails — or vice versa?*

## The side-effect map

| Side effect | Trigger | Mechanism | Timing vs commit | Retry? |
| --- | --- | --- | --- | --- |
| Route entries refresh | content publish/unpublish/remove | `_entries.UpdateEntriesAsync()` in Autoroute handler — [`AutoroutePartHandler.cs:63-113`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) (comment: "after the session is committed") | post-commit (deferred) | on next update |
| Cache tag eviction | publish etc. | `ITagCache.RemoveTagAsync("slug:...")` (`:199-202`) | with handler | none found |
| Homepage route change | publish with `SetHomepage` | `ISiteService.UpdateSiteSettingsAsync` (`:177-197`) | in-transaction (same session) | n/a |
| Scheduled publish | cron | `IBackgroundTask` + distributed lock ([Flow 5](../01-codebase-cartography/05-key-flows.md)) | own scope/txn | next tick (predicate-idempotent) |
| Search/Lucene/Elastic indexing | content events → background | `OrchardCore.Indexing` + tasks (dir surveyed, not read) | eventual | *verify per module* |
| Email, webhooks (Workflows) | module-specific | `OrchardCore.Email*`, `OrchardCore.Workflows` (not read) | — | *verify* |

## The repo's three side-effect tools (in order of increasing safety)

**1. Inline in the same session** — the effect joins the transaction. Right when the effect is *also* YesSql state (homepage route above). Atomic; couples latency.

**2. `ShellScope.AddDeferredTask(...)`** — run after the scope disposes, i.e. **after commit**, in a *fresh scope* — [`ShellScope.cs:387-392, 468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs). Failures are logged and *dropped* (`:497-504`). This is at-most-once delivery: commit-then-effect with no persistence of intent. Compare: a **transactional outbox** would write the intent in the same transaction and drain it with retries — Orchard chose simplicity + idempotent-refresh effects instead (route entries are *recomputed from the index*, so a dropped task heals on the next publish). That's the mature framing: **at-most-once effects are fine when effects are self-healing recomputations; you need an outbox when effects are non-idempotent external calls.**

**3. `ShellScope.AddDeferredSignal(key)`** — cache-token invalidation after scope end (`:377-382, 459-466`) — the lightest tool: invalidate, let readers rebuild.

Also note `HttpBackgroundJob` ([`src/OrchardCore/OrchardCore.Abstractions/BackgroundJobs/HttpBackgroundJob.cs`](../../../src/OrchardCore/OrchardCore.Abstractions/BackgroundJobs/HttpBackgroundJob.cs)) — fire a job after the current request, surveyed not fully read.

## Idempotency, spelled out

- **Scheduled publishing** is idempotent by *predicate*: `!Published && due` ([`ScheduledPublishingBackgroundTask.cs:29-32`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)) — a rerun finds nothing to redo.
- **Content import** is idempotent by *identity*: batches keyed on `ContentItemVersionId` so re-importing skips existing versions ([`DefaultContentManager.cs:673-700`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
- **PublishAsync** is idempotent by *guard*: `if (contentItem.Published) return true` ([`DefaultContentManager.cs:380-383`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).

Three idempotency techniques, one file away from each other — a ready-made interview answer.

## Concurrency control inventory

| Mechanism | Where | Protects |
| --- | --- | --- |
| `SemaphoreSlim` per tenant name | [`ShellHost.cs:87-106`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) | duplicate container builds |
| Distributed lock | background tasks ([`ModularBackgroundService.cs:133-157`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)), shell activation ([`ShellScope.cs:314-329`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)) | multi-instance double-run |
| Optimistic concurrency | `SaveAsync(x, checkConcurrency: true)` | lost updates on version rows |
| Retry loop + version compare | shell reload ([`ShellHost.cs:208-244`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)) | settings races across processes |
| Ref counting | `ShellContext` via scope ctor/dispose ([`ShellScope.cs:28-37, 553-556`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)) | container disposal under in-flight requests |

## Failure visibility (the weak flank)

Background task failures: logged, `context.Exception` set for `IBackgroundTaskEventHandler`s, then continue — [`ModularBackgroundService.cs:222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs). Deferred task failures: logged only. There is no dead-letter queue, no retry budget, no metric emission in the kernel (health checks module exists; task-level metrics don't). **"How would I know this broke?" → you grep logs.** This is the most legitimate critique of the design and a strong improvement-project candidate ([06-contribution-practice/03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md), Project 2).

## Backpressure and timeouts

- Background loop paces with `MinimumIdleTime` + `PollingTime` ([`ModularBackgroundService.cs:78-96, 348-358`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)).
- Lock acquisition has timeout semantics (skip tick on timeout, log Information — not Error: an intentional "someone else is doing it" signal).
- No global queue: burst work (e.g. 10k due items) runs in one batch/one commit — see Flow 5 risk. Batching exists in import (`_importBatchSize = 500` — [`DefaultContentManager.cs:21`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) but not in scheduled publishing. Inconsistency = ticket material.

## Drill

Design "send a notification email when a post is published" for this codebase three ways: (a) inline handler, (b) deferred task, (c) outbox table + background task. For each: what happens on commit failure, on SMTP failure, on process crash between commit and send? Which would you ship? Self-grade — Strong: chooses (c) for email (non-idempotent external effect), *and* can defend (b) if the product tolerates rare misses, citing how Autoroute gets away with it.

Interview angle: "exactly-once vs at-least-once vs at-most-once" and "what is a transactional outbox" → [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) variation 3 and [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q10.
