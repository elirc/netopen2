# Command Cheatsheet

Legend: __verified__ = run in this workspace; __inferred__ = sourced from `AGENTS.md`, `global.json`, `package.json`, `test/OrchardCore.Tests.Functional/README.md`, or CI workflows but not executed here. When in doubt, treat as inferred and confirm locally.

## Run

```bash
cd src/OrchardCore.Cms.Web && dotnet run -f net10.0     # __inferred__ — CMS on http://localhost:5000
# Reset to setup screen: delete src/OrchardCore.Cms.Web/App_Data/   # __inferred__
cd src/OrchardCore.Mvc.Web && dotnet run                 # __inferred__ — minimal modular-MVC host
```

## Build

```bash
dotnet build OrchardCore.slnx                            # __inferred__ — full solution (slow)
dotnet build OrchardCore.slnx -c Release                 # __inferred__
dotnet build src/OrchardCore.Modules/OrchardCore.Contents/OrchardCore.Contents.csproj  # __inferred__ — scoped, fast
```
Prereqs (__verified__ from files): .NET SDK 10.0.200+ ([`global.json`](../../../global.json)); Node 22 + Yarn 4 for assets ([`package.json`](../../../package.json), [`AGENTS.md`](../../../AGENTS.md)).

## Test (xUnit v3 on Microsoft.Testing.Platform)

```bash
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj                       # __inferred__ — unit
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-class "*ShellHostTests*"    # __inferred__
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj --filter-method "*.MyTest"           # __inferred__
# Functional (Playwright, in-process host):
dotnet build -c Release test/OrchardCore.Tests.Functional/OrchardCore.Tests.Functional.csproj    # __inferred__
dotnet exec test/OrchardCore.Tests.Functional/bin/Release/net10.0/Microsoft.Playwright.dll install chromium   # __inferred__ (README)
dotnet test test/OrchardCore.Tests.Functional/OrchardCore.Tests.Functional.csproj -c Release --no-build --filter-class "*Cms*"   # __inferred__
```
Filter notes (__verified__ from `AGENTS.md`): `--filter-class`/`--filter-method` accept fully-qualified names, `*` wildcards at start/end, multiple = OR; simple and query filters can't mix.

## Frontend assets

```bash
yarn                # __inferred__ — install workspaces (Yarn 4)
yarn build          # __inferred__ — assets-manager build → each module's wwwroot/
yarn watch          # __inferred__ — rebuild on change
yarn lint           # __inferred__ — ESLint 9 (eslint.config.mjs)
yarn check          # __inferred__ — vue-tsc --noEmit
```

## Git / inspection (__verified__ — run here)

```bash
git log --oneline -5            # HEAD 29649872e
git status --short
```

## Migrations / codegen

- Migrations run **automatically** per tenant at startup (`AutomaticDataMigrations`, [`OrchardCoreBuilderExtensions.cs:44`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)) — no manual migrate command. __inferred__
- To exercise a migration path locally: run with a pre-upgrade `App_Data`, watch logs. __inferred__

## Docker / docs

```bash
docker build -f Dockerfile -t orchardcore .    # __inferred__
mkdocs serve                                   # __inferred__ — docs site from mkdocs.yml
```

## Database env vars (functional tests, __inferred__ from README)

```bash
OrchardCore__DatabaseProvider=Postgres OrchardCore__ConnectionString="..." \
  dotnet test test/OrchardCore.Tests.Functional/...   # captured once at process start, then cleared per fixture
```

None of the build/run/test commands were executed in this workspace — verify locally before relying on exact flag behavior.
