# Runtime Deep-Dive Cards (C#/.NET, with JS mappings)

14 cards. Format: what it's really testing → repo anchor → junior/mid/senior answer contrast → follow-ups → drill. Practice aloud, 90 seconds per answer.

---

## Q1: "Explain async/await. What actually happens at an `await`?"
Round: deep-dive. Testing: mental model vs syntax knowledge.
Repo anchor: [`ShellHost.cs:80-119`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs) (awaits inside a semaphore).
Junior: "it waits without blocking." Mid: continuation scheduling — the method returns a Task, resumes on the thread pool; contrast with JS's single-threaded loop: **.NET continuations run concurrently on many threads**. Senior: adds sync-context capture rules, why `lock` can't span `await` (thread affinity of Monitor) and the `SemaphoreSlim.WaitAsync` alternative this repo uses.
Follow-ups: "what breaks if two awaits touch shared state?" "Task vs ValueTask?"
Drill: explain lines 87-106 aloud without reading them.

## Q2: "How do you prevent an expensive resource being created twice under concurrency?"
Round: deep-dive/coding. Testing: concurrency patterns beyond `lock`.
Repo anchor: same as Q1 — per-key semaphore + double-check + retry loop.
Junior: one big lock. Mid: per-key lock, double-check inside, and *why* the global lock is wrong (head-of-line blocking across tenants). Senior: the residual race the `while` handles (released-after-created), and when to prefer `Lazy<Task<T>>` per key.
Follow-ups: "and across multiple servers?" → distributed lock ([`ModularBackgroundService.cs:133-157`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)).
JS mapping: memoized promises give you dedup for free on one event loop; the multi-instance question is identical.

## Q3: "How does context flow through async code without passing parameters?"
Testing: AsyncLocal / ALS knowledge (rarely known at junior level — differentiator card).
Repo anchor: [`ShellScope.cs:14, 227-230, 580-588`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs).
Junior: statics or DI. Mid: `AsyncLocal<T>` flows with the execution context, like Node's `AsyncLocalStorage`; used here for "current tenant scope." Senior: the holder-object trick so clearing propagates to all captured contexts; failure mode — ambient context as hidden coupling, null outside a scope.
Follow-up: "why not a scoped DI service?" (background code creates scopes *before* DI resolution is possible).

## Q4: "Exceptions: how should a long-running worker handle them?"
Repo anchor: [`ModularBackgroundService.cs:78-96, 222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs).
Junior: try/catch and log. Mid: catch at unit-of-work boundaries; exception filters (`when (!ex.IsFatal())`) to let fatal errors escape; keep the loop alive. Senior: what "log-and-continue" costs (silent degradation, R3), the retry/dead-letter upgrade path, and `ExceptionDispatchInfo` for rethrow-with-stack ([`RecipeExecutor.cs:95-123`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs)).

## Q5: "What are DI lifetimes, and tell me about a bug class from misusing them."
Repo anchor: [`OrchardCoreBuilderExtensions.cs:56-173`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) (singleton store, scoped session).
Junior: lists three lifetimes. Mid: captive dependency — singleton capturing scoped `ISession` = one shared unit of work across requests; how this repo dodges it with lazy `IServiceProvider` resolution ([`ContentTypeAuthorizationHandler.cs:69-70`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)). Senior: "singleton per *what*?" — per tenant container here; lifetimes are relative to their container.

## Q6: "Reference vs value identity: when does object identity bite you?"
Repo anchor: [`ContentElementTests.cs:9-38`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs) — two `Get<T>` calls, two instances.
Junior: == vs Equals. Mid: hydrated-projection models (JSON-backed parts) return fresh instances; mutate-then-`Apply()` is the contract. Senior: generalizes to ORMs/DTOs everywhere: identity map vs projection, and how the repo's `_contentManagerSession` identity cache ([`DefaultContentManager.cs:144-148`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) restores identity where it matters.

## Q7: "Explain cancellation. Where do you check the token?"
Repo anchor: [`ModularBackgroundService.cs:114-117`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) vs the missing check in [`ScheduledPublishingBackgroundTask.cs:41-57`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs).
Mid: cooperative; check between units of work; pass it down. Senior: cancellation vs consistency — cancelling mid-batch here still commits partial work at scope end; predicate idempotency makes that safe, *and you should be able to say why*.

## Q8: "What is a unit of work? Where should SaveChanges/commit live?"
Repo anchor: [`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs).
Junior: repository calls Save. Mid: commit owned by the request boundary; code *cancels* to opt out ([`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). Senior: tradeoffs — long transactions, post-commit side-effect problem and its deferred-task/outbox solutions.

## Q9: "Multicast delegates/events — any gotcha with async handlers?"
Repo anchor: [`ShellHost.cs:190-196`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs).
Niche differentiator: awaiting a multicast delegate awaits only the last target; iterate `GetInvocationList()`. JS mapping: emitting to N listeners returning promises — you must `Promise.all` (or sequence) explicitly too.

## Q10: "How do you make code testable without a DB or clock?"
Repo anchor: `IClock` injection ([`ScheduledPublishingBackgroundTask.cs:21-25`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs)).
Mid: seams via interfaces; time/randomness/IO injected. Senior: which seams *not* to mock (the session — its semantics are the thing under test; see [05-quality-engineering/01](../05-quality-engineering/01-testing-strategy.md)).

## Q11: "Explain `IDisposable`/`await using` and shared ownership."
Repo anchor: ref-counted `ShellContext` ([`ShellScope.cs:28-37, 553-575`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)).
Junior: using = cleanup. Mid: ownership protocol; async disposal for async cleanup. Senior: ref-counting when lifetime is shared (in-flight requests vs tenant reload) and idempotent disposal via state flags.

## Q12: "String handling / JSON: merge vs replace semantics in PATCH?"
Repo anchor: `MergeArrayHandling.Replace` ([`DefaultContentManager.cs:23-26`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
Mid: arrays replace (else duplicates accumulate), objects merge; contract must be documented. Senior: JSON Merge Patch (RFC 7386) vs JSON Patch (6902) framing and when each fits an API.

## Q13: "How would you throttle an endpoint?"
Repo anchor: `[RateLimitGroup(PasswordAuthentication)]` ([`AccountController.cs:105`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)).
Mid: middleware-level limiter, keyed by user/IP, 429 + Retry-After. Senior: which endpoints need *different* keys (login by username+IP to stop credential stuffing without cross-user DoS) and fail-open vs fail-closed choice.

## Q14: "Parallelism: when do you use Parallel.ForEachAsync / Promise.all, and when not?"
Repo anchor: parallel across tenants, sequential within one ([`ModularBackgroundService.cs:100`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)); `Task.WhenAll` for the two cache writes ([`DefaultDynamicCacheService.cs:82-85`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)).
Mid: parallelize independent units; keep dependent work sequential. Senior: the unit choice *is* the design — tenant = isolation boundary = safe parallel unit; plus bounded parallelism/backpressure for large fan-outs.

---

Coverage check: 14/14 repo-anchored. Do three cards a day; the [cram plan](07-two-week-cram-plan.md) schedules them.
