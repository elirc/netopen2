# Two-Week Cram Plan

For a mid-level fullstack interview ~14 days out, using this repo as your evidence base. ~2–3 focused hours/day. Adjust for your weak spots after the day-7 checkpoint.

Ground rule: **every session ends with one answer spoken aloud, on a timer.** Reading is not practice; retrieval is.

## Week 1 — Build the base

**Day 1 — Orientation.** Read [README](../README.md) + [00-fast-track](../00-fast-track.md). Run the app on SQLite (or read the flow if you can't build). Skim [01-system-map](../01-codebase-cartography/01-system-map.md). Teach-back the architecture paragraph aloud.

**Day 2 — The two core flows.** [05-key-flows](../01-codebase-cartography/05-key-flows.md) Flows 1 & 2. Fill the Flow 2 trace table yourself. Aloud: "life of a publish" in 2 min.

**Day 3 — Runtime deep-dive.** [02-01 language model](../02-stack-and-language-mastery/01-language-runtime-model.md) + [08-01 cards](01-js-ts-node-deep-dive.md) Q1–Q5. Speak all five.

**Day 4 — Framework + data.** [02-02](../02-stack-and-language-mastery/02-framework-mental-models.md), [03-02 data model](../03-architecture-and-patterns/02-data-model-and-persistence.md). [08-03 cards](03-api-and-data-modeling-questions.md) Q1–Q3, Q6, Q8 aloud.

**Day 5 — Authz + behavioral seed.** [03-03 validation/authz](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) + [08-03](03-api-and-data-modeling-questions.md) Q4/Q5. Draft [08-06 STAR](06-behavioral-star-stories.md) S1, S2, S4 (write S/T/A/R).

**Day 6 — Async/reliability.** [03-04 side effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) + Flow 5. [08-01](01-js-ts-node-deep-dive.md) Q7/Q8, [08-03](03-api-and-data-modeling-questions.md) Q10 aloud (the outbox ladder).

**Day 7 — CHECKPOINT.** Self-assess (rubric below). Full [08-04 system design](04-system-design-from-this-repo.md) walkthrough at a whiteboard, 40 min, recorded. Score it. Your lowest-scoring dimension sets Week 2's emphasis.

## Week 2 — Sharpen and simulate

**Day 8 — Debugging rounds.** [05-quality/03](../05-quality-engineering/03-systematic-debugging.md) all five scenarios; then [08-05](05-debugging-and-code-review-rounds.md) D1–D3 timed and recorded.

**Day 9 — Pattern fluency + behavioral.** [03-05 pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) flashcard mode (all 16). Rehearse STAR S3, S5, S6.

**Day 10 — Frontend/rendering + remaining API cards.** [08-02](02-frontend-framework-questions.md) all; [08-03](03-api-and-data-modeling-questions.md) Q7, Q9, Q11, Q12. Speak the ones you fumble.

**Day 11 — Code-review rounds.** [04-code-reading-gym/04 katas](../04-code-reading-gym/04-review-katas.md) 1–4; then [08-05](05-debugging-and-code-review-rounds.md) R1–R3 timed, tone included.

**Day 12 — System design variations.** [08-04](04-system-design-from-this-repo.md) all 5 variation prompts, 10 min each, aloud. This is the highest-leverage day for mid→senior signal.

**Day 13 — Behavioral full pass + weak spots.** All 10 STAR stories under 2 min each. Redo any Week-1 card you still read instead of recall.

**Day 14 — FINAL CHECKPOINT + rest.** One mock of each round type (30 min design, 2 debugging, 1 review, 3 behavioral). Then stop early — walk in rested.

## Daily card budget

3–4 question cards/day from [08-01](01-js-ts-node-deep-dive.md)/[02](02-frontend-framework-questions.md)/[03](03-api-and-data-modeling-questions.md), always spoken. By day 14 you'll have voiced all 36 at least once, most twice.

## Self-assessment checkpoints

**Day 7 — can you, without notes:**
- [ ] Explain the tenant request→response flow with the container swap (Flow 1)
- [ ] Narrate a publish and the Latest/Published invariant (Flow 2)
- [ ] Answer async/await, DI lifetimes, unit-of-work with a repo example each
- [ ] Give the documents+indexes content-model design with one alternative
- [ ] Deliver system-design steps 0–2 crisply

**Day 14 — add:**
- [ ] System design steps 0–6 + at least 2 variations in 40 min
- [ ] Two debugging rounds with explicit hypotheses + cheapest probe
- [ ] Two review rounds: blocking issue found, kind tone, staged fix
- [ ] All 10 STAR stories under 2 min, each ending in impact
- [ ] Every answer follows example → tradeoff → failure mode

## Scoring dimension (apply to recordings)

Rate 1–5 each: **structure** (requirements/hypothesis before diving), **concreteness** (repo anchors, not definitions), **tradeoffs** (alternative + when-it-wins), **failure modes** (named unprompted), **communication** (2-min, impact-first). Average ≥4 across dimensions = ready. Below 3 on concreteness is the most common gap — fix it by re-anchoring every answer to a file:line before the interview.

If you have less than two weeks: do Days 1, 2, 7 (system design), 8 (debugging), 11 (review), 13 (behavioral) — the spine of the plan.
