# Tooling and Build System

## Solution and package management

- **`OrchardCore.slnx`** — the new XML solution format holding ~200 projects. You will rarely open all of it; work per-project (`dotnet build src/OrchardCore.Modules/OrchardCore.Contents/...csproj` builds its dependency subgraph).
- **Central Package Management**: versions live once in [`Directory.Packages.props`](../../../Directory.Packages.props); csproj files reference packages without versions. Transferable: one lockstep upgrade point, no version drift between 200 projects — the .NET analog of a pnpm catalog.
- **`Directory.Build.props`** cascades common MSBuild settings (there are three: root, `src/`, `src/OrchardCore.Modules/` — each layer narrowing scope). When a property seems to come from nowhere, walk up the directory tree.
- **`src/OrchardCore.Build/TargetFrameworks.props`** pins `net10.0` per `AGENTS.md` (__inferred__, file not read).

## The dependency law of the repo

Modules depend on `*.Abstractions`/`*.Core` libraries, *not* on other modules' implementation projects. Cross-module integration happens through DI interfaces resolved at runtime within a tenant. That's why `CommonPermissions` lives in `OrchardCore.Contents.Core` — "so they can be used in other modules without having to reference OrchardCore.Contents by itself" (the file says so itself: [`CommonPermissions.cs:5-8`](../../../src/OrchardCore/OrchardCore.Contents.Core/CommonPermissions.cs)).

## Frontend asset pipeline

From [`package.json`](../../../package.json) (__verified__ read):
- Yarn 4 workspaces: `.scripts/assets-manager`, `.scripts/bloom`, `src/Frontend/` (listed in `package.json` but absent from this snapshot), plus **every module/theme `Assets/` folder** is its own workspace.
- `yarn build` runs the custom `@orchardcore/assets-manager` which compiles each module's `Assets.json`-described sources into that module's `wwwroot/`.
- `yarn lint` = ESLint 9 flat config ([`eslint.config.mjs`](../../../eslint.config.mjs)); `yarn check` = `vue-tsc` (some admin UIs are Vue).

Rule that saves you a bad PR: **never edit `wwwroot/` files** — they're build outputs; edit `Assets/` sources and rebuild. (Convention inferred from the workspace layout and `Assets.json` files, e.g. [`src/OrchardCore.Modules/OrchardCore.Contents/Assets.json`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Assets.json).)

## Tests and CI

| Layer | Project | Runner |
| --- | --- | --- |
| Unit | `test/OrchardCore.Tests` | xUnit v3 on Microsoft.Testing.Platform ([`global.json`](../../../global.json) `"test": { "runner": "Microsoft.Testing.Platform" }`) |
| Integration | `test/OrchardCore.Tests.Integration` | CI workflow `integration_tests.yml` |
| E2E | `test/OrchardCore.Tests.Functional` | Playwright + xUnit, in-process host, parallel fixtures with isolated `App_Data_Tests_{Fixture}` dirs ([README](../../../test/OrchardCore.Tests.Functional/README.md)) |
| All-DB E2E | same | `functional_all_db.yml` runs against SQLite/SQL Server/MySQL/Postgres |

Useful flags (xUnit v3 style, from `AGENTS.md`): `--filter-class "*ShellHostTests*"`, `--filter-method "*.MyTest"`.

## Docker & docs

- [`Dockerfile`](../../../Dockerfile) / `Dockerfile-CI` build the CMS image (not analyzed further here).
- Docs site: mkdocs ([`mkdocs.yml`](../../../mkdocs.yml)) sourcing `src/docs/` — PRs that change behavior are expected to update these docs (see [CONTRIBUTING.md](../../../CONTRIBUTING.md)).

## Sharp edges

- First `dotnet build` of the full solution is long; prefer project-scoped builds during iteration.
- `dotnet run` must happen in `src/OrchardCore.Cms.Web` (tenant data lands in that project's `App_Data/`). Delete `App_Data/` to reset to the setup screen — that's the standard "factory reset" while learning.
- Renovate manages dependency PRs ([`renovate.json5`](../../../renovate.json5)) — don't hand-bump packages in unrelated PRs.

## Drill

Trace how a CSS change in `src/OrchardCore.Themes/TheBlogTheme/Assets/` (if you make one) reaches the browser: workspace → assets-manager build → theme `wwwroot/` → static files middleware → page. Which steps run at build time vs request time?

## Interview angle

1. "How do you manage dependencies across a large monorepo?" → central package management + abstractions-only references.
2. "Build vs runtime artifacts — how do you keep them straight?" → Assets/ vs wwwroot/.
3. "How is your test pyramid structured?" → the table above, → [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).
