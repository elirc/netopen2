# Writing PRs and RFCs

## PR description template (fits this repo's expectations)

```markdown
## What
One paragraph: the user-visible or maintainer-visible change.

## Why
Link the issue. If none exists, open one first for anything non-trivial —
maintainers prefer discussing direction before reviewing code.

## How
- Key decisions and the pattern followed (e.g. "index + migration mirrors OrchardCore.PublishLater")
- Anything you considered and rejected, in one line each

## How tested
- Unit: `dotnet test ... --filter-class "*X*"` (paste the command)
- Manual: steps on a fresh SQLite site (Blog recipe)
- All-DB concerns: none / migration reviewed against provider limits

## Risks / follow-ups
- Behavior changes, perf notes, out-of-scope items as issues
```

Rules that keep reviews fast here: one concern per PR; no drive-by refactors mixed with features; user-facing strings localized; docs updated (`src/docs/`) when behavior changes; screenshots for admin-UI changes.

## Commit messages

Imperative summary ≤72 chars, body = why. The repo squashes typical PRs (observed: single-commit-per-PR history like `29649872e Fix theme extensions not appearing when manifest includes...`), so your PR title is the artifact that survives — write it like a changelog line.

## When to RFC (issue/discussion first)

RFC-first when any is true: touches `*.Abstractions` (public contract), adds a dependency, changes migration behavior, spans >1 module, changes a default. The Kata/Project pipeline maps: T-tickets → straight PR; M-tickets → issue first; Projects → RFC.

## RFC template tailored to this repo

```markdown
# RFC: <title>

## Problem
What breaks or is missing today. Anchor with file:line evidence
(e.g. deferred tasks drop failures silently — ShellScope.cs:497-504).

## Proposal
The design in one page. Name the existing patterns composed
(document+index, IBackgroundTask+distributed lock, deferred tasks...).

## Tenancy impact          ← this section is Orchard-specific and mandatory
Per-tenant or host-level? What happens on shell release/reload?

## Data & migration
New tables/indexes; portability across SqlServer/Sqlite/MySql/Postgres;
expand/contract steps.

## Compatibility
Abstractions changes (additive?); behavior changes; recipe/deployment formats.

## Alternatives considered
Two minimum, with the one-line reason each lost.

## Test & rollout plan
Layers, all-DB needs, feature-flag/default-off strategy, rollback.
```

## Worked example (abbreviated): RFC for Project 2

> **Problem**: background task failures are visible only in logs ([`ModularBackgroundService.cs:222-230`](../../../src/OrchardCore/OrchardCore/Modules/ModularBackgroundService.cs)); operators can't answer "has scheduled publishing succeeded today?" without grepping.
> **Proposal**: `OrchardCore.BackgroundTasks.Monitoring` feature: an `IBackgroundTaskEventHandler` persisting per-task state transitions to a tenant document; admin page + health check read it. No kernel changes.
> **Tenancy impact**: state is per-tenant; farm aggregation out of scope v1.
> **Compatibility**: additive only.
> **Alternatives**: (a) kernel-level metrics — rejected: forces an OTel dependency; (b) log-scraping guidance — rejected: not actionable in admin UI.

Interview angle: bring a printed/remembered RFC like this to "how do you propose technical changes?" — structure beats vocabulary. The behavioral pair ("a time you disagreed on design") is the Alternatives section told as a story.
