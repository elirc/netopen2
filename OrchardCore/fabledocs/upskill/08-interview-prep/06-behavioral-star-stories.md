# Behavioral STAR Stories

Ten worksheets. If you complete tickets/projects from [06-contribution-practice](../06-contribution-practice/README.md), fill Situation/Task/Action/Result with *your* work. If you're studying the repo without contributing yet, the "studying a complex codebase" and "found a risk" stories below are sourced from this analysis itself — legitimate and specific.

Rule: every story ends with **impact**, stays under 2 minutes, and cites concrete evidence (a PR, a test, a metric, a file). Map each to the prompts it answers.

---

## S1 — Learning a large unfamiliar codebase
Prompts: "walk me through a complex codebase you learned"; "how do you get up to speed?"
Source: this curriculum's cartography work.
S: Handed a ~200-project modular CMS (Orchard Core) with no prior context. T: understand it well enough to reason about changes. A: mapped the runtime spine first (request → tenant container swap → shell scope → commit-at-scope-end), traced 8 end-to-end flows with file anchors, then read one full vertical module (PublishLater) to see all layers at once. R: could locate any concern and explain the publish invariant in one paragraph within days.
Evidence: the flow traces / system map. Senior-signal: "I learn spine-first, not file-by-file."
Resume bullet: *Onboarded to a 200-project modular ASP.NET Core CMS by mapping its tenancy and content-versioning flows end to end.*

## S2 — Found a risk before it bit
Prompts: "time you prevented a problem"; "attention to detail."
Source: risk R1 (API publish-authz asymmetry).
S: reviewing content APIs. T: verify permission checks matched across surfaces. A: compared the admin controller's `PublishContent` gate against the API endpoint's `EditContent`-only path; documented the exact principal+request that would exercise the gap; labeled it needing runtime confirmation rather than crying "vulnerability." A(cont): proposed routing both surfaces through one operation-level check. R: turned a latent inconsistency into a scoped, testable fix with a disclosure-aware process.
Senior-signal: calibrated claim ("investigate", private channel if confirmed), not alarmism.
Resume bullet: *Identified a cross-surface authorization inconsistency in a content API and proposed a single-seam fix with regression tests.*

## S3 — Technical tradeoff you weighed
Prompts: "a hard technical decision"; "tradeoff you made."
Source: side-effects analysis (deferred task vs outbox).
S: needed a post-publish side effect (e.g. webhook). T: choose delivery semantics. A: analyzed at-most-once (existing deferred tasks — safe only because route refresh is a recomputation) vs at-least-once outbox; chose based on whether the effect was idempotent. R: shipped the simple option where safe, reserved the outbox for non-idempotent external calls — and could defend both.
Senior-signal: the decision rule, not the mechanism.

## S4 — Debugging under uncertainty
Prompts: "toughest bug"; "debug something intermittent."
Source: slug-collision scenario (or your own).
S: two pages intermittently served for one URL. T: find whether it was a race or bad data. A: queried the index for duplicate paths, checked timestamps to distinguish TOCTOU from an import bypass, reproduced with parallel requests. R: root-caused a check-then-write race; fixed the instance (resave) and proposed the class fix (unique constraint + retry).
Senior-signal: separated instance repair from class fix.

## S5 — Improving reliability/observability
Prompts: "made a system more reliable"; "operational improvement."
Source: R3 / Project 2.
S: background task failures were invisible (log-only). T: make "did scheduled work succeed?" answerable. A: designed an event-handler that records per-task state + a health check, no kernel changes. R: operators get a dashboard instead of grep.
Resume bullet: *Designed background-task health monitoring surfacing per-tenant last-run/last-error, closing a log-only observability gap.*

## S6 — Disagreement with a teammate/maintainer
Prompts: "conflict"; "disagreed with a decision."
Source: review kata (static-lock slug fix) / [07-career/03](../07-career-and-collaboration/03-maintainer-communication.md).
S: a proposed fix used a process-wide lock. T: get to the right fix without alienating the author. A: restated their goal, added the missing info (multi-instance farms + head-of-line blocking, citing the distributed-lock service), offered the decision back with a staged roadmap. R: aligned on constraint-plus-retry; author stayed engaged.
Senior-signal: treated it as asymmetric information, not a contest.

## S7 — Mistake and recovery
Prompts: "a mistake you made"; "how you handle being wrong."
Source: any drill where your first read was wrong (be honest).
S: assumed a "Get" method was read-only. T: — A: shipped/reviewed on that assumption, then discovered `GetAsync(DraftRequired)` writes a version row. Owned it immediately, added a doc comment (T3) so the next person wouldn't repeat it. R: turned my error into a permanent guardrail.
Senior-signal: fix the class (docs/test), not just the instance.

## S8 — Ambiguity / incomplete information
Prompts: "unclear requirements"; "start with little context."
Source: verifying an unverified risk (R6 GraphQL draft gating).
S: unclear whether draft data was permission-gated. T: resolve without assuming. A: traced the resolver + filter seam as far as the code allowed, labeled the untraced part explicitly, and scoped a test to prove-or-fix rather than guessing. R: converted ambiguity into a concrete verification task.
Senior-signal: comfortable saying "I don't know yet; here's how I'd find out."

## S9 — Mentoring / making others better
Prompts: "helped a teammate grow"; "leadership without authority."
Source: writing the review katas / annotation drills (or explaining a flow to a peer).
S: teammates struggled to review CMS PRs safely. T: give them a repeatable method. A: distilled the review into five layers + a repo checklist + example comments, tied to real anchors. R: reviews caught the load-bearing issues (missing cancel, authz drift) instead of bikeshedding names.

## S10 — Scoping / saying no
Prompts: "pushed back on scope"; "prioritized under pressure."
Source: the architecture critique's "what I would NOT change" (shape system, kernel complexity).
S: pressure to "clean up" the stringly-typed rendering system. T: decide what's worth touching in a quarter. A: ranked risks by blast radius; explicitly deferred the high-radius, working-as-designed areas and shipped the low-radius high-value fixes (authz, observability). R: focused effort where risk/reward was best.
Senior-signal: naming what you *won't* do is the seniority tell.

---

## Filling and rehearsing

For each: write 4 sentences (S/T/A/R) + one evidence citation + one resume line. Rehearse against a timer; if you exceed 2 minutes, cut the Situation, never the Result. Then map each story to 2–3 common prompts so you can redeploy one story across questions. The [cram plan](07-two-week-cram-plan.md) rehearses these on days 5, 9, and 13.
