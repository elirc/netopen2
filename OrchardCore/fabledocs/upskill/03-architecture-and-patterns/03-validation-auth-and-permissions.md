# Validation, Authentication, and Permissions

## The validation layers (all of them)

Input crosses up to five checkpoints before persisting. Knowing *which layer rejects what* is the difference between junior and mid answers.

| # | Layer | Where | Rejects |
| --- | --- | --- | --- |
| 1 | Model binding | MVC / minimal-API binder | unparseable input (wrong types) |
| 2 | Driver `UpdateAsync` | per part/field, e.g. [`TitlePartDisplayDriver.cs:49-64`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) | field-level rules (required title) → `ModelState` |
| 3 | Handler `ValidatingAsync` | domain events, e.g. [`AutoroutePartHandler.cs:115-133`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) | cross-record invariants (slug uniqueness) → `context.Fail` |
| 4 | Operation veto | Publishing/Creating handlers can `Cancel` — [`DefaultContentManager.cs:396-414`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs) | workflow-level "not now" |
| 5 | Persistence | optimistic concurrency (`checkConcurrency:true`), DB constraints (few) | lost updates |

The controller pattern gluing 1–4: update → `if (!ModelState.IsValid ...) { await _session.CancelAsync(); return View(model); }` — [`AdminController.cs:757-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs). Cancel-the-session is the load-bearing move: without it, invalid mutations from step 2–3 would still commit at scope end. **What a junior misses: validation failure without explicit cancel still persists.** What a senior checks first in any new endpoint here.

API variant: [`CreateEndpoint.cs:73-93`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs) — `ValidateAsync` → errors copied into ModelState → `session.CancelAsync()` → `ValidationProblem` (RFC-style error payload).

## Authentication (who are you)

ASP.NET Identity per tenant: `SignInManager`/`UserManager` over YesSql-backed stores. The login flow ([Flow 3](../01-codebase-cartography/05-key-flows.md)) adds:
- lockout on failure (`CheckPasswordSignInAsync(..., lockoutOnFailure: true)` — [`AccountController.cs:133`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs))
- endpoint rate limiting (`[RateLimitGroup(PasswordAuthentication)]`, `:105`)
- timing normalization against user enumeration (`:182-188`)
- a separate `"Api"` authentication scheme for API endpoints ([`CreateEndpoint.cs:33`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)) — cookie auth and token auth are distinct schemes; an API endpoint challenging the `Api` scheme won't silently accept a CSRF-able cookie session (`ChallengeOrForbid("Api")`, `:45-47`).

## Authorization (what may you do)

Three cooperating mechanisms:

**1. Named permissions with implication chains.**
[`CommonPermissions.cs:16-44`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs): `PublishContent` implies `EditContent` implies... — declared via the constructor's implied-by list. Roles grant permission sets; `IAuthorizationService.AuthorizeAsync(User, permission)` walks the chain.

**2. Owner variants (the anti-IDOR machinery).**
`EditOwnContent` etc. exist for every core permission (`OwnerPermissionsByName`, `:46-56`). When a check carries a **resource** (the content item), [`ContentTypeAuthorizationHandler.cs:38-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs) swaps in the Own-variant *iff* `user.FindFirstValue(NameIdentifier) == content.Owner` (`:88-96`). So "authors can edit their own posts" is not an if-statement in a controller — it's a policy the whole system inherits. **Resource-based authorization is the IDOR defense**: the check binds identity to the *specific row*, not just the route.

**3. Dynamic per-content-type permissions.**
The same handler converts `EditContent` into `EditContent_{ContentType}` (`:48-62`) so admins can grant "edit only BlogPosts." Permission templates registered per type — that's why `Permissions.cs` files exist per module.

The pattern every controller action follows: load item → `AuthorizeAsync(User, permission, item)` → 403/404 — e.g. [`AdminController.cs:431-443`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs). Note it returns `NotFound()` *before* authorization for missing items and `Forbid()` after — leaking existence of items to unauthorized users is accepted here (admin context); public APIs often prefer uniform 404. Have an opinion ready.

## Tenant isolation

The strongest isolation is structural: separate DI containers, separate databases or table prefixes ([`OrchardCoreBuilderExtensions.cs:104-109`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)). Cross-tenant IDOR in the classic sense (guess another tenant's id) mostly *can't happen through the ORM* because the session is constructed against the tenant's own store. The residual risks are host-level services and anything sharing a database with prefixes — see [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| Demands `EditOwnContent` directly (the file's own comment says don't — [`CommonPermissions.cs:11-14`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs)) | Demands the main permission **with the resource**, lets the handler downgrade to Own |
| Checks permission for the page, forgets per-item checks in bulk actions | Bulk path authorizes *each* item ([`AdminController.cs:280-292`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) |
| Assumes validation == ModelState | Knows layers 3–4 exist and that cancel-the-session is mandatory |
| One auth scheme for everything | Separate `Api` scheme; antiforgery deliberately disabled only where token auth replaces it |
| "Publish is just a save" | Publish is a separately-permissioned operation (`PublishContent` ≠ `EditContent`) — and hunts for paths where that separation slipped (see risk register R1) |

## Drill

The API `CreateEndpoint` update path authorizes `EditContent` then may call `PublishAsync` when `draft=false` ([`CreateEndpoint.cs:95-127`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)) — while `AdminController` demands `PublishContent` before publishing ([`AdminController.cs:876-879`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)). Trace both paths yourself and either (a) find the mitigating check we didn't, or (b) write the two-line fix and the test proving it. Self-grade — Strong: you can state precisely which principal (has EditContent, lacks PublishContent) and which request (`POST api/content` with existing id, no `draft` flag) exercises the gap, and you label your conclusion as needing runtime confirmation.

Interview angle: "Design authorization for a multi-author CMS" → this file is the full answer arc: named permissions → implication → resource-based owner checks → per-type dynamic grants. Cross-links: [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q4/Q5, [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) review round 2.
