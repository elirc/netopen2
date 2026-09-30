# Orchard Core: reasoning across tenant boundaries

This independent workbook extends the [original learning route](../README.md). Read the original fast track and codebase cartography first. The new route starts with executable control flow, then builds toward persistence, publication, background work, and recovery. Existing application code and learner material are preserved.

## Available chapters

1. [Selecting a tenant is an ordered decision](01-tenant-selection.md)
2. [A request crosses two middleware boundaries](02-request-scope-and-paths.md)

Attempt each chapter's exercises before opening [review guide 01](reviews/01-routing-and-request-boundaries.md). Exercise identifiers beginning OC belong to this course. Answers are separate so prediction remains an independent activity.

## Continuing route

The next installments cover shell scope lifetime and deferred tasks, content identities and versions, YesSql indexing, publication and autoroute collisions, recipe execution, background scheduling, feature and permission boundaries, rendering, and independent incident and design capstones. Planned topics are not claims of completed material.

## Evidence and safe working practices

Source links point into this local checkout. Named methods are more durable anchors than historical line numbers. Each chapter distinguishes what this checkout does, what an educational model assumes, and what a proposed change would need to establish. Read a test before treating its name as evidence; a test file that exists has not necessarily been executed in this environment.

This workbook is outside the canonical MkDocs documentation tree. Its navigation is maintained here; adding it does not alter the application documentation site or runtime. No application setup, tenant creation, migrations, package installation, or production access is required for the opening chapters. Keep notes in a separate learner workspace. Do not overwrite repository files to answer an exercise.
