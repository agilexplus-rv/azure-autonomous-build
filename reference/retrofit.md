# Retrofit — adopting the method onto work already under way

The method usually arrives late: a build exists, some of it is good, nobody can say which parts
were verified and by what. **Do not resume forward.** Resuming inherits the drift and buries it
under new work. Run the reconstruction audit first — it is hours, not days, and it is the only
way the plan you write next is about the real system.

**The audit's output is evidence and gates, not a refactor.** Do not rewrite working code to make
it match the method. Rewriting is how a retrofit turns into a second build.

## 1. Reconstruct the ledger

What exists, when it landed, and — the question that is always missing — **what built it**:

```bash
git log --format="%h %ad %an | %s" --date=short
git log --format="%h %s%n  %(trailers:key=Co-Authored-By,valueonly)" -- <path>
git log --format="%an" | sort | uniq -c | sort -rn
```

Commit trailers, PR bodies and branch names are usually the only surviving record of provenance.
If they do not answer it either, that is finding number one: **the record has no model column.**
Fix it going forward (`loop.md`, "Put the model on the record") before you add a single feature.

## 2. Tier-audit what was built

Classify every shipped capability against the tier table in SKILL.md §1. You are looking for one
thing: **work on the never-delegate list that was built at a lower tier, in a cheaper harness, or
with no gate at all.** Order what you find by consequence, and that ordering is your back-gate
queue.

The queue runs at **the tier the work deserved originally**, not the tier that produced it. A
security control written by the cheapest model is not made safe by a cheap review.

## 3. Sweep for the unreachable

The cheapest high-yield pass there is. Four classes, all of which pass CI:

| Class | How to find it |
|---|---|
| component with no caller | grep for the export, then for its import |
| route never registered in the release/feature gate | enumerate routes, diff against the gate's list |
| issuer/handler with no caller | grep the function name; one hit means it is dead |
| primary call-to-action pointing at a 404 | walk the real journey, no hand-constructed URLs |

## 4. Check that the guards can fail

A guard that cannot fail is the most expensive silent defect there is, because it reads as
coverage. For each check in the `verify` contract, **plant a violation and prove it fails**. Keep
the planted violations as fixtures so the proof is repeatable.

Then check the CI shape: is the required status check an aggregating job, and does a **skipped**
job count as passing inside it? A docs-only fast path becomes a hole the moment a PR touches
anything else.

## 5. Audit the evidence, not the code

For every requirement already marked done, ask one question: **is there a test that could have
failed?** Downgrade every requirement that cannot answer it to UNVERIFIED, with what would settle
it. The three that never survive this:

- a dev-mode check for a production-only defect;
- a test asserting the *absence* of an error string;
- a permission check run as the database owner.

This is a re-examination of evidence, not of code. Resist reopening design decisions here; that
is a different argument and it will eat the audit.

## 6. Reconstruct the decisions

Write the decision register retroactively as far as the history supports, and mark plainly where
the reasoning is gone. **"Reasoning lost" is itself a finding** — it predicts which decisions get
relitigated or silently reversed in the next month.

## 7. Produce the same artefacts as a fresh start

The audit ends where `intake.md` ends, plus one thing:

1. the pass boundary and the Pass 1 exit bar, in this project's terms;
2. the tier map, with model and harness named, recorded in the repository;
3. the phase list for what remains, with gate weights;
4. sprint goals as exit tests for the next phase only;
5. **the back-gate queue as the first phase** — before any new capability.

Then ask the operator the intake questions that the existing work does not already answer. A
retrofit that skips the interview inherits the assumptions that caused the drift.
