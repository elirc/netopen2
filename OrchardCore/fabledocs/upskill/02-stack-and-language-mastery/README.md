# 02 — Stack and Language Mastery

The stack here is **C# 13 / .NET 10, ASP.NET Core, YesSql, Liquid (Fluid), and a Yarn-managed JS asset layer**. This track builds mental models from first principles, then shows where this repo uses them well and where the sharp edges are.

| File | Covers |
| --- | --- |
| [01-language-runtime-model.md](01-language-runtime-model.md) | async/await & the task model, AsyncLocal, threading, exceptions, disposal |
| [02-framework-mental-models.md](02-framework-mental-models.md) | ASP.NET Core DI lifetimes, middleware, MVC binding, hosted services |
| [03-type-system-and-contracts.md](03-type-system-and-contracts.md) | generics, JSON-backed dynamic models, interfaces-as-contracts, versioned abstractions |
| [04-tooling-and-build-system.md](04-tooling-and-build-system.md) | slnx, central package management, assets-manager, CI |

Every concept anchors to a real file. Every file ends with an **Interview angle** listing the questions this material answers, cross-linked into [08-interview-prep/](../08-interview-prep/).

If you come from Node/TS: the biggest mental shifts are (1) real threads — concurrency bugs exist that Node can't have, (2) DI lifetimes as a first-class design axis, (3) exceptions instead of error values as the dominant style.
