# Runtime and Tooling Map

## Toolchain

| Tool | Version / config | Evidence |
| --- | --- | --- |
| .NET SDK | 10.0.200, `rollForward: latestMajor`; tests use Microsoft.Testing.Platform runner | [`global.json`](../../../global.json) |
| Target framework | `net10.0` (set in `src/OrchardCore.Build/TargetFrameworks.props`) | [`AGENTS.md`](../../../AGENTS.md) |
| C# solution | `OrchardCore.slnx` (~200 projects); central package versions in `Directory.Packages.props` | repo root |
| Node / Yarn | Node 22.x, Yarn 4.17 (`packageManager` field), workspaces for module/theme assets | [`package.json`](../../../package.json) |
| Asset pipeline | custom `@orchardcore/assets-manager` (`yarn build/watch/clean`), ESLint 9, vue-tsc typecheck | [`package.json` scripts](../../../package.json) |
| Docs | mkdocs (`mkdocs.yml`), content under `src/docs/` | repo root |
| CI | GitHub Actions: `pr_ci.yml`, `main_ci.yml`, `integration_tests.yml`, `functional_all_db.yml` (functional tests against all four DBs), `release_ci.yml` | `.github/workflows/` |

## Commands (see [../09-reference/command-cheatsheet.md](../../upskill/09-reference/command-cheatsheet.md) for the full annotated list)

```bash
# All __inferred__ from AGENTS.md / README files — not executed in this workspace
cd src/OrchardCore.Cms.Web && dotnet run -f net10.0     # run the CMS on :5000
dotnet build OrchardCore.slnx                            # full build (slow: ~200 projects)
dotnet test test/OrchardCore.Tests/OrchardCore.Tests.csproj
dotnet test ... --filter-class "*ShellHostTests*"        # xUnit v3 simple filters
yarn && yarn build                                       # frontend assets
```

## Runtime boundaries

| Boundary | What runs there | Sharp edge |
| --- | --- | --- |
| **Host process** (singleton services) | `ShellHost`, `ModularBackgroundService`, `RunningShellTable`, `IStore`-per-tenant singletons | A singleton must never capture a tenant-scoped service; tenant containers come and go on reload. See how `ModularBackgroundService` re-resolves everything per scope ([`ModularBackgroundService.cs:159-231`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)). |
| **Tenant container** (per shell) | Everything a module registers in `Startup.ConfigureServices` | Rebuilt on feature enable/disable — `ShellHost.ChangedAsync` releases the shell ([`ShellHost.cs:173-174`](../../../src/OrchardCore/OrchardCore/Shell/ShellHost.cs)). Cached state dies with it. |
| **Shell scope** (per request / per background run) | `ISession`, `IContentManager`, controllers | The session commits **once**, at scope end ([`OrchardCoreBuilderExtensions.cs:152-170`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)). `await session.CancelAsync()` is how you "roll back". |
| **Browser** | Per-module JS/CSS under `Assets/`, compiled to `wwwroot/` by assets-manager; Vue in some admin UIs | Assets are *built artifacts*; edit sources under `Assets/`, not `wwwroot/`. |
| **Background execution** | `IBackgroundTask` per tenant, cron-scheduled, fake `HttpContext` injected ([`ModularBackgroundService.cs:108-109`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)) | No real request: anything reading the URL needs the site `BaseUrl` fallback (lines 169-181). |

## Configuration and environment

- Host config: standard ASP.NET Core `appsettings.json` in `src/OrchardCore.Cms.Web/`.
- Tenant registry: `App_Data/tenants.json` (created at first run). Per-tenant config: `App_Data/Sites/{tenant}/appsettings.json` — documented in [`ShellSettings.cs:9-13`](../../../src/OrchardCore/OrchardCore.Abstractions/Shell/ShellSettings.cs).
- Any tenant setting is also reachable via regular .NET configuration sources prefixed `OrchardCore__...` (functional tests use `OrchardCore__ConnectionString` / `OrchardCore__DatabaseProvider` — [`test/OrchardCore.Tests.Functional/README.md`](../../../test/OrchardCore.Tests.Functional/README.md)).
- Database per tenant: provider + connection string + optional `TablePrefix`/`Schema` chosen at setup ([`OrchardCoreBuilderExtensions.cs:56-118`](../../../src/OrchardCore/OrchardCore.Data.YesSql/OrchardCoreBuilderExtensions.cs)). SQLite is the zero-config default (line 51).
- No secrets are committed; connection strings live in `App_Data` at runtime. (Do not paste `tenants.json` contents into issues.)

## Where state lives at runtime

```
App_Data/
├── tenants.json               # all tenants' settings
├── Sites/{tenant}/
│   ├── appsettings.json       # per-tenant config
│   └── yessql.db              # SQLite DB (default provider)
└── logs/                      # NLog output (UseNLogHost in Program.cs)
```

## Drill

Answer without looking: (1) If you enable a feature on tenant A, why does tenant B's traffic never pause? (2) Why can't a singleton hold an `ISession`? Check yourself against the boundary table. **Strong** answers mention: per-tenant containers are released and rebuilt independently; `ISession` is scoped, stateful, and committed at scope end — a singleton would share one uncommitted unit of work across requests and tenants.
