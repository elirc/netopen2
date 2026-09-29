# Frontend / Rendering Questions

This repo's UI is **server-rendered Razor + the shape system**, with per-module JS (some Vue) compiled by the assets pipeline. That's an asset in interviews: composition, state, and caching questions are framework-independent, and you can contrast the server-side answer with the React answer you already know. 10 cards.

---

## Q1: "How do you compose a page from components owned by different teams/plugins?"
Round: frontend/design. Testing: composition models.
Repo anchor: drivers → shapes → zones/placement ([`TitlePartDisplayDriver.cs:29-35`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs) `.Location("Detail", "Header")`, [`placement.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/placement.json)).
Junior: components import each other. Mid: inversion — contributors *declare* fragments + positions; a host composes; React analog: slot/portal patterns, plugin registries. Senior: the contract question — what's compiler-checked vs convention (shape names aren't; props are in React), and how overrides work (theme template beats module template = specificity system).

## Q2: "Explain template overriding / theming without forking."
Repo anchor: alternates ([`ContentItemDisplayManager.cs:75-83`](../../../src/OrchardCore/OrchardCore.ContentManagement.Display/ContentItemDisplayManager.cs)): `Content_Summary__BlogPost` falls back to `Content_Summary`.
Mid: most-specific-template-wins resolution chains. Senior: names the cost — render-time resolution failures instead of compile-time (risk R5) — and the mitigation (functional tests over key pages).

## Q3: "Where should form validation live?"
Repo anchor: driver `UpdateAsync` adds ModelState errors server-side ([`TitlePartDisplayDriver.cs:53-57`](../../../src/OrchardCore.Modules/OrchardCore.Title/Drivers/TitlePartDisplayDriver.cs)).
Junior: client-side for UX. Mid: **both**, server is authoritative; client mirrors for latency. Senior: single-source-of-truth strategies (validation metadata driving both sides) and the multi-writer wrinkle here: any part's server error rejects the whole form — partial-failure UX must be designed.

## Q4: "How do you cache rendered UI safely?"
Repo anchor: DynamicCache contexts/discriminators ([`DefaultDynamicCacheService.cs:137-150`](../../../src/OrchardCore.Modules/OrchardCore.DynamicCache/Services/DefaultDynamicCacheService.cs)).
Mid: key must include everything output varies by (user/role/culture/route) — under-keying = cross-user leak. Senior: tag-based invalidation from the write path ([`AutoroutePartHandler.cs:199-202`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs)); React analog: SWR/React-Query cache keys + invalidation — same discipline, different tier.

## Q5: "Multiple submit buttons, one form — how do you route the action?"
Repo anchor: `[FormValueRequired("submit.Publish")]` ([`AdminController.cs:473-479`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)).
Junior: JS sets a hidden field. Mid: button `name`/`value` posts; server dispatches — works without JS (progressive enhancement). Senior: contract risk (button names are stringly-typed API), and idempotency of each handler under double-click (+ the antiforgery/PRG pattern the controller uses — redirect after POST).

## Q6: "How does the asset pipeline serve per-module frontend code?"
Repo anchor: Yarn workspaces per module `Assets/` → assets-manager → `wwwroot/` ([`package.json`](../../../package.json)).
Mid: build-time compilation, content-addressed/cached static files, never edit outputs. Senior: tradeoff vs central bundle — module autonomy vs duplicate dependencies and admin-page bundle weight; measurement first ([05-04](../05-quality-engineering/04-performance-thinking.md) domain table).

## Q7: "Server-rendered vs SPA vs islands — how do you choose?" (conceptual — no direct anchor)
Mid: data-freshness, SEO, team skills, interactivity density; this CMS: server-rendered pages + JS islands for editors — matching interactivity where it lives. Senior: names the *state duplication* cost of SPAs over CRUD (every entity shape twice) and when RSC/islands models reduce it. Tie back: Orchard's admin uses server pages + targeted scripts — the original islands.

## Q8: "How do you keep UI in sync with authorization?"
Repo anchor: creatable-types menu filtered per permission ([`AdminController.cs:790-812`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)).
Junior: hide buttons. Mid: UI filtering is UX, server checks are security — both, always (the repo does both: menu filter + per-action authorize). Senior: capability-driven UI (server sends allowed actions with the resource) to avoid drift — and where this repo would put that (the SummaryAdmin shape's action buttons).

## Q9: "A page renders wrong for exactly one user. Debug it."
Repo anchor: cache discriminators (Q4) — likely a cache context missing role/culture; also Liquid `MemberAccessStrategy` allowlists ([`Contents/Startup.cs:73-80`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Startup.cs)).
Mid: bisect: with cache off? with their role? Senior: instrument the cache key components; classic answer structure from [05-03](../05-quality-engineering/03-systematic-debugging.md) Scenario 3.

## Q10: "XSS: where's the line between encoding and sanitizing?"
Repo anchor: Razor auto-encodes; HTML-body parts are raw by design; Fluid member allowlist limits template reach ([`Contents/Startup.cs:73-80`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Startup.cs)).
Junior: "escape everything." Mid: encode-on-output by default; sanitize only where HTML input is a feature; permission-gate raw HTML authoring. Senior: threat-models the *template* layer too (who can edit Liquid patterns? — [`AutoroutePartHandler.cs:386-409`](../../../src/OrchardCore.Modules/OrchardCore.Autoroute/Handlers/AutoroutePartHandler.cs) renders admin-defined Liquid), which is the answer that stands out.

---

8/10 repo-anchored (Q7 conceptual, Q3 partly). If your target role is React-heavy, answer each card twice: once with the repo anchor, once translated to React idioms — the translation practice is the interview skill.
