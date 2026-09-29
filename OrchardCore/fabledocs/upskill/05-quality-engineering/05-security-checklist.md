# Security Checklist

Each threat mapped to where this repo handles it (or where to verify). Then the pre-merge checklist.

## Threat map

| Threat | This repo's defense | Anchor | Residual/verify |
| --- | --- | --- | --- |
| **Broken authz / IDOR** | resource-based checks + owner variants + per-type dynamic permissions | [`ContentTypeAuthorizationHandler.cs:20-96`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Security/ContentTypeAuthorizationHandler.cs); per-item checks even in bulk ([`AdminController.cs:280-292`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)) | endpoints that omit the resource skip owner logic; **R1**: API update path can publish with `EditContent` only ([`CreateEndpoint.cs:95-127`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs)) — investigate |
| **Authn weaknesses** | Identity + lockout, rate-limit group, 2FA controllers, timing normalization | [`AccountController.cs:102-206`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs) | external login flows not reviewed here |
| **CSRF** | MVC antiforgery on admin forms (framework default); API routes explicitly `DisableAntiforgery()` **because** they use the `Api` token scheme, not cookies | [`CreateEndpoint.cs:26-33`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/CreateEndpoint.cs) | any endpoint that disables antiforgery *and* accepts cookie auth would be a finding — check new endpoints for this pairing |
| **Open redirect** | `Url.IsLocalUrl(returnUrl)` before redirecting | [`AdminController.cs:509-516`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs); login clears returnUrl for authenticated users ([`AccountController.cs:75-78`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)) | every new `returnUrl` param must repeat the check |
| **XSS** | Razor auto-encoding; Liquid rendering via Fluid with member-access allowlists (`MemberAccessStrategy.Register<...>` — [`Contents/Startup.cs:73-80`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Startup.cs)) | HTML-body fields (HtmlPart, Markdown) are by-design raw output — editor permission = HTML injection permission; sanitization module exists (`OrchardCore.Infrastructure` / sanitizer, not reviewed) |
| **SQL injection** | YesSql parameterizes; GraphQL where-clauses bind via `WithParameter` | [`ContentItemsFieldType.cs:190-198`](../../../src/OrchardCore/OrchardCore.ContentManagement.GraphQL/Queries/ContentItemsFieldType.cs) | any hand-built SQL via `IDbConnectionAccessor` needs review |
| **Tenant isolation** | container + store per tenant; table prefix option | [`OrchardCoreBuilderExtensions.cs:56-118`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs) | host-level singletons and shared caches are the crossing points; key caches per tenant |
| **Draft/data exposure** | published-only defaults everywhere (`VersionOptions.Published` default — [`DefaultContentManager.cs:109`](../../../src/OrchardCore/OrchardCore.ContentManagement/DefaultContentManager.cs)) | **R6**: GraphQL `status: DRAFT` gating not traced — verify before public exposure |
| **Secrets** | tenant connection strings in `App_Data` (not repo); DataProtection modules for key storage (Azure variants exist) | [`ShellSettings.cs:9-13`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs) | never log `ShellSettings` config values; `tenants.json` is sensitive |
| **Uploads/media** | dedicated Media module with antivirus hook (`OrchardCore.Antivirus`), S3/Azure backends | module dirs (not reviewed) | verify content-type sniffing + path traversal handling before citing |
| **User enumeration** | timing normalization + uniform errors | [`AccountController.cs:182-190`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs) | registration/reset flows not reviewed |
| **Recipes as code execution** | recipes evaluate `[js:...]` — import is a privileged operation | [`RecipeExecutor.cs:233-259`](../../../src/OrchardCore/OrchardCore.Recipes.Core/Services/RecipeExecutor.cs) | restrict who can run imports/deployments; treat recipe files like deploy scripts |
| **Existence oracles** | — | 404-before-authz on API delete ([`DeleteEndpoint.cs:36-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/DeleteEndpoint.cs)) reveals ids to any `AccessContentApi` holder | decide + document the intended contract |

Reporting note: this repo has a [SECURITY.md](../../../SECURITY.md) and asks for private disclosure (contact@orchardcore.net) — if you ever *confirm* R1/R6-class issues, that's the channel, not a public issue.

## Pre-merge security checklist (for your PRs here or anywhere)

- [ ] Every endpoint: which **auth scheme**, which **permission**, is the **resource** passed to the check?
- [ ] Separate permission for separate operations (edit ≠ publish ≠ delete) — same check on *every* surface (UI, API, GraphQL)
- [ ] `returnUrl`/redirect targets validated as local
- [ ] Antiforgery: disabled only where cookie auth is impossible
- [ ] New user input: encoded on output; if intentionally raw HTML, permission-gated and documented
- [ ] No new existence oracles (404 vs 403 ordering deliberate)
- [ ] Errors/logs don't leak connection strings, tokens, or other tenants' names
- [ ] Bulk operations authorize **per item**
- [ ] Anything cached: does the key include user/role/tenant contexts it varies by?
- [ ] Migrations/imports: no privilege escalation via recipe steps

Interview angle: "how do you review a PR for security?" — walk this checklist against Kata 3/4 in [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md); citing the timing-normalization and 404-ordering examples makes the answer concrete and memorable.
