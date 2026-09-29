# Observability and Operations

The organizing question: **"How would I know this broke?"** — asked per flow, answered with what exists and what's missing.

## What exists

| Facility | Where | Notes |
| --- | --- | --- |
| Structured logging | NLog host-wide ([`Program.cs:5`](../../../src/OrchardCore.Cms.Web/Program.cs) `UseNLogHost()`); `ILogger<T>` everywhere with tenant/task context in messages (e.g. [`ModularBackgroundService.cs:209-224`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)) | log files under `App_Data/logs/` |
| Log-level guards | `if (_logger.IsEnabled(LogLevel.Debug))` before interpolation ([`ShellHost.cs:337-340`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)) | perf-conscious logging idiom worth copying |
| Health checks | `OrchardCore.HealthChecks` module (dir verified) | HTTP liveness per tenant |
| Profiling | `OrchardCore.MiniProfiler` module | per-request SQL/step timings on dev |
| Audit trail | `OrchardCore.AuditTrail` module + Contents integration ([`Contents/AuditTrail/`](../../../src/OrchardCore.Modules/OrchardCore.Contents/AuditTrail)) | who did what to which content |
| Error surfaces | `UseExceptionHandler("/Error")` in production ([`Program.cs:13-16`](../../../src/OrchardCore.Cms.Web/Program.cs)); scope exception handlers cancel the session ([`OrchardCoreBuilderExtensions.cs:164-169`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)) | request errors also reach scope handlers via the middleware ([`ModularTenantContainerMiddleware.cs:62-66`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs)) |
| Diagnostics UI | `OrchardCore.Diagnostics` module | error pages |

Not found in the kernel: metrics/tracing emission (no OpenTelemetry wiring observed in files read — the Aspire host may add some; not verified). Assume **logs are the only signal** unless a deployment adds APM.

## "How would I know this broke?" per key flow

| Flow | Breakage | Signal today | Gap |
| --- | --- | --- | --- |
| 1 Tenant routing | tenant 503s or unmatched host | 503s in access logs; "initializing" message | no per-tenant request metrics; unmatched hosts fail silently — add a catch-all log if you operate this |
| 2 Publish | validation storm / concurrency exceptions | error notifications (user-facing), exception logs | no rate metric; TempData-carried ModelState ([`AdminController.cs:853-864`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) surfaces bulk errors to the *user*, not to ops |
| 3 Login | credential stuffing / lockout waves | `LoggingInFailedAsync` events exist as a **seam** — but only if a module subscribes; rate limiter rejections | wire an event handler → metrics; that's a one-file contribution ([06-contribution-practice/01](../06-contribution-practice/01-good-first-tickets.md) T9) |
| 4 Autoroute | duplicate slugs | user reports wrong page | log + metric on collision-suffixing would make R2 measurable |
| 5 Scheduled publish | task fails every tick | `LogError` per failure ([`ModularBackgroundService.cs:222-228`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)) | nothing aggregates "task X hasn't succeeded in N hours" — the flagship observability gap (R3) |
| 6 GraphQL | slow/abusive queries | none specific observed | result cap exists; no query timing/complexity logging traced |
| 8 DynamicCache | Redis down | `LogError` + 30s failover ([`DefaultDynamicCacheService.cs:120-129`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)) | failover state itself isn't exported — origin load spike is your alarm |

## Deploy and rollback story

- App deploy = standard ASP.NET Core (container image via [`Dockerfile`](../../../Dockerfile)); tenants re-initialize on first request; `ShellWarmup` option pre-warms for background processing.
- **Migrations run automatically at tenant startup** (`AutomaticDataMigrations` — [`OrchardCoreBuilderExtensions.cs:44`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)). Consequence: rolling back a deploy does **not** roll back schema — migrations must be backward-compatible one version (expand/contract), which the repo's additive-only migration style ([`PublishLater/Migrations.cs:44-61`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)) supports.
- Config rollback: tenant settings are files (`App_Data`) + DB documents — snapshot both; deployment plans (export) are the supported "config as artifact" path.

## The operational drill

You operate a 200-tenant Orchard farm. Write the four alerts you'd define first, each with its signal source. Model answer: (1) rate of 5xx per tenant (access logs), (2) background task error-log rate > 0 for 3 ticks (log query — because no metric exists), (3) cache failover log lines (leading indicator of origin overload), (4) login failure spike per tenant (requires adding the `ILoginFormEvent` metrics handler — admit the gap). Strong answers distinguish *signals that exist* from *signals you must build*, which is exactly the R3 critique.

Interview angle: "how do you make a system observable?" — the honest gap analysis above is a better answer than reciting logs/metrics/traces; it shows you audit before you instrument. Cross-link: [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) step 6.
