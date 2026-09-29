# 08 — Interview Prep

Target: **mid-level fullstack interviews** (.NET-centric, but every concept transfers). The edge this module gives you: answers anchored to a real codebase you can cite — "in Orchard Core, publish demotes the previous version inside one transaction; here's why" beats any definition.

## How mid-level loops are typically structured

1. **Screen** (30–45 min): one coding exercise + "walk me through a project/codebase."
2. **Technical deep-dive** (60): your stack — async, DI, data access, HTTP. Files 01–03 here.
3. **Practical coding** (60): build/extend a small feature; tests expected.
4. **System design** (45–60): scoped for mid-level — one service done well, not global scale. File 04.
5. **Debugging / code review** (increasingly common): File 05.
6. **Behavioral** (45): STAR stories. File 06.

## The golden rule

Answer with **concrete example → tradeoff → failure mode**, never a definition alone.

> Weak: "Optimistic concurrency uses version numbers to detect conflicts."
> Strong: "Orchard Core saves content versions with `checkConcurrency: true` — a version check at commit. Two parallel edits: the second commit throws instead of silently overwriting. The tradeoff is you must handle the exception path — the admin UI surfaces it as an error rather than merging. For low-contention CRUD that's the right cost; for high-contention counters you'd redesign instead."

Same fact; the second one is a mid-level answer. Every card in this module drills that shape.

## Contents

| File | Cards | Notes |
| --- | --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | 14 | C#/.NET runtime deep-dive (the JS mapping noted per card) |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | 10 | rendering/composition — server-side here, React-transferable |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | 12 | REST/validation/schema/transactions/caching |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | 1 full walkthrough + 4 variations | reverse-engineered whiteboard exercise |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | 3 debugging + 3 review sims | timed, with rubrics |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | 10 worksheets | sourced from tracks 05–06 |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | — | day-by-day schedule of all of the above |

36 question cards + 6 simulation prompts + 5 design prompts ≈ 47 items; ~80% repo-anchored. Interview in two weeks? Go straight to [07-two-week-cram-plan.md](07-two-week-cram-plan.md).
