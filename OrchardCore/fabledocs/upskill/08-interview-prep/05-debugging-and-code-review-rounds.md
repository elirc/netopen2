# Debugging and Code-Review Rounds (Timed)

Three debugging sims + three review sims, converted from earlier modules into interview format with interviewer follow-ups and rubrics. Time-box strictly; narrate aloud (interviewers score your *process*, not just the answer).

---

## Debugging Round D1 — "Saved changes vanish" (10 min)
Prompt: "A user reports that editing a content item's custom field, then saving, discards their change. No error. Walk me through debugging it."
(Full scenario: [05-quality-engineering/03](../05-quality-engineering/03-systematic-debugging.md) Scenario 1.)
Interviewer follow-ups you should expect:
- "How do you know it's not the database?" → bisect at the session-commit boundary.
- "The value reaches the model but not the DB — now what?" → `ModelState.IsValid`? *another* part failing validation cancels the whole session ([`AdminController.cs:759-768`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Controllers/AdminController.cs)).
- "How would you prevent regressions?" → driver-level unit test.
Rubric — Pass: bisects binding vs persistence with concrete probes. Strong: identifies the unit-of-work cancel as the mechanism and names the cross-part coupling *before* being led there.

## Debugging Round D2 — "Scheduled publish didn't fire" (10 min)
Prompt: "Content scheduled for 10:00 is still a draft at 10:15. Debug." ([05-03](../05-quality-engineering/03-systematic-debugging.md) Scenario 4.)
Follow-ups: "cheapest first check?" (query index eligibility — timezone bug is #1); "logs show the task ran but skipped it?" (poison-item batch abort R4 / lock timeout); "two servers?" (distributed lock → exactly one runs).
Rubric — Strong: separates *eligibility → execution → outcome*, and the first probe is a query, not "add logging."

## Debugging Round D3 — "Login slow for some users" (8 min)
Prompt: [05-03](../05-quality-engineering/03-systematic-debugging.md) Scenario 5.
The trap: the slow path for unknown users is *intentional* (timing normalization, [`AccountController.cs:182-188`](../../../src/OrchardCore.Modules/OrchardCore.Users/Controllers/AccountController.cs)).
Rubric — Strong: segments by outcome first and recognizes the deliberate security property instead of "fixing" it. This round tests whether you break security by reflex.

---

## Review Round R1 — "Friendlier login errors" (10 min)
Give the candidate Kata 3 ([04-code-reading-gym/04](../04-code-reading-gym/04-review-katas.md)): a PR that says which of username/password was wrong and deletes the "dead" timing normalizer.
Expected: **Blocking** on enumeration (message *and* timing); offer a safe alternative (rate-limited username recovery).
Follow-up: "the author insists the normalizer is dead code — convince them." → explain the timing side-channel in two sentences with the dummy-hash mechanism.
Rubric — Strong: kind, specific, teaches the vuln, offers the safe path — doesn't just say "rejected."

## Review Round R2 — "Move the 404 up" (8 min)
Kata 4: reorder so missing ids 404 before authorization on the delete endpoint ([`DeleteEndpoint.cs:31-46`](../../../src/OrchardCore.Modules/OrchardCore.Contents/Endpoints/Api/DeleteEndpoint.cs), `AllowAnonymous` route).
Expected: **Blocking** — existence oracle for anonymous callers; auth must precede existence signals.
Follow-up: "what's the ideal contract?" → uniform 403/404 policy, documented.
Rubric — Strong: distinguishes the *new* leak (anonymous) from the *existing* narrower one (any `AccessContentApi` holder) and files the latter as a question, not a block.

## Review Round R3 — "Slug race fix with a static lock" (10 min)
Kata 6: a PR wrapping the uniqueness check+save in a process-wide lock.
Expected: **Blocking** — doesn't compile around `await` / if `SemaphoreSlim`, still single-process (farms race) and serializes all saves globally.
Follow-up: "so what's the real fix?" → DB unique constraint + violation retry preserving the `-N` suffix logic, staged behind a dedupe migration (Project 1).
Rubric — Strong: names both defects (multi-instance + head-of-line blocking) and gives the staged roadmap, not just "wrong."

---

## How to run these solo

Record yourself. Score each on: (1) clarifying/repro before diving, (2) explicit competing hypotheses, (3) cheapest discriminating probe, (4) root cause vs symptom, (5) regression/fix proposal, (6) tone (review rounds). 5–6/6 across all six = interview-ready. Map to the [cram plan](07-two-week-cram-plan.md) days 8 and 11.
