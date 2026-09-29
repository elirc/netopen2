# Mid-Level Feature Tickets

12 cross-layer tickets. **Rule: write a half-page design note before coding** (data → API → UI → tests → risk → rollback) and self-review it against the pattern catalog. M1 is the full-template model; others condensed. Each includes Risk and Rollback because mid-level = "can change systems safely," not "can write features."

---

## Ticket M1: Add `PublishContent` check to the API update-publish path (risk R1)

Difficulty: Medium — Estimated time: 1–2 days
Skills practiced: authorization, API contracts, regression testing, security writeups.
Story: As a site owner, I want API clients with edit-only rights to be unable to publish, so that the publish permission means the same thing on every surface.
Why this is a good contribution: closes a real authz asymmetry ([`CreateEndpoint.cs:95-127`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs) vs [`AdminController.cs:876-879`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). **Process note: verify exploitability first, then follow SECURITY.md private disclosure if confirmed** — the PR may need to be coordinated, not public. That judgment is part of the exercise.
Acceptance criteria:
- [ ] `POST api/content` with existing item + `draft=false` demands `PublishContent` on the item
- [ ] `draft=true` still requires only `EditContent`
- [ ] Create branch behavior unchanged (already demands `PublishContent`, `:67-70`)
- [ ] Integration tests: editor-without-publish → 403 on publish, 200 on draft
Read these anchors first: `CreateEndpoint.cs` whole file; `CommonPermissions.cs:16-22` (implication direction!).
Files likely touched: `CreateEndpoint.cs`, new test file under the API tests.
Implementation plan: 1. repro test proving current behavior; 2. add check before the `if (!draft) PublishAsync` branch; 3. release-note wording.
What could go wrong: breaking legitimate clients whose tokens hold `PublishContent` via implication — verify implication still passes (it does: PublishContent ⇒ EditContent, so publish-capable clients pass both checks).
Suggested checks: run the Contents API functional tests; grep other endpoints for the same pattern.
Review questions: is 403 vs ValidationProblem the right failure shape? does the error leak draft existence?
Rollback: revert commit; no data shape changes.
Interview story potential: "found and fixed a cross-surface authorization inconsistency, with the disclosure-process judgment call."

---

## M2 — Batch + poison-item isolation in scheduled publishing (R4)
Medium, 2–3 days. Page the due-item query (batch 500, mirroring [`DefaultContentManager.cs:21`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)), per-item try/catch that logs and continues, counter in the completion log. Risk: changed failure semantics (previously all-or-nothing per tick) — document in PR. Rollback: revert; no schema change. Test: poison item (handler that throws for a marker title) + assert others published. Story: "made a batch job partial-failure tolerant."

## M3 — Slug collision telemetry + dedupe report (R2 phase 1)
Medium, 2–3 days. Build on T1: add an admin Diagnostics page listing duplicate `AutoroutePartIndex` paths (`GROUP BY Path HAVING COUNT > 1` via `QueryIndex`). No behavior change to routing. Risk: query cost on huge sites — paginate. Story: "instrumented a suspected race before proposing the fix" — *this ordering is the senior signal*.

## M4 — `UnpublishLater` feature (done right)
Medium-Hard, 3–5 days. The Kata-1 feature, implemented properly: own part, own index + migration (three-column pattern from [`PublishLater/Migrations.cs:24-39`](../../../src/OrchardCore.Modules/OrchardCore.PublishLater/Migrations.cs)), own background task with `[BackgroundTask]` schedule, driver + placement + views, feature entry in a Manifest. Design note must answer: same module or new? (Upstream chose: check if `OrchardCore.PublishLater` already has it — if so, build it as a private exercise.) Full vertical slice — the best single practice ticket in this file.

## M5 — Publish-event webhook module (deferred-task edition)
Medium-Hard, 3–5 days. New module: on content publish, POST JSON to configured URLs. Design decisions the note must cover: deferred task vs outbox (start deferred; document at-most-once — [03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)); secrets storage for signing key; SSRF guard on admin-entered URLs (allowlist/deny-private-ranges); timeout + no retry v1. Risk: outbound calls from request-adjacent scopes — must be deferred, never inline. Story: "designed delivery semantics explicitly and wrote them down."

## M6 — Per-type default sort for admin content list
Medium, 2–3 days. Content-type setting (via a settings driver on the type editor — pattern: existing `*Settings` drivers in ContentTypes module) that sets the default `ContentsOrder` in [`AdminController.cs:152-158`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs). Cross-layer: settings model → editor driver → list query. Risk: interacting with saved user filters — define precedence.

## M7 — Health check for background task staleness (R3 phase 1)
Medium, 2–4 days. An `IHealthCheck` (HealthChecks module pattern) reporting unhealthy when a `[BackgroundTask]`-scheduled task hasn't succeeded within N× its cron interval. Requires recording last-success — an `IBackgroundTaskEventHandler` writing a small document per task ([`ModularBackgroundService.cs:202-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs) seam). Risk: write-per-tick churn — throttle to state *changes*.

## M8 — GraphQL: draft-status permission gate audit + test (R6)
Medium, 2–3 days. Trace `status: DRAFT` authorization end-to-end ([`ContentItemsFieldType.cs:69-76, 100-121`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs) → `IGraphQLFilter` implementations in the GraphQL module). Deliverable: either the missing gate + test, or a test *proving* the existing gate + doc note. "Prove it's safe" tickets are underrated interview stories.

## M9 — Content version pruning: dry-run mode
Medium, 2–3 days. The VersionPruning feature ([manifest](../../../src/OrchardCore.Modules/OrchardCore.Contents/Manifest.cs), `VersionPruning/` dir) deletes old versions; add a dry-run setting that logs what *would* be pruned. Read the existing background task first. Risk: none (additive flag). Trains: safe-by-default destructive operations.

## M10 — Admin list saved filters
Medium-Hard, 4–6 days. Persist named filter queries (the `q` filter string, [`AdminController.cs:177-184`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) per user. Data: user-properties document. UI: dropdown in the filter bar (ContentOptions display driver). Risk: leaking filter names between users — key by user id; test that.

## M11 — Import: per-batch progress events
Medium, 2–3 days. `ImportAsync` processes 500-item batches silently ([`DefaultContentManager.cs:673-700`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)); add an event/callback seam reporting batch completion for UI progress. Contract design: new interface in Abstractions (versioning discipline! additive only). Risk: Abstractions surface change — smallest possible interface.

## M12 — Rate-limit group for content API endpoints
Medium, 2–3 days. Apply the `RateLimitGroup` pattern from login ([`AccountController.cs:105`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)) to the anonymous-routed API endpoints (`api/content`). Design note: which limits, per-user vs per-IP, and the failure shape (429 + Retry-After). Risk: throttling legitimate imports — separate group for authenticated bulk clients.

---

Self-review rubric for your design notes — Basic: layers listed. Solid: risks with mechanisms, rollback stated, one alternative considered. Strong: names the repo pattern each part follows and what a maintainer would push back on first.
