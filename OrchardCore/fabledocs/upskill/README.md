# Orchard Core Upskill Curriculum

A training lab built from the real Orchard Core codebase, for one specific learner: a **junior fullstack engineer** (CRUD web dev — .NET/C#, React/Node/TS or similar) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, when it fails, and how to talk about it in an interview.

## What this repo is

Orchard Core (v3.0.1) is an open-source, **modular, multi-tenant application framework and CMS for ASP.NET Core**, maintained by the Orchard Core team under the .NET Foundation. It is two products in one repository: a framework for building modular multi-tenant ASP.NET Core apps (`src/OrchardCore/*`), and a full CMS built on that framework as ~95 pluggable modules (`src/OrchardCore.Modules/*`). Each tenant gets its **own dependency-injection container and request pipeline**, built lazily from that tenant's enabled features. Persistence is **YesSql** — a document store over SQL Server/SQLite/MySQL/Postgres where documents are JSON and queries run against explicit *index tables*. UI is rendered through a **display-management** system (drivers produce "shapes", placement files position them, templates can be overridden per theme). It is one of the most instructive large .NET codebases in the wild: real multi-tenancy, real modularity, real versioned-content invariants, real background scheduling — all things interviews probe.

## The learning tracks

| Track | Directory | What you get |
| --- | --- | --- |
| Cartography | `01-codebase-cartography/` | Maps: system, reading order, glossary, tooling, key flows |
| Stack mastery | `02-stack-and-language-mastery/` | C#/.NET/ASP.NET Core/YesSql mental models anchored to real files |
| Architecture | `03-architecture-and-patterns/` | Boundaries, persistence, authz, async/reliability, pattern catalog, critique |
| Reading gym | `04-code-reading-gym/` | Annotation drills, trace tables, fake-code contrasts, review katas |
| Quality | `05-quality-engineering/` | Testing, debugging, performance, security, observability |
| Contribution | `06-contribution-practice/` | Junior tickets → mid-level features → senior projects → katas |
| Career | `07-career-and-collaboration/` | Reviews, PRs/RFCs, maintainer communication |
| **Interview prep** | `08-interview-prep/` | 40+ question cards, system design, debugging rounds, STAR stories, cram plan |
| Reference | `09-reference/` | Commands, risk register, rubrics, verification log |

## How to use it

- **One weekend**: [00-fast-track.md](00-fast-track.md) end to end.
- **Two weeks (interview soon)**: [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) — it schedules everything else.
- **Eight weeks (deep upskill)**: one track per week in order 01 → 05, then 06 (do 3 tickets + 1 project), then 07–08.
- **Ongoing contributor**: live in `06-contribution-practice/` and `07-career-and-collaboration/`, use the rest as reference.

### Recommended paths by profile

- **Brand-new junior**: 00 → 01 (all) → 02 → 04 (drills as you read) → 05-01/05-03 → first ticket from 06-01.
- **Junior who knows .NET**: 00 → 01-05 (key flows) → 03 (all) → 04 → 06.
- **Mid-level, new to this repo**: 01-01, 01-05, 03-05 (pattern catalog), 03-06 (critique), then 06-02/06-03.
- **Senior doing architecture review**: 03-06, 09-reference/risk-register.md, 05-05, 05-06.
- **Candidate with an interview in two weeks**: go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md).

## Conventions used everywhere

- **File anchors**: `src/path/File.cs:24-60` — path relative to repo root, with line ranges verified against this working copy (commit `29649872e`, 2026-07-16). Line numbers drift as the repo moves; the anchor tells you *where to look*, the surrounding prose tells you *what to find*.
- **Fake code is labeled**: every snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is quoted from real files.
- **Drills and self-grading**: most sections end with a drill and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" is the mid-level interview bar.
- **Verification labels**: commands are marked __verified__ (run here) or __inferred__ (from docs/manifests). Behavior claims that were not fully traced are labeled "investigate" or "possible risk" — never presented as confirmed bugs.
- **Senior vocabulary** — *invariant, boundary, contract, idempotency, isolation, authorization, consistency, blast radius* — is used where it truly applies and defined in context. These are interview words; learn to use them precisely, not decoratively.

## The mindset ladder

- **Junior asks**: "How do I make it work?"
- **Mid-level asks**: "Is this the right pattern? What breaks if I'm wrong?"
- **Senior asks**: "What does this commit us to, who pays the cost, and how do we reduce the risk?"

Mid-level interviews test the second and third questions, not the first. Every flow, pattern card, and question card in this curriculum is practice at climbing that ladder with concrete evidence from a real codebase.
