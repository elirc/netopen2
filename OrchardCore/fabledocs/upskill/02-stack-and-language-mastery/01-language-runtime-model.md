# C# / .NET Runtime Model

## 1. async/await is cooperative scheduling over real threads

Mental model: `await` splits a method into continuations scheduled on the thread pool. Unlike Node, **many awaits run truly in parallel on different threads** — shared mutable state needs synchronization.

Where this repo uses it well:
- `ShellHost` guards per-tenant creation with a `SemaphoreSlim` per key and double-checked reads — [`ShellHost.cs:80-119`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs). `lock` can't contain an `await`; `SemaphoreSlim.WaitAsync` is the async-compatible mutex.
- `ModularBackgroundService` fans out per tenant with `Parallel.ForEachAsync` — [`ModularBackgroundService.cs:100`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) — real parallelism across tenants, sequential within one.
- Concurrent dictionaries hold cross-request state: `ConcurrentDictionary<string, ShellContext>` ([`ShellHost.cs:27-29`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)).

Sharp edges to check anywhere you touch:
- **Atomicity gaps**: `TryGetValue` then `TryRemove` on a concurrent dictionary is two atomic steps, not one atomic step — `ShellHost` handles the gap with retry loops and "Consistency:" comments ([`ShellHost.cs:219-240`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)). Read those comments; they're a masterclass.
- **Sync-over-async** (`.Result`, `.Wait()`) deadlocks or starves the pool. This repo avoids it; you should too.
- Fire-and-forget tasks lose exceptions. The repo's alternative is deferred tasks with logging ([`ShellScope.cs:489-506`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)).

## 2. AsyncLocal: ambient context that flows with awaits

`ShellScope.Current` is an `AsyncLocal<ShellScopeHolder>` — [`ShellScope.cs:14, 227-230`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs). Any code, however deep, can ask "which tenant am I?" without parameter-threading. Node analog: `AsyncLocalStorage`.

Why the *holder* indirection (line 599-602): `AsyncLocal` values are captured per execution context; mutating a field inside the shared holder clears the scope in *all* captured contexts at once ([`Terminate()`, :580-588]). That's a subtle-but-real trick to know.

Failure mode: ambient context is invisible coupling. `ShellScope.Services.GetRequiredService<T>()` works until someone calls it outside a scope and gets a null — which is why kernel code null-checks `Current`.

## 3. Exceptions, filters, and "fatal vs recoverable"

Pattern used throughout: `catch (Exception ex) when (!ex.IsFatal())` — e.g. [`ModularBackgroundService.cs:89-92`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs). The `when` filter keeps stack traces intact and lets OOM/thread-abort style exceptions escape. Compare `RecipeExecutor` capturing with `ExceptionDispatchInfo` to rethrow *outside* a catch block with original stack — [`RecipeExecutor.cs:95-123`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs).

Transferable rule: catch broadly only at *unit-of-work boundaries* (per background task, per recipe step, per request), log with context, and rethrow or record — never swallow silently mid-flow.

## 4. Disposal and lifetime = resource ownership

`ShellScope` implements both `IDisposable` and `IAsyncDisposable` with idempotent state flags ([`ShellScope.cs:542-578`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)); `ShellContext` is **ref-counted** (`AddRef`/`Release`) so a tenant reload doesn't yank the container out from under in-flight requests. `ShellSettings` even has a finalizer as a last resort ([`ShellSettings.cs:173-199`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs)).

Junior take: `using` = "close my file." Senior take: disposal is an ownership protocol — *who* disposes, *when*, and what may still hold a reference. Ref-counting appears exactly when ownership is shared.

## 5. Delegates and multicast events as extension points

`ShellHost` exposes `ShellEvent` delegates and iterates `GetInvocationList()` to await each handler ([`ShellHost.cs:190-196`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)) — because `await eventDelegate()` on a multicast would await only the last. Niche but a great "do you actually know the language" interview nugget.

## Pitfall checklist (run it on your own PRs)

- [ ] No `.Result`/`.Wait()`/`GetAwaiter().GetResult()` on hot paths
- [ ] Shared mutable state is concurrent-safe or scope-confined
- [ ] Every `catch` either rethrows, cancels the unit of work, or logs with tenant/task context
- [ ] `CancellationToken` is passed through loops that can be long (compare `ScheduledPublishingBackgroundTask.DoWorkAsync` — the per-item loop at [`:41-57`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) checks no token — is that fine for a 1-minute cron? form an opinion)
- [ ] Disposal ownership is explicit when an object outlives the method that created it

## Drill

In `GetOrCreateShellContextAsync` ([`ShellHost.cs:80-119`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)), explain: why the second `TryGetValue` inside the semaphore, and why the outer `while` loop still needed after that. Self-grade — Basic: "double-checked locking." Solid: names the released-shell race the loop covers. Strong: explains why a single global lock would be wrong (head-of-line blocking across tenants).

## Interview angle

1. "Explain async/await vs threads" → [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q1–Q3 (asked in .NET terms there).
2. "How do you share context across async calls without parameters?" → AsyncLocal / ShellScope story.
3. "Double-checked locking — when is it safe?" → this drill.
4. "How do you handle exceptions in background jobs?" → §3 + Flow 5.
