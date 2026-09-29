# Refactor and Design Katas (Senior)

Eight katas. No PRs required — the artifact is a design note, a diff sketch, or an RFC. Self-grading criteria after each. Time-box: 45–90 minutes each.

---

## K1 — Fix the boundary leak: `IUpdateModelAccessor` in `DefaultContentManager`
Task: design (don't implement) removing MVC `ModelState` knowledge from the domain ([`DefaultContentManager.cs:398-414`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)): e.g. `PublishAsync` returns a result object with cancellation reasons; web layer translates to ModelState.
Self-grade — Basic: result-object sketch. Solid: inventories all callers (GraphQL, API endpoints, workflows) and the Abstractions-breaking-change problem. Strong: staged plan (new overload returning result, old method delegates + obsolete, remove in vNext) and honest verdict on whether the churn is worth it (defensible: no).

## K2 — Design the outbox (paper edition of Project 3)
Task: one-page RFC: table shape, write path (same session — cite [`OrchardCoreBuilderExtensions.cs:152-163`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)), drain loop (compose [`ModularBackgroundService`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) patterns), delivery semantics, poison handling, pruning.
Self-grade — Strong: states *invariants* ("a committed trigger row implies an eventual dispatch attempt"), not just components; includes the consumer idempotency contract.

## K3 — Split a module: extract `OrchardCore.Contents.Api`
Task: the minimal-API endpoints ([`Endpoints/Api/`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api)) as a separate *feature* (not project) with its own manifest entry, permission set, and rate limits (M12).
Self-grade — Solid: feature dependency graph correct ([`Manifest.cs`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs) syntax). Strong: back-compat story for sites with the endpoints already enabled (feature auto-enable migration? document the trap).

## K4 — Remove duplication kata: the six notifier blocks
Task: actually implement T2 locally; then write the paragraph explaining why the obvious `string.Format` helper is wrong (localizer extraction needs literal format strings in `H[...]`) and what shape survives that constraint.
Self-grade — Strong: your final helper keeps every user-visible string literal and greppable; diff is boring.

## K5 — Improve type safety: shape names
Task: design compile-time-checked shape references (constants class? source generator? nameof-conventions?) for the stringly-typed rendering contract (risk R5) without breaking theme template resolution.
Self-grade — Basic: constants file. Solid: analyzes why constants don't help templates (the other half of the contract lives in `.cshtml`/Liquid file *names*). Strong: proposes an analyzer/test that walks placement.json + Views folders and fails CI on orphaned shape names — checking the *seam*, not decorating the code.

## K6 — Design a migration: `ContentItemIndex.DisplayText` widening (Kata 8 done right)
Task: full expand/contract plan for wider display text: new column `DisplayTextLong` (expand), dual-write via index provider, backfill strategy (re-save vs SQL per provider), reader switch, contract (drop old — or never, per portability rules).
Self-grade — Strong: names what happens on each of the four providers and where the plan dies on one of them (that discovery is the point).

## K7 — Reduce N+1: admin list count
Task: eliminate/soften `CountAsync` on huge sites ([`AdminController.cs:186-197`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)): options — `MaxPagedCount` default, approximate counts, "page N of many", filtered-index counts.
Self-grade — Solid: measures first (what % of page time is count on 2M rows — how would you find out?). Strong: UX tradeoff stated (losing exact totals) and per-provider count performance differences acknowledged.

## K8 — Review a flawed PR, senior edition
Task: Kata 6 from [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md) (the static-lock slug fix), but your deliverable is the *full review*: what you'd write, including the better design, a offer to pair, and the maintainer-tone rejection that keeps the contributor coming back.
Self-grade — Strong: your review teaches the TOCTOU/multi-instance reasoning in ≤10 sentences and proposes the staged fix (telemetry → dedupe → constraint) as a shared roadmap, not a dunk.

---

Meta-drill: after any three katas, write the one-line lesson each taught you about *this* repo and the one-line transferable lesson. If they're the same line, redo the kata.
