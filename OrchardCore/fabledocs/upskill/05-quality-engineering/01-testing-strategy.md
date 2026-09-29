# Testing Strategy

## The layers this repo actually has

| Layer | Project | Scope | Speed | What belongs here |
| --- | --- | --- | --- | --- |
| Unit | `test/OrchardCore.Tests` (xUnit v3) | classes, serialization, kernel logic | ms | invariants provable without HTTP/DB: content element casting ([`ContentElementTests.cs`](../../../test/OrchardCore.Tests/ContentManagement/ContentElementTests.cs)), shell table matching (`Shell/RunningShellTableTests.cs`), permission handlers (`Security/PermissionHandlerTests.cs`) |
| Integration | `test/OrchardCore.Tests.Integration` | subsystems with real infrastructure | s | *(project listed; contents not reviewed here)* |
| Functional/E2E | `test/OrchardCore.Tests.Functional` (Playwright + xUnit) | full CMS in-process, headless Chromium | 10s+ | setup wizard, login, multi-tenancy, theming — per its [README](../../../test/OrchardCore.Tests.Functional/README.md) |
| Benchmarks | `test/OrchardCore.Benchmarks` | perf regressions | — | BenchmarkDotNet suites |

CI runs unit tests per-PR (`pr_ci.yml`), functional tests against **all four database providers** (`functional_all_db.yml`) — the portability constraints in migrations ([`PublishLater/Migrations.cs:47-48`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)) are enforced by this matrix, not by review vigilance alone.

## What the E2E layer does architecturally (worth stealing)

From the functional README (__verified__ read): each test fixture boots Orchard **in-process** via `OrchardTestServer` on a dynamic port with an isolated `App_Data_Tests_{Fixture}` directory and per-fixture database; recipes come from **embedded resources** to avoid shared filesystem state; fixtures parallelize safely because *all* state is fixture-scoped. That is the checklist for hermetic E2E anywhere: own data dir, own DB, own port, no shared mutable fixtures.

## What belongs at which layer (decision rules for this codebase)

- **Invariant on pure logic** (flag transitions, path suffixing, permission implication) → unit. If you need a DB to test it, first ask if the logic can be extracted.
- **Anything crossing the session/commit boundary** (does cancel actually roll back? does publish demote the previous version *in the database*?) → integration/functional. Unit-mocking `ISession` here tests your mocks.
- **Anything involving shapes/templates/placement** → functional. The rendering contract is stringly-typed (risk R5); only a real render catches a broken alternate.
- **Migrations** → the all-DB functional matrix is the only honest test; SQLite locally as smoke.

## Test-double policy you can infer from the code

The kernel is interface-everything (`IClock`, `IContentManager`, `ISession` …) — [`ScheduledPublishingBackgroundTask.cs:21-25`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) takes `IClock` precisely so due-time logic is testable without real time. `test/OrchardCore.Tests/Stubs/` exists for shared fakes. Time and randomness are injected (`IClock`, `IdGenerator`); if you write a test with `DateTime.UtcNow`, you've missed the seam the codebase already gave you.

## Flake prevention

- No shared static state between fixtures (see fixture isolation above).
- Parallelism is explicit: `xunit.runner.json` + `maxParallelThreads`.
- Background/polling behavior (e.g. `ModularBackgroundService`) is not E2E-tested by sleeping — prefer unit-level scheduler tests (`BackgroundTaskScheduler` is a plain class) and functional tests that trigger work synchronously.

## What NOT to test (and why saying so matters)

- Framework behavior (MVC binding, Identity hashing) — trust the platform, test your configuration of it.
- Generated/dynamic permissions for every content type — test the *generator* once.
- Rendering pixel output — assert on semantic markers, not markup details (themes own markup).

## Drill

Classify these five into layers, then justify: (1) "Latest flag is unique per item after publish", (2) "permalink collision shows a form error", (3) "UpdateFrom2Async adds three columns on MySQL", (4) "login lockout after N failures", (5) "GetOrCreateShellContextAsync never builds two containers for one tenant under 100 parallel calls".
Answers: 1 integration (needs session semantics; could be unit if invariant logic extracted), 2 functional (drivers+ModelState+render), 3 all-DB matrix, 4 functional (Identity pipeline config) or integration, 5 unit — it's pure concurrency logic, and `ShellHostTests.cs` lives at unit level for a reason.

Interview angle: "how do you decide unit vs integration vs E2E?" — answer with the *session-boundary* rule and the *stringly-typed rendering* rule; they generalize to any stack ([08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q9).
