# 01 — Codebase Cartography

You can't reason about a system you can't locate yourself in. This track builds your map of Orchard Core before you form opinions about it.

| File | What it gives you |
| --- | --- |
| [01-system-map.md](01-system-map.md) | The three-layer shape: host → framework libs → modules; ownership map |
| [02-file-reading-order.md](02-file-reading-order.md) | 28 files in reading order, junior/mid/senior paths |
| [03-domain-glossary.md](03-domain-glossary.md) | Tenant, shell, content item, part, field, shape, recipe… with code locations |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | SDK, build, tests, assets pipeline, runtime boundaries |
| [05-key-flows.md](05-key-flows.md) | **The core asset**: 8 end-to-end flows with trace tables and drills |

Suggested order: 01 → 03 (skim) → 05 (deep) → 02 (as a guided tour) → 04 (reference).

Interview note: "walk me through a large codebase you know" is a standard mid-level screen question. The system map + one key flow, told concretely, is a complete answer.
