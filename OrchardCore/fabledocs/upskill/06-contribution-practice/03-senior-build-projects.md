# Senior Build Projects

Six projects, 2 days–4 weeks. Each is scoped to be *plausibly acceptable* upstream (coordinate via issue/RFC first — see [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) or a portfolio piece on a fork. Format per project: problem → value → architecture decisions → likely files → migration/test/security/performance/rollout plans → open questions → stretch.

---

## Project 1: Slug uniqueness hardening (R2, end-to-end)
**Problem**: path uniqueness is check-then-write ([`AutoroutePartHandler.cs:465-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)); concurrent publishes can collide. **Value**: correctness guarantee for every Orchard site.
**Architecture decisions**: (a) DB unique index vs distributed lock vs serializable txn — chosen: unique index on `AutoroutePartIndex.Path` scoped rows `Published || Latest`… except flags make partial-unique indexes provider-dependent → *decision point*: normalized "active path" table vs per-provider conditional index. (b) Violation → retry with existing `-N` suffix logic, bounded attempts. (c) Case sensitivity per provider — define canonical casing at write.
**Likely files**: Autoroute Core indexes + migrations, `AutoroutePartHandler`, YesSql migration per provider, tests.
**Migration plan**: release 1 = M3 telemetry + dedupe migration (rewrite colliding paths with suffixes, audit-trail entries); release 2 = constraint + retry. Never constraint-first (existing dupes would brick migration).
**Test plan**: parallel-publish integration test (two sessions); all-DB matrix mandatory; dedupe migration test with seeded collisions.
**Security plan**: none new. **Performance plan**: index adds write cost per autoroute save — benchmark vs the existing 4-variant `IsIn` query it can partially replace.
**Rollout/rollback**: feature-flag the constraint check path; rollback = drop index (additive-only rule bends here — document provider caveats).
**Open questions**: contained-item relative paths (same treatment?); multi-culture path uniqueness scope.
**Stretch**: collision metrics dashboard.
Interview story potential: full expand/contract migration narrative under a four-database portability constraint — a *complete* senior system-design story.

## Project 2: Background task observability + retry policy (R3)
**Problem**: task failures are log-only; no retry, no aggregation ([`ModularBackgroundService.cs:222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). **Value**: operability for every deployment.
**Architecture decisions**: state store = per-tenant document (last success/failure/duration per task) written by an `IBackgroundTaskEventHandler` — kernel untouched; retry = opt-in per `[BackgroundTask]` attribute (max attempts, backoff) implemented *in the event-handler layer* vs kernel scheduler — decision: kernel-adjacent library, RFC required.
**Likely files**: new module `OrchardCore.BackgroundTasks.Monitoring` (admin UI listing task health), Abstractions addition (attribute properties — additive).
**Test plan**: unit (scheduler math), integration (failing task → recorded), functional (admin page).
**Security**: admin-permission-gated UI. **Performance**: state writes throttled to transitions.
**Rollout**: pure feature module; rollback = disable feature.
**Open questions**: farm-wide aggregation (per-instance vs shared store); retry + distributed lock interaction (retry must re-acquire).
**Stretch**: OpenTelemetry metrics emission.

## Project 3: Transactional outbox module
**Problem**: deferred tasks are at-most-once ([`ShellScope.cs:468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)); integrations (webhooks/email) need at-least-once. **Value**: a reusable reliability primitive the ecosystem lacks.
**Architecture decisions**: outbox = YesSql documents + index (due/attempts/status) written *in the same session* as the triggering change; drain = `IBackgroundTask` with distributed lock (both patterns exist — compose them); handler contract = named payload handlers (mirror recipe step dispatch, [`RecipeExecutor.cs:181-184`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs)); idempotency keys for consumers.
**Likely files**: new module + Abstractions; M5's webhook module becomes the first consumer.
**Migration**: new tables only. **Test**: crash-between-commit-and-dispatch simulation; duplicate-delivery consumer test.
**Security**: payloads may contain content — encrypt-at-rest option; SSRF rules from M5.
**Performance**: poll frequency vs latency; batch dispatch.
**Rollout**: standalone feature; rollback = disable (undelivered rows persist — document).
**Open questions**: ordering guarantees (per-key FIFO?); retention/pruning policy (steal VersionPruning's approach).
**Stretch**: exactly-once-effect via consumer dedupe store.
Interview story: *the* outbox story every senior loop wants, implemented rather than recited.

## Project 4: GraphQL cost control & authz audit (R6 grown up)
**Problem**: draft gating unverified; no query cost limits traced. **Value**: makes the GraphQL surface safe to expose publicly.
**Decisions**: complexity scoring (depth × fields) vs result-cap-only; per-permission field visibility (draft/status arg hidden without permission — schema-level vs resolver-level enforcement); persisted-query allowlist option.
**Files**: `OrchardCore.Apis.GraphQL` options + validation rules; `ContentItemsFieldType` ([`:69-76`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)).
**Tests**: authz matrix (anon/editor/publisher × published/draft); abusive query corpus.
**Rollout**: default-off validation rules first release. **Open questions**: back-compat for existing clients querying drafts legitimately.

## Project 5: Tenant cold-start budget
**Problem**: first request per tenant pays container+pipeline build (Flow 1); farms with many tenants see post-deploy latency storms. **Value**: measurable p99 improvement.
**Decisions**: warmup strategies (background pre-build of top-N tenants by traffic vs all — `ShellWarmup` exists as a hint: [`ModularBackgroundService.cs:52-63`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)); measure first: instrument container build time per tenant (BenchmarkDotNet + production histogram).
**Files**: kernel-adjacent warmup host service; diagnostics.
**Test**: benchmark suite in `test/OrchardCore.Benchmarks` following existing style.
**Open questions**: memory ceiling vs warm set size — an explicit cost curve, which is the deliverable.
Interview story: "measure → model → mitigate" performance arc with real constraints.

## Project 6: End-to-end multi-tenant test kit
**Problem**: cross-tenant isolation (the crown property) lacks a dedicated test suite a contributor can run in minutes. **Value**: regression net for the scariest bug class.
**Decisions**: build on the functional suite's fixture isolation ([README](../../../test/OrchardCore.Tests.Functional/README.md)); scenario matrix: content isolation, user isolation, feature isolation, cache isolation (DynamicCache keys), background-task isolation.
**Files**: new functional test class(es) + helpers.
**Test plan is the project.** Security: this *is* security work. Rollout: tests only — the ideal first big PR because rejection risk is style, not design.
**Stretch**: table-prefix shared-DB variant of every scenario (the weaker isolation mode deserves the stronger tests).

---

Choosing guidance: portfolio depth → Project 3; upstream acceptance likelihood → Project 6 then 2; interview-story density per day → Project 1.
