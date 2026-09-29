# 03 — Architecture and Patterns

This track turns "I can find things" into "I can judge things."

| File | What it teaches |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | The real layer diagram, what each layer owns, and two boundary leaks worth studying |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | YesSql document+index model, transactions, how to change schema safely |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | Every validation layer; authn vs authz; the ownership/IDOR machinery |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Side-effect map; deferred tasks; idempotency; what has no retry |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 16 pattern cards — recognition training |
| [06-architecture-critique.md](06-architecture-critique.md) | Strongest choices, real risks, what I'd change owning this for 3 months |

Read 01 → 02 → 03 → 04 in order (each builds on the last), then treat 05 as flashcards and 06 as the model answer for system-design interviews ([08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) builds on it directly).
