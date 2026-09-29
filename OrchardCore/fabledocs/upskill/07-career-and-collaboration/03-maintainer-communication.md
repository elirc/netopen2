# Maintainer Communication

Channels for this project: GitHub Issues (bugs/features), GitHub Discussions (questions/feedback), Discord (real-time), security email for vulnerabilities ([README's Help section](../../../README.md), [SECURITY.md](../../../SECURITY.md)). Choosing the right channel is itself a signal.

## Asking for help without outsourcing thinking

The format that gets answers:

```markdown
**What I'm trying to do**: schedule unpublishing (mirror of PublishLater).
**What I've found**: PublishLater's task queries PublishLaterPartIndex
(ScheduledPublishingBackgroundTask.cs:29-32); I've built the same shape.
**Where I'm stuck**: my index rows never appear. I've confirmed the migration
ran (table exists) and the part saves (JSON contains it).
**My current hypothesis**: my IndexProvider isn't registered per-tenant —
does RegisterIndexes require the scoped IScopedIndexProvider variant?
**Environment**: main @ 29649872e, SQLite, fresh Blog recipe.
```

Five lines of evidence beats five paragraphs of frustration. The hypothesis line is what separates "help me think" from "think for me."

## Reporting a bug (repro-first)

```markdown
**Steps**: 1. Fresh site, Blog recipe. 2. Create two posts titled "About".
   3. Publish both within ~1s (two tabs).
**Expected**: second gets /about-2.
**Actual**: both have /about; menu links route to the first.
**Evidence**: AutoroutePartIndex rows attached; both Published=1, same Path.
**Suspected area**: IsAbsolutePathUniqueAsync check-then-write
   (AutoroutePartHandler.cs:465-477) — race between validation and commit?
**Version/DB**: main @ 29649872e / SQLite.
```

Minimal, reproducible, anchored, hypothesized-but-humble. If it's security-sensitive (real data exposure, authz bypass like R1): **security email, not a public issue** — offering that judgment is a trust signal.

## Proposing a feature

Lead with the user problem, offer the smallest version, volunteer the work: "Operators can't see background task health (details in issue). Smallest useful slice: last-run/last-error per task in the admin UI, no retry logic. I'm happy to implement behind a feature if maintainers agree with the shape — draft design attached."

## Responding to review

- Every comment gets a response: fixed (link commit), pushback (with a failure scenario or measurement), or question.
- Batch pushes; don't force re-review per nit.
- When wrong, say so plainly and move: "You're right — the deferred task re-resolves the session, my capture was stale. Fixed in abc123."
- When right but overruled on style: take it. Bank disagreement capital for correctness issues.

## Respectful disagreement (script)

1. Restate their concern until they'd endorse the restatement.
2. Add the missing information, anchored: "the reason I avoided a static lock is multi-instance farms — the distributed lock service at ModularBackgroundService.cs:133-157 exists for exactly this."
3. Offer the decision back: "if you still prefer X, I'll do X and note the farm limitation in docs."
Most "disagreements" are asymmetric information; step 2 resolves them. Step 3 keeps you easy to work with — and is *verbatim* a strong behavioral-interview beat.

## Timezone/async etiquette for OSS

One thorough message beats four fragments. Include everything a maintainer in another timezone needs to act without a round-trip: repro, environment, hypothesis, and what you'll try next if no answer in a few days (then actually do it and report back — the follow-through is what builds the reputation).

Interview angle: behavioral questions about conflict, feedback, and ambiguity are all answerable from these habits — see [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md) stories S6–S8.
