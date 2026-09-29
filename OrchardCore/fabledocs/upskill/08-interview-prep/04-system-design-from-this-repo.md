# System Design From This Repo

The exercise: **"Design a multi-tenant CMS platform"** — reverse-engineered from Orchard Core so that at every step you can say what a production system actually chose, why, and what you'd do differently. Practice at a whiteboard, 40 minutes, speaking aloud.

## Step 0 — Requirements (3 min)

Functional: multiple sites (tenants) per deployment; user-defined content types; draft/publish workflow; friendly URLs; roles/permissions incl. "own content"; scheduled publishing; import/export; theming.
Non-functional: tenant isolation (data + features), 4 SQL backends, horizontal scale (web farm), sub-second page renders, operable by small teams.

*Junior answers* jump to boxes. *Mid* states requirements and one explicit non-goal (e.g., "not designing global CDN delivery"). *Senior* also ranks: isolation and correctness of publish workflow are the two hills to die on.

## Step 1 — Tenancy model (8 min)

Options ladder: (a) row-level tenant_id, (b) schema/prefix per tenant, (c) DB per tenant, (d) process per tenant.
**What this repo chose**: (b)/(c) selectable per tenant, *plus* container-per-tenant in one process — settings in `tenants.json`, matched by host/prefix, DI container swapped per request ([`ModularTenantContainerMiddleware.cs:27-69`](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs), [`OrchardCoreBuilderExtensions.cs:104-109`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)).
Why: tenants differ in *enabled features*, not just data — that forces container-level isolation.
Cost you must name: memory per tenant, cold starts (mitigated by placeholders — [`ShellHost.cs:335-374`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)).
Simpler alternative when tenants are homogeneous: row-level tenancy + a feature-flag table. Say when you'd take it (SaaS with one product surface).

## Step 2 — Content model (8 min)

Options: rigid tables per type / EAV / JSON documents + query projections.
**Chosen**: documents + index tables, atomic together ([02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)). Versioning: version rows with `Latest`/`Published` flags; publish = transactional flag flip ([`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
Whiteboard the two tables (Document, ContentItemIndex) and narrate one publish (Trace 2 in [04-code-reading-gym/02](../04-code-reading-gym/02-trace-tables.md)).
Stronger/simpler alternatives: Postgres-only? JSONB + partial unique indexes gets you constraint-backed invariants the portable design can't have — a sharp senior observation.

## Step 3 — API + rendering (6 min)

Admin: server-rendered, per-field validation distributed to part owners (drivers). Public API: REST endpoints + GraphQL over the index tables with a whitelisted filter grammar and result caps ([`ContentItemsFieldType.cs:203-222`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)).
Design call to defend: **same permission for the same operation on every surface** (this is where you cite R1 as the failure mode).

## Step 4 — AuthZ (5 min)

Named permissions + implication chains; owner variants via resource-based checks; per-type dynamic permissions ([03-validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)). Draw the check path (Trace 3).

## Step 5 — Async work (5 min)

Scheduled publishing: cron task per tenant, distributed lock for farm-safety, idempotent predicate ([Flow 5](../01-codebase-cartography/05-key-flows.md)). Post-commit effects: deferred tasks; name the at-most-once property and when you'd upgrade to an outbox.

## Step 6 — Operations (5 min)

What exists (logs, health checks, audit trail) vs gaps (task metrics — R3); deploy story: auto-migrations at tenant start ⇒ expand/contract discipline ([05-06](../05-quality-engineering/06-observability-and-operations.md)).

---

## What junior / mid / senior sound like (summary)

| Step | Junior | Mid | Senior |
| --- | --- | --- | --- |
| Tenancy | "add tenant_id" | container-per-tenant with costs | selectable isolation per tenant tier; placeholder trick for scale |
| Content | one table per type | docs+indexes, versions as rows | constraint-vs-portability tension made explicit |
| API | CRUD endpoints | validation layers + uniform authz | operation-level permission seam so surfaces can't drift |
| Async | setTimeout/cron | locked cron + idempotent predicate | delivery-semantics ladder, chooses per effect |
| Ops | "add logging" | signals per flow | names what must be *built* vs what exists |

## Variation prompts (practice each as a 10-min delta)

1. **"Now add real-time collaborative editing."** Talk: version rows vs OT/CRDT granularity; draft locking as the cheap v1 (audit trail + optimistic concurrency already exist); where SignalR-per-tenant lives (careful: per-tenant hubs in per-tenant containers).
2. **"Now handle 10× tenants (10,000 sites)."** Placeholder startup already O(n)-cheap; the new costs: `tenants.json` as a single file, background service polling every tenant ([`ModularBackgroundService.cs:360-364`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)), memory per warm container → tenant eviction policy (release idle shells — the API exists: `ReleaseShellContextAsync` "can be used to free up resources after a given time of inactivity", [`ShellHost.cs:247-253`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)).
3. **"Now guarantee webhooks on publish."** The outbox conversation (Project 3 is your model answer).
4. **"Now make search work."** Index-on-publish via handlers → eventual consistency window → the "how stale is acceptable" negotiation; Lucene/Elastic modules exist as precedent.
5. **"Multi-region?"** Distributed lock provider and cache become region-pinned; tenants as the sharding unit (clean because of isolation); `tenants.json` → distributed settings store (Azure/Redis providers exist in-tree — `OrchardCore.Shells.Azure`).

Rubric for self-recording: did you (1) state requirements before boxes, (2) give one alternative per major choice with a *when-it-wins*, (3) name one failure mode per component unprompted, (4) finish with operations? That's the mid-level pass bar; (2)+(3) done crisply is the senior signal.
