# API and Data-Modeling Questions

12 cards. This is usually the make-or-break round for fullstack mid-level.

---

## Q1: "Design storage for user-defined content types (CMS/no-code style)."
Testing: schema flexibility strategies.
Repo anchor: JSON documents + projected index tables ([`ContentItemIndex.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs), [02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
Junior: EAV or "a JSON column". Mid: document store for shape freedom + index tables for queries, committed atomically; names the truncation/lossiness caveat. Senior: compares EAV / JSONB+GIN / dedicated tables-per-type; picks per query patterns; write amplification of indexes.

## Q2: "Model drafts + published versions + history."
Repo anchor: version rows with `Latest`/`Published` flags; publish transition ([`DefaultContentManager.cs:376-428`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
Junior: `is_draft` boolean on the row. Mid: separate version rows; the two-flag invariant; one-transaction flag flips. Senior: invariant is code-enforced — argues for/against a DB constraint; retention (version pruning feature exists for a reason); non-versionable types as the escape hatch ([`:168-171`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).

## Q3: "How do you page results correctly?"
Repo anchor: admin list pager ([`AdminController.cs:186-197`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)); GraphQL `first`/`skip` with default cap ([`ContentItemsFieldType.cs:203-222`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs)).
Mid: offset paging + total-count cost (`CountAsync`/`MaxPagedCount` tradeoff); default page caps on APIs. Senior: keyset/cursor pagination for deep pages and why offset breaks under concurrent writes; also the authz×paging interaction (post-filters under-fill pages — Flow 6).

## Q4: "Prevent IDOR in a multi-author system."
Repo anchor: resource-based authorize + owner-variant swap ([`ContentTypeAuthorizationHandler.cs:38-46, 88-96`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs)).
Junior: check user id in the query. Mid: centralized resource-based policy; the check binds identity to the row; every surface uses the same check. Senior: cites R1 as what drift looks like, and 404-vs-403 ordering as the existence-oracle subtlety ([`DeleteEndpoint.cs:36-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/DeleteEndpoint.cs)).

## Q5: "Where does validation belong in an API?"
Repo anchor: the five-layer table in [03-validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md); API error shape ([`CreateEndpoint.cs:88-93`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs) `ValidationProblem`).
Mid: binding → field rules → cross-record invariants → operation veto; failures roll back (`CancelAsync`). Senior: which layer owns *uniqueness* (and the TOCTOU consequence of app-level checks — R2).

## Q6: "Schema migrations: how do you ship a breaking change safely?"
Repo anchor: `UpdateFromN` ladder + portability comment ([`PublishLater/Migrations.cs:44-61`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)); auto-migrate at startup ([`OrchardCoreBuilderExtensions.cs:44`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)).
Mid: expand/contract; additive first; code tolerant of both shapes for one release. Senior: rollback reality (deploy rollback ≠ schema rollback), multi-provider constraints, migrations that can't use app services.

## Q7: "Design an import/bulk API."
Repo anchor: `ImportAsync` — 500-item batches, version-id dedupe ([`DefaultContentManager.cs:673-700`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)).
Mid: batching, idempotency by natural key, per-item results. Senior: restart semantics (retry-safe because dedupe), progress reporting seam (ticket M11), and transactional scope per batch vs per item.

## Q8: "Transactions in a web app: what's your default?"
Repo anchor: request = one transaction; commit at scope end; explicit cancel ([`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)).
Mid: unit-of-work per request; optimistic concurrency for conflicts. Senior: what escapes the transaction (external calls → deferred/outbox), isolation level as configuration, and long-request risks.

## Q9: "How do you test data-layer behavior?"
Repo anchor: [05-quality-engineering/01](../05-quality-engineering/01-testing-strategy.md) — the session-boundary rule; all-DB functional matrix.
Mid: don't mock the ORM's semantics; integration tests for commit/rollback; unit-test extracted logic. Senior: hermetic fixtures (per-fixture DB/data dir — functional README) and migration testing = run on every provider.

## Q10: "A write must trigger an email/webhook. Walk me through the reliability options."
Repo anchor: deferred tasks and their at-most-once semantics ([`ShellScope.cs:468-508`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/Scope/ShellScope.cs)); [03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).
Junior: send after saving. Mid: after-commit hook; names the crash window; idempotent consumer. Senior: the ladder — inline (same txn) / post-commit hook / outbox / CDC — with the rule "outbox when effects are non-idempotent external calls," citing that Orchard's route refresh is safe *because* it's a recomputation.

## Q11: "REST design: PUT vs PATCH vs POST for content updates?"
Repo anchor: `POST api/content` acts as upsert with merge semantics ([`CreateEndpoint.cs:55-104`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs), arrays-replace merge).
Mid: idempotency expectations per verb; documents the merge contract explicitly. Senior: critiques the anchor — upsert-POST is pragmatic but couples create/update authz paths (see R1) and complicates caching/idempotency; proposes PUT (full) + PATCH (merge-patch RFC 7386) split with `If-Match` concurrency.

## Q12: "Design cache invalidation for an API over mutable content."
Repo anchor: tag-based eviction from the publish handler ([`AutoroutePartHandler.cs:87-88, 199-202`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)); failover circuit breaker ([`DefaultDynamicCacheService.cs:93-135`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)).
Mid: write-path emits invalidations keyed by tags; TTL as backstop. Senior: staleness windows (post-commit eviction timing), fail-open vs fail-closed on cache outage, and ETag/validation caching at the HTTP tier as the complementary layer.

---

12/12 repo-anchored. These cards + the system-design walkthrough cover ~80% of observed mid-level API rounds.
