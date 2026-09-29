# Data Model and Persistence

## The core idea: documents + index tables

YesSql stores each entity as a **JSON document** in a `Document` table, and maintains **index tables** — plain SQL tables projected from documents by `IndexProvider`s — for querying. You get schema-flexible writes *and* relational reads.

- The projection: [`ContentItemIndex.cs:28-73`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs) — `Describe(...).Map(contentItem => new ContentItemIndex { ... })`. Every save re-maps affected index rows in the same transaction.
- The query: `session.Query<ContentItem, ContentItemIndex>().Where(x => x.ContentItemId == id && x.Latest)` — [`DefaultContentManager.cs:113-118`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs). The lambda targets index columns; the document is rehydrated from JSON after the index row matches.
- `QueryIndex<T>` when you need only the projection, no documents: [`ScheduledPublishingBackgroundTask.cs:29-32`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Services/ScheduledPublishingBackgroundTask.cs) — cheaper, and the right default for existence/due-date scans.

Consistency property worth quoting: **documents and their index rows commit atomically** (same session/transaction), so an index can't reference a missing document — but the index *content* can be lossy (255-char truncation, [`ContentItemIndex.cs:50-68`](../../../src/OrchardCore/OrchardCore.ContentManagement/Records/ContentItemIndex.cs)).

## Key entities and relationships

| Entity | Identity | Versioning | Index tables (found) |
| --- | --- | --- | --- |
| ContentItem | `ContentItemId` (26-char generated) | rows per version, `Latest`/`Published` flags | `ContentItemIndex`, plus per-part indexes (`AutoroutePartIndex`, `PublishLaterPartIndex`, …) |
| User | ASP.NET Identity over YesSql (`OrchardCore.Users`) | none | `UserIndex` etc. (module dir listed, not read) |
| Site settings | singleton document via `ISiteService` | none | — |
| Content definitions | documents via `IContentDefinitionManager` (DB or file store — [`DatabaseContentDefinitionStore.cs`](../../../src/OrchardCore/OrchardCore.ContentManagement/DatabaseContentDefinitionStore.cs) / `FileContentDefinitionStore.cs`) | — | — |

The "join" idiom: there are no foreign keys between documents. References are stored ids (`ContentItemId` strings), resolved by additional queries — or denormalized into an index designed for the read (Autoroute stores container *and* contained item ids per path row: [`AutoroutePartHandler.cs:465-477`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)).

## Transactions and the unit of work

One `ISession` per shell scope; **commit happens once, at scope disposal** — [`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs); an exception registered handler calls `CancelAsync` instead. Consequences:

1. "Rollback" = `await _session.CancelAsync()` — see the validation-failure path in [`AdminController.cs:764-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs) and bulk actions (`:283-292`).
2. Everything a request writes commits **together** — a request is transactional by default. Rare and valuable.
3. Anything that must happen *only after* commit (cache invalidation, search indexing, webhooks) must be a **deferred task** ([04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md)).
4. Optimistic concurrency is opt-in per save: `_session.SaveAsync(contentItem, checkConcurrency: true)` ([`DefaultContentManager.cs:423`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) — version column check at flush; loser gets a concurrency exception rather than lost-update.

Isolation is configurable (`YesSqlOptions.IsolationLevel` passed to each provider — [`OrchardCoreBuilderExtensions.cs:74-98`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)); default not chased here — check YesSql docs before relying on read semantics.

## Migrations: how schema changes ship

Per-module `DataMigration` classes with `CreateAsync` (fresh installs) and `UpdateFromNAsync` (upgrades), each returning the next version number. The **best specimen** is PublishLater: [`Migrations.cs:18-89`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs) —

- `CreateAsync` builds the index table + composite DB index and returns 3 (fresh installs jump straight to final shape).
- `UpdateFrom1Async`/`UpdateFrom2Async` replay history for existing sites; note the comment at lines 47-48: a column is *kept* because "dropping an index and altering a column don't work on all providers" — cross-database portability constrains migrations, a very interview-worthy detail.
- Migrations also alter *content definitions* (data, not schema): `AlterPartDefinitionAsync` ([`Contents/Migrations.cs:16-23`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Migrations.cs)).

Migrations run automatically per tenant at startup (`IModularTenantEvents, AutomaticDataMigrations` — [`OrchardCoreBuilderExtensions.cs:44`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)).

## How to change the schema safely here (checklist)

1. Add/modify the `MapIndex` class and its provider mapping.
2. Write `UpdateFromNAsync` that *adds* — avoid destructive alters (4-provider portability). Fold the final shape into `CreateAsync` and bump its return.
3. Additive first, remove later (two-release deprecation) — index consumers may exist in other modules.
4. New query patterns → add a DB index (`CreateIndex`) covering `DocumentId` + filter columns; copy the composite style at [`PublishLater/Migrations.cs:31-39`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs).
5. Existing documents don't rewrite themselves: index rows refresh on next save, or ship a migration that re-saves affected items. Know which you need.
6. Test against SQLite at minimum; CI runs functional tests on all four providers (`functional_all_db.yml`).

## What juniors miss vs seniors check

- Junior: treats the index as the data. Truncated/missing index values ≠ document truth.
- Senior: asks "which reads hit documents vs indexes?", "is this invariant schema-enforced or code-enforced?" (here: almost all code-enforced — no unique constraint on slugs or Published flags), and "what's the write amplification of a new index on a hot type?" (every content save re-maps every registered index for it).

## Drill

Design the index + migration for "find all content items expiring in the next hour" (an `ExpiresLater` feature). Write the `MapIndex` class, the provider predicate, the migration, and the background-task query. Compare structurally against PublishLater. Self-grade — Strong: your DB index covers the exact background query; your provider maps only for items *having* the part (see how `PublishLaterPartIndexProvider` would need a null return — check [`Indexes/PublishLaterPartIndexProvider.cs`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Indexes/PublishLaterPartIndexProvider.cs)).

Interview angle: "Why would you pair a document store with SQL tables?" and "How do drafts/versions live in one table?" → [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q1–Q3.
