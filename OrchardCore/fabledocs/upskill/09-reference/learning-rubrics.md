# Learning Rubrics

Observable behaviors, not vague traits. Use these to place yourself and to define "done" for each skill. The **Interview-ready** column is the bar a mid-level loop tests.

## Skill: Understanding the codebase

| | Junior | Mid | Senior | Interview-ready (mid target) |
| --- | --- | --- | --- | --- |
| Navigation | finds a file when told its name | locates the owner of a concern from symptoms | predicts where an unseen feature lives from conventions | traces any of the 8 key flows from memory |
| Flows | can read one flow with the doc open | narrates publish/tenant flows unaided | explains *why* each step exists + its failure mode | Flow 1 & 2 spoken in <2 min each with anchors |
| Boundaries | sees layers when pointed out | names what each layer owns | identifies leaks and judges whether they're acceptable | can cite Leak A (ModelState in domain) as a real example |

## Skill: C#/.NET & framework

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Async | uses async/await | reasons about parallelism + shared state | picks concurrency primitives per contention shape | explains the per-tenant semaphore + retry unaided |
| DI | injects services | knows lifetimes + captive deps | designs lifetime/container boundaries | names a captive-dependency bug with the repo example |
| Data access | writes a query | knows documents vs index reads, unit of work | designs indexes with write-amplification budget | contrasts `Query<T,TIndex>` vs `QueryIndex` and when |

## Skill: Correctness & judgment

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Validation | adds a required check | knows all 5 layers + cancel-the-session | designs where each invariant is enforced | can state why validation failure without cancel persists |
| AuthZ | checks a permission | uses resource-based + owner variants | spots authz drift across surfaces | explains the owner-variant IDOR defense + R1 |
| Reliability | happy path works | knows idempotency + post-commit effects | chooses delivery semantics per effect | delivers the at-most/least/exactly-once ladder |

## Skill: Debugging

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Method | tries fixes | reproduce→narrow→hypothesize→fix→regress | bisects at boundaries, cheapest probe first | narrates a debugging round with explicit hypotheses |
| Probes | adds prints everywhere | uses logs/queries/breakpoints deliberately | picks the one discriminating probe | first probe is a query/log line, not "add logging" |

## Skill: Contribution & collaboration

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Scope | changes what's asked | keeps PRs single-concern, tested | scopes to small blast radius by design | can justify a ticket's blast radius unprompted |
| Review | finds style issues | grades blocking vs optional, anchors comments | teaches through review, kind tone | delivers a review kata with a staged fix + tone |
| Communication | asks vague questions | evidence + hypothesis + environment | disagrees via asymmetric-info framing | tells a conflict STAR ending in alignment |

## Self-assessment checklist (tick honestly)

Foundational (target before interviewing):
- [ ] I can draw the request→tenant→scope→commit spine from memory
- [ ] I can explain the Latest/Published invariant and where it's maintained
- [ ] I can name the 5 validation layers and the cancel-the-session rule
- [ ] I can give a repo example for async/await, DI lifetimes, unit of work
- [ ] I can pick documents-vs-index reads and justify it

Mid-level (target for the loop):
- [ ] I deliver every answer as example → tradeoff → failure mode
- [ ] I can run a system-design walkthrough (steps 0–6) + 2 variations
- [ ] I can debug two scenarios aloud with explicit competing hypotheses
- [ ] I can review a PR: find the blocking issue, grade the rest, stay kind
- [ ] I have 8+ STAR stories, each <2 min, each ending in impact

Senior signals (stretch):
- [ ] I can critique the architecture with ranked risks + costs (R1–R4)
- [ ] I can say what I would *not* change and why
- [ ] I can design a migration under the 4-provider portability constraint

Re-run this monthly if you're upskilling over 8 weeks; before an interview, the Mid-level block is the gate.
